# Chapter 15: Persistent L2ARC Cache

> **Source:** `include/sys/arc_impl.h`, `include/sys/arc.h`, `include/sys/vdev_impl.h`, `include/sys/fs/zfs.h`, `include/sys/zio.h`, `module/zfs/arc.c`, `cmd/zdb/zdb.c`, `module/os/linux/zfs/zfs_sysfs.c`

L2ARC is a secondary read cache stored on cache vdevs. Persistent L2ARC adds on-device metadata so ARC headers can be reconstructed after pool import.

Unlike most modern OpenZFS format changes, this is **not** a pool feature flag in `zfeature_common.c`. The format is versioned by `L2ARC_PERSISTENT_VERSION` in `fs/zfs.h` and stored only on L2ARC devices.

## 15.1 On-Device Layout

Persistent metadata is written directly to each cache vdev:

- Device header at offset `VDEV_LABEL_START_SIZE` (4 MiB)
- Rotating log blocks and cached payload in `[l2ad_start, l2ad_end)`

`l2ad_start` and `l2ad_end` are initialized as:

- `l2ad_start = VDEV_LABEL_START_SIZE + l2ad_dev_hdr_asize`
- `l2ad_end = VDEV_LABEL_START_SIZE + vdev_get_min_asize(vd)`
- `l2ad_dev_hdr_asize = max(sizeof (l2arc_dev_hdr_phys_t), 1 << ashift)`

So the header always occupies at least one physical sector, even though the on-disk struct itself is 512 bytes.

## 15.2 Device Header (`l2arc_dev_hdr_phys_t`)

`l2arc_dev_hdr_phys_t` is exactly `SPA_MINBLOCKSIZE` (512 bytes).

| Offset | Size | Field |
|--------|------|-------|
| `0x000` | 8 | `dh_magic` (`L2ARC_DEV_HDR_MAGIC`, ASCII `"ZFSCACHE"`) |
| `0x008` | 8 | `dh_version` (`L2ARC_PERSISTENT_VERSION`) |
| `0x010` | 8 | `dh_spa_guid` |
| `0x018` | 8 | `dh_vdev_guid` |
| `0x020` | 8 | `dh_log_entries` |
| `0x028` | 8 | `dh_evict` |
| `0x030` | 8 | `dh_flags` |
| `0x038` | 8 | `dh_start` |
| `0x040` | 8 | `dh_end` |
| `0x048` | 64 | `dh_start_lbps[0]` (newest log block pointer) |
| `0x088` | 64 | `dh_start_lbps[1]` (second newest pointer) |
| `0x0C8` | 8 | `dh_lb_asize` |
| `0x0D0` | 8 | `dh_lb_count` |
| `0x0D8` | 8 | `dh_trim_action_time` |
| `0x0E0` | 8 | `dh_trim_state` |
| `0x0E8` | 240 | `dh_pad[30]` |
| `0x1D8` | 40 | `dh_tail` (`zio_eck_t`) |

Header flag bits:

- Bit `0`: `L2ARC_DEV_HDR_EVICT_FIRST` (mirror of `l2ad_first`)

Byte order note:

- If `dh_magic == BSWAP_64(L2ARC_DEV_HDR_MAGIC)`, OpenZFS byteswaps the 64-bit fields before validation. Log blocks use the same magic-based byteswap rule.

Import-time validation (`l2arc_dev_hdr_read`) checks at least:

- magic (with byteswap fallback)
- pool/vdev GUID match
- persistent version match
- `dh_log_entries` match current device settings
- `dh_end` match current computed end
- `dh_evict` in valid device range
- trim state compatibility when `l2arc_trim_ahead > 0`

If these checks fail, rebuild is skipped and the header may be reinitialized.

## 15.3 Log Block Pointer (`l2arc_log_blkptr_t`)

`l2arc_log_blkptr_t` is 64 bytes:

| Field | Size | Description |
|-------|------|-------------|
| `lbp_daddr` | 8 | Log block device offset (bytes) |
| `lbp_payload_asize` | 8 | Aligned payload size represented by this log block |
| `lbp_payload_start` | 8 | Offset of first payload buffer |
| `lbp_prop` | 8 | Packed properties |
| `lbp_cksum` | 32 | Checksum of stored log block bytes |

`lbp_prop` uses `L2BLK_*` packing macros:

```
63      61 60      57 56 55       48 47      40 39 38      32 31      16 15       0
+---------+----------+--+-----------+----------+--+----------+----------+----------+
|RESERVED |  STATE   |P |   TYPE    | CHECKSUM |PF| COMPRESS |  PSIZE   |  LSIZE   |
+---------+----------+--+-----------+----------+--+----------+----------+----------+
```

For `lbp_prop`, only these fields are populated by current code:

- `LSIZE` (set to `sizeof (l2arc_log_blk_phys_t)`, 64 KiB)
- `PSIZE` (actual aligned stored size)
- `COMPRESS` (`LZ4` if compressed, otherwise `OFF`)
- `CHECKSUM` (currently `ZIO_CHECKSUM_FLETCHER_4`)

The other bit ranges are reserved in pointer context.

## 15.4 Log Entry (`l2arc_log_ent_phys_t`)

Each log entry is 64 bytes and describes one cached ARC buffer:

| Offset | Size | Field |
|--------|------|-------|
| `0x00` | 16 | `le_dva` |
| `0x10` | 8 | `le_birth` |
| `0x18` | 8 | `le_prop` |
| `0x20` | 8 | `le_daddr` (L2ARC payload offset) |
| `0x28` | 8 | `le_complevel` |
| `0x30` | 16 | `le_pad[2]` |

`le_prop` uses the same bit layout as above, but entry-specific fields are used:

- `LSIZE`, `PSIZE`, `COMPRESS`
- `TYPE` (ARC buffer content type; encoded with +1 compatibility shim)
- `P` (`PROTECTED` bit for encryption/protection state)
- `PF` (`PREFETCH`)
- `STATE` (ARC state enum value used during restore)
- `CHECKSUM` is not currently populated for `le_prop`

## 15.5 Log Block (`l2arc_log_blk_phys_t`)

`l2arc_log_blk_phys_t` is 64 KiB total:

- `lb_magic` (`L2ARC_LOG_BLK_MAGIC`, ASCII `"LOGBLKHD"`)
- `lb_prev_lbp` (pointer to previous block in same chain)
- `lb_pad[7]` (header padded to 128 bytes)
- `lb_entries[L2ARC_LOG_BLK_MAX_ENTRIES]`

`L2ARC_LOG_BLK_MAX_ENTRIES` is `1022`, so payload size is:

- `1022 * 64 = 65408` bytes

Header + entries:

- `128 + 65408 = 65536` bytes (64 KiB)

## 15.6 Two Interleaved Log Chains

Persistent L2ARC uses two linked lists of log blocks, rooted at `dh_start_lbps[0]` and `dh_start_lbps[1]`.

On each commit (`l2arc_log_blk_commit`):

1. `lb_prev_lbp = dh_start_lbps[1]`
2. `dh_start_lbps[1] = dh_start_lbps[0]`
3. `dh_start_lbps[0] = newly committed block pointer`

This produces two interleaved time-ordered chains so rebuild can prefetch one block ahead while decoding another.

## 15.7 Rebuild Algorithm

At import / cache-vdev online (`l2arc_rebuild`):

1. Read and validate device header.
2. Seed traversal from `dh_start_lbps[0..1]`.
3. For each valid pointer:
   - read log block
   - verify `lbp_cksum`
   - decompress if needed (`OFF` or `LZ4`)
   - validate `lb_magic`
   - restore entries in reverse order to preserve temporal order in `l2ad_buflist`
4. Stop on invalid pointer, overwrite/eviction overlap, or any I/O/checksum/decode error.

Safety property: stale entries are harmless. ARC lookups include both DVA and birth TXG, so old cache payload cannot satisfy requests for newer blocks at the same offset.

## 15.8 Small-Device Behavior and Log Entry Count

Persistent metadata is size-gated per L2ARC device:

- if `l2ad_end < l2arc_rebuild_blocks_min_l2size` (default 1 GiB):
  - `l2ad_log_entries = 0` (no log blocks written)
- else:
  - `l2ad_log_entries = min((l2ad_end - l2ad_start) >> SPA_MAXBLOCKSHIFT, L2ARC_LOG_BLK_MAX_ENTRIES)`

With `SPA_MAXBLOCKSHIFT = 24` (16 MiB), this scales entry count down on smaller cache devices so a log block never tracks more max-size payload than the device can reasonably cycle.

Even when `l2ad_log_entries == 0`, the device header is still maintained.

## 15.9 Encryption Interaction

For protected/encrypted ARC buffers:

- log entries persist protection state (`PROTECTED` bit in `le_prop`)
- L2ARC writes encrypted bytes (not plaintext)
- data is transformed with dataset crypto context before write
- if key lookup fails at write time, buffer is skipped for L2ARC

No wrapping keys or master keys are stored in L2ARC metadata structures.

## 15.10 Compatibility and Visibility

- There is no `feature@l2arc_persistent` pool feature in `zfeature_common.c`.
- Format compatibility is carried by `dh_version` (`L2ARC_PERSISTENT_VERSION`, currently `1`).
- Linux exposes support as kernel feature string `org.openzfs:l2arc_persistent` under `/sys/module/zfs/features/kernel`.
- `zdb -l`, `zdb -ll`, and `zdb -lll` can decode header/log blocks/log entries directly from a cache device.
