# Glossary

## Acronyms and Terms

| Term | Full Name | Description |
|------|-----------|-------------|
| **ACE** | Access Control Entry | A single entry in an ACL specifying permissions for a user or group |
| **ACL** | Access Control List | A table of ACEs controlling access to a ZFS object |
| **ARC** | Adaptive Replacement Cache | ZFS's in-memory read cache |
| **COW** | Copy On Write | ZFS's transactional write model; data is never overwritten in place |
| **DMU** | Data Management Unit | Layer that organizes blocks into objects and object sets |
| **DSL** | Dataset and Snapshot Layer | Manages relationships between datasets, snapshots, and clones |
| **DVA** | Data Virtual Address | A (vdev, offset) pair identifying a block's physical location |
| **MOS** | Meta Object Set | The pool-wide object set (type `DMU_OST_META`) pointed to by the uberblock; contains all pool metadata |
| **SA** | System Attributes | Modern replacement for the bonus buffer znode format |
| **SPA** | Storage Pool Allocator | Top-level layer managing the storage pool, vdevs, and I/O |
| **TXG** | Transaction Group | A group of writes committed atomically; identified by a monotonically increasing 64-bit number |
| **ZAP** | ZFS Attribute Processor | Module for storing name-value pair attributes in objects |
| **ZIL** | ZFS Intent Log | Records synchronous operations for replay after crash |
| **ZIO** | ZFS I/O Pipeline | The I/O scheduling and execution framework |
| **L2ARC** | Level 2 Adaptive Replacement Cache | Secondary read cache on a dedicated device (SSD) |
| **MMP** | Multi-Modifier Protection | Heartbeat mechanism preventing simultaneous pool import on multiple hosts |
| **SLOG** | Separate Log | A dedicated device for ZIL (intent log) writes |
| **ZPL** | ZFS POSIX Layer | Makes DMU objects appear as a POSIX-compliant filesystem |
| **ZVOL** | ZFS Volume | A logical block device backed by a ZFS object set |

## DMU Object Types

These constants identify the type of data stored in a dnode. Values are stored in the `dn_type` field of `dnode_phys_t` and in the `type` field of `blkptr_t`.

| Value | Constant | Description |
|-------|----------|-------------|
| 0 | `DMU_OT_NONE` | Unallocated / free dnode |
| 1 | `DMU_OT_OBJECT_DIRECTORY` | MOS object directory (ZAP) |
| 2 | `DMU_OT_OBJECT_ARRAY` | Array of object numbers |
| 3 | `DMU_OT_PACKED_NVLIST` | Packed nvlist data |
| 4 | `DMU_OT_PACKED_NVLIST_SIZE` | Size of a packed nvlist object |
| 5 | `DMU_OT_BPOBJ` | Block pointer object (formerly BPLIST) |
| 6 | `DMU_OT_BPOBJ_HDR` | Block pointer object header |
| 7 | `DMU_OT_SPACE_MAP_HEADER` | Space map header |
| 8 | `DMU_OT_SPACE_MAP` | Space map |
| 9 | `DMU_OT_INTENT_LOG` | ZFS intent log |
| 10 | `DMU_OT_DNODE` | Array of dnodes (metadnode) |
| 11 | `DMU_OT_OBJSET` | Object set |
| 12 | `DMU_OT_DSL_DIR` | DSL directory |
| 13 | `DMU_OT_DSL_DIR_CHILD_MAP` | DSL child directory map (ZAP) |
| 14 | `DMU_OT_DSL_DS_SNAP_MAP` | DSL snapshot map (ZAP) |
| 15 | `DMU_OT_DSL_PROPS` | DSL properties (ZAP) |
| 16 | `DMU_OT_DSL_DATASET` | DSL dataset |
| 17 | `DMU_OT_ZNODE` | ZPL file metadata |
| 18 | `DMU_OT_OLDACL` | Legacy ACL data |
| 19 | `DMU_OT_PLAIN_FILE_CONTENTS` | Regular file data |
| 20 | `DMU_OT_DIRECTORY_CONTENTS` | Directory entries (ZAP) |
| 21 | `DMU_OT_MASTER_NODE` | ZPL master node (ZAP) |
| 22 | `DMU_OT_UNLINKED_SET` | Unlinked (delete queue) set |
| 23 | `DMU_OT_ZVOL` | ZVOL data |
| 24 | `DMU_OT_ZVOL_PROP` | ZVOL properties (ZAP) |
| 25 | `DMU_OT_PLAIN_OTHER` | Other uint8 data (test) |
| 26 | `DMU_OT_UINT64_OTHER` | Other uint64 data (test) |
| 27 | `DMU_OT_ZAP_OTHER` | Other ZAP object (test) |
| 28 | `DMU_OT_ERROR_LOG` | Persistent error log (ZAP) |
| 29 | `DMU_OT_SPA_HISTORY` | Pool history |
| 30 | `DMU_OT_SPA_HISTORY_OFFSETS` | Pool history offsets |
| 31 | `DMU_OT_POOL_PROPS` | Pool properties (ZAP) |
| 32 | `DMU_OT_DSL_PERMS` | DSL permissions (ZAP) |
| 33 | `DMU_OT_ACL` | ZFS ACL (v1) |
| 34 | `DMU_OT_SYSACL` | ZFS system ACL |
| 35 | `DMU_OT_FUID` | FUID table |
| 36 | `DMU_OT_FUID_SIZE` | FUID table size |
| 37 | `DMU_OT_NEXT_CLONES` | Next clones (ZAP) |
| 38 | `DMU_OT_SCAN_QUEUE` | Scan work queue (ZAP) |
| 39 | `DMU_OT_USERGROUP_USED` | User/group space used (ZAP) |
| 40 | `DMU_OT_USERGROUP_QUOTA` | User/group quota (ZAP) |
| 41 | `DMU_OT_USERREFS` | Snapshot user reference holds (ZAP) |
| 42 | `DMU_OT_DDT_ZAP` | Dedup table (ZAP) |
| 43 | `DMU_OT_DDT_STATS` | Dedup statistics (ZAP) |
| 44 | `DMU_OT_SA` | System attributes |
| 45 | `DMU_OT_SA_MASTER_NODE` | SA master node (ZAP) |
| 46 | `DMU_OT_SA_ATTR_REGISTRATION` | SA attribute registration (ZAP) |
| 47 | `DMU_OT_SA_ATTR_LAYOUTS` | SA attribute layouts (ZAP) |
| 48 | `DMU_OT_SCAN_XLATE` | Scan translations (ZAP) |
| 49 | `DMU_OT_DEDUP` | Deduplicated block |
| 50 | `DMU_OT_DEADLIST` | DSL deadlist map (ZAP) |
| 51 | `DMU_OT_DEADLIST_HDR` | DSL deadlist header |
| 52 | `DMU_OT_DSL_CLONES` | DSL directory clones (ZAP) |
| 53 | `DMU_OT_BPOBJ_SUBOBJ` | Block pointer object subobject |

Values 0x80 and above use the new-type encoding; see [Chapter 3, Section 3.5](03-dmu.md#35-new-type-encoding).

## Object Set Types

| Value | Constant | Description |
|-------|----------|-------------|
| 0 | `DMU_OST_NONE` | Uninitialized |
| 1 | `DMU_OST_META` | Meta Object Set (pool-level, one per pool) |
| 2 | `DMU_OST_ZFS` | ZPL filesystem, snapshot, or clone |
| 3 | `DMU_OST_ZVOL` | ZFS volume |

## Checksum Algorithms

| Value | Name | Algorithm | Added |
|-------|------|-----------|-------|
| 1 | `on` | fletcher2 (legacy default) | v1 |
| 2 | `off` | none | v1 |
| 3 | `label` | SHA-256 (labels only) | v1 |
| 4 | `gang_header` | SHA-256 (gang blocks only) | v1 |
| 5 | `zilog` | fletcher2 (ZIL only) | v1 |
| 6 | `fletcher2` | fletcher2 | v1 |
| 7 | `fletcher4` | fletcher4 | v1 |
| 8 | `SHA-256` | SHA-256 | v1 |
| 9 | `SHA-512/256` | SHA-512 truncated to 256 bits | feature |
| 10 | `skein` | Skein-512/256 | feature |
| 11 | `edonr` | Edon-R/256 | feature |
| 12 | `blake3` | BLAKE3 | feature |

## Compression Algorithms

| Value | Name | Algorithm | Added |
|-------|------|-----------|-------|
| 1 | `on` | lzjb (legacy default) | v1 |
| 2 | `off` | none | v1 |
| 3 | `lzjb` | lzjb | v1 |
| 5 | `gzip-1` | GZIP level 1 | v5 |
| 6 | `gzip-2` | GZIP level 2 | v5 |
| 7 | `gzip-3` | GZIP level 3 | v5 |
| 8 | `gzip-4` | GZIP level 4 | v5 |
| 9 | `gzip-5` | GZIP level 5 | v5 |
| 10 | `gzip-6` | GZIP level 6 | v5 |
| 11 | `gzip-7` | GZIP level 7 | v5 |
| 12 | `gzip-8` | GZIP level 8 | v5 |
| 13 | `gzip-9` | GZIP level 9 | v5 |
| 14 | `zle` | Zero-Length Encoding | v20 |
| 15 | `lz4` | LZ4 | feature |
| 16 | `zstd` | Zstandard | feature |

## ZIL Transaction Types

| Value | Constant | Description |
|-------|----------|-------------|
| 1 | `TX_CREATE` | Create file |
| 2 | `TX_MKDIR` | Create directory |
| 3 | `TX_MKXATTR` | Create extended attribute directory |
| 4 | `TX_SYMLINK` | Create symbolic link |
| 5 | `TX_REMOVE` | Remove file |
| 6 | `TX_RMDIR` | Remove directory |
| 7 | `TX_LINK` | Create hard link |
| 8 | `TX_RENAME` | Rename file or directory |
| 9 | `TX_WRITE` | Write to file |
| 10 | `TX_TRUNCATE` | Truncate file |
| 11 | `TX_SETATTR` | Set file attributes |
| 12 | `TX_ACL` | Set ACL |

## Pool States

| Value | Constant | Description |
|-------|----------|-------------|
| 0 | `POOL_STATE_ACTIVE` | Pool is active and in use |
| 1 | `POOL_STATE_EXPORTED` | Pool has been exported |
| 2 | `POOL_STATE_DESTROYED` | Pool has been destroyed |

## ZAP Block Type Identifiers

| Value | Constant | Description |
|-------|----------|-------------|
| `(1ULL << 63) + 3` | `ZBT_MICRO` | Microzap block |
| `(1ULL << 63) + 1` | `ZBT_HEADER` | Fat ZAP header (first block) |
| `(1ULL << 63) + 0` | `ZBT_LEAF` | Fat ZAP leaf block |

## ZAP Leaf Chunk Types

| Value | Constant | Description |
|-------|----------|-------------|
| 251 | `ZAP_LEAF_ARRAY` | Name or value data chunk |
| 252 | `ZAP_LEAF_ENTRY` | Attribute entry chunk |
| 253 | `ZAP_LEAF_FREE` | Free chunk |

## Vdev Type Strings

| Type String | Description |
|-------------|-------------|
| `"disk"` | Leaf vdev: block storage device |
| `"file"` | Leaf vdev: file-backed storage |
| `"mirror"` | Interior vdev: mirror (N-way replication) |
| `"raidz"` | Interior vdev: RAID-Z (parity-based redundancy) |
| `"replacing"` | Interior vdev: temporary mirror during disk replacement |
| `"root"` | Interior vdev: root of the vdev tree |
| `"spare"` | Leaf vdev: hot spare device |
| `"log"` | Interior vdev: dedicated ZIL log device (SLOG) |
| `"l2cache"` | Leaf vdev: L2ARC read cache device |
| `"hole"` | Placeholder for a removed top-level vdev slot |
| `"missing"` | Placeholder for a device not present at import |
| `"indirect"` | Remapped vdev after device removal |
| `"draid"` | Distributed-parity RAID with integrated spares |
| `"dspare"` | Distributed spare within a dRAID group |

## Pool Versions

See [Chapter 1, Section 1.6](01-vdevs.md#16-pool-versioning) for full descriptions of each version.

| Version | Key Features |
|---------|-------------|
| 1 | Initial ZFS format |
| 2 | Ditto blocks |
| 3 | Hot spares, RAID-Z2 |
| 4 | Pool history |
| 5 | GZIP compression |
| 6 | Boot filesystem |
| 7 | Separate intent log devices (SLOG) |
| 8 | Delegated administration |
| 9 | FUID, refquota, refreservation |
| 10 | L2ARC |
| 11 | Next clones, origin, DSL scrub |
| 12 | Snapshot properties |
| 13 | Used space breakdown |
| 14 | Passthrough-x |
| 15 | User/group space accounting |
| 16 | STMF properties |
| 17 | RAID-Z3 |
| 18 | User reference holds |
| 19 | Hole support |
| 20 | ZLE compression |
| 21 | Deduplication |
| 22 | Received properties |
| 23 | Slim ZIL |
| 24 | System attributes (SA) |
| 25 | Improved scrub/scan |
| 26 | Directory clones, improved deadlists |
| 27 | Fast snapshots |
| 28 | Multiple device replacement (last numbered version) |
| 5000 | Feature flags system |

## Feature Flags

Feature flags are identified by reverse-DNS GUIDs. See [Chapter 1, Section 1.6](01-vdevs.md#16-pool-versioning) for the feature flag mechanism.

> **Source:** `module/zcommon/zfeature_common.c`

| Feature Name | GUID | Description |
|-------------|------|-------------|
| async_destroy | `com.delphix:async_destroy` | Destroy filesystems asynchronously |
| empty_bpobj | `com.delphix:empty_bpobj` | Snapshots use less space |
| lz4_compress | `org.illumos:lz4_compress` | LZ4 compression algorithm |
| multi_vdev_crash_dump | `com.joyent:multi_vdev_crash_dump` | Crash dumps to multiple vdev pools |
| spacemap_histogram | `com.delphix:spacemap_histogram` | Spacemaps maintain space histograms |
| enabled_txg | `com.delphix:enabled_txg` | Record txg at which a feature is enabled |
| hole_birth | `com.delphix:hole_birth` | Retain hole birth txg for more precise send |
| zpool_checkpoint | `com.delphix:zpool_checkpoint` | Pool state can be checkpointed |
| spacemap_v2 | `com.delphix:spacemap_v2` | Efficient space maps for large segments |
| extensible_dataset | `com.delphix:extensible_dataset` | Enhanced dataset functionality |
| bookmarks | `com.delphix:bookmarks` | `zfs bookmark` command |
| filesystem_limits | `com.joyent:filesystem_limits` | Filesystem and snapshot limits |
| embedded_data | `com.delphix:embedded_data` | Embedded-data block pointers |
| livelist | `com.delphix:livelist` | Improved clone deletion performance |
| log_spacemap | `com.delphix:log_spacemap` | Log-structured metaslab space maps |
| large_blocks | `org.open-zfs:large_blocks` | Blocks larger than 128KB |
| large_dnode | `org.zfsonlinux:large_dnode` | Variable on-disk dnode sizes |
| sha512 | `org.illumos:sha512` | SHA-512/256 checksum |
| skein | `org.illumos:skein` | Skein checksum |
| edonr | `org.illumos:edonr` | Edon-R checksum |
| redaction_bookmarks | `com.delphix:redaction_bookmarks` | Bookmarks with redaction lists |
| redacted_datasets | `com.delphix:redacted_datasets` | Redacted datasets from redacted send |
| bookmark_written | `com.delphix:bookmark_written` | written#\<bookmark\> property |
| device_removal | `com.delphix:device_removal` | Top-level vdev removal |
| obsolete_counts | `com.delphix:obsolete_counts` | Reduced memory for removed devices |
| userobj_accounting | `org.zfsonlinux:userobj_accounting` | User/group object accounting |
| bookmark_v2 | `com.datto:bookmark_v2` | Larger bookmarks |
| encryption | `com.datto:encryption` | Dataset-level encryption |
| project_quota | `org.zfsonlinux:project_quota` | Project-based space/object accounting |
| allocation_classes | `org.zfsonlinux:allocation_classes` | Separate allocation classes (special vdev) |
| resilver_defer | `com.datto:resilver_defer` | Defer new resilvers during active resilver |
| device_rebuild | `org.openzfs:device_rebuild` | Sequential mirror/dRAID device rebuilds |
| zstd_compress | `org.freebsd:zstd_compress` | Zstandard compression |
| draid | `org.openzfs:draid` | Distributed spare RAID |
| zilsaxattr | `org.openzfs:zilsaxattr` | xattr=sa logging in ZIL |
| head_errlog | `com.delphix:head_errlog` | Per-dataset on-disk error logs |
| blake3 | `org.openzfs:blake3` | BLAKE3 checksum |
| block_cloning | `com.fudosecurity:block_cloning` | Block cloning via Block Reference Table |
| vdev_zaps_v2 | `com.klarasystems:vdev_zaps_v2` | Root vdev ZAP |
| redaction_list_spill | `com.delphix:redaction_list_spill` | Increased redaction snapshot arguments |
| raidz_expansion | `org.openzfs:raidz_expansion` | RAIDZ vdev expansion |
| fast_dedup | `com.klarasystems:fast_dedup` | Advanced deduplication |
| longname | `org.zfsonlinux:longname` | Filenames up to 1024 bytes |
| large_microzap | `com.klarasystems:large_microzap` | Microzaps larger than 128KB |
