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
| 30 | Encryption | `blkptr_t` field repurposing, key metadata | low | `E-000` |
| 37 | LZ4 compression | compression enum / property semantics | low | `E-000` |
| 44 | Device removal | indirect mapping / removal state records | low | `E-000` |

> Note: this table is intentionally conservative until each row has a concrete before/after capture set.

## A.6 Hypothesis Log

Track assumptions explicitly and force them through reproducible tests.

| ID | Hypothesis | Predicted Structure Change | Test Plan | Status | Evidence Ref |
|----|------------|----------------------------|-----------|--------|--------------|
| H-001 | Oracle encryption reused different `blkptr_t` bits than OpenZFS | block pointer field interpretation differs under encrypted datasets | Create encrypted/non-encrypted pools, diff `zdb` block dumps | open | `E-001` |
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

## A.9 Compatibility Statement

This appendix is informational and comparative.
It does **not** define a compatibility target for this project.

OpenZFS remains the implementation reference for all normative on-disk format
documentation in this repository.

## A.10 Open Questions

- Which Oracle pool versions introduced on-disk encryption metadata changes?
- How do Oracle checkpoint/removal/accounting records map (or not) to OpenZFS concepts?
- Which differences are semantic only vs physically incompatible?

---

### Evidence IDs

- `E-000`: initial observed `zfs upgrade -v` / `zpool upgrade -v` output snapshot
