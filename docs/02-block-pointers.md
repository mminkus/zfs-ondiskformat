# Chapter 2: Block Pointers and Indirect Blocks

> **Source:** `include/sys/spa.h` (blkptr_t layout and macros), `include/sys/blkptr.h`, `module/zfs/zio.c`

Data is transferred between disk and memory in units called **blocks**. A **block pointer** (`blkptr_t`) is a 128-byte structure used to physically locate, verify, and describe a block of data on disk.

## Block Pointer Layout

The 128-byte `blkptr_t` is organized as 16 x 8-byte words. Each word is referenced by its index (0-f). Fields are packed using bit manipulation macros, not C bitfields.

```
Word   64      56      48      40      32      24      16      8       0
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0    |  pad  |         vdev1          | pad   |       ASIZE           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  1    |G|                       offset1                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  2    |  pad  |         vdev2          | pad   |       ASIZE           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  3    |G|                       offset2                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  4    |  pad  |         vdev3          | pad   |       ASIZE           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  5    |G|                       offset3                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  6    |BDX|lvl|  type  | cksum |E| comp|     PSIZE     |    LSIZE      |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  7    |R|                       padding                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  8    |                         padding                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  9    |                    physical birth txg                          |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  a    |                    logical birth txg                           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  b    |                      fill count                               |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  c    |                     checksum[0]                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  d    |                     checksum[1]                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  e    |                     checksum[2]                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  f    |                     checksum[3]                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
```

**Legend:**

| Field | Bits | Description |
|-------|------|-------------|
| **vdev** | 24 | Virtual device ID |
| **ASIZE** | 24 | Allocated size (in 512-byte sectors, no bias) |
| **G** | 1 | Gang block indicator |
| **offset** | 63 | Offset into vdev (in 512-byte sectors) |
| **B** | 1 | Byte order (0=big-endian, 1=little-endian) |
| **D** | 1 | Dedup indicator |
| **X** | 1 | Encryption indicator |
| **lvl** | 5 | Indirection level |
| **type** | 8 | DMU object type (see [glossary](glossary.md#dmu-object-types)) |
| **cksum** | 8 | Checksum algorithm (see [glossary](glossary.md#checksum-algorithms)) |
| **E** | 1 | Embedded data indicator |
| **comp** | 7 | Compression algorithm (see [glossary](glossary.md#compression-algorithms)) |
| **PSIZE** | 16 | Physical size (in 512-byte sectors, +1) |
| **LSIZE** | 16 | Logical size (in 512-byte sectors, +1) |
| **R** | 1 | Rewrite indicator (block was reallocated at physical birth txg) |

## 2.1 DVA -- Data Virtual Address

A **DVA** (Data Virtual Address) is the combination of a vdev ID and offset that uniquely identifies a block's location on a specific device.

Each block pointer contains up to three DVAs (dva1, dva2, dva3), allowing up to three copies of the block data. All copies are identical. The number of DVAs used is called the block pointer's "wideness":

| Wideness | DVAs Used | Typical Use |
|----------|-----------|-------------|
| Single-wide | 1 (dva1) | User data |
| Double-wide | 2 (dva1, dva2) | Metadata |
| Triple-wide | 3 (dva1, dva2, dva3) | Critical metadata (e.g., uberblock root) |

**DVA encoding** (per DVA, 16 bytes = 2 words):

- **vdev**: 24-bit integer uniquely identifying the vdev containing this block.
- **offset**: 63-bit integer, the offset in 512-byte sectors from the start of the allocatable region of the vdev.

To compute the physical byte address on the vdev:

```
physical_address = (offset << 9) + 0x400000
```

The `0x400000` (4 MB) offset accounts for the two front vdev labels (2 x 256 KB) and the boot block (3.5 MB).

## 2.2 GRID

Reserved field for RAID-Z layout information. In the original specification this was reserved for future use. The space occupied by GRID in the original layout is now used as padding between the vdev and ASIZE fields.

## 2.3 Gang Blocks

A **gang block** is used when a contiguous allocation of the requested size is unavailable. ZFS allocates several smaller blocks totaling the requested size, then creates a gang block containing block pointers to those smaller blocks. The requester sees a single logical block.

Gang blocks are identified by the **G bit** in the DVA:

| G Bit | Meaning |
|-------|---------|
| 0 | Normal (non-gang) block |
| 1 | Gang block |

A gang block is a self-checksumming header (`zio_gbh_phys_t`). The header size is at least `SPA_MINBLOCKSIZE` (512 bytes) and can be larger; it contains an array of block pointers followed by a `zio_eck_t` trailer. The number of block pointers is `(size - sizeof(zio_eck_t)) / sizeof(blkptr_t)`.

```
Gang Block (size = SPA_MINBLOCKSIZE or larger)
+─────────────────────────────────────────────+
| blkptr_t  zg_blkptr[N]                    |
+─────────────────────────────────────────────+
| padding                                     |
+─────────────────────────────────────────────+
| zio_eck_t  zg_eck                           |   checksum trailer
+─────────────────────────────────────────────+
```

The checksum trailer (`zio_eck_t`) consists of:

- **`zec_magic`**: Magic number `0x210da7ab10c7a11` (`ZEC_MAGIC`)
- **`zec_cksum`**: A `zio_cksum_t` (four `uint64_t` words) containing the gang header checksum

## 2.4 Checksum

ZFS checksums all data and metadata by default. The checksum algorithm is identified by the 8-bit `cksum` field in the block pointer. See the [glossary](glossary.md#checksum-algorithms) for the full table of algorithms and their numeric values.

The 256-bit checksum is stored across four 64-bit words: `checksum[0]` through `checksum[3]` (words c-f of the block pointer).

If checksumming is disabled (`cksum` = 2, "off"), all four checksum words are zero.

The checksum is always computed over the data the block pointer references. Gang blocks and ZIL blocks are self-checksumming: their checksums are stored in a `zio_eck_t` trailer embedded within the block itself, not in the parent block pointer.

## 2.5 Compression

The compression algorithm used for a block is identified by the 7-bit `comp` field. See the [glossary](glossary.md#compression-algorithms) for the full table.

When compression is enabled, data is compressed before being written to disk. The block pointer records both the logical (uncompressed) size and the physical (compressed) size.

## 2.6 Block Size

Three size fields describe each block:

| Field | Description |
|-------|-------------|
| **LSIZE** | Logical size: the uncompressed data size |
| **PSIZE** | Physical size: the size on disk after compression |
| **ASIZE** | Allocated size: total space consumed including RAID-Z parity and gang block overhead |

LSIZE and PSIZE are stored as the number of 512-byte sectors **minus one**. To get the byte size: `(stored_value + 1) * 512`.  
ASIZE is stored as the number of 512-byte sectors **with no bias**. To get the byte size: `stored_value * 512`.

When compression is off and no RAID-Z or gang overhead applies, LSIZE = PSIZE = ASIZE.

### Large Blocks (`feature@large_blocks`)

The LSIZE and PSIZE fields are each 16 bits wide, encoding sizes up to 32 MB (`2^16 * 512 bytes`). The original ZFS format limited record sizes to 128 KB, but modern OpenZFS supports blocks up to 16 MB when the `org.open-zfs:large_blocks` feature is enabled. The encoding did not change -- larger values in the same 16-bit fields simply became valid.

## 2.7 Endianness

The **B** (byteorder) bit indicates the byte order in which the block's data was written:

| Value | Byte Order |
|-------|------------|
| 0 | Big-endian |
| 1 | Little-endian |

Blocks are always written in the machine's native byte order. When a pool is imported on a machine with different endianness, ZFS byte-swaps on read.

## 2.8 Type

The 8-bit `type` field identifies what kind of data the block holds. This corresponds to the DMU object type. See the [glossary](glossary.md#dmu-object-types) for the complete list.

## 2.9 Level

The 5-bit `lvl` field indicates the number of levels of indirection between this block pointer and the actual data. Level 0 block pointers point directly to data. Level 1 block pointers point to indirect blocks containing level 0 block pointers, and so on. See [Chapter 3](03-dmu.md) for a detailed description of indirection.

## 2.10 Fill Count

The `fill` field (word b) counts the number of non-zero block pointers beneath this block pointer.

- For a level-0 data block pointer, the fill count is 1.
- For indirect blocks, it is the sum of fill counts of all block pointers within.
- For block pointers of type `DMU_OT_DNODE`, the fill count instead represents the number of allocated dnodes beneath this block pointer.

## 2.11 Birth Transaction

- **Physical birth txg** (word 9): The transaction group in which `dva[0]` was written to disk. Zero if same as logical birth txg.
- **Logical birth txg** (word a): The transaction group in which this block was logically created.

The birth transaction is used for incremental operations such as `zfs send` -- only blocks born after a given txg need to be included.

## 2.12 Dedup

The **D** (dedup) bit (bit 62 of word 6) indicates that this block is deduplicated. When set, the block's checksum serves as a lookup key into the **Dedup Table** (DDT), allowing multiple block pointers to reference the same physical data without storing duplicate copies.

Dedup was added in pool version 21. See [glossary](glossary.md#pool-versions) for pool version details.

## 2.13 Rewrite

The **R** (rewrite) bit (bit 63 of word 7) indicates that a block was physically rewritten (reallocated) without changing its logical contents. This preserves the original logical birth txg while recording a new physical birth txg, which is important for incremental `zfs send` correctness.

Added by `feature@physical_rewrite` (`com.truenas:physical_rewrite`).

## 2.14 Padding

Words 7-8 (except for the R bit in word 7) contain padding reserved for future use.

## 2.15 Embedded Block Pointers

> **Source:** `include/sys/spa.h`, `module/zfs/blkptr.c`

When the **E** (embedded) bit is set (bit 39 of word 6), the block pointer does not point to an on-disk block. Instead, it stores up to **112 bytes of payload data** inline within the 128-byte block pointer structure itself. This is used for very small blocks that compress to 112 bytes or less, eliminating the need for a separate disk allocation.

Added by `feature@embedded_data` (`com.delphix:embedded_data`).

### Embedded Block Pointer Layout

The embedded layout repurposes the DVA, padding, fill count, and checksum fields for payload storage:

```
Word   64      56      48      40      32      24      16      8       0
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0    |                         payload                                |
  1    |                         payload                                |
  2    |                         payload                                |
  3    |                         payload                                |
  4    |                         payload                                |
  5    |                         payload                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  6    |BDX|lvl| type  | etype |E| comp| PSIZE |         LSIZE         |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  7    |                         payload                                |
  8    |                         payload                                |
  9    |                         payload                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  a    |                    logical birth txg                           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  b    |                         payload                                |
  c    |                         payload                                |
  d    |                         payload                                |
  e    |                         payload                                |
  f    |                         payload                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
```

14 of the 16 words carry payload data (112 bytes total). Word 6 (blk_prop) and word a (logical birth txg) retain their standard meanings.

### Embedded vs. Standard Block Pointers

| Field | Standard BP | Embedded BP |
|-------|------------|-------------|
| **cksum** (bits 40-47) | Checksum algorithm | Embedded type (`etype`) |
| **LSIZE** (bits 0-24) | Sectors minus one (16 bits) | Bytes (25 bits, no sector encoding) |
| **PSIZE** (bits 25-31) | Sectors minus one (16 bits) | Bytes (7 bits, no sector encoding) |
| DVAs, fill, checksum | Present | Replaced by payload |
| Physical birth txg | Present | Absent |

### Embedded Types (`etype`)

| Value | Constant | Description |
|-------|----------|-------------|
| 0 | `BP_EMBEDDED_TYPE_DATA` | Regular embedded data |
| 1 | `BP_EMBEDDED_TYPE_RESERVED` | Reserved |
| 2 | `BP_EMBEDDED_TYPE_REDACTED` | Redacted block marker (see below) |

### Redacted Block Pointers

A **redacted block pointer** is an embedded block pointer with `etype` set to `BP_EMBEDDED_TYPE_REDACTED`. It marks a block whose data was intentionally excluded from a redacted `zfs send` stream. The receiving side stores this marker so it knows the data is missing and cannot be read.

Added by `feature@redacted_datasets` (`com.delphix:redacted_datasets`).

## 2.16 Encrypted Block Pointers

> **Source:** `include/sys/spa.h`, `include/sys/zio_crypt.h`

When the **X** (crypt) bit is set (bit 61 of word 6), the block pointer stores encryption metadata alongside the block location. The X bit has three interpretations depending on context:

| Condition | Interpretation |
|-----------|---------------|
| X=1, level 0, encrypted object type | **Encrypted**: data is encrypted and authenticated |
| X=1, level 0, non-encrypted object type | **Authenticated**: data is authenticated (MAC) but not encrypted |
| X=1, level > 0 | **Indirect MAC**: checksum covers MACs of child blocks |

Added by `feature@encryption` (`com.datto:encryption`).

### Encrypted Block Pointer Layout

Encrypted block pointers sacrifice DVA[2] (the third copy) and half the checksum space to store cryptographic metadata:

```
Word   64      56      48      40      32      24      16      8       0
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0    |  pad  |         vdev1          | pad   |       ASIZE           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  1    |G|                       offset1                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  2    |  pad  |         vdev2          | pad   |       ASIZE           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  3    |G|                       offset2                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  4    |                          salt (64 bits)                        |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  5    |                     IV1 (first 64 bits of IV)                  |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  6    |BDX|lvl| type  | cksum |E| comp|     PSIZE     |    LSIZE      |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  7    |R|                       padding                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  8    |                         padding                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  9    |                    physical birth txg                          |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  a    |                    logical birth txg                           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  b    |       IV2 (32 bits)            |       fill count (32 bits)    |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  c    |                     checksum[0]                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  d    |                     checksum[1]                                |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  e    |                       MAC[0]                                   |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  f    |                       MAC[1]                                   |
       +-------+-------+-------+-------+-------+-------+-------+-------+
```

### Encrypted vs. Standard Block Pointers

| Field | Standard BP | Encrypted BP |
|-------|------------|--------------|
| DVA[2] (words 4-5) | Third block copy | Salt (64 bits) + IV1 (64 bits) |
| Fill count (word b) | 64 bits | IV2 (upper 32 bits) + fill count (lower 32 bits) |
| Checksum (words c-f) | 256-bit checksum | 128-bit checksum (words c-d) + 128-bit MAC (words e-f) |

### Cryptographic Fields

| Field | Size | Description |
|-------|------|-------------|
| **Salt** | 8 bytes | Random per-block salt for key derivation |
| **IV** (IV1 + IV2) | 12 bytes | Initialization vector for AES encryption (96 bits total) |
| **MAC** | 16 bytes | Message Authentication Code for tamper detection |

### Consequences

- Encrypted blocks support a **maximum of 2 DVAs** (not 3), since DVA[2] stores the salt and IV.
- The checksum is reduced from 256 bits to 128 bits to make room for the MAC.
- Embedded block pointers (`E=1`) **cannot** be encrypted (`X=1`); the two are mutually exclusive.
- The encryption algorithm (AES-128/192/256-CCM or AES-128/192/256-GCM) is stored in the dataset properties, not in the block pointer.
