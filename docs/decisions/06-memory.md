# Memory

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| M1 | How are physical pages allocated? | Bitmap · Free list · Buddy allocator | Buddy allocator. | Claude's pick, approved |
| M2 | How does the kernel heap work? | Simple free list · Slab caches | Slab caches with a general-purpose allocator on top. | Claude's pick, approved |
| M3 | Is there a metadata record per physical page? | Yes · No | Yes: reference count and flags. Copy-on-write and the page cache both need it. | Claude's pick, approved |
| M4 | How is a process's address space modeled? | Page tables only · Regions backed by memory objects | Regions backed by memory objects, either anonymous or file. | Claude's pick, approved |
| M5 | Is memory allocated lazily? | On first touch · Up front | On first touch. | Claude's pick, approved |
| M6 | How does fork copy memory? | Copy-on-write · Full copy | Copy-on-write. | Claude's pick, approved |
| M7 | When do file mappings and shared memory arrive? | Phase 2 · Phase 3 · Phase 5 | Anonymous mappings in phase 2, file mappings in phase 3, shared memory before the GUI in phase 5. | Claude's pick, approved |
| M8 | How is file data cached? | One unified page cache · Separate block cache | One unified page cache. | Claude's pick, approved |
| M9 | Is there swap? | Yes · No | Yes. Swap arrives in phase 6, once the block layer and page cache have settled. | Graeson, detailed by Claude |
| M10 | What happens when memory runs out? | Strict accounting, allocations fail · Overcommit with a process killer | Overcommit with a simple killer that logs loudly. Ported software expects it. | Claude's pick, approved |
| M11 | Are large pages used? | Kernel only · Userland too · Never | Kernel's own mapping of RAM only. | Claude's pick, approved |
| M12 | How does the kernel reach physical memory? | Direct map of all RAM · Map on demand | Direct map of all RAM in the higher half. | Claude's pick, approved |
| M13 | How are page tables abstracted? | One generic walker with per-port entry encoding · Separate code per port | One generic walker. All three targets use the same tree shape, so only the entry bits differ. | Claude's pick, approved |
| M14 | How do user stacks grow? | Fixed size · Lazy up to a limit with a guard page | Lazy up to a limit, guard page below. | Claude's pick, approved |
| M15 | Where does userland `malloc` come from? | Our own · Ported allocator | Our own size-class allocator in libc. | Claude's pick, approved |
| M16 | What memory debugging exists? | Poisoned frees and red zones in debug builds · None | Both, in debug builds. | Claude's pick, approved |
