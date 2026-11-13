# Detail held for specs

Design detail the agents worked out but cut from the notes to stay within length. Fold each block into the spec that builds it, then delete it here.

## yartfs v1 (phase 2)

- Extent block: `uint32_t magic, count; uint64_t next; uint8_t reserved[16]; YartfsExtent extents[253]`.
- Superblock also carries `label[32]`, `createdNs`, `mountedNs`. State constants `YARTFS_CLEAN`, `YARTFS_DIRTY`. Zero-padded to 4096 bytes. Root inode is 1.
- Inodes: 16 per block, one per 16 KiB by default.
- Directory entries: padded to 8 bytes, never cross a block, inode 0 marks a free slot, `.` and `..` are stored.
- Allocation: first-fit from a moving hint, taking the longest free run as one extent. A write past the end allocates zeroed blocks.
- Limits: 2^64 blocks, 2^32 links.
- Library entry points: `yartfsFormat`, `yartfsInodeRead`, `yartfsResize`, `yartfsDirLookup`, `yartfsDirAdd`, `yartfsCheck`. The library follows T11 and K7.
- Tools: fsck also checks the superblock; `--repair` rebuilds counts. The image manifest uses the U13 syntax and output is byte-for-byte reproducible.

## VFS and block layer (phase 2)

- `vfsRegister`, `vfsMount`, `vfsAccess`; remaining `VnodeOps`: `readDir`, `readLink`, `getAttr`, `setAttr`, `resize`, `sync`, `release`.
- The name cache evicts oldest first. A removed file lives until its last descriptor closes.
- devfs has `input/event0`. `/proc` files are U13 text built at open.
- `blockAdd`; sectors in 512-byte units; operations `BLOCK_READ`, `BLOCK_WRITE`, `BLOCK_FLUSH`; no I/O scheduler until a benchmark asks (principle 10); `done` never runs in the interrupt handler.
- Tests: remove-while-open in the userland suite.

## Later storage

- Watch records: create, remove, rename, modify, attribute, overflow, fed by a `vfsNotify` hook (phase 5).
- FAT covers FAT12, 16 and 32 with long names; fuzz it (phase 6).

## Drivers

- Helpers: `regRead32`, `regWrite32`, `driverUnregister`, `virtQueueKick`, `blockAdd`, `inputAdd`, `netAdd`, and the virtio transport's `getStatus` and `getFeatures`.
- ACPI: the static parser also reads the root pointer and the MADT interrupt overrides. The phase 6 interpreter sits behind our own layer for memory, locks and register access.
- AHCI: 48-bit reads and writes plus cache flush, one command per port to start. NVMe: one queue pair.
- PS/2 mouse: three-byte packets with the wheel extension.
- Reset per port through `ArchResetOps`: UEFI runtime on x86_64, our firmware on RISC-V, PSCI on AArch64.
