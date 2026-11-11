# Storage and filesystems

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| F1 | What is the first root filesystem? | Our own design · ext2 · FAT32 | Our own filesystem, yartfs, from phase 2. We write the host-side tools that create and check images, sharing code with the kernel driver. | Graeson, detailed by Claude |
| F2 | Do we design our own filesystem later? | After phase 5 · Never | yartfs gets a second on-disk version after phase 5 that adds checksums and crash safety. The format carries a version field from day one. | Graeson, detailed by Claude |
| F3 | How crash-safe is storage before then? | Journaling · None, repair with host tools | None through phase 5. | Claude's pick, approved |
| F4 | When does FAT arrive? | Phase 2 · Phase 6 | Phase 6, read-write, for USB sticks and the firmware partition. | Claude's pick, approved |
| F5 | Which partition tables? | GPT only · GPT and MBR | GPT only. | Claude's pick, approved |
| F6 | How is the virtual filesystem layer built? | Minimal dispatch table · Vnodes, mount table and name cache | Vnodes, mount table and name cache. | Claude's pick, approved |
| F7 | Which pseudo-filesystems exist? | In-memory temp · Device nodes · Process info · Hardware tree | In-memory temp, device nodes and a small process-info tree. | Claude's pick, approved |
| F8 | What is the directory layout? | Classic Unix with `/usr` · Unix names, merged layout · Our own names | Unix names, merged: `/bin`, `/lib`, `/etc`, `/home`, `/dev`, `/proc`, `/tmp`, `/var`, and no `/usr` split. | Claude's pick, approved |
| F9 | Are file names case-sensitive? | Sensitive · Insensitive but preserving | Sensitive. | Claude's pick, approved |
| F10 | What permission model? | Unix owner, group and mode · Access control lists | Unix owner, group and mode. | Claude's pick, approved |
| F11 | Which link types? | Hard · Symbolic · Both | Both. | Claude's pick, approved |
| F12 | Is block I/O synchronous? | Synchronous · Async requests with a simple blocking wrapper | Async requests with a blocking wrapper. Retrofitting async later is painful. | Claude's pick, approved |
| F13 | How are disk images built? | Host script with standard tools · Our own image tool | Host script that needs no root access, using our own image and filesystem tools, producing one bootable image per architecture. | Graeson, detailed by Claude |
| F14 | How precise are timestamps? | Seconds · Nanoseconds | Nanoseconds, 64-bit. | Claude's pick, approved |
| F15 | Can programs watch files for changes? | Phase 3 · Phase 5 · Never | Phase 5, for the file manager. | Claude's pick, approved |
| F16 | How is removable media mounted? | Manual `mount` · Automatic in the GUI | Manual through phase 5, automatic in the GUI in phase 6. | Claude's pick, approved |
