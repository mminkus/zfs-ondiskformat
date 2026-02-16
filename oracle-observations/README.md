# Oracle Observations Evidence Layout

This directory stores raw evidence bundles referenced by
`docs/oracle-zfs-lineage.md`.

## Naming

- Use `E-###` IDs (`E-000`, `E-001`, ...).
- Keep one experiment family per evidence ID.

## Expected Bundle Shape

```text
oracle-observations/
  E-001/
    env.md
    command-log.txt
    before-*.txt
    after-*.txt
    notes.md
```

Not every bundle needs every file, but `notes.md` should always summarize:

- what was tested
- exact commands
- key observations
- confidence level

## Formatting Rules

- Keep full command output in `.txt` files.
- In docs, link to raw files and include only short excerpts.
- Prefer append-only updates to preserve chronology.

## Safety / Licensing

- Do not commit Oracle binaries.
- If licensing/publication status of raw output is unclear, keep full artifacts
  private and publish only short, necessary excerpts in docs.
