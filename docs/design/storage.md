# Storage

Status: draft      Implements: F1 to F16

## Purpose
Storage covers everything between a file call and a disk sector: the VFS, the pseudo-filesystems, our own filesystem yartfs with its host tools, the block layer and GPT. Disk controllers are in drivers.md; the page cache is in memory.md.

## Decisions applied
- F1, Q1, Q8: all yartfs format logic is in `fs/yartfs/`, built for kernel, userland and host. The kernel driver and the tools are thin shells.
- F3: no crash safety in v1; a clean flag and fsck are the whole answer.
- M8, findings R4: file data is cached in the vnode's `MemoryObject` from phase 2.

## Design

### yartfs v1 layout
Blocks are 4 KiB (P9), so one block is one cache page. Fields are little-endian (P2). The volume is one flat run:

```
superblock | block bitmap | inode bitmap | inode table | data ... | superblock copy
```

No block groups: they buy locality on spinning disks and cost layout arithmetic in the kernel, mkfs and fsck (principle 4). The price is an inode count fixed at mkfs.

```c
typedef struct YartfsSuperblock {        /* block 0 */
    uint8_t  magic[8];                   /* "YARTFS\0\0" */
    uint32_t version, blockSize;         /* 1, 4096 */
    uint64_t requiredFeatures, readOnlyFeatures;
    uint64_t blockCount, inodeCount, freeBlocks, freeInodes;
    uint64_t blockBitmapStart, inodeBitmapStart, inodeTableStart, dataStart;
    uint64_t rootInode;
    uint32_t state, reserved;
    uint8_t  uuid[16];
} YartfsSuperblock;

typedef struct YartfsExtent {
    uint64_t diskBlock;
    uint32_t fileBlock, count;
} YartfsExtent;

typedef struct YartfsInode {
    uint16_t mode, flags;
    uint32_t linkCount, uid, gid;
    uint64_t size, blockCount;
    int64_t  accessNs, modifyNs, changeNs, createNs;
    uint32_t extentCount, generation;
    uint64_t extentChain;
    YartfsExtent extents[10];
    uint8_t  reserved[16];
} YartfsInode;

typedef struct YartfsDirEntry {
    uint64_t inode;
    uint16_t recordLength;
    uint8_t  nameLength, type;
    char     name[];
} YartfsDirEntry;
```

- **Extents, not block pointers.** An extent list is one sorted array: lookup is a search, freeing is a loop, and fsck only checks for overlap. Indirect blocks need three levels of tree code for the same job. Ten extents sit in the 256-byte inode; more go into a linked chain of extent blocks.
- **Directories** are files of variable-length entries, searched linearly. Names compare as raw bytes (F9, A8).
- **Symbolic links** keep their target as file data. Device nodes exist only in devfs.
- **Free space** is two bitmaps, allocated first-fit.
- **Limits:** 255-byte names (A9), 16 TiB files (32-bit `fileBlock`).

Left out of v1 for v2: checksums, a journal or copy-on-write, holes, extended attributes, directory indexes. Reserved bytes are written as zero and ignored on read, and every region is found through a superblock pointer, so v2 can add one. An unknown required feature bit refuses the mount; an unknown read-only bit forces read-only. v2 sets `version` to 2, and its driver still mounts v1.

Mounting read-write sets `state` dirty; a clean unmount clears it.

### Shared library and tools
```c
typedef struct YartfsBlockOps {
    int  (*get)(void *context, uint64_t block, void **data);
    void (*put)(void *context, void *data, bool dirty);
} YartfsBlockOps;

int yartfsOpen(YartfsVolume *volume, const YartfsBlockOps *ops, void *context, uint32_t flags);
int yartfsMapBlock(YartfsVolume *volume, const YartfsInode *inode, uint64_t fileBlock,
                   uint64_t *diskBlock, uint32_t *runLength);
```
Format, check, resize and the directory calls follow the same shape. The library does no I/O itself: in the kernel `get` pins a page of the block device's cache object; on the host it reads a file. It bounds-checks everything read from disk.

Host tools in `tools/`, since nothing else reads the format:
- `yartfs-mkfs` formats a file, or a byte range inside one.
- `yartfs-fsck` checks bitmaps against reachable blocks, extent overlap, link counts and directory structure. `--repair` rebuilds bitmaps and frees orphans.
- `yartfs-dump` lists and extracts.
- `yart-image` (F13) writes the protective MBR, GPT and partitions, then fills a yartfs root from a staging directory, with no mounts and no root. A manifest gives owner, group and mode, because the host user cannot chown. The EFI partition comes from the host FAT writer (findings R3).

### VFS (F6)
```c
typedef struct Vnode {
    const VnodeOps *ops;
    Mount          *mount, *mountedHere;
    uint64_t        number;
    uint32_t        type, refCount;
    Mutex           lock;
    MemoryObject   *object;
    List            watches;
    void           *fsData;
} Vnode;

typedef struct VnodeOps {
    int (*lookup)(Vnode *dir, const Name *name, Vnode **out);
    int (*create)(Vnode *dir, const Name *name, const VnodeAttr *attr, Vnode **out);
    int (*link)(Vnode *dir, const Name *name, Vnode *target);
    int (*remove)(Vnode *dir, const Name *name);
    int (*rename)(Vnode *from, const Name *oldName, Vnode *to, const Name *newName);
    /* also readDir, readLink, getAttr, setAttr, resize, sync, release */
} VnodeOps;

int vfsWalk(Vnode *start, const char *path, uint32_t flags, Vnode **out);
```
- File data moves through the `MemoryObject`; the filesystem supplies `readPage` and `writePage`. Device and `/proc` nodes carry their own ops.
- `vfsWalk` is one loop over components. It crosses mounts through `mountedHere`, follows at most 8 symbolic links (F11), and checks search permission at each step.
- The mount table lists each mount's type, root, covered vnode and device.
- The name cache maps (directory vnode, name) to a vnode or to "absent", and is bounded. Create, remove, rename and unmount invalidate entries.
- Permissions (F10): one function checks owner, group and mode. User 0 passes (X3). Set-user and set-group bits are never honoured (X4).
- `Name` is a pointer and a length. Hard links never name directories or cross mounts.
- Times (F14) are signed nanoseconds since 1970 UTC.

### Pseudo-filesystems (F7) and layout (F8)
- **tmpfs**: data lives only in the `MemoryObject`. Holds the unpacked boot archive (B8), then `/tmp`.
- **devfs**: drivers call `devfsAdd(name, ops, context, mode)`: `null`, `zero`, `random`, `klog`, `console`, `disk0`, `disk0p2`.
- **/proc** (S14): the files in processes.md plus `mounts` and `memory`.

The kernel starts on the tmpfs archive and mounts the `root=` device at `/`. The image builder creates the F8 directories; `/dev`, `/proc` and `/tmp` are mount points.

### Block layer (F12) and GPT (F5)
```c
typedef struct BlockRequest {
    uint32_t operation, pageCount;       /* read, write, flush */
    uint64_t sector;
    Page   **pages;
    void   (*done)(struct BlockRequest *request, int status);
    void    *context;
} BlockRequest;

void blockSubmit(BlockDevice *device, BlockRequest *request);
int  blockSubmitWait(BlockDevice *device, BlockRequest *request);
```
A device has a `submit` op, a size and a first-in first-out queue. `done` runs in the driver's deferred work (K4). `blockSubmitWait` sleeps until then. A partition is a child device that adds an offset.

The GPT reader in `lib/` checks both CRCs, falls back to the backup header, and reports entries; the kernel adds one child device each. yartfs gets its own partition type GUID.

### Later parts
- **Notification (F15):** `fileWatch(dirFd, path, mask)` returns a descriptor (K12) yielding change records for a node or a directory's direct children. One VFS hook feeds `Vnode.watches`.
- **FAT (F4):** `fs/fat/` is a shared library like yartfs. The phase 5 host writer and the phase 6 driver share it. Owner and mode come from mount options.
- **Removable media (F16):** `mount` by hand. In phase 6 a service watches `/dev` (F15) and mounts through the elevation service (X4).

## Per-architecture notes
Nothing is per-port except the image (F13): x86_64 has an EFI partition and a yartfs root, riscv64 a yartfs root only.

## Build order
- Phase 2: block layer with a RAM disk, GPT, VFS and name cache, tmpfs, devfs, the yartfs library with mkfs, fsck and fuzzing, `yart-image`, the kernel yartfs driver, root mount.
- Phase 3: `/proc`, file mappings, `mount` and `df`, fsck built for yart.
- Phase 5: notification, host FAT writer.
- Phase 6: FAT driver, yartfs v2, swap, automatic mounting.

## Test plan
- Host: library tests on a memory-backed volume. mkfs then fsck is clean. Image builder then dump round-trips a tree. fsck reports each seeded corruption. Fuzz open, walk and check, plus GPT (Q8). A committed v1 image must mount forever.
- In-kernel: block requests on a RAM disk, partition offsets, path walk, name cache.
- Userland: one suite on tmpfs and yartfs: create, resize, rename, links, link loops, permissions, times, full disk.
- Boot: both ports run a program from a yartfs root. A test writes files and powers off; host fsck and dump must then find a clean image holding them.

## Risks and open questions
- F2: journal or copy-on-write is not chosen, nor whether v1 volumes convert in place.
- F3, F1, G4: what mounting a dirty volume does is undecided. Installed machines have no host, so fsck and mkfs must also build for yart.
- B9, F5: the `root=` syntax is unsettled.
- F8: no directory is named for mount points, the EFI partition, root's home or app directories (W26).
- F9, F4, F6: FAT is case-insensitive, so the name cache must let a filesystem supply its own comparison. Large USB sticks often ship as exFAT, which no decision covers.
- F10, X4: whether set-user and set-group bits are stored or rejected is open.
- F14: whether access times are written on read is undecided.
- F10, F11, F15, U8: syscalls-and-libraries.md lists no rename, link, attribute, resize, sync, mount or watch calls.
- M9, F5: swap partition or swap file is undecided.
