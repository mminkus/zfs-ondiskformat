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
| **ASIZE** | 24 | Allocated size (in 512-byte sectors, +1) |
| **G** | 1 | Gang block indicator |
| **offset** | 63 | Offset into vdev (in 512-byte sectors) |
| **B** | 1 | Byte order (0=big-endian, 1=little-endian) |
| **D** | 1 | Dedup indicator |
| **X** | 1 | Encryption indicator |
| **lvl** | 7 | Indirection level |
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

A gang block is a 512-byte self-checksumming structure (`zio_gbh_phys_t`) containing up to three block pointers plus a block tail with checksum:

```
Gang Block (512 bytes)
+─────────────────────────────────────────────+
| blkptr_t  zg_blkptr[SPA_GBH_NBLKPTRS]      |   3 x 128 bytes = 384 bytes
+─────────────────────────────────────────────+
| uint64_t  zg_filler[]                       |   padding for alignment
+─────────────────────────────────────────────+
| zio_block_tail_t  zg_tail                   |   checksum trailer
+─────────────────────────────────────────────+
```

The block tail (`zio_block_tail_t`) consists of:

- **`zbt_magic`**: Magic number `0x210da7ab10c7a11` ("zio-data-bloc-tail")
- **`zbt_cksum`**: A `zio_cksum_t` (four `uint64_t` words) containing the SHA-256 checksum of the gang block

## 2.4 Checksum

ZFS checksums all data and metadata by default. The checksum algorithm is identified by the 8-bit `cksum` field in the block pointer. See the [glossary](glossary.md#checksum-algorithms) for the full table of algorithms and their numeric values.

The 256-bit checksum is stored across four 64-bit words: `checksum[0]` through `checksum[3]` (words c-f of the block pointer).

If checksumming is disabled (`cksum` = 2, "off"), all four checksum words are zero.

The checksum is always computed over the data the block pointer references. Gang blocks and ZIL blocks are self-checksumming: their checksums are stored in a block tail embedded within the block itself, not in the parent block pointer.

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

All three are stored as the number of 512-byte sectors **minus one**. To get the byte size: `(stored_value + 1) * 512`.

When compression is off and no RAID-Z or gang overhead applies, LSIZE = PSIZE = ASIZE.

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

The 7-bit `lvl` field indicates the number of levels of indirection between this block pointer and the actual data. Level 0 block pointers point directly to data. Level 1 block pointers point to indirect blocks containing level 0 block pointers, and so on. See [Chapter 3](03-dmu.md) for a detailed description of indirection.

## 2.10 Fill Count

The `fill` field (word b) counts the number of non-zero block pointers beneath this block pointer.

- For a level-0 data block pointer, the fill count is 1.
- For indirect blocks, it is the sum of fill counts of all block pointers within.
- For block pointers of type `DMU_OT_DNODE`, the fill count instead represents the number of allocated dnodes beneath this block pointer.

## 2.11 Birth Transaction

- **Physical birth txg** (word 9): The transaction group in which `dva[0]` was written to disk. Zero if same as logical birth txg.
- **Logical birth txg** (word a): The transaction group in which this block was logically created.

The birth transaction is used for incremental operations such as `zfs send` -- only blocks born after a given txg need to be included.

## 2.12 Padding

Words 7-8 contain padding reserved for future use. Bit 63 of word 7 is the **R** (rewrite) flag indicating the block was reallocated at the physical birth txg.
