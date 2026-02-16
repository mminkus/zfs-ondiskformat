# Appendix: Oracle ZFS Lineage Notes

> **Status:** Working notes / evidence log
> **Scope:** Historical and structural comparison only (not an interoperability target)

This appendix documents observed Oracle ZFS behavior and likely on-disk
evolution points, compared with modern OpenZFS.

## A.1 Project Position

- This repository uses **OpenZFS source** as the canonical reference for on-disk format documentation.
- We are **not** targeting compatibility or interoperability with Oracle ZFS.
- Oracle ZFS is still valuable to study as a separate lineage, especially where
  it diverged earlier (for example, encryption-related block-pointer evolution).
- Goal: identify ideas, tradeoffs, and on-disk patterns that may be useful for
  understanding modern ZFS design choices.

## A.2 What We Record Here

- Direct observations from Oracle systems (`zpool`, `zfs`, `zdb`, controlled experiments).
- Version and feature behavior visible from tooling output.
- Hypotheses about affected on-disk structures.
- Evidence-backed confirmations and contradictions.

Priority is always **observed data first**, then inference.

## A.2.1 Output Formatting Convention

Long command output should be stored in evidence files, not pasted inline in
full, to keep this appendix readable.

Use this pattern:

- Put raw output in `oracle-observations/E-###/*.txt`
- In this appendix, keep:
  - a short summary
  - key deltas/findings
  - links to the raw files

Optional:

- Use a short inline excerpt when a specific line is important.

Conventions for `oracle-observations/` are documented in:

- `oracle-observations/README.md`

## A.3 Environment Snapshot (Template)

Fill this section for each test host used in analysis.

| Field | Value |
|------|-------|
| OS | `SunOS solaris 5.11 11.4.81.193.1 i86pc i386 i86pc kvm` |
| Hostname | `solaris` |
| `zpool` version output date | `2026-02-16` |
| `zdb` build/version notes | `<fill>` |
| Pool used for tests | `<fill>` |
| Test media type | `<file-backed vdev / disk / VM disk>` |

## A.4 Captured Baseline Output

Evidence bundle directory for this baseline:

- `oracle-observations/E-000/`

Raw captures:

- `oracle-observations/E-000/zfs-upgrade-v.txt`
- `oracle-observations/E-000/zpool-upgrade-v.txt`
- `oracle-observations/E-000/zdb-help.txt`
- `oracle-observations/E-000/zdb-rpool.txt`
- `oracle-observations/E-000/zdb-uberblock-rpool.txt`
- `oracle-observations/E-000/zdb-M-all-rpool.txt`
- `oracle-observations/E-000/zpool-get-all-rpool.txt`
- `oracle-observations/E-000/zfs-get-all-rpool.txt`

Quick highlights:

- Oracle filesystem version line reports up to `zfs` version `8`.
- Oracle pool version line reports up to `zpool` version `53`.
- `zdb -M all` output shows Oracle-specific MOS key names and reporting
  structure that differ from current OpenZFS tooling conventions.

### A.4.1 Inline Excerpt (`zpool upgrade -v`)

```text
This system is currently running ZFS pool version 53.
...
44  Device removal
45  Lazy deadlists
46  Compact file metadata for encryption
47  Property support for ZVOLs
48  File Retention
49  Unicode versioning
50  Raw crypto replication
51  Retention onexpiry
52  Clonedir
53  Maximize space
```

## A.5 Observed Pool Version Track (Oracle)

Seed table from currently observed output.
Add evidence references as each entry is validated by experiment.

| Pool VER | Oracle Description (Observed) | Suspected On-Disk Area | Confidence | Evidence Ref |
|---------|--------------------------------|-------------------------|------------|--------------|
| 30 | Encryption | `blkptr_t` field repurposing, key metadata | medium | `E-000`, `E-001` |
| 37 | LZ4 compression | compression enum / property semantics | low | `E-000` |
| 44 | Device removal | indirect mapping / removal state records | low | `E-000` |
| 45 | Lazy deadlists | deadlist lifecycle/accounting behavior | low | `E-000` |
| 46 | Compact file metadata for encryption | encrypted metadata placement and density | low | `E-000` |
| 50 | Raw crypto replication | send/recv stream + key handling interactions | low | `E-000` |
| 53 | Maximize space | allocator / block packing behavior (unknown) | low | `E-000` |

> Note: this table is intentionally conservative until each row has a concrete before/after capture set.

## A.6 Hypothesis Log

Track assumptions explicitly and force them through reproducible tests.

| ID | Hypothesis | Predicted Structure Change | Test Plan | Status | Evidence Ref |
|----|------------|----------------------------|-----------|--------|--------------|
| H-001 | Oracle encryption reused different `blkptr_t` bits than OpenZFS | block pointer field interpretation differs under encrypted datasets | Create encrypted/non-encrypted pools, diff `zdb` block dumps | partial | `E-001` |
| H-002 | Oracle device removal persistence differs from OpenZFS `spa_removing_phys_t` + indirect mapping model | different MOS keys and/or removal accounting semantics | Perform controlled remove operation, capture MOS object/key transitions | open | `E-002` |

## A.7 Evidence Capture Workflow

For each tested operation:

1. Create a minimal, disposable test pool.
2. Capture **baseline** (`zdb`/`zpool`/`zfs`) before operation.
3. Apply exactly one operation.
4. Capture **post-change** output.
5. Diff and annotate only fields/objects that changed.
6. Record confidence and competing explanations.

Suggested evidence bundle layout:

```text
oracle-observations/
  E-001/
    env.txt
    before-zdb.txt
    after-zdb.txt
    command-log.txt
    notes.md
```

## A.8 Safety and Legal Notes

- Use disposable test pools only.
- Do not mutate production pools for reverse-engineering.
- Respect Oracle licensing and platform terms.
- Do not rely on undocumented behavior for operational decisions.
- Do not publish Oracle binaries or large proprietary-tooling dumps beyond
  short excerpts; keep full raw artifacts private when licensing is unclear.

## A.9 Compatibility Statement

This appendix is informational and comparative.
It does **not** define a compatibility target for this project.

OpenZFS remains the implementation reference for all normative on-disk format
documentation in this repository.

## A.10 Open Questions

- Which Oracle pool versions introduced on-disk encryption metadata changes?
- How do Oracle checkpoint/removal/accounting records map (or not) to OpenZFS concepts?
- Which differences are semantic only vs physically incompatible?

## A.11 Oracle Encrypted `blkptr_t` (Provisional, v30-era)

This model is a **forensic deduction** based on:

- Oracle public write-up ([ZFS Encryption: What Is On Disk](https://blogs.oracle.com/solaris/zfs-encryption-what-is-on-disk))
- observed mdb/zdb examples
- local decoder experiments in patched Linux `zdb` tooling (`E-001`)

It should be treated as "best current model", not a normative ABI.
It is currently most defensible for the v30 encryption lineage and may diverge
in later Oracle pool versions.
When `xbit=1`, `DVA[2]` must not be interpreted as a normal data DVA.

### A.11.1 Deduced Layout

```
Word   64      56      48      40      32      24      16      8       0
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0    |                      DVA[0] word0                              |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  1    |                      DVA[0] word1                              |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  2    |                      DVA[1] word0                              |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  3    |                      DVA[1] word1                              |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  4    |      IV[95:64] (32 bits)       |    reserved/unknown (32)      |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  5    |                      IV[63:0]                                   |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  6    |                      blk_prop (X bit set)                       |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  7    |                         padding                                 |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  8    |                         padding                                 |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  9    |                    physical birth txg                           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  a    |                    logical birth txg                            |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  b    |                      fill count                                 |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  c    |                   SHA256-trunc (part 1)                         |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  d    |                   SHA256-trunc (part 2)                         |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  e    | SHA256-trunc tail (32) |            MAC[95:64] (32)            |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  f    |                        MAC[63:0]                               |
       +-------+-------+-------+-------+-------+-------+-------+-------+
```

### A.11.2 Key Differences vs OpenZFS Native Encryption

- Oracle model uses `DVA[2]` as IV carrier (`96-bit IV`) rather than OpenZFS
  salt+IV split design.
- Candidate checksum+MAC split is `SHA256-trunc160 + MAC96` (vs OpenZFS
  checksum128 + MAC128 for encrypted/authenticated BP paths).
- Practical consequence remains the same: encrypted blocks are effectively
  limited to two physical DVAs.

### A.11.3 Field Extraction Candidate

Given raw words for `DVA[2]` and checksum words:

- `iv96 ~= high32(word4) || word5`
- `mac96 ~= low32(word14) || word15`
- `sha256_trunc160 ~= word12 || word13 || high32(word14)`

All extraction formulas are defined in word order as stored in `blkptr_t`.
Byte-sliced renderings are diagnostic only and may vary by dump tooling/endian
presentation.

### A.11.4 Observed Invariants (Current)

- When Oracle decode mode triggers with `xbit=1`, only `DVA[0..1]` are treated
  as data copies (`data_copies(max2)=2`).
- Checksum algorithm IDs may appear as `0` while checksum words remain
  populated, consistent with checksum-field repurposing in this lineage.

## A.12 Decoder Support Status

Read-only forensic decoder support for Oracle encrypted-block candidates is
implemented in local tooling (outside this docs repo), including:

- Oracle layout banner and extraction heuristics
- IV/MAC candidate extraction
- plausibility scoring and fingerprints
- comparison with normal OpenZFS decode path

Patchset references:

- `96a649f85db001442b8a955ef382de6e8810681c`
- `ca1312c86c111b3d0ce5d68c32d74073f9027367`
- fork branch: [mminkus/zfs `oracle-zfs`](https://github.com/mminkus/zfs/tree/oracle-zfs)
- evidence note: `oracle-observations/E-001/notes.md`

Scope note:

- This is for **forensic decode and structure analysis only**.
- It does not imply full interoperability or authoritative Oracle crypto
  semantics.

---

### Evidence IDs

- `E-000`: initial observed `zfs upgrade -v` / `zpool upgrade -v` output snapshot
- `E-001`: Oracle encrypted `blkptr_t` forensic decode experiments (`dumpbp.py --oracle-layout`)
