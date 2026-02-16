# Chapter 13: Feature Flags Reference

> **Source:** `include/zfeature_common.h`, `include/sys/zfeature.h`, `include/sys/dmu.h`, `include/sys/fs/zfs.h`, `include/sys/spa_impl.h`, `module/zfs/zfeature.c`, `module/zcommon/zfeature_common.c`, `module/zfs/spa.c`, `module/zfs/spa_misc.c`, `module/zfs/spa_config.c`, `lib/libzfs/libzfs_pool.c`

Feature flags are the modern versioning mechanism for ZFS on-disk format changes. Instead of a single monotonically increasing pool version, each independent format change is tracked by a named feature GUID and a reference count.

## 13.1 Versioning Model

When feature flags are in use, `spa_version` is `SPA_VERSION_FEATURES` (`5000`).
Pools are first upgraded to `SPA_VERSION_BEFORE_FEATURES` (`28`), then feature-gated changes are tracked per feature.

Each feature has:

- A stable GUID (on-disk ID), e.g. `com.delphix:async_destroy`
- A user-facing short name (`fi_uname`), exposed as `feature@<name>` pool property
- Flags describing compatibility/load behavior
- Optional dependencies on other features

Feature GUIDs are required to use reverse-DNS form with a single colon.

## 13.2 On-Disk Objects

Feature metadata is stored in MOS directory entries (`DMU_POOL_DIRECTORY_OBJECT`):

| MOS Key | Purpose |
|---------|---------|
| `features_for_read` (`DMU_POOL_FEATURES_FOR_READ`) | GUID -> refcount for features required to read |
| `features_for_write` (`DMU_POOL_FEATURES_FOR_WRITE`) | GUID -> refcount for features required to write |
| `feature_descriptions` (`DMU_POOL_FEATURE_DESCRIPTIONS`) | GUID -> human-readable description |
| `feature_enabled_txg` (`DMU_POOL_FEATURE_ENABLED_TXG`) | GUID -> txg when feature was enabled (optional; created lazily) |

All enabled features appear in either `features_for_read` or `features_for_write`, never both.

Selection is determined by `ZFEATURE_FLAG_READONLY_COMPAT`:

- If `READONLY_COMPAT` is set: feature is stored in `features_for_write`
- Otherwise: feature is stored in `features_for_read`

In other words, unsupported `READONLY_COMPAT` features still allow read-only imports; unsupported non-`READONLY_COMPAT` features do not.

Import checks only require support for **active** features (refcount `> 0`). Features with refcount `0` are enabled but inactive, and do not block import.

## 13.3 Feature States and Refcounts

A feature's state is encoded by presence/value of its refcount entry:

- `disabled`: no entry in feature ZAP
- `enabled`: entry exists with refcount `0`
- `active`: entry exists with refcount `> 0`

The framework does not interpret non-zero values beyond "active"; individual features may use refcounts as meaningful counters.

## 13.4 Label Copy of Read-Critical Features

The vdev label nvlist includes `ZPOOL_CONFIG_FEATURES_FOR_READ` so import can validate read-critical support before opening MOS.

The label stores only feature names (no refcounts). In-memory this is `spa_label_features`.

`spa_activate_mos_feature()` and `spa_deactivate_mos_feature()` maintain this label set for active MOS-critical features.

During import, ZFS checks label `features_for_read`; unknown entries fail load with unsupported-feature error.

## 13.5 Flags and Types

`zfeature_flags_t`:

- `ZFEATURE_FLAG_READONLY_COMPAT` (`RO`): unsupported feature still allows read-only open
- `ZFEATURE_FLAG_MOS` (`MOS`): needed to load/read MOS metadata path
- `ZFEATURE_FLAG_ACTIVATE_ON_ENABLE` (`AOE`): initial refcount is `1` when enabled
- `ZFEATURE_FLAG_PER_DATASET` (`PDS`): tracked per dataset
- `ZFEATURE_FLAG_NO_UPGRADE` (`NUP`): not auto-enabled by `zpool upgrade`

`zfeature_type_t`:

- `ZFEATURE_TYPE_BOOLEAN`
- `ZFEATURE_TYPE_UINT64_ARRAY`

`UINT64_ARRAY` is currently used by per-dataset features that store array payloads (currently `feature@redacted_datasets`).

## 13.6 Dependency and Enable Semantics

`spa_feature_enable()` recursively enables dependencies first, then the requested feature.

Enable-time behavior:

- Feature description is written to `feature_descriptions`
- Refcount is initialized to `0`, or `1` if `AOE`
- If pool feature `enabled_txg` is enabled, `feature_enabled_txg` records the enabling txg

Compatibility note: features should not rely on enable-time object creation; on-disk objects are generally created on first real use.

## 13.7 Per-Dataset Features On Disk

Per-dataset features (`PDS`) are recorded in the dataset object ZAP (`dsl_dataset` object ID), keyed by feature GUID:

- Boolean type: ZAP entry exists with `uint64` payload
- `UINT64_ARRAY` type: ZAP entry stores a `uint64[]` payload

Activating/deactivating a dataset feature also increments/decrements the corresponding pool feature refcount.

## 13.8 Compatibility Sets and `NO_UPGRADE`

Pool compatibility settings (`ZPOOL_PROP_COMPATIBILITY`) use feature-set files from:

- `ZPOOL_SYSCONF_COMPAT_D` (system config)
- `ZPOOL_DATA_COMPAT_D` (distribution data)

Hard-wired compatibility values include:

- `legacy`
- `off`

`NO_UPGRADE` features are excluded from default `zpool upgrade` behavior. They are enabled only by explicit `feature@name=enabled` requests, or when explicitly selected by policy during pool creation workflows.

## 13.9 Tooling Note: `zhack`

OpenZFS ships `zhack` (`cmd/zhack.c`) as a low-level debugging tool. It is
often packaged by distributions (for example, Debian and FreeBSD builds).

For this chapter, the most useful command is:

```bash
zhack feature stat <pool>
```

Its output maps directly to on-disk feature objects:

- `for_read_obj` -> `features_for_read` (`DMU_POOL_FEATURES_FOR_READ`)
- `for_write_obj` -> `features_for_write` (`DMU_POOL_FEATURES_FOR_WRITE`)
- `descriptions_obj` -> `feature_descriptions` (`DMU_POOL_FEATURE_DESCRIPTIONS`)
- `enabled_txg_obj` -> `feature_enabled_txg` (`DMU_POOL_FEATURE_ENABLED_TXG`)
- `label config` -> label `features_for_read` set (`ZPOOL_CONFIG_FEATURES_FOR_READ`)

`zhack` can also mutate feature state (`feature enable`, `feature ref`) and
even inject synthetic feature GUIDs for testing. It is intended for debugging,
not normal administration; misuse can corrupt pools.

## 13.10 Current Feature Registry (OpenZFS)

Legend:

- `RO` = `ZFEATURE_FLAG_READONLY_COMPAT`
- `MOS` = `ZFEATURE_FLAG_MOS`
- `AOE` = `ZFEATURE_FLAG_ACTIVATE_ON_ENABLE`
- `PDS` = `ZFEATURE_FLAG_PER_DATASET`
- `NUP` = `ZFEATURE_FLAG_NO_UPGRADE`

Scope notes:

- `pool`: activation is tracked only at pool level.
- `dataset`: feature has per-dataset on-disk markers (`PDS`) in addition to the pool-level feature entry/refcount.

| Feature Property | GUID | Scope | Flags | Dependencies |
|------------------|------|-------|-------|--------------|
| `feature@async_destroy` | `com.delphix:async_destroy` | pool | RO | - |
| `feature@empty_bpobj` | `com.delphix:empty_bpobj` | pool | RO | - |
| `feature@lz4_compress` | `org.illumos:lz4_compress` | pool | AOE | - |
| `feature@multi_vdev_crash_dump` | `com.joyent:multi_vdev_crash_dump` | pool | - | - |
| `feature@spacemap_histogram` | `com.delphix:spacemap_histogram` | pool | RO | - |
| `feature@enabled_txg` | `com.delphix:enabled_txg` | pool | RO | - |
| `feature@hole_birth` | `com.delphix:hole_birth` | pool | MOS,AOE | `feature@enabled_txg` |
| `feature@zpool_checkpoint` | `com.delphix:zpool_checkpoint` | pool | RO | - |
| `feature@spacemap_v2` | `com.delphix:spacemap_v2` | pool | RO,AOE | - |
| `feature@extensible_dataset` | `com.delphix:extensible_dataset` | pool | - | - |
| `feature@bookmarks` | `com.delphix:bookmarks` | pool | RO | `feature@extensible_dataset` |
| `feature@filesystem_limits` | `com.joyent:filesystem_limits` | pool | RO | `feature@extensible_dataset` |
| `feature@embedded_data` | `com.delphix:embedded_data` | pool | MOS,AOE | - |
| `feature@livelist` | `com.delphix:livelist` | pool | RO | `feature@extensible_dataset` |
| `feature@log_spacemap` | `com.delphix:log_spacemap` | pool | RO | `feature@spacemap_v2` |
| `feature@large_blocks` | `org.open-zfs:large_blocks` | dataset | PDS | `feature@extensible_dataset` |
| `feature@large_dnode` | `org.zfsonlinux:large_dnode` | dataset | PDS | `feature@extensible_dataset` |
| `feature@sha512` | `org.illumos:sha512` | dataset | PDS | `feature@extensible_dataset` |
| `feature@skein` | `org.illumos:skein` | dataset | PDS | `feature@extensible_dataset` |
| `feature@edonr` | `org.illumos:edonr` | dataset | PDS | `feature@extensible_dataset` |
| `feature@redaction_bookmarks` | `com.delphix:redaction_bookmarks` | pool | - | `feature@bookmark_v2`, `feature@extensible_dataset`, `feature@bookmarks` |
| `feature@redacted_datasets` | `com.delphix:redacted_datasets` | dataset | PDS | `feature@extensible_dataset` |
| `feature@bookmark_written` | `com.delphix:bookmark_written` | pool | - | `feature@bookmark_v2`, `feature@extensible_dataset`, `feature@bookmarks` |
| `feature@device_removal` | `com.delphix:device_removal` | pool | MOS | - |
| `feature@obsolete_counts` | `com.delphix:obsolete_counts` | pool | RO | `feature@extensible_dataset`, `feature@device_removal` |
| `feature@userobj_accounting` | `org.zfsonlinux:userobj_accounting` | dataset | RO,PDS | `feature@extensible_dataset` |
| `feature@bookmark_v2` | `com.datto:bookmark_v2` | pool | - | `feature@extensible_dataset`, `feature@bookmarks` |
| `feature@encryption` | `com.datto:encryption` | dataset | PDS | `feature@extensible_dataset`, `feature@bookmark_v2` |
| `feature@project_quota` | `org.zfsonlinux:project_quota` | dataset | RO,PDS | `feature@extensible_dataset` |
| `feature@allocation_classes` | `org.zfsonlinux:allocation_classes` | pool | RO | - |
| `feature@resilver_defer` | `com.datto:resilver_defer` | pool | RO | - |
| `feature@device_rebuild` | `org.openzfs:device_rebuild` | pool | RO | - |
| `feature@zstd_compress` | `org.freebsd:zstd_compress` | dataset | PDS | `feature@extensible_dataset` |
| `feature@draid` | `org.openzfs:draid` | pool | MOS | - |
| `feature@zilsaxattr` | `org.openzfs:zilsaxattr` | dataset | RO,PDS | `feature@extensible_dataset` |
| `feature@head_errlog` | `com.delphix:head_errlog` | pool | AOE | - |
| `feature@blake3` | `org.openzfs:blake3` | dataset | PDS | `feature@extensible_dataset` |
| `feature@block_cloning` | `com.fudosecurity:block_cloning` | pool | RO | - |
| `feature@block_cloning_endian` | `com.truenas:block_cloning_endian` | pool | RO | - |
| `feature@vdev_zaps_v2` | `com.klarasystems:vdev_zaps_v2` | pool | MOS | - |
| `feature@redaction_list_spill` | `com.delphix:redaction_list_spill` | pool | - | `feature@redaction_bookmarks` |
| `feature@raidz_expansion` | `org.openzfs:raidz_expansion` | pool | MOS | - |
| `feature@fast_dedup` | `com.klarasystems:fast_dedup` | pool | RO | - |
| `feature@longname` | `org.zfsonlinux:longname` | dataset | PDS | `feature@extensible_dataset` |
| `feature@large_microzap` | `com.klarasystems:large_microzap` | dataset | RO,PDS | `feature@extensible_dataset`, `feature@large_blocks` |
| `feature@dynamic_gang_header` | `com.klarasystems:dynamic_gang_header` | pool | MOS,NUP | - |
| `feature@physical_rewrite` | `com.truenas:physical_rewrite` | dataset | RO,PDS | `feature@extensible_dataset` |
