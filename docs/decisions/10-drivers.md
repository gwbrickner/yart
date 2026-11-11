# Drivers and hardware

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| H1 | How do drivers find devices? | Hand-written init calls · Buses enumerate, drivers register match tables | Buses enumerate and drivers register match tables. | Claude's pick, approved |
| H2 | Which virtio devices come first? | Block · Console · Network · Display · Input · Random | All six over time, block and console in phase 2. One virtio core supports both the PCI and memory-mapped transports. | Claude's pick, approved |
| H3 | How is PCI handled? | Memory-mapped config only · Legacy port access too | Memory-mapped config only, with message-signaled interrupts preferred. | Claude's pick, approved |
| H4 | Which interrupt controllers? | Modern ones only · Legacy x86 PIC as well | Modern only: APIC on x86, PLIC on RISC-V, GICv3 on ARM. | Claude's pick, approved |
| H5 | How deep does ACPI support go? | Static tables only · Our own bytecode interpreter · A ported interpreter | Static tables through phase 5. A ported interpreter in phase 6, since writing one is a known tar pit. | Claude's pick, approved |
| H6 | Which real storage controllers? | NVMe · SATA · Both | SATA (AHCI) in phase 5, because VirtualBox needs it (D9). NVMe in phase 6. | Claude, following D9 |
| H7 | What USB support? | xHCI only · Older controllers too | xHCI only, in phase 6, with keyboard, mouse and storage classes. | Claude's pick, approved |
| H8 | How does input work before USB exists? | Virtio input in QEMU · PS/2 on real machines | Both. PS/2 is the cheap path to a working keyboard on real x86 hardware. | Claude's pick, approved |
| H9 | How do pixels reach the screen? | Firmware framebuffer · Virtio display · Native GPU drivers | Firmware framebuffer on real machines and virtio display in QEMU. Native GPU drivers are a stretch goal after 1.0. | Graeson |
| H10 | Is there GPU acceleration? | Yes · Not before 1.0 | After 1.0. Everything is software rendered until then. | Graeson |
| H11 | What audio hardware? | None · Virtio sound · Intel HD Audio | Virtio sound in phase 6, Intel HD Audio after. | Claude's pick, approved |
| H12 | Which network cards? | Virtio only · One real card too | Virtio first, then whichever wired card your test machine has. | Claude's pick, approved |
| H13 | Is there Wi-Fi? | Yes · Not before 1.0 | Not before 1.0. It needs vendor firmware and a very large stack. | Claude's pick, approved |
| H14 | How do shutdown and reboot work? | Firmware services · ACPI | Firmware services on each platform: UEFI on x86, our own machine-mode firmware on RISC-V, PSCI on ARM. | Claude, following P6 |
| H15 | Is there sleep, battery or backlight support? | Yes · After the ACPI interpreter lands | After the ACPI interpreter lands. Laptop features depend on it. | Claude's pick, approved |
| H16 | Where does randomness come from? | Virtio device · CPU instructions · Interrupt timing | All three, mixed into one pool. | Claude's pick, approved |
| H17 | How are drivers tested? | By hand · Each driver has an automated QEMU test | Each driver has an automated QEMU test. | Claude's pick, approved |
