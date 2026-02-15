# ZFS On-Disk Format

An independently written description of the ZFS on-disk format for modern [OpenZFS](https://github.com/openzfs/zfs).

This documentation is derived from the publicly available OpenZFS source code and publicly available materials. It is not a reproduction of any proprietary documentation.

## Documentation

The documentation is organized into chapters covering each layer of the ZFS on-disk format:

- [Conventions](docs/conventions.md) -- Notation, byte order, diagram style
- [Glossary](docs/glossary.md) -- Terms, type enumerations, constants
- [Introduction](docs/index.md) -- Architecture overview and data traversal
- [Chapter 1: Virtual Devices, Labels, and Boot Block](docs/01-vdevs.md)
- [Chapter 2: Block Pointers and Indirect Blocks](docs/02-block-pointers.md)
- [Chapter 3: Data Management Unit](docs/03-dmu.md)
- [Chapter 4: Dataset and Snapshot Layer](docs/04-dsl.md)
- [Chapter 5: ZFS Attribute Processor](docs/05-zap.md)
- [Chapter 6: ZFS POSIX Layer](docs/06-zpl.md)
- [Chapter 7: ZFS Intent Log](docs/07-zil.md)
- [Chapter 8: ZVOL](docs/08-zvol.md)

## Status

Phase 1 (initial documentation of the on-disk format) is complete. Phase 2 (modernization with post-2006 OpenZFS features: encryption, dRAID, block cloning, feature flags, etc.) is planned.

## License

See [LICENSE](LICENSE).
