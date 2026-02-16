# Chapter 11: RAID-Z and dRAID

> **Source:** `include/sys/vdev_raidz.h`, `include/sys/vdev_raidz_impl.h`, `include/sys/vdev_draid.h`, `include/sys/uberblock_impl.h`, `include/sys/fs/zfs.h`, `module/zfs/vdev_raidz.c`, `module/zfs/vdev_draid.c`, `module/zcommon/zfeature_common.c`

RAID-Z and dRAID are top-level vdev types that map a logical block to multiple child devices with parity. The on-disk layout is deterministic: given the vdev configuration, parity level, ashift, and logical offset, the physical sector locations are implied. Parity blocks have no blkptr of their own and are located by the same mapping rules as data.

## 11.1 RAID-Z Vdev Configuration (On Disk)

RAID-Z vdevs are stored in the pool config nvlist (vdev labels) with:

| Key | Type | Description |
|-----|------|-------------|
| `ZPOOL_CONFIG_TYPE` | string | `raidz` |
| `ZPOOL_CONFIG_NPARITY` | uint64 | Parity columns (1, 2, or 3) |

Children are listed in `ZPOOL_CONFIG_CHILDREN`. `VDEV_RAIDZ_MAXPARITY` is 3.

## 11.2 RAID-Z Stripe Mapping

RAID-Z divides a logical block into a stripe (row) that spans the child vdevs. During expansion reflow the mapping can be split into multiple rows, but each row uses the same column rules. The mapping is computed from the logical offset and size:

```
b  = io_offset >> ashift        // logical sector offset
s  = io_size >> ashift          // sectors in this I/O
f  = b % dcols                  // first column index
o  = (b / dcols) << ashift      // starting offset on each child
q  = s / (dcols - nparity)      // full-stripe data sectors per column
r  = s - q * (dcols - nparity)  // remainder sectors
bc = (r == 0 ? 0 : r + nparity) // big columns with an extra sector
tot = s + nparity * (q + (r == 0 ? 0 : 1)) // data + parity sectors
```

`dcols` is the number of child vdevs and `ashift` is the sector shift.

Parity columns are placed first in the row: columns `0..nparity-1` are parity, and data columns start at `nparity`. The `bc` big columns receive `q + 1` sectors, and the remaining accessed columns receive `q` sectors. Column `c` is assigned to child `(f + c) % dcols`, and its offset is `o` with an extra sector added for each wrap past the end of the child list.

If `q == 0` (the I/O does not span all children), only `bc` columns are accessed. The stripe is padded out to a multiple of `nparity + 1` columns as described in 11.4.

## 11.3 Parity Placement and Rotation

RAID-Z1 uses P parity, RAID-Z2 uses P and Q parity, and RAID-Z3 uses P, Q, and R parity. Parity is computed from the data columns and stored as regular sectors in the stripe; there is no separate on-disk metadata for parity.

To avoid always placing parity on the same device, RAID-Z1 rotates the first two columns every 1 MiB of logical offset. This is implemented by swapping the first two columns when `io_offset` has bit 20 set. This is part of the on-disk format for single-parity RAID-Z.

## 11.4 Skip Sectors and Padding

When a stripe is not a full-width multiple of `nparity + 1` columns, RAID-Z inserts padding sectors:

```
nskip     = roundup(tot, nparity + 1) - tot
skipstart = bc
```

Skip sectors are logical padding. They can be materialized as zero-filled sectors to improve I/O aggregation, but they are not part of the logical block and are not referenced by any blkptr.

## 11.5 RAID-Z Expansion and Reflow

`feature@raidz_expansion` (`org.openzfs:raidz_expansion`) allows a RAID-Z vdev to add a child and reflow data into the new width. During expansion, blocks written before the expansion keep the old logical stripe width, while new blocks use the new width. This produces time-dependent geometry: the correct layout for a block is chosen based on its birth txg and the expansion history.

On-disk fields used by reflow:

### Uberblock Reflow Info

`uberblock_t` stores `ub_raidz_reflow_info` with two subfields:

```
63                    55 54                           0
+----------------------+-------------------------------+
|  Scratch State (9b) |  Reflow Offset (55b)          |
+----------------------+-------------------------------+
```

Offset is stored in units of `1 << SPA_MINBLOCKSHIFT` bytes.

Scratch state uses `raidz_reflow_scratch_state_t`:

- `RRSS_SCRATCH_NOT_IN_USE`
- `RRSS_SCRATCH_VALID`
- `RRSS_SCRATCH_INVALID_SYNCED`
- `RRSS_SCRATCH_INVALID_SYNCED_ON_IMPORT`
- `RRSS_SCRATCH_INVALID_SYNCED_REFLOW`

### Scratch Area

During the initial phase of reflow, RAID-Z uses the reserved boot block area as scratch space. This is the 3.5 MiB region immediately following the vdev labels. The scratch area is persisted on disk so interrupted reflows can resume safely.

### Vdev Config NVList

The top-level RAID-Z vdev config includes:

| Key | Type | Description |
|-----|------|-------------|
| `ZPOOL_CONFIG_RAIDZ_EXPANDING` | boolean | Present while reflow is in progress |
| `ZPOOL_CONFIG_RAIDZ_EXPAND_TXGS` | uint64 array | Txg values marking completed expansions |

`raidz_expand_txgs` is used after reflow completes to determine the logical width for a block by birth txg.

### Top Vdev ZAP Entries

Each top-level RAID-Z vdev also stores expansion progress in its ZAP object:

| ZAP Entry | Type | Description |
|-----------|------|-------------|
| `org.openzfs:raidz_expand_state` | uint64 | `dsl_scan_state_t` value |
| `org.openzfs:raidz_expand_start_time` | uint64 | Start time (seconds) |
| `org.openzfs:raidz_expand_end_time` | uint64 | End time (seconds) |
| `org.openzfs:raidz_expand_bytes_copied` | uint64 | Bytes copied so far |

These entries are informational and allow progress reporting and restart.

## 11.6 dRAID Vdev Configuration (On Disk)

dRAID is a top-level vdev type (`draid`) that combines RAID-Z parity with distributed spares. The vdev config nvlist contains:

| Key | Type | Description |
|-----|------|-------------|
| `ZPOOL_CONFIG_TYPE` | string | `draid` |
| `ZPOOL_CONFIG_NPARITY` | uint64 | Parity columns (1, 2, or 3) |
| `ZPOOL_CONFIG_DRAID_NDATA` | uint64 | Data columns per group |
| `ZPOOL_CONFIG_DRAID_NSPARES` | uint64 | Distributed spares |
| `ZPOOL_CONFIG_DRAID_NGROUPS` | uint64 | Groups per permutation |

Children are listed in the `ZPOOL_CONFIG_CHILDREN` nvlist array (the child count is derived from the array length). `VDEV_DRAID_MAXPARITY` is 3.

## 11.7 dRAID Mapping and Permutations

dRAID divides address space into fixed-size groups and maps them across disks using a permutation map.

Derived geometry:

- `groupwidth = ndata + nparity`
- `rowheight = VDEV_DRAID_ROWHEIGHT` (1 << `SPA_MAXBLOCKSHIFT`)
- `groupsz = groupwidth * rowheight`
- `ndisks = children - nspares`

The permutation map (`draid_map_t`) is a static table keyed by child count. The on-disk config does not store the map; it is regenerated at import time from the compiled map seed and checksum.

Logical to physical mapping uses group and permutation indexes. For a logical byte offset:

```
b_offset = (logical_offset >> ashift)
rowheight_sectors = VDEV_DRAID_ROWHEIGHT >> ashift
group = logical_offset / groupsz
groupstart = (group * groupwidth) % ndisks
perm = group / ngroups
row = (perm * ((groupwidth * ngroups) / ndisks)) +
      (((group % ngroups) * groupwidth) / ndisks)

physical_offset = (rowheight_sectors * row +
                   (b_offset % (rowheight_sectors * groupwidth)) / groupwidth)
                  << ashift
```

The permutation row selects which child indices hold data, parity, and spare columns for that group. `groupstart` is the base position into the permutation row for the group.

## 11.8 dRAID Distributed Spares

Distributed spares are represented as leaf vdevs of type `dspare` (`VDEV_TYPE_DRAID_SPARE`) in the pool spare list. Each spare stores:

| Key | Type | Description |
|-----|------|-------------|
| `ZPOOL_CONFIG_TOP_GUID` | uint64 | Top-level dRAID vdev GUID |
| `ZPOOL_CONFIG_SPARE_ID` | uint64 | Spare index within the dRAID |

Spare columns are chosen by the permutation map and are normally empty. During a rebuild or resilver they receive reconstructed data without changing the logical mapping of existing blocks.

## 11.9 Feature Flags

| Feature | Description | On-Disk Impact |
|---------|-------------|----------------|
| `feature@raidz_expansion` | RAID-Z expansion and reflow | Adds `ub_raidz_reflow_info`, `raidz_expand_txgs`, `raidz_expanding`, and top-vdev ZAP progress fields |
| `feature@draid` | Distributed spare RAID | Enables `draid` vdev type and its configuration keys |
