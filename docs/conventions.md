# Document Conventions

This document describes the on-disk format of ZFS as implemented in OpenZFS. It is an independently written description derived from the publicly available OpenZFS source code.

## Terminology

- **On-disk format**: The byte-level layout of data structures as stored on physical media. This is distinct from the in-memory (C struct) representation, which may differ due to padding, alignment, or platform-specific layout.
- **Structure names**: We use OpenZFS type names throughout (e.g., `blkptr_t`, `dva_t`, `uberblock_t`, `dnode_phys_t`). The `_phys_t` suffix typically denotes an on-disk structure.
- **Object numbers**: 64-bit integers identifying objects within an object set. Object 0 is reserved for the metadnode; object 1 is typically the first user-visible object.

## Byte Order (Endianness)

ZFS uses an **adaptive-endian** format. Data is written to disk in the native byte order of the machine that writes it. The byte order is recorded in each block pointer's `B` (byteorder) bit:

| Value | Byte Order |
|-------|------------|
| 0 | Big Endian |
| 1 | Little Endian |

When a pool is imported on a machine with a different byte order, ZFS byte-swaps data on read. Multi-byte integer fields in ZAP leaf arrays are always stored big-endian regardless of the machine's native order.

## Size Units

Unless otherwise noted:

- **LSIZE** and **PSIZE** in block pointers are stored as the number of 512-byte sectors minus one. To get the byte size: `(stored_value + 1) << 9`.  
  **ASIZE** is stored as the number of 512-byte sectors with no bias. To get the byte size: `stored_value << 9`.
- **Offsets** in DVAs are stored in 512-byte sector units. The physical byte address on a vdev is: `(offset << 9) + 0x400000` (where 0x400000 = 4 MB accounts for the two front labels and boot block).
- **Block sizes** range from 512 bytes (2^9) to 16 MB (2^24), with 128 KB (2^17) being the traditional maximum record size.

## Diagrams

### Byte Layout Diagrams

On-disk structure layouts use ASCII art with byte offsets and field sizes:

```
Offset  +----------+----------+----------+----------+
  0x00  | field_a  | field_b  |       field_c        |
        | (1 byte) | (1 byte) |       (2 bytes)      |
        +----------+----------+----------+----------+
  0x04  |                  field_d                    |
        |                 (4 bytes)                   |
        +----------+----------+----------+----------+
```

### Structural Diagrams

Relationships between on-disk structures use Mermaid diagrams for rendering on GitHub.

## Source References

Each chapter references the relevant OpenZFS source files. References use the format:

> **Source:** `include/sys/uberblock_impl.h`, `module/zfs/uberblock.c`

All paths are relative to the OpenZFS source tree root.

## On-Disk vs. C Layout

The C struct definitions in OpenZFS headers are provided for reference, but the authoritative format is the on-disk byte layout. In most cases they match, but be aware that:

- C compilers may insert padding between struct members.
- Bitfield layout is compiler-dependent and should not be relied upon for on-disk format. OpenZFS uses explicit bit manipulation macros (e.g., `BF64_GET`, `BF64_SET`) for encoding/decoding packed fields in block pointers.
- Conditional fields (e.g., encryption parameters that replace the 3rd DVA) change the interpretation of the same byte range.
