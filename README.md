# ZFS On-Disk Format

An independently written description of the ZFS on-disk format for modern [OpenZFS](https://github.com/openzfs/zfs).

> This document is derived from the ZFS On-Disk Specification (2006),
> Copyright 1994–2006 Sun Microsystems, Inc.,
> licensed under the Berkeley License (see [LICENSE-SUN.md](LICENSE-SUN.md)).

The original specification is maintained by Matthew Ahrens at [github.com/ahrens/zfsondisk](https://github.com/ahrens/zfsondisk/). This project extends it with all changes made by OpenZFS since 2006, written from the publicly available OpenZFS source code.

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
- [Chapter 9: Space Maps and Metaslabs](docs/09-space-maps.md)
- [Chapter 10: Native Encryption](docs/10-encryption.md)
- [Chapter 11: RAID-Z and dRAID](docs/11-raidz.md)

## Status

Phase 1 (initial documentation of the on-disk format) is complete. Phase 2 (modernization with post-2006 OpenZFS features: encryption, dRAID, block cloning, feature flags, etc.) is in progress.

## Resources

- [OpenZFS](https://openzfs.org/wiki/Main_Page)
- [OpenZFS Developer Resources](https://openzfs.org/wiki/Developer_resources)
- [ZFS On-Disk Specification (original)](https://github.com/ahrens/zfsondisk/) -- Matthew Ahrens
- [ZFS On-Disk Specification (PDF)](https://www.giis.co.in/Zfs_ondiskformat.pdf)
- [Examining ZFS On-Disk Format Using mdb and zdb](https://www.youtube.com/watch?v=BIxVSqUELNc) -- Max Bruning
- [The Zettabyte File System](https://www.cs.hmc.edu/~rhodes/cs134/readings/The%20Zettabyte%20File%20System.pdf)
- [ZFS: The Last Word in Filesystems](https://www.slideshare.net/slideshow/zfs-the-last-word-in-filesystems/6893781)
- [The ZFS File System (CS308 Slides)](https://cs.uml.edu/~bill/cs308/ZFS_File_Sys.pdf)

## License

This project is dual-licensed:

- New content: [MIT License](LICENSE)
- Derived from: [Sun Microsystems Berkeley License](LICENSE-SUN.md)
