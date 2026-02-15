# Chapter 7: ZFS Intent Log (ZIL)

> **Source:** `include/sys/zil.h`, `include/sys/zil_impl.h`, `module/zfs/zil.c`

The ZFS Intent Log (ZIL) records filesystem operations that require synchronous semantics (e.g., `fsync`, `O_DSYNC`). These log records are kept in memory until either the DMU transaction group (txg) commits them to the pool -- at which point they can be discarded -- or they are flushed to stable storage on the pool due to a synchronous requirement. In the event of a crash or power failure, the log records are replayed to bring the filesystem up to date.

There is one ZIL per filesystem. Its on-disk format consists of three parts:

1. **ZIL header** -- entry point stored in the object set
2. **ZIL blocks** -- variable-size blocks chained together
3. **ZIL records** -- individual transaction records within blocks

```mermaid
graph LR
    HDR["ZIL Header<br/>(in objset_phys_t)"]
    BLK1["ZIL Block 1"]
    BLK2["ZIL Block 2"]
    BLK3["ZIL Block 3"]

    HDR -->|zh_log| BLK1
    BLK1 -->|zit_next_blk| BLK2
    BLK2 -->|zit_next_blk| BLK3
```

Each block contains one or more log records followed by a trailer. The trailer contains a block pointer to the next block in the chain.

## 7.1 ZIL Header (`zil_header_t`)

The ZIL header is stored in the `os_zil_header` field of the `objset_phys_t` (see [Chapter 3](03-dmu.md#32-object-sets)).

```c
typedef struct zil_header {
    uint64_t    zh_claim_txg;   /* txg in which log blocks were claimed */
    uint64_t    zh_replay_seq;  /* highest replayed sequence number */
    blkptr_t    zh_log;         /* block pointer to first log block */
} zil_header_t;
```

| Field | Description |
|-------|-------------|
| `zh_claim_txg` | The transaction group in which the ZIL blocks were claimed during import/replay. |
| `zh_replay_seq` | The highest sequence number that has been replayed. Used to avoid replaying the same records twice. |
| `zh_log` | A block pointer to the first ZIL block in the chain. If the ZIL is empty, this is a zero-filled block pointer. |

## 7.2 ZIL Blocks

ZIL blocks are variable-size, allocated on demand from the pool (or from a separate intent log device, if configured). The size of each block is encoded in the `blkptr_t` that points to it. Each block is filled with log records and ends with a `zil_trailer_t`.

### ZIL Trailer (`zil_trailer_t`)

The trailer is located at the end of each ZIL block:

```c
typedef struct zil_trailer {
    blkptr_t            zit_next_blk;   /* next block in chain */
    uint64_t            zit_nused;      /* bytes used in this log block */
    zio_block_tail_t    zit_bt;         /* block tail with checksum */
} zil_trailer_t;
```

| Field | Description |
|-------|-------------|
| `zit_next_blk` | Block pointer to the next ZIL block in the chain. A zero block pointer indicates the end of the chain. |
| `zit_nused` | Number of bytes used in this block (log records + trailer). |
| `zit_bt` | Self-checksum for this block (see gang block description in [Chapter 2](02-block-pointers.md#23-gang-blocks) for the `zio_block_tail_t` format). |

ZIL blocks are **self-checksumming**: the checksum stored in `zit_bt` covers the block contents, not the parent block pointer's checksum fields.

## 7.3 ZIL Records

Each log record represents a filesystem operation that can be replayed. Records are packed sequentially within ZIL blocks.

### Common Header (`lr_t`)

Every record begins with a common header:

```c
typedef struct {
    uint64_t    lrc_txtype;     /* transaction type */
    uint64_t    lrc_reclen;     /* total record length in bytes */
    uint64_t    lrc_txg;        /* transaction group number */
    uint64_t    lrc_seq;        /* log sequence number */
} lr_t;
```

All fields are 64 bits for consistent alignment across architectures.

### Transaction Types

| Value | Constant | Description | Record Structure |
|-------|----------|-------------|-----------------|
| 1 | `TX_CREATE` | Create file | `lr_create_t` |
| 2 | `TX_MKDIR` | Create directory | `lr_create_t` |
| 3 | `TX_MKXATTR` | Create xattr directory | `lr_create_t` |
| 4 | `TX_SYMLINK` | Create symbolic link | `lr_create_t` |
| 5 | `TX_REMOVE` | Remove file | `lr_remove_t` |
| 6 | `TX_RMDIR` | Remove directory | `lr_remove_t` |
| 7 | `TX_LINK` | Create hard link | `lr_link_t` |
| 8 | `TX_RENAME` | Rename file/directory | `lr_rename_t` |
| 9 | `TX_WRITE` | Write to file | `lr_write_t` |
| 10 | `TX_TRUNCATE` | Truncate file | `lr_truncate_t` |
| 11 | `TX_SETATTR` | Set file attributes | `lr_setattr_t` |
| 12 | `TX_ACL` | Set ACL | `lr_acl_t` |

### Record-Specific Structures

Each record type embeds the common `lr_t` header followed by type-specific fields. All non-string fields are 64 bits wide.

#### `lr_create_t` (TX_CREATE, TX_MKDIR, TX_MKXATTR, TX_SYMLINK)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_doid` | Object number of the parent directory |
| `lr_foid` | Object number of the created object |
| `lr_mode` | File mode |
| `lr_uid` | Owner UID |
| `lr_gid` | Owner GID |
| `lr_gen` | Generation number (creation txg) |
| `lr_crtime[2]` | Creation time (seconds + nanoseconds) |
| `lr_rdev` | Device number (for special files) |

The object name follows the fixed fields as a variable-length string. For symlinks, the link target follows the name.

#### `lr_remove_t` (TX_REMOVE, TX_RMDIR)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_doid` | Object number of the parent directory |

The name of the object to remove follows the fixed fields.

#### `lr_link_t` (TX_LINK)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_doid` | Object number of the directory |
| `lr_link_obj` | Object number of the link target |

The link name follows the fixed fields.

#### `lr_rename_t` (TX_RENAME)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_sdoid` | Source directory object number |
| `lr_tdoid` | Target directory object number |

Two strings follow: the source name and the destination name.

#### `lr_write_t` (TX_WRITE)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_foid` | Object number of the file |
| `lr_offset` | Byte offset to write at |
| `lr_length` | Number of bytes to write |
| `lr_blkoff` | Offset represented by `lr_blkptr` |
| `lr_blkptr` | Block pointer for replay (large writes) |

For small writes, the write data follows the fixed fields inline. For large writes, the data is referenced by `lr_blkptr`.

#### `lr_truncate_t` (TX_TRUNCATE)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_foid` | Object number of the file |
| `lr_offset` | Byte offset to truncate from |
| `lr_length` | Length to truncate |

#### `lr_setattr_t` (TX_SETATTR)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_foid` | Object number of the file |
| `lr_mask` | Bitmask of attributes to set |
| `lr_mode` | New file mode |
| `lr_uid` | New UID |
| `lr_gid` | New GID |
| `lr_size` | New file size |
| `lr_atime[2]` | New access time |
| `lr_mtime[2]` | New modification time |

#### `lr_acl_t` (TX_ACL)

| Field | Description |
|-------|-------------|
| `lr_common` | Common log record header |
| `lr_foid` | Object number of the file |
| `lr_aclcnt` | Number of ACE entries |

The ACE entries (`ace_t` structures) follow the fixed fields.
