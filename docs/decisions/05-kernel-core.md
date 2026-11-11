# Kernel core

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| K1 | Are there loadable kernel modules? | Yes · Static only · Later | Yes, loadable kernel modules. Drivers use the module interface from phase 2 and are linked in at first; runtime loading arrives in phase 4. | Graeson, detailed by Claude |
| K2 | Can kernel code be preempted? | Only user code is preempted · Kernel is preemptible | Preemptible kernel from the start. Multicore forces real locking anyway, so design it once. | Claude's pick, approved |
| K3 | Which locking primitives exist? | Spinlocks · Sleeping mutexes · Read-write locks · RCU | Spinlocks, sleeping mutexes and wait queues. A debug build checks lock ordering. | Claude's pick, approved |
| K4 | How much work happens inside an interrupt handler? | Everything · Minimal, with the rest deferred | Minimal handlers, with the rest deferred to kernel threads. | Claude's pick, approved |
| K5 | How does the kernel keep time? | Fixed periodic tick · One-shot timers on demand | One-shot timers. All three architectures support them, and idle CPUs stay idle. | Claude's pick, approved |
| K6 | Where do the clocks come from? | CPU counter for monotonic time · Real-time clock for wall time · Network time | CPU counter for monotonic time, real-time clock read at boot for wall time, network sync in phase 6. | Claude's pick, approved |
| K7 | How do kernel functions report errors? | Negative error codes · Result structs | Negative error codes, one list shared with libc. | Claude's pick, approved |
| K8 | How does the kernel log? | Plain prints · Levelled ring buffer readable from userland | Levelled ring buffer, mirrored to serial, readable from a `dmesg`-style tool. | Claude's pick, approved |
| K9 | What happens on a kernel panic? | Halt with register dump · Add a symbolized backtrace · Reboot | Register dump plus a symbolized backtrace from an embedded symbol table, then halt. | Claude's pick, approved |
| K10 | How is the kernel debugged? | QEMU's GDB server · Built-in debug shell · Both | Both. The debug shell on serial inspects tasks, memory and locks when GDB isn't attached. | Claude's pick, approved |
| K11 | Are assertions compiled into release builds? | Always · Debug only | Always, until 1.0. | Claude's pick, approved |
| K12 | How does userland name kernel objects? | File descriptors for everything · Typed handles | File descriptors for everything. | Claude's pick, approved |
| K13 | How are internal interfaces documented? | Header comments are the contract · Separate design notes only | Header comments are the contract, and the design note explains why. | Claude's pick, approved |
| K14 | How does the kernel touch user memory? | Dedicated copy functions with fault recovery · Direct pointer access | Dedicated copy functions only. CPU features that block direct access stay switched on. | Claude's pick, approved |
