# Chapter 12: Allocation Classes, Special Vdevs, and Device Removal

> **Source:** `include/sys/vdev_impl.h`, `include/sys/metaslab_impl.h`, `include/sys/fs/zfs.h`, `include/sys/vdev_indirect_mapping.h`, `include/sys/vdev_indirect_births.h`, `include/sys/spa_impl.h`, `include/sys/dmu.h`, `module/zfs/vdev.c`, `module/zfs/vdev_label.c`, `module/zfs/vdev_indirect.c`, `module/zfs/vdev_indirect_mapping.c`, `module/zfs/vdev_indirect_births.c`, `module/zfs/vdev_removal.c`, `module/zcommon/zfeature_common.c`

ZFS routes block allocations through **metaslab classes**, which are categories of top-level vdevs. Allocation classes are the on-disk mechanism behind special and dedup vdevs, and they also interact with device removal via indirect vdevs.

## 12.1 Allocation Classes and Allocation Bias

Each top-level vdev can be tagged with an allocation bias that selects its metaslab class. This bias is stored in the vdev's top ZAP object:

| ZAP Entry | Value Type | Description |
|-----------|------------|-------------|
| `org.zfsonlinux:allocation_bias` (`VDEV_TOP_ZAP_ALLOCATION_BIAS`) | string | Allocation class (`log`, `special`, `dedup`) |

Absence of this entry implies the **normal** class. The vdev's config type remains `mirror`, `raidz`, `draid`, etc. Allocation bias is an additional on-disk tag, not a vdev type.

`feature@allocation_classes` (`org.zfsonlinux:allocation_classes`) is activated for
special/dedup allocation classes. Log bias also uses the allocation-bias string,
but does not itself require activating this feature.

## 12.2 Special and Dedup Classes

- **Special class**: used for metadata, optionally small data blocks, and DDT
  blocks when no dedicated dedup class is available.
- **Dedup class**: reserved for DDT metadata when dedicated dedup vdevs exist.

On disk, both are represented by the allocation bias string on the top vdev ZAP.
There is no separate vdev type for special or dedup devices.

For DDT allocations, routing preference is:

1. Dedup class (if present)
2. Special class (if configured to host DDT data)
3. Normal class

Datasets can opt in to small-block placement on the special class via the dataset property `special_small_blocks` (`ZFS_PROP_SPECIAL_SMALL_BLOCKS`), which is stored in the dataset properties ZAP (see Chapter 4). The objset caches this value as `os_zpl_special_smallblock`.

## 12.3 Device Removal Overview

`feature@device_removal` (`com.delphix:device_removal`) allows removing devices
without rewriting block pointers. For top-level singleton/mirror vdev removal,
the removed vdev is replaced by an **indirect vdev**, which holds a mapping from
old offsets to new locations. (Log vdev removal follows a different path and is
not converted to indirect.)

On disk, removal state is tracked in the MOS by the `DMU_POOL_REMOVING` entry (`com.delphix:removing`), which stores a `spa_removing_phys_t` structure:

```
spa_removing_phys_t
Offset  Size  Field              Description
0x00    8     sr_state           dsl_scan_state_t
0x08    8     sr_removing_vdev   vdev ID being removed (-1 if none)
0x10    8     sr_prev_indirect_vdev  last removed vdev ID (-1 if none)
0x18    8     sr_start_time      removal start time (seconds)
0x20    8     sr_end_time        removal end time (seconds)
0x28    8     sr_to_copy         bytes to copy
0x30    8     sr_copied          bytes copied or freed
```

When a removal is in progress, the vdev config records `ZPOOL_CONFIG_REMOVING`
on the removing vdev. A dedicated slog is the exception: slog removals are
synchronous and do not persist this flag.

## 12.4 Indirect Vdev Configuration (On Disk)

Indirect vdevs use vdev type `indirect` (`VDEV_TYPE_INDIRECT`). Their config (stored in the vdev tree) includes:

| Key | Type | Description |
|-----|------|-------------|
| `com.delphix:indirect_object` (`ZPOOL_CONFIG_INDIRECT_OBJECT`) | uint64 | Object ID of indirect mapping |
| `com.delphix:indirect_births` (`ZPOOL_CONFIG_INDIRECT_BIRTHS`) | uint64 | Object ID of indirect births table |
| `com.delphix:prev_indirect_vdev` (`ZPOOL_CONFIG_PREV_INDIRECT_VDEV`) | uint64 | Previous removed vdev ID, or absent |

## 12.5 Indirect Mapping Object

The indirect mapping object is a `DMU_OTN_UINT64_METADATA` object (block size `SPA_OLD_MAXBLOCKSIZE`) containing an array of `vdev_indirect_mapping_entry_phys_t` entries, sorted by source offset. The bonus buffer stores `vdev_indirect_mapping_phys_t`:

```
vdev_indirect_mapping_phys_t (bonus)
Offset  Size  Field              Description
0x00    8     vimp_max_offset    max mapped offset
0x08    8     vimp_bytes_mapped  total bytes mapped
0x10    8     vimp_num_entries   number of entries
0x18    8     vimp_counts_object obsolete counts object (if present)
```

If `feature@obsolete_counts` is not enabled, the bonus size is `VDEV_INDIRECT_MAPPING_SIZE_V0` (24 bytes, first three fields only).

Each mapping entry is:

```
vdev_indirect_mapping_entry_phys_t
Offset  Size  Field      Description
0x00    8     vimep_src  source offset (low 63 bits), 1-bit mark
0x08    16    vimep_dst  destination DVA (vdev ID, offset, asize)
```

`vimep_src` stores the source offset in units of `1 << SPA_MINBLOCKSHIFT` bytes. The high bit is a mark used by tooling (e.g., zdb) for garbage collection.

## 12.6 Indirect Births Object

The indirect births object tracks the physical birth time of the copied data. It is a `DMU_OTN_UINT64_METADATA` object (block size `SPA_OLD_MAXBLOCKSIZE`) with bonus type `DMU_OTN_UINT64_METADATA` and bonus size `sizeof (vdev_indirect_birth_phys_t)`.

Entries are stored as an array of `vdev_indirect_birth_entry_phys_t`:

```
vdev_indirect_birth_entry_phys_t
Offset  Size  Field              Description
0x00    8     vibe_offset        end offset of this range
0x08    8     vibe_phys_birth_txg physical birth txg for range
```

Each entry indicates that everything up to (but not including) `vibe_offset` was copied in `vibe_phys_birth_txg`.

## 12.7 Obsolete Tracking and Condensing

Removed vdev mappings shrink over time as blocks are freed or remapped. On-disk structures used to track obsolescence include:

- **Per-vdev obsolete space map**: `com.delphix:indirect_obsolete_sm` (`VDEV_TOP_ZAP_INDIRECT_OBSOLETE_SM`) stores obsolete ranges as space map entries.
- **Optional obsolete counts**: when `feature@obsolete_counts` is active, `vimp_counts_object` stores a `uint32` array with per-entry obsolete byte counts, and the top vdev ZAP entry `com.delphix:obsolete_counts_are_precise` indicates precise tracking.
- **Per-dataset remap deadlist**: `ds_remap_deadlist` stores remapped blocks
  that are still referenced by older snapshots until snapshot destruction moves
  eligible entries into pool-level obsolete processing.
- **Pool-wide obsolete bpobj**: `com.delphix:obsolete_bpobj` (`DMU_POOL_OBSOLETE_BPOBJ`) accumulates block pointers whose indirect mapping entries should be marked obsolete.

If a mapping condense is in progress, the MOS directory has `DMU_POOL_CONDENSING_INDIRECT` (`com.delphix:condensing_indirect`) containing:

```
spa_condensing_indirect_phys_t
Offset  Size  Field                        Description
0x00    8     scip_vdev                    vdev ID being condensed
0x08    8     scip_prev_obsolete_sm_object previous obsolete space map
0x10    8     scip_next_mapping_object     new mapping object
```

## 12.8 Feature Flags

| Feature | Description | On-Disk Impact |
|---------|-------------|----------------|
| `feature@allocation_classes` | Allocation classes (special, dedup) | Adds per-vdev allocation bias ZAP entry |
| `feature@device_removal` | Top-level vdev removal | Adds `DMU_POOL_REMOVING` entry and indirect vdev config keys |
| `feature@obsolete_counts` | Precise obsolete tracking | Expands indirect mapping bonus and adds counts object |
