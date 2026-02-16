# Chapter 14: Deduplication Tables and Block Cloning

> **Source:** `include/sys/ddt.h`, `include/sys/ddt_impl.h`, `include/sys/brt.h`, `include/sys/brt_impl.h`, `include/sys/dmu.h`, `include/sys/fs/zfs.h`, `module/zfs/ddt.c`, `module/zfs/ddt_log.c`, `module/zfs/ddt_zap.c`, `module/zfs/ddt_stats.c`, `module/zfs/brt.c`, `module/zcommon/zfeature_common.c`

OpenZFS has two distinct reference-tracking mechanisms for shared blocks:

- **DDT (Deduplication Table):** checksum-keyed, automatic dedup.
- **BRT (Block Reference Table):** offset-keyed, explicit block cloning.

Both are on-disk metadata systems and can coexist on one pool.

## 14.1 DDT Key Format (`ddt_key_t`)

A DDT key is:

- `ddk_cksum` (`zio_cksum_t`, 32 bytes)
- `ddk_prop` (`uint64_t`, 8 bytes)

Total key size: **40 bytes**.

`ddk_prop` bit layout:

```
63                           40 39 38        32 31        16 15         0
+-----------------------------+--+------------+------------+-------------+
|             0               |X |  COMPRESS  |   PSIZE    |    LSIZE    |
+-----------------------------+--+------------+------------+-------------+
```

- `LSIZE` and `PSIZE` are encoded in `SPA_MINBLOCKSHIFT` units.
- `COMPRESS` is the compression algorithm id.
- `X` is the crypt flag (`DDK_GET_CRYPT`), indicating block-pointer crypt semantics (`BP_USES_CRYPT`), including encrypted/authenticated crypt handling.

## 14.2 DDT Value Formats (`ddt_univ_phys_t`)

`ddt_univ_phys_t` has two on-disk payload modes.

### Traditional payload (`ddp_trad`)

- Four 64-byte slots (`DDT_PHYS_DITTO`, `SINGLE`, `DOUBLE`, `TRIPLE`)
- Total value size: **256 bytes**

Each slot layout:

| Offset | Size | Field |
|--------|------|-------|
| `0x00` | 48 | `ddp_dva[3]` |
| `0x30` | 8 | `ddp_refcnt` |
| `0x38` | 8 | `ddp_phys_birth` |

Notes:

- `DDT_PHYS_DITTO` is obsolete and not generated for new entries.
- Class selection is based on total refcount (`UNIQUE` for `1`, `DUPLICATE` for `>1`).

### Flat payload (`ddp_flat`, `feature@fast_dedup`)

Single value layout (72 bytes):

| Offset | Size | Field |
|--------|------|-------|
| `0x00` | 48 | `ddp_dva[3]` |
| `0x30` | 8 | `ddp_refcnt` |
| `0x38` | 8 | `ddp_phys_birth` |
| `0x40` | 8 | `ddp_class_start` (realtime seconds) |

In flat mode, new entries have one physical payload (`DDT_PHYS_FLAT`).

## 14.3 DDT Objects in MOS

DDT metadata is per-checksum (`sha256`, `sha512`, `skein`, `edonr`, `blake3`).

### Legacy layout (`DDT_VERSION_LEGACY`)

Objects are referenced directly from `DMU_POOL_DIRECTORY_OBJECT` with names:

- `DDT-<checksum>-<type>-<class>` (`DMU_POOL_DDT`)

Current type is `zap`, classes are `ditto`, `duplicate`, `unique`.

### Fast dedup layout (`DDT_VERSION_FDT`, `feature@fast_dedup`)

Each checksum gets a directory object in MOS:

- `DDT-<checksum>` (`DMU_POOL_DDT_DIR`)

Directory entries include:

- `version` (`DDT_DIR_VERSION`)
- `flags` (`DDT_DIR_FLAGS`)
- per-object links named `DDT-<checksum>-<type>-<class>`
- optional dedup log object links (`DDT-log-<checksum>-0/1`, created lazily on first log use)

Version/flags mapping in current OpenZFS:

- `DDT_VERSION_LEGACY` => no flags
- `DDT_VERSION_FDT` => `DDT_FLAG_FLAT | DDT_FLAG_LOG`

`DMU_POOL_DDT_STATS` (`DDT-statistics`) stores histogram payloads (`ddt_histogram_t`) keyed by DDT object name.

Allocation routing note:

- DDT objects (`DMU_OT_DDT_ZAP`) are routed by allocation class policy: dedicated dedup class first (if present), then special class when DDT-on-special is enabled, otherwise normal class.

## 14.4 DDT Log On Disk (`DDT_FLAG_LOG`)

Fast dedup uses an append log to avoid heavy random ZAP rewrites every txg.

Each log object:

- Type: `DMU_OTN_UINT64_METADATA`
- Data block size: `SPA_OLD_MAXBLOCKSIZE`
- Bonus: `ddt_log_header_t` (64 bytes)

`ddt_log_header_t`:

| Offset | Size | Field |
|--------|------|-------|
| `0x00` | 8 | `dlh_info` (version + flags) |
| `0x08` | 8 | `dlh_length` |
| `0x10` | 8 | `dlh_first_txg` |
| `0x18` | 40 | `dlh_checkpoint` (`ddt_key_t`) |

`dlh_info` bits:

- Bits `7:0`: log version (currently `1`)
- Bits `15:8`: log flags (`DDL_FLAG_FLUSHING`, `DDL_FLAG_CHECKPOINT`)

Log records are packed entries with `ddt_log_record_t` headers (`dlr_info`) plus payload.

For `DLR_ENTRY` records:

- record type in bits `7:0`
- record length in bits `23:8` (16-bit field)
- entry type in bits `55:48`
- entry class in bits `63:56`

Record payload is `ddt_log_record_entry_t`:

- `dlre_key` (`ddt_key_t`)
- `dlre_phys` (`ddt_univ_phys_t` payload sized for current DDT mode)

Typical record sizes (8-byte aligned):

- flat mode: **120 bytes**
- traditional mode: **304 bytes**

In current OpenZFS FDT mode (`DDT_FLAG_FLAT | DDT_FLAG_LOG`), dedup log records are flat-format.

## 14.5 DDT ZAP Entry Encoding

DDT storage type `zap` uses uint64-array keys and compressed values:

- Key: `ddt_key_t` interpreted as `uint64[5]`
- Value: compressed phys payload (`72` or `256` bytes before compression)

A one-byte value prefix stores:

- compression function id (`DDT_ZAP_COMPRESS_FUNCTION_MASK`)
- host byteorder bit (`DDT_ZAP_COMPRESS_BYTEORDER_MASK`)

Compression path uses ZLE by default, with fallback to uncompressed (`ZIO_COMPRESS_OFF`) if no gain.

## 14.6 BRT On-Disk Objects (Block Cloning)

BRT tracks explicit clone references by physical offset, per top-level vdev.

### MOS directory anchor

For each top-level vdev id `N` with active BRT entries:

- `com.fudosecurity:brt:vdev:<N>` -> object id of that vdev's BRT metadata object

(`BRT_OBJECT_VDEV_PREFIX` in `DMU_POOL_DIRECTORY_OBJECT`.)

### Per-vdev metadata object

Object type: `DMU_OTN_UINT64_METADATA`

- Data area: `uint16` entry-count array
- Data block size: `BRT_BLOCKSIZE` (`32 KiB`)
- Bonus: `brt_vdev_phys_t` (56 bytes)

`brt_vdev_phys_t` layout:

| Offset | Size | Field |
|--------|------|-------|
| `0x00` | 8 | `bvp_mos_entries` (entries ZAP object id) |
| `0x08` | 8 | `bvp_size` (number of `uint16` counters) |
| `0x10` | 8 | `bvp_byteorder` |
| `0x18` | 8 | `bvp_totalcount` |
| `0x20` | 8 | `bvp_rangesize` |
| `0x28` | 8 | `bvp_usedspace` |
| `0x30` | 8 | `bvp_savedspace` |

Entry-count array semantics:

- One `uint16` counter per region of size `bvp_rangesize`.
- Default range size is `BRT_RANGESIZE` (`16 MiB`).
- Counter value is the number of BRT entries whose offsets fall in that region.

### Per-vdev entries ZAP

`bvp_mos_entries` points to a ZAP object keyed by offset:

- Key: one `uint64` (first DVA offset within this top-level vdev)
- Value: `uint64` reference count

This object is created with `ZAP_FLAG_HASH64 | ZAP_FLAG_UINT64_KEY`.
Its dnode storage type is set to `DMU_OT_DDT_ZAP`.

BRT object lifecycle is lazy:

- Created on first real clone references for that top-level vdev
- Destroyed when the per-vdev BRT entry count drops to zero

## 14.7 `feature@block_cloning_endian`

Early block cloning pools stored BRT ZAP value metadata with legacy integer layout.
The `feature@block_cloning_endian` fix changes new BRT ZAP values to native uint64 layout.

On-disk compatibility behavior in `brt.c`:

- Pools without active endian-fix feature use legacy lookup/update parameterization.
- Pools with active fix use corrected uint64 parameterization.

This preserves readability of older pools while fixing new writes.

## 14.8 DDT and BRT Interaction

When cloning a block whose BP has dedup (`D`) bit set, BRT defers to DDT:

- `brt_pending_apply()` calls `ddt_addref()`.
- If DDT entry exists, no BRT entry is created.
- If DDT entry is missing (e.g., pruned), BRT can still track it via normal BRT path.

This avoids maintaining duplicate reference state in both tables for the same block.

## 14.9 Feature Flags and On-Disk Impact

| Feature | Description | On-Disk Impact |
|---------|-------------|----------------|
| `feature@fast_dedup` (`com.klarasystems:fast_dedup`) | Flat + logged DDT format | Enables per-checksum `DDT-<checksum>` directory objects, `version`/`flags`, flat phys payloads, and `DDT-log-*` objects |
| `feature@block_cloning` (`com.fudosecurity:block_cloning`) | Block Reference Tables | Adds per-vdev BRT objects anchored by `com.fudosecurity:brt:vdev:<id>` and offset->refcount ZAP entries |
| `feature@block_cloning_endian` (`com.truenas:block_cloning_endian`) | BRT ZAP endianness fix | Switches BRT ZAP value encoding path to corrected uint64 layout for new pools/entries |

Notes:

- Dedup itself is not a standalone `feature@dedup` flag in OpenZFS. The base dedup mechanism predates feature flags; modern format upgrades are represented by `feature@fast_dedup`.
- DDT and BRT both remain backward-compatible with older on-disk entries through format/version checks at load time.
