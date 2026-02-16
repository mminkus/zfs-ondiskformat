# Chapter 16: Pool Operations State

> **Source:** `include/sys/dmu.h`, `include/sys/dsl_scan.h`, `include/sys/vdev_rebuild.h`, `include/sys/spa_impl.h`, `include/sys/spa_checkpoint.h`, `include/sys/fs/zfs.h`, `include/sys/zio.h`, `module/zfs/dsl_scan.c`, `module/zfs/vdev_rebuild.c`, `module/zfs/spa_errlog.c`, `module/zfs/spa_history.c`, `module/zfs/spa_checkpoint.c`, `module/zfs/vdev_removal.c`, `module/zfs/vdev.c`, `module/zcommon/zfeature_common.c`

ZFS persists long-running pool operation state in two places:

- MOS directory entries (`DMU_POOL_DIRECTORY_OBJECT`, object `1`)
- Top-level vdev ZAP objects (for per-vdev state)

This chapter covers scrub/resilver scan state, sequential rebuild state, error logs, command history, checkpoint state, and vdev removal progress.

## 16.1 Persistent Anchors

| Scope | Key | Payload |
|-------|-----|---------|
| MOS dir ZAP | `scan` (`DMU_POOL_SCAN`) | `dsl_scan_phys_t` as `uint64[SCAN_PHYS_NUMINTS]` |
| MOS dir ZAP | `error_scrub` (`DMU_POOL_ERRORSCRUB`) | `dsl_errorscrub_phys_t` as `uint64[ERRORSCRUB_PHYS_NUMINTS]` |
| MOS dir ZAP | `errlog_last` (`DMU_POOL_ERRLOG_LAST`) | `uint64` object ID of `DMU_OT_ERROR_LOG` ZAP |
| MOS dir ZAP | `errlog_scrub` (`DMU_POOL_ERRLOG_SCRUB`) | `uint64` object ID of `DMU_OT_ERROR_LOG` ZAP |
| MOS dir ZAP | `history` (`DMU_POOL_HISTORY`) | `uint64` object ID of pool history object |
| MOS dir ZAP | `com.delphix:zpool_checkpoint` (`DMU_POOL_ZPOOL_CHECKPOINT`) | checkpointed `uberblock_t` stored as `uint64[]` |
| MOS dir ZAP | `com.delphix:removing` (`DMU_POOL_REMOVING`) | `spa_removing_phys_t` as `uint64[]` |
| Top-level vdev ZAP | `org.openzfs:vdev_rebuild` (`VDEV_TOP_ZAP_VDEV_REBUILD_PHYS`) | `vdev_rebuild_phys_t` as `uint64[12]` |
| Top-level vdev ZAP | `com.delphix:pool_checkpoint_sm` (`VDEV_TOP_ZAP_POOL_CHECKPOINT_SM`) | `uint64` object ID of checkpoint space map |

## 16.2 Scan / Scrub / Resilver State (`DMU_POOL_SCAN`)

`dsl_scan_phys_t` is the durable scan state record:

- On-disk size: `24 * 8 = 192` bytes (`SCAN_PHYS_NUMINTS`)
- Stored inline in MOS dir key `scan` (`DMU_POOL_SCAN`)
- Written with `zap_update(..., sizeof(uint64_t), SCAN_PHYS_NUMINTS, ...)`

Core fields:

- `scn_func`: `pool_scan_func_t` (`NONE`, `SCRUB`, `RESILVER`, `ERRORSCRUB`)
- `scn_state`: `dsl_scan_state_t` (`NONE`, `SCANNING`, `FINISHED`, `CANCELED`, `ERRORSCRUBBING`)
- `scn_queue_obj`: object ID of scan queue ZAP
- `scn_min_txg` / `scn_max_txg`: scan txg window
- `scn_cur_min_txg` / `scn_cur_max_txg`: current traversal window
- `scn_start_time` / `scn_end_time`: unix timestamps
- `scn_to_examine`, `scn_examined`, `scn_skipped`, `scn_processed`, `scn_errors`
- `scn_ddt_class_max` and `scn_ddt_bookmark` (`ddt_bookmark_t`) for DDT phase progress
- `scn_bookmark` (`zbookmark_phys_t`) for traversal resume position
- `scn_flags` (`dsl_scan_flags_t`)

`scn_queue_obj` details:

- Queue object is a ZAP created as `DMU_OT_SCAN_QUEUE` (or legacy `DMU_OT_ZAP_OTHER` on very old pools)
- Key: dataset object ID (integer key)
- Value: txg

### 16.2.1 Scan Flags

`scn_flags` currently defines:

- `DSF_VISIT_DS_AGAIN` (bit 0)
- `DSF_SCRUB_PAUSED` (bit 1)

`DSF_SCRUB_PAUSED` is used for durable pause/resume behavior of scrub operations.

### 16.2.2 Error-Scrub State (`DMU_POOL_ERRORSCRUB`)

Error-scrub has a separate persisted struct, `dsl_errorscrub_phys_t`:

- On-disk size: `9 * 8 = 72` bytes (`ERRORSCRUB_PHYS_NUMINTS`)
- Stored inline under MOS dir key `error_scrub`

Fields:

- `dep_func`, `dep_state`
- `dep_cursor` (serialized ZAP cursor for resumed traversal)
- `dep_start_time`, `dep_end_time`
- `dep_to_examine`, `dep_examined`, `dep_errors`
- `dep_paused_flags`

## 16.3 Sequential Rebuild State (`org.openzfs:vdev_rebuild`)

Sequential rebuild (device rebuild) is tracked per top-level vdev by
`vdev_rebuild_phys_t`:

- On-disk size: `12 * 8 = 96` bytes
- Stored as `uint64[REBUILD_PHYS_ENTRIES]` in top-level vdev ZAP key
  `VDEV_TOP_ZAP_VDEV_REBUILD_PHYS` (`"org.openzfs:vdev_rebuild"`)

Fields:

- `vrp_rebuild_state` (`vdev_rebuild_state_t`: `NONE`, `ACTIVE`, `CANCELED`, `COMPLETE`)
- `vrp_last_offset`
- `vrp_min_txg`, `vrp_max_txg`
- `vrp_start_time`, `vrp_end_time`
- `vrp_scan_time_ms`
- `vrp_bytes_scanned`, `vrp_bytes_issued`, `vrp_bytes_rebuilt`, `vrp_bytes_est`
- `vrp_errors`

Behavior notes:

- State is updated during rebuild progress (`vdev_rebuild_update_sync()`).
- `feature@device_rebuild` is incremented at start and decremented at completion/cancel.
- During import, missing/corrupt rebuild ZAP payload (`ENOENT`, `EOVERFLOW`, `ECKSUM`) is tolerated by clearing rebuild state rather than failing pool load.

## 16.4 Persistent Error Logs (`errlog_last`, `errlog_scrub`)

MOS dir entries:

- `DMU_POOL_ERRLOG_LAST` -> object ID
- `DMU_POOL_ERRLOG_SCRUB` -> object ID

Both point to `DMU_OT_ERROR_LOG` ZAP objects.

Rotation model:

- Errors are accumulated while pool is active.
- At scrub completion, scrub log rotates into "last" and previous "last" is discarded.

### 16.4.1 Legacy Key Format

Without `feature@head_errlog`, error-log entries are keyed by a string form of
`zbookmark_phys_t`:

- key: `"objset:object:level:blkid"` (hex fields)
- value: optional NUL-terminated name string (often empty)

### 16.4.2 `feature@head_errlog` Format

With `feature@head_errlog` enabled:

- Root error-log object maps `head_dataset_obj` -> child error-log object ID
- Child error-log object keys use `zbookmark_err_phys_t`:
  `"object:level:blkid:birth"`
- Value remains optional name string

This adds birth txg to persisted error identity, improving correctness when
blocks are rewritten or snapshots are involved.

`spa_error_entry_t` is an in-memory staging record flushed into these on-disk
ZAPs at sync time.

## 16.5 Pool History (`DMU_POOL_HISTORY`)

`DMU_POOL_HISTORY` points to a history object created as:

- Data object type: `DMU_OT_SPA_HISTORY`
- Bonus type: `DMU_OT_SPA_HISTORY_OFFSETS`
- Bonus payload: `spa_history_phys_t` (40 bytes)

`spa_history_phys_t` fields:

- `sh_pool_create_len`: logical end of initial `zpool create` record
- `sh_phys_max_off`: physical ring size
- `sh_bof`, `sh_eof`: logical begin/end offsets
- `sh_records_lost`: overwrite counter

Record encoding in the data object:

- `uint64` record length in little-endian
- packed nvlist bytes (`nvlist_pack(..., NV_ENCODE_NATIVE, ...)`)

History is a ring buffer, but the original create record is preserved by
keeping `sh_pool_create_len` as a permanent floor.

Size policy at creation:

- target: `0.1%` of pool size
- clamped to `[128 KiB, 1 GiB]`

## 16.6 Pool Checkpoint State (`DMU_POOL_ZPOOL_CHECKPOINT`)

There is no dedicated `pool_checkpoint_phys_t` struct in current OpenZFS.
Checkpoint durability uses:

- MOS dir entry `DMU_POOL_ZPOOL_CHECKPOINT` containing a full `uberblock_t`
  payload (as `uint64[]`)
- Per-top-vdev checkpoint space maps referenced by
  `VDEV_TOP_ZAP_POOL_CHECKPOINT_SM`
- In-memory accounting (`spa_checkpoint_info_t`)

Lifecycle:

1. Checkpoint create writes checkpointed uberblock into MOS dir key
   `DMU_POOL_ZPOOL_CHECKPOINT`, sets `spa_checkpoint_txg`, and activates
   `feature@zpool_checkpoint`.
2. While checkpoint exists, freed checkpoint-owned extents are logged into
   vdev checkpoint space maps (not returned to normal allocatable space).
3. Discard removes `DMU_POOL_ZPOOL_CHECKPOINT` first, then asynchronously
   drains/deletes per-vdev checkpoint space maps; feature refcount drops only
   after drain completes.

Operational state detection:

- feature active + key present => checkpoint exists
- feature active + key absent => checkpoint discard in progress

## 16.7 Vdev Removal Progress (`DMU_POOL_REMOVING`)

Device removal progress is persisted as `spa_removing_phys_t`:

- On-disk size: `7 * 8 = 56` bytes
- Stored inline in MOS dir key `com.delphix:removing`

Fields:

- `sr_state` (`dsl_scan_state_t`)
- `sr_removing_vdev`: vdev currently being removed, or `UINT64_MAX` (stored as `-1`) when none
- `sr_prev_indirect_vdev`: most recently removed indirect vdev, or `UINT64_MAX` if none
- `sr_start_time`, `sr_end_time`
- `sr_to_copy`: bytes that must be migrated
- `sr_copied`: bytes copied or freed while removal runs

Notes:

- The `sr_prev_indirect_vdev` chain is used at import to reopen indirect mapping state in newest-to-oldest order.
- `sr_to_copy`/`sr_copied` are explicit counters (not derived from mappings) so completion accounting remains correct when blocks are freed before copy.

## 16.8 Related Feature Flags

| Feature Property | GUID | On-Disk Role |
|------------------|------|--------------|
| `feature@device_rebuild` | `org.openzfs:device_rebuild` | Enables persistent per-top-vdev rebuild state (`org.openzfs:vdev_rebuild`) |
| `feature@zpool_checkpoint` | `com.delphix:zpool_checkpoint` | Enables checkpoint uberblock persistence and checkpoint space-map lifecycle |
| `feature@head_errlog` | `com.delphix:head_errlog` | Switches persistent error-log format to per-head-dataset layout with birth-aware keys |
| `feature@device_removal` | `com.delphix:device_removal` | Enables vdev removal machinery that persists `DMU_POOL_REMOVING` progress state |
