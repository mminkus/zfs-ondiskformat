# Chapter 9: Space Maps and Metaslabs

> **Source:** `include/sys/space_map.h`, `include/sys/metaslab_impl.h`, `include/sys/vdev_impl.h`, `include/sys/fs/zfs.h`, `include/sys/dmu.h`, `module/zfs/space_map.c`, `module/zfs/metaslab.c`, `module/zfs/vdev.c`, `module/zfs/spa_log_spacemap.c`, `module/zcommon/zfeature_common.c`

Space maps are the on-disk allocation logs used by ZFS metaslabs. Each top-level vdev is divided into metaslabs, and each metaslab has an associated space map object that records alloc/free segments. The space map is append-only and is periodically condensed into a smaller, more efficient form.

Space maps are the **on-disk** representation of free/allocated space. In memory, ZFS uses **range trees** (`zfs_range_tree_t`) to represent the same information as sorted, balanced trees of contiguous ranges. Range trees are built from space map entries at load time and have no on-disk representation.

## 9.1 Metaslab Array (On Disk)

Each top-level vdev stores a metaslab array object in the MOS. The vdev config (nvlist) stores:

| Key | Value Type | Description |
|-----|------------|-------------|
| `metaslab_array` | `uint64` | Object ID of the metaslab array (MOS object) |
| `metaslab_shift` | `uint64` | log2 of metaslab size (bytes) |

The metaslab array is a `DMU_OT_OBJECT_ARRAY` object containing `uint64` entries. Each entry is the object ID of a space map for one metaslab. The array index corresponds to the metaslab index for that vdev.

```
metaslab_count = vdev_asize >> metaslab_shift
metaslab_size  = 1 << metaslab_shift
```

Metaslabs are grouped in memory into **metaslab groups** (one per top-level vdev) for allocation purposes, but metaslab groups have no on-disk representation -- they are reconstructed at pool load time from the per-metaslab space maps.

Metaslab groups belong to **metaslab classes** (allocation classes). Each class has its own allocation policy and set of top-level vdevs. Common classes include:

- `normal` (default data allocations)
- `log` (separate intent log devices)
- `special` (metadata and small blocks)
- `dedup` (deduplication tables)

Allocation class membership is reflected on disk via each top-level vdev's ZAP entry `org.zfsonlinux:allocation_bias` (`VDEV_TOP_ZAP_ALLOCATION_BIAS`), which stores one of `log`, `special`, or `dedup` (absence implies `normal`).

## 9.2 Space Map Object (`DMU_OT_SPACE_MAP`)

A metaslab space map is a `DMU_OT_SPACE_MAP` object whose bonus buffer is typed as `DMU_OT_SPACE_MAP_HEADER` and holds `space_map_phys_t`:

```
space_map_phys_t (bonus data)
Offset  Size    Field           Description
0x00    8       smp_object      Space map object ID (deprecated, kept for compat)
0x08    8       smp_length      Space map data length in bytes
0x10    8       smp_alloc       Net space allocated from the map (signed)
0x18    40      smp_pad         Reserved (5 x uint64)
0x40    256     smp_histogram   32-bucket free space histogram
```

If `feature@spacemap_histogram` is **not** enabled, the bonus size is only `SPACE_MAP_SIZE_V0` (24 bytes -- the first three fields). When the feature is enabled, the full `space_map_phys_t` (320 bytes) is used.

### Histogram Buckets

Each bucket tracks free regions by size. Bucket `i` counts regions whose size is:

```
2^(i + sm_shift) <= size < 2^(i + sm_shift + 1)
```

`sm_shift` is the allocation unit shift (typically the vdev's `ashift`). It is not stored in `space_map_phys_t` -- it is passed as a parameter when opening the space map and is derived from the metaslab's configuration. All one-word entry offsets and runs are in units of `2^sm_shift` bytes.

## 9.3 Space Map Entry Encoding

Space map entries are 64-bit words stored in the space map object's data blocks. There are three encodings:

### Debug Entry (prefix `10b`)

```
63 62 61 60 59        50 49                      0
+-----+-----+----------+-------------------------+
|  1  |  0  |  action  |        syncpass         |
|     |     |  (2 b)   |        (10 b)           |
+-----+-----+----------+-------------------------+
|                txg (low 50 bits)              |
+------------------------------------------------+
```

- Bits 63-62: prefix `10` (debug entry marker)
- Bits 61-60: action (2 bits)
- Bits 59-50: sync pass (10 bits)
- Bits 49-0: txg, lower 50 bits

Debug entries are not allocation records. They carry metadata and are also used as padding so that two-word entries do not cross block boundaries.

### One-Word Entry (bit 63 = `0`)

```
62                          16 15 14             0
+----------------------------+----+---------------+
| offset (47 bits)           |type|  run (15 b)   |
+----------------------------+----+---------------+
```

- Bit 63: `0` (one-word marker; bit 62 is the MSB of offset)
- Bits 62-16: offset in `sm_shift` units, relative to `sm_start` (47 bits)
- Bit 15: type -- `0` = `SM_ALLOC`, `1` = `SM_FREE`
- Bits 14-0: run length in `sm_shift` units, encoded as `run - 1` (15 bits)

### Two-Word Entry (prefix `11b`, `feature@spacemap_v2`)

```
Word 0:
63 62 61 60 59                     24 23        0
+-----+-----+-----+-----------------+-----------+
|  1  |  1  | pad |   run (36 b)    | vdev (24b)|
+-----+-----+-----+-----------------+-----------+

Word 1:
63 62                                           0
+----+-------------------------------------------+
|type|              offset (63 bits)             |
+----+-------------------------------------------+
```

- Word 0 bits 63-62: prefix `11` (two-word marker)
- Word 0 bits 23-0: vdev ID (24 bits, `SPA_VDEVBITS`), or `SM_NO_VDEVID` (`1 << 24`) if not applicable
- Word 0 bits 59-24: run length in `sm_shift` units (36 bits)
- Word 1 bit 63: type -- `0` = `SM_ALLOC`, `1` = `SM_FREE`
- Word 1 bits 62-0: offset in `sm_shift` units (63 bits)

Two-word entries are used when the offset or run does not fit in a single word, or when a vdev ID is required (e.g., log space maps that reference multiple vdevs).

A two-word entry never straddles a block boundary; if necessary, the last word of a block is padded with a debug entry.

## 9.4 Condensing

Because space maps are append-only logs, they grow over time as alloc/free entries accumulate. **Condensing** rewrites a space map as a compact summary of the current state, discarding the historical log of individual operations.

Condensing is triggered during `metaslab_sync()` when the on-disk space map is sufficiently inefficient. The common case requires both:

- `space_map_length >= (optimal_size * zfs_condense_pct / 100)`
- `space_map_length > zfs_metaslab_condense_block_threshold * record_size`

Where `record_size` is `max(space_map_block_size, vdev_block_size)`. Empty metaslabs and explicit condense requests bypass these heuristics.

When a space map is condensed:

1. The existing space map object is truncated (`space_map_truncate`).
2. New entries are written representing only the current allocation state -- all allocated regions as `SM_ALLOC` entries and all free regions as `SM_FREE` entries.
3. The `smp_length` and `smp_alloc` fields in `space_map_phys_t` are updated to reflect the new compact size.
4. The histogram is recalculated.

Condensing is transparent to the rest of the system -- the space map object ID does not change, only its contents are rewritten.

## 9.5 Feature Flags

Relevant feature flags and their on-disk impact:

| Feature | Description | On-Disk Impact |
|---------|-------------|----------------|
| `feature@spacemap_histogram` | Maintain free-space histograms | Expands `space_map_phys_t` bonus from 24 to 280 bytes |
| `feature@spacemap_v2` | More efficient encoding for large segments | Enables two-word space map entries |
| `feature@log_spacemap` | Pool-wide log spacemap | Adds MOS entry `com.delphix:log_spacemap_zap` and per-vdev unflushed txg tracking |
| `feature@allocation_classes` | Separate allocation classes | Adds per-vdev allocation bias (`org.zfsonlinux:allocation_bias`) |

`feature@log_spacemap` depends on `feature@spacemap_v2`.

## 9.6 Log Space Maps (Pool-Wide)

When `feature@log_spacemap` is enabled, ZFS writes a **pool-wide** space map each TXG that records metaslab changes. This avoids random writes to individual metaslab space maps under random-free workloads -- changes are batched in the log and flushed to individual metaslabs later.

### On-Disk Structures

**MOS directory entry** `com.delphix:log_spacemap_zap` (`DMU_POOL_LOG_SPACEMAP_ZAP`):
A ZAP object mapping `txg -> log spacemap object ID`. Each value is the object ID of a `DMU_OT_SPACE_MAP` object containing all alloc/free entries for that TXG across the entire pool.

Log spacemap entries are always encoded as **two-word entries** (with a vdev ID to identify which top-level vdev each entry belongs to).

**Per-top-vdev ZAP entry** `com.delphix:ms_unflushed_phys_txgs` (`VDEV_TOP_ZAP_MS_UNFLUSHED_PHYS_TXGS`):
An array stored in each top-level vdev's ZAP object, with one `uint64` per metaslab. Each value is the `ms_unflushed_txg` -- the TXG up to which that metaslab's space map has been flushed. Log spacemap entries older than a metaslab's unflushed txg are ignored during replay for that metaslab.

### Lifecycle

A log spacemap becomes obsolete once all metaslabs across all vdevs have flushed changes past its TXG. At that point, the log spacemap object is destroyed and its entry removed from the ZAP. ZFS periodically flushes the oldest unflushed metaslabs to retire old log spacemaps and bound the total log size.

### Flushing and `spa_log_summary` (In-Memory)

When log spacemaps are active, per-metaslab changes are accumulated in in-memory range trees and periodically flushed back to each metaslab's on-disk space map. The number of metaslabs flushed per TXG is guided by two heuristics:

- **Memory heuristic**: tracks memory used by unflushed trees (`spa_unflushed_stats.sus_memused`).
- **Block heuristic**: tracks total log spacemap blocks in the pool, using per-log block counts.

To avoid scanning all log spacemap objects every TXG, ZFS maintains an in-memory **`spa_log_summary`** list which summarizes metaslab and block counts for the existing log spacemaps. This summary is used to decide how aggressively to flush and which metaslabs to flush first. It has no on-disk representation.

## 9.7 Checkpoint Space Map

When a pool checkpoint is active (`feature@zpool_checkpoint`), freed blocks must be tracked so the pool can be rewound. Each top-level vdev stores a **checkpoint space map** referenced by:

**Per-top-vdev ZAP entry** `com.delphix:pool_checkpoint_sm` (`VDEV_TOP_ZAP_POOL_CHECKPOINT_SM`):
The object ID of a standard `DMU_OT_SPACE_MAP` object that records blocks freed since the checkpoint was created (written as `SM_FREE` entries).

On checkpoint discard, these space maps are destroyed and the tracked space is released. On checkpoint rewind, the entries are reapplied to restore the freed blocks.

## 9.8 Device Removal Space Maps

When a top-level vdev is removed (`feature@device_removal`), its blocks are remapped to other vdevs via indirect mappings. Two per-vdev ZAP entries track obsolete indirect mappings:

| ZAP Entry | Description |
|-----------|-------------|
| `com.delphix:indirect_obsolete_sm` | Space map tracking obsolete indirect mapping entries |
| `com.delphix:obsolete_counts_are_precise` | Boolean flag indicating whether obsolete counts are exact |

These are standard `DMU_OT_SPACE_MAP` objects used to track which portions of the indirect mapping have become obsolete (e.g., because the remapped blocks were subsequently freed).
