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

## Object Set Types

| Value | Constant | Description |
|-------|----------|-------------|
| 0 | `DMU_OST_NONE` | Uninitialized |
| 1 | `DMU_OST_META` | Meta Object Set (pool-level, one per pool) |
| 2 | `DMU_OST_ZFS` | ZPL filesystem, snapshot, or clone |
| 3 | `DMU_OST_ZVOL` | ZFS volume |

## Checksum Algorithms

| Value | Name | Algorithm |
|-------|------|-----------|
| 1 | `on` | fletcher2 (default for data) |
| 2 | `off` | none |
| 3 | `label` | SHA-256 |
| 4 | `gang_header` | SHA-256 |
| 5 | `zilog` | fletcher2 |
| 6 | `fletcher2` | fletcher2 |
| 7 | `fletcher4` | fletcher4 |
| 8 | `SHA-256` | SHA-256 |

## Compression Algorithms

| Value | Name | Algorithm |
|-------|------|-----------|
| 1 | `on` | lzjb (legacy default) |
| 2 | `off` | none |
| 3 | `lzjb` | lzjb |

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
