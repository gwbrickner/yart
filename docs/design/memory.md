# Memory

Status: draft      Implements: M1 to M16

## Purpose
Memory management owns physical pages, the kernel heap, page tables and user address spaces. It sits on the MMU ops table (P8, arch-contract.md). Userland `malloc` (M15) belongs to the libc note.

## Decisions applied
- M4: a region is backed by a file or shared object, by an anonymous object holding its private pages, or by both (a private file mapping).
- M6, S1: fork shares pages by reference count (M3). Spawn is the native call, so fork stays simple.
- M14: a stack is a lazy region of the limit's size with an unmapped guard region below. Nothing grows.
- X6: no mapping is both writable and executable; addresses are randomized.

## Design

### Physical pages (M1, M3)
```c
typedef struct Page {
    uint32_t flags, refCount;
    ListNode list;
    union { uint32_t order; SlabCache *cache; MemoryObject *object; };
    uint64_t index;
} Page;
Page *pageAlloc(uint32_t order, uint32_t flags);
void  pageRef(Page *page);
void  pageUnref(Page *page);
```
The buddy allocator keeps one free list per order, 0 to 9. `pageUnref` frees at zero. `BootInfo` usable regions become a table of RAM ranges, each with its own `Page` array, so holes cost nothing. A boot-time bump allocator places the arrays.

### Kernel heap (M2, M16)
```c
SlabCache *slabCacheCreate(const char *name, size_t size, size_t align);
void *slabAlloc(SlabCache *cache, uint32_t flags);
void *heapAlloc(size_t size, uint32_t flags);
void  heapFree(void *pointer);
```
Slabs are direct-map pages whose `Page` names the cache, so `heapFree` needs no size. `heapAlloc` picks a power-of-two size-class cache; large requests go to the buddy. One spinlock per cache; per-CPU caches wait for a benchmark. With `CONFIG_DEBUG_MEMORY`, freed objects and pages are filled with a poison pattern that the next allocation verifies, and red zones around each object are checked on free.

### Kernel address space (M11, M12)
From the bottom of the higher half: the direct map of all RAM, the only mapping with large leaves; a kernel virtual area for guarded stacks and device mappings; the image in the top 2 GiB (boot.md), then the module area. Bases are computed at boot (B11). In phase 1 the kernel builds its own root with the walker and switches to it. Kernel-half root entries are all created then and never change, because two ports copy them into every user root.

### Page tables (M13)
```c
int ptMap(uint64_t root, uintptr_t virt, uint64_t phys, uint32_t flags, uint32_t level);
int ptQuery(uint64_t root, uintptr_t virt, uint64_t *phys, uint32_t *flags);
```
Every port has 512 eight-byte entries per table (P9). One iterative loop (T11) descends `arch->mmu->levels` levels, 9 bits each, through the port's `makeLeaf`, `makeTable` and `decode`. Replacing a live entry is always clear, flush, write, as ARM requires. Dirty tracking is software: a writable shared page stays read-only until its first write fault.

### Address spaces (M4 to M6, M14)
```c
typedef struct MemoryObject {
    const MemoryObjectOps *ops;    /* readPage, writePage */
    PageTree pages;
    List regions;
} MemoryObject;

typedef struct Region {
    uintptr_t start, end;
    uint32_t protection, flags;    /* REGION_SHARED, REGION_GUARD */
    MemoryObject *object;          /* file or shared, or NULL */
    MemoryObject *overlay;
    uint64_t offset;
} Region;

int vmMap(AddressSpace *space, uintptr_t *address, size_t length, uint32_t protection,
          uint32_t flags, MemoryObject *object, uint64_t offset);
int vmFork(AddressSpace *parent, AddressSpace **child);
int vmHandleFault(AddressSpace *space, uintptr_t address, uint32_t access);
```
An `AddressSpace` is a root table, a mutex, a region tree and the set of CPUs running it. `vmMap` only records a `Region` (M5). `vmHandleFault` takes the page from `overlay`, else from `object` (`readPage` fills a miss), else a zeroed page. A write to a private page that is not solely the overlay's copies it first. `vmFork` gives the child overlays that reference the same pages and makes both sides read-only. A fault in a guard region delivers the fault signal (S7) and logs a stack overflow (S15).

### Page cache, file mappings, shared memory (M7, M8)
Phase 3: each vnode owns one file `MemoryObject`, and its `PageTree` is the page cache. `read` and `write` copy to and from those pages and mappings map them. Filesystem metadata is cached through the block device's object. A kernel thread writes dirty pages back through `writePage`; clean pages sit on a reclaim list. Phase 5: shared memory is an anonymous object held by a file descriptor (K12), mapped `REGION_SHARED` and passed over local sockets (S12).

### Running out (M10)
`vmMap` never refuses for lack of RAM. When `pageAlloc` fails it reclaims clean cache pages, then kills the process with the most resident pages (never init), logs it at error level, and retries.

### TLB shootdown
```c
void tlbFlushRange(AddressSpace *space, uintptr_t start, uintptr_t end);
void tlbFlushKernel(uintptr_t start, uintptr_t end);
```
`kernel/` flushes only through these. Until phase 4 they call the port's local flush. Then they queue a request for each other CPU in the space's set, send an IPI and wait for its `ipi` upcall to acknowledge. Fixed now: a page is freed only after the flush returns, and no spinlock is held across one.

### Swap (M9)
Phase 6 adds anonymous-object ops that read and write swap slots. For that to be an addition, these hold earlier:
- Phase 2: every user page lives in an object slot, and page tables are a cache that faults rebuild. Slots are tagged words, so one can later name a swap location. Kernel memory is never in an object.
- Phase 3: the fault path may sleep on I/O. An object's region list can unmap a page from every address space. The reclaim list and software dirty tracking cover file pages.

## Per-architecture notes
- x86_64: 4 levels.
- riscv64: 3 levels (Sv39) or 4 (Sv48), undecided.
- aarch64: 4 levels, 4K granule, separate user and kernel roots.

## Build order
- Phase 1: boot allocator, `Page` arrays, buddy, kernel root, slab and heap, poisoning, flush interface.
- Phase 2: address spaces, anonymous regions, lazy faults, fork, stacks, the killer.
- Phase 3: file objects, page cache, file mappings, writeback, reclaim.
- Phase 4: shootdown by IPI, AArch64 encoding. Phase 5: shared memory. Phase 6: swap.

## Test plan
- Host: buddy, slab, page tree, region tree, fault logic with fake objects, the walker on the fake port at 3 and 4 levels.
- In-kernel: map and query at each leaf size; poison and red zones catch a deliberate use-after-free and overflow.
- Userland: fork isolation, stack overflow raises the fault signal, a file write shows through a mapping, two processes share memory, allocating until killed leaves the system running.
- Boot: all of it in 64 MiB (P11).

## Risks and open questions
- M13, P3: Sv39 or Sv48 is undecided. The walker handles both; Sv39 suits a small custom CPU.
- M13, P8: the walker needs the user address limit, the kernel-half base, the largest leaf level and, on AArch64, which root an address belongs to. `ArchMmuOps` reports only `levels`.
- M12, B5: the direct-map base is not in `BootInfo`, and boot.md does not say whether the loader's tables may be freed.
- M11, P4: 1 GiB leaves are not baseline x86_64.
- M8, M7, F1: yartfs arrives in phase 2, the page cache in phase 3, and M8 names no phase. Either phase 2 file I/O is uncached and reworked, or file objects move to phase 2.
- M6, M9: a page shared by fork sits in several overlays with no list of owners, so swap cannot unmap it everywhere. Phase 6 skips such pages or adds owner tracking.
- M9, M13: hardware accessed bits differ per port, so how swap finds cold pages is unsettled.
- M10, S16: M10 names no victim rule, and S16 does not say whether its limit counts mapped or resident memory.
- X6, H16: randomized layout in phase 2 needs entropy before any driver supplies it.
