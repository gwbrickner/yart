# Storage

Status: draft      Implements: F1 to F16

## Purpose
Storage covers everything between a file call and a disk sector: the VFS, the pseudo-filesystems, our own filesystem yartfs with its host tools, the block layer and GPT. It lives in `kernel/fs/`, `fs/yartfs/`, `lib/` and `tools/`. Disk controllers are in drivers.md; the page cache is in memory.md.

## Decisions applied
- F1, Q1, Q8: every byte of yartfs format logic is in `fs/yartfs/`, built for kernel, userland and host. The kernel driver and the tools are thin shells around it.
- F2: v1 carries a version and feature masks. v2 changes are gated by them, never guessed.
- F3: v1 has no crash safety. A clean flag and fsck are the whole answer.
- F12: drivers see only asynchronous requests. Filesystems use the blocking wrapper.
- F13: the image builder writes every byte itself: no loop devices, no mounts, no root.
- M8, findings R4: file data is cached in the vnode's `MemoryObject` from phase 2.
- P2: disk fields are little-endian, read through `lib/` byte-order helpers.

## Design

### yartfs v1 layout
Blocks are 4 KiB (P9), so one block is one cache page. The volume is one flat run:

```
block 0: superblock | block bitmap | inode bitmap | inode table | data ... | last block: superblock copy
```

No block groups. They buy locality on spinning disks and cost layout arithmetic in the kernel, mkfs and fsck. Our volumes are small images and solid-state disks, and flat is what a new reader understands fastest (principle 4). The price is an inode count fixed at mkfs, one per 16 KiB by default.

```c
typedef struct YartfsSuperblock {        /* padded with zeros to 4096 bytes */
    uint8_t  magic[8];                   /* "YARTFS\0\0" */
    uint32_t version;                    /* 1 */
    uint32_t blockSize;                  /* 4096 */
    uint64_t requiredFeatures;           /* unknown bit: refuse to mount */
    uint64_t readOnlyFeatures;           /* unknown bit: mount read-only */
    uint64_t blockCount, inodeCount;
    uint64_t freeBlocks, freeInodes;     /* advisory; the bitmaps are the truth */
    uint64_t blockBitmapStart, inodeBitmapStart, inodeTableStart, dataStart;
    uint64_t rootInode;                  /* 1 */
    uint32_t state;                      /* YARTFS_CLEAN or YARTFS_DIRTY */
    uint32_t reserved;
    uint8_t  uuid[16];
    char     label[32];
    int64_t  createdNs, mountedNs;
} YartfsSuperblock;

typedef struct YartfsExtent {            /* 16 bytes */
    uint64_t diskBlock;
    uint32_t fileBlock, count;
} YartfsExtent;

typedef struct YartfsInode {             /* 256 bytes, 16 per block */
    uint16_t mode;                       /* type and permission bits (F10) */
    uint16_t flags;
    uint32_t linkCount;
    uint32_t uid, gid;
    uint64_t size, blockCount;
    int64_t  accessNs, modifyNs, changeNs, createNs;   /* F14 */
    uint32_t extentCount, generation;
    uint64_t extentChain;                /* first extent block, or 0 */
    YartfsExtent extents[10];
    uint8_t  reserved[16];
} YartfsInode;

typedef struct YartfsExtentBlock {
    uint32_t magic, count;
    uint64_t next;
    uint8_t  reserved[16];
    YartfsExtent extents[253];
} YartfsExtentBlock;

typedef struct YartfsDirEntry {          /* padded to 8 bytes; never crosses a block */
    uint64_t inode;                      /* 0: free slot */
    uint16_t recordLength;
    uint8_t  nameLength, type;
    char     name[];
} YartfsDirEntry;
```

- **Extents, not block pointers.** An extent list is one sorted array: lookup is a search, freeing is a loop, and fsck only checks that ranges do not overlap. Indirect blocks need three levels of tree code for the same job. Ten extents sit in the inode; more go into a singly linked chain of extent blocks. No tree. v1 has no holes: extents cover the file from block 0, and a write past the end allocates zeroed blocks.
- **Directories** are files of variable-length entries, searched linearly, with `.` and `..` stored. Names are compared as raw bytes (F9, A8). The name cache hides the linear search.
- **Symbolic links** keep their target as file data. Device nodes exist only in devfs.
- **Free space** is two bitmaps, allocated first-fit from a moving hint. The longest free run found is taken as a single extent.
- **Limits:** 255-byte names (A9), 16 TiB files (32-bit `fileBlock`), 2^64 blocks, 2^32 links.

Left out of v1 so v2 can add it: checksums, a journal or copy-on-write, holes, extended attributes, directory indexes and online resize. Room is kept: reserved bytes in the superblock, inode and extent block are written as zero and ignored on read, and every region is found through a superblock pointer, so v2 can add one. v2 sets `version = 2` plus feature bits. A v1 driver refuses it; the v2 driver still mounts v1.

Mounting read-write sets `state` to dirty; a clean unmount or shutdown clears it.

### Shared library and tools
```c
typedef struct YartfsBlockOps {
    int  (*get)(void *context, uint64_t block, void **data);
    void (*put)(void *context, void *data, bool dirty);
} YartfsBlockOps;

int yartfsOpen(YartfsVolume *volume, const YartfsBlockOps *ops, void *context, uint32_t flags);
int yartfsFormat(const YartfsBlockOps *ops, void *context, const YartfsFormatArgs *args);
int yartfsInodeRead(YartfsVolume *volume, uint64_t number, YartfsInode *out);
int yartfsMapBlock(YartfsVolume *volume, const YartfsInode *inode, uint64_t fileBlock,
                   uint64_t *diskBlock, uint32_t *runLength);
int yartfsResize(YartfsVolume *volume, uint64_t number, uint64_t newSize);
int yartfsDirLookup(YartfsVolume *volume, uint64_t dir, const char *name, size_t length, uint64_t *out);
int yartfsDirAdd(YartfsVolume *volume, uint64_t dir, const char *name, size_t length, uint64_t inode, uint8_t type);
int yartfsCheck(YartfsVolume *volume, uint32_t flags, YartfsReport *report);
```
The library does no I/O of its own. In the kernel `get` pins a page of the block device's cache object; on the host it reads a file. It obeys the kernel rules: no recursion (T11), negative errors (K7), and nothing on disk is trusted before it is bounds-checked.

Host tools in `tools/`, all ours since nothing else reads the format:
- `yartfs-mkfs`: formats a file, or a byte range inside one.
- `yartfs-fsck`: runs `yartfsCheck`: superblock, bitmaps against reachable blocks, extent overlap, link counts, directory structure. `--repair` rebuilds bitmaps and counts and frees orphans.
- `yartfs-dump`: lists and extracts, for tests.
- `yart-image` (F13): writes the protective MBR, GPT and partitions, then fills a yartfs root from a staging directory. A manifest in U13 syntax gives owner, group and mode, because the host user cannot chown. Output is byte-for-byte reproducible. The EFI partition comes from the host FAT writer (findings R3).

### VFS (F6)
```c
typedef struct Vnode {
    const VnodeOps *ops;
    Mount          *mount, *mountedHere;
    uint64_t        number;
    uint32_t        type, refCount;
    Mutex           lock;
    MemoryObject   *object;              /* file pages (memory.md) */
    List            watches;             /* F15 */
    void           *fsData;
} Vnode;

typedef struct VnodeOps {
    int (*lookup)(Vnode *dir, const char *name, size_t length, Vnode **out);
    int (*create)(Vnode *dir, const char *name, size_t length, const VnodeAttr *attr, Vnode **out);
    int (*link)(Vnode *dir, const char *name, size_t length, Vnode *target);
    int (*remove)(Vnode *dir, const char *name, size_t length);
    int (*rename)(Vnode *from, const char *oldName, size_t oldLength, Vnode *to, const char *newName, size_t newLength);
    int (*readDir)(Vnode *dir, uint64_t *cookie, DirEntry *out);
    int (*readLink)(Vnode *node, char *buffer, size_t size);
    int (*getAttr)(Vnode *node, VnodeAttr *out);
    int (*setAttr)(Vnode *node, const VnodeAttr *attr, uint32_t mask);
    int (*resize)(Vnode *node, uint64_t size);
    int (*sync)(Vnode *node);
    void (*release)(Vnode *node);
} VnodeOps;

int vfsRegister(const FileSystemType *type);
int vfsMount(const char *source, const char *target, const char *type, uint32_t flags);
int vfsWalk(Vnode *start, const char *path, uint32_t flags, Vnode **out);
int vfsAccess(const Vnode *node, const Credentials *who, uint32_t want);
```
- A regular file's data moves through its `MemoryObject`; the filesystem supplies `readPage` and `writePage`. Device and `/proc` nodes carry their own read and write ops.
- `vfsWalk` is one loop over components. It crosses mounts through `mountedHere`, follows at most 8 symbolic links (F11), and checks search permission at each step.
- The name cache maps (directory vnode, name) to a vnode or to "absent". Create, remove, rename and unmount invalidate entries; size is bounded, oldest out first.
- Permissions (F10): one check, `vfsAccess`. User 0 passes (X3). Set-user and set-group bits are never honoured (X4).
- Hard links never name directories or cross mounts. A removed file lives until its last descriptor closes.
- Times (F14) are signed 64-bit nanoseconds since 1970 UTC.

### Pseudo-filesystems (F7) and layout (F8)
- **tmpfs**: vnodes and child lists on the heap; data lives only in the `MemoryObject`. Holds the unpacked boot archive (B8), then `/tmp`.
- **devfs**: drivers call `devfsAdd(name, ops, context, mode)`: `null`, `zero`, `random`, `klog`, `console`, `disk0`, `disk0p2`, `input/event0`.
- **/proc** (S14): the per-process files in processes.md plus `mounts` and `memory`, as U13 text built at open.

The kernel starts on the tmpfs archive. When `root=` names a device, it mounts that at `/`. The image builder creates the F8 directories; `/dev`, `/proc` and `/tmp` are mount points.

### Block layer (F12) and GPT (F5)
```c
typedef struct BlockRequest {
    uint32_t operation;                  /* BLOCK_READ, BLOCK_WRITE, BLOCK_FLUSH */
    uint32_t pageCount;
    uint64_t sector;                     /* 512-byte units */
    Page   **pages;
    void   (*done)(struct BlockRequest *request, int status);
    void    *context;
    ListNode node;
} BlockRequest;

int  blockAdd(BlockDevice *device);
void blockSubmit(BlockDevice *device, BlockRequest *request);
int  blockSubmitWait(BlockDevice *device, BlockRequest *request);
```
A device has a `submit` op, a size and a first-in first-out queue; no scheduler until a benchmark asks (principle 10). `done` runs in the driver's deferred work (K4), never in the interrupt handler. `blockSubmitWait` submits and sleeps on a wait queue. A partition is a child device that adds an offset.

The GPT reader in `lib/` checks the header and entry-array CRCs, falls back to the backup header, and reports entries; the kernel adds one child device each. yartfs gets its own partition type GUID.

### Later parts
- **Notification (F15):** `fileWatch(dirFd, path, mask)` returns a descriptor (K12) that yields create, remove, rename, modify and attribute records for a node or a directory's direct children, plus an overflow record. One `vfsNotify` hook in the VFS feeds `Vnode.watches`.
- **FAT (F4):** `fs/fat/` is a shared library like yartfs: FAT12, 16 and 32 with long names. The phase 5 host writer and the phase 6 kernel driver are the same code. Owner and mode come from mount options.
- **Removable media (F16):** `mount` by hand. In phase 6 a service watches `/dev` through F15 and mounts through the elevation service (X4).

## Per-architecture notes
Nothing here is per-port. Each port gets its own image (F13): x86_64 has an EFI partition and a yartfs root, riscv64 a yartfs root only.

## Build order
- Phase 2: block layer with a RAM disk, GPT, VFS and name cache, tmpfs, devfs, the yartfs library with mkfs, fsck and fuzzing, `yart-image`, the kernel yartfs driver (read, then write), root mount.
- Phase 3: `/proc`, file mappings, `mount` and `df`, fsck built for yart.
- Phase 5: notification, host FAT writer.
- Phase 6: FAT driver, yartfs v2, swap, automatic mounting.

## Test plan
- Host: library unit tests on a memory-backed volume. mkfs then fsck is clean. Image builder then dump round-trips a tree. fsck reports each seeded corruption. Fuzz `yartfsOpen`, walk and check, plus GPT and later FAT (Q8). A committed v1 image must mount forever.
- In-kernel: block requests on the RAM disk, partition offsets, path walk and name cache on tmpfs.
- Userland: one file suite on tmpfs and yartfs: create, resize, rename, links, link loops, permissions, times, remove while open, full disk.
- Boot: both ports run a program from a yartfs root (phase 2 milestone). A test writes files and powers off; host fsck must find the image clean and dump must find the files.

## Risks and open questions
- F2: journal or copy-on-write is not chosen, nor whether v1 volumes convert in place. v1 leaves both open.
- F3, F1, G4: nothing says what mounting a dirty volume does. VirtualBox and installed machines have no host, so fsck and mkfs must also build for yart. Proposed: mount dirty volumes with a loud warning until fsck runs on yart, then check at boot.
- B9, F5: the `root=` syntax is unsettled. Proposed: GPT partition label.
- F8: no directory is named for mount points, the EFI partition, root's home or app directories (W26).
- F9, F4, F6: FAT is case-insensitive. The name cache must let a filesystem supply its own comparison.
- F4, F16: large USB sticks often ship as exFAT, which no decision covers.
- F10, X4, R5: whether set-user and set-group bits are stored or rejected is open; ported `tar` may expect them kept.
- F14: no decision says whether access times are written on read. Proposed: only alongside another inode write.
- F15, S13: the watch call is not in the syscall table, and recursion is unspecified. This note assumes non-recursive.
- F10, F11, U8: syscalls-and-libraries.md lists no rename, link, symbolic link, attribute, resize, sync or mount calls.
- M9, F5: swap partition or swap file is undecided.
- F12, P9: v1 fixes the block size at 4 KiB. Disks with larger sectors are unsupported.
