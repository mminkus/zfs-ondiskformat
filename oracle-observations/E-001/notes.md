# E-001: Oracle Encrypted `blkptr_t` Forensic Decode

## Summary

This evidence bundle records the first consistent read-only decode heuristics
for Oracle encrypted block-pointer candidates (pool version <= 30 test focus)
using a patched Linux `zdb` toolchain and helper script.

Observed outcome: a consistent candidate mapping for IV/MAC extraction from
Oracle encrypted `blkptr_t` words.

## Tooling Context

- Repository: `/home/martin/development/zfs-explorer/zfs`
- Fork branch: [mminkus/zfs `oracle-zfs`](https://github.com/mminkus/zfs/tree/oracle-zfs)
- Commits:
  - `96a649f85db001442b8a955ef382de6e8810681c`
  - `ca1312c86c111b3d0ce5d68c32d74073f9027367`

## Command

```bash
python3 dumpbp.py --oracle-layout --oracle --limit 2 crypt_L1_0
```

## Key Decoder Output (excerpt)

```text
Oracle Encrypted blkptr_t (deduced, word-level)

  word 0-1 : DVA[0] data copy
  word 2-3 : DVA[1] data copy
  word 4-5 : DVA[2] repurposed for IV96
             iv96 ~= high32(word4) || word5
             word4 low32 appears reserved/unused in Oracle examples
             DVA[2] is not treated as a normal data DVA when xbit=1
  word 6   : blk_prop (X bit indicates encrypted/authenticated semantics)
  word 7-8 : pad
  word 9   : phys_birth
  word 10  : birth
  word 11  : fill
  word 12-15: checksum field repurposed:
              sha256_trunc160 ~= word12 || word13 || high32(word14)
              mac96           ~= low32(word14) || word15
```

## Current Confidence

- Provisional structural confidence: **medium**
- Crypto/semantic confidence: **low-to-medium** (needs more pools and known
  plaintext/ciphertext correlation)

## Observed Invariants

- Under Oracle decode mode with `xbit=1`, only `DVA[0..1]` are treated as data
  copies (`data_copies(max2)=2`).
- Checksum algorithm IDs can appear as `0` while checksum words remain
  populated, consistent with checksum field repurposing.

## Follow-up

- Capture larger sample set across multiple datasets / txgs.
- Compare known plaintext writes against decoded block candidates.
- Validate whether mapping remains stable for post-v30 pools.
