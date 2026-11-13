# Drivers

Status: draft      Implements: H1 to H17

## Purpose
Drivers turn the hardware a machine reports into the block, input, display, network and console devices the rest of the kernel uses. The device model lives in `drivers/bus/`, the table parsers in `lib/`.

## Decisions applied
- H1, K1: every driver is a module whose `init` registers one `Driver`.
- H2: virtio drivers see only the virtio core, never a transport. Version 1 devices and split queues only.
- H3, H5: without the interpreter x86 has no interrupt routing table, so through phase 5 an x86 PCI device must offer message-signaled interrupts.
- P7: ACPI tables and the device tree both produce platform devices; drivers cannot tell which.

## Design

### Driver model (H1)
```c
typedef struct DeviceMatch {
    uint32_t    bus;                     /* platform, PCI, virtio, USB */
    uint32_t    vendor, device, classCode, classMask;
    const char *compatible;
} DeviceMatch;

typedef struct Driver {
    const char        *name;
    const DeviceMatch *matches;
    size_t             matchCount;
    int  (*probe)(Device *device, const DeviceMatch *match);
    void (*remove)(Device *device);
    void (*shutdown)(Device *device);
} Driver;

int driverRegister(Driver *driver);
int deviceAdd(Bus *bus, Device *device);
int deviceMapRegisters(Device *device, uint32_t index, DeviceRegisters *out);
int deviceIrq(Device *device, uint32_t index, uint32_t *irqOut);
```
A bus enumerates and calls `deviceAdd`. Matching runs both ways: a new device is offered to every driver, a new driver to every unbound device, so load order never matters. A `Device` carries its parent, identifiers and resources. `DeviceRegisters` and its read and write accessors hide whether a range is memory or x86 I/O ports, so one 16550 driver serves both ports and no driver includes a port header (principle 1). Handlers follow K4.

### Discovery (P7, H5)
- **ACPI static tables**, parsed in `lib/`: XSDT, MADT (interrupt controllers), MCFG (PCI config base), FADT flags. The x86 port also adds fixed platform devices: serial port, real-time clock and, when the FADT reports one, the PS/2 controller.
- **Device tree**: each node with a `compatible` string becomes a platform device; `reg` and `interrupts` become its resources.
- **ACPI interpreter**, phase 6: ported (D7). It adds interrupt routing, the power button, battery and backlight (H15).

### PCI and interrupts (H3, H4)
The PCI bus walks memory-mapped config space iteratively (T11) and adds one device per function. It prefers MSI-X, then MSI, then a pin where the platform describes routing. Interrupt controllers sit behind `ArchIrqOps`; drivers see only the number `deviceIrq` returns.

### Virtio (H2)
```c
typedef struct VirtioTransportOps {
    void (*setStatus)(VirtioDevice *device, uint8_t status);
    void (*setFeatures)(VirtioDevice *device, uint64_t features);
    void (*readConfig)(VirtioDevice *device, uint32_t offset, void *buffer, size_t length);
    int  (*setupQueue)(VirtioDevice *device, VirtQueue *queue);
    void (*notify)(VirtioDevice *device, uint32_t queueIndex);
    /* also getStatus, getFeatures */
} VirtioTransportOps;

int   virtQueueAdd(VirtQueue *queue, const VirtioBuffer *buffers, uint32_t outCount,
                   uint32_t inCount, void *token);
void *virtQueueTake(VirtQueue *queue, uint32_t *lengthOut);
```
The PCI transport matches the virtio vendor on the PCI bus; the memory-mapped transport matches `virtio,mmio` platform devices. Each adds a device to the virtio bus, where the six H2 drivers and sound match by device type.

### Device classes
```c
void inputReport(InputDevice *device, uint16_t type, uint16_t code, int32_t value);
int  displayAdd(Display *display, const DisplayOps *ops);
void entropyAdd(const void *data, size_t length, uint32_t creditBits);
```
- **Input (H8):** virtio input and PS/2 (controller, keyboard, mouse) translate to one key code table and report events read from `/dev/input/`.
- **Display (H9, H10):** `DisplayOps` has `getMode`, `setMode` and `flush(damage)`. The firmware framebuffer driver wraps `BootInfo.framebuffer`: one fixed mode, no flush. Virtio display is 2D only. Neither reports refresh, so the compositor uses its timer (W4).
- **Storage (H6):** virtio block, AHCI and NVMe complete the `BlockRequest`s of storage.md.
- **USB (H7):** xHCI is a bus. Keyboard, mouse and storage drivers match on class codes.
- **Audio (H11):** virtio sound playback behind `/dev/audio0`.
- **Network (H12):** virtio net, then one real card, behind one `NetDevice` interface (N3). No Wi-Fi (H13).
- **Randomness (H16):** virtio random, CPU instructions and interrupt timing call `entropyAdd`; our own hash (X8) turns the pool into `/dev/random`.
- **Shutdown (H14):** sync filesystems, call each driver's `shutdown`, then the port's `ArchResetOps`.

### What each environment needs
Numbers are phases.

| Environment | Console | Storage | Input | Display | Network |
| --- | --- | --- | --- | --- | --- |
| QEMU x86_64 (`q35`, UEFI) | 16550 serial (0), virtio console (2) | virtio block on PCI (2) | virtio input (3) | virtio display (3) | virtio net (6) |
| QEMU riscv64 (`virt`) | firmware console (0), virtio console (2) | virtio block, memory-mapped (2) | virtio input (3) | virtio display (3) | virtio net (6) |
| QEMU aarch64 (`virt`) | PL011 (4) | virtio block (4) | virtio input (4) | virtio display (4) | virtio net (6) |
| VirtualBox, x86_64 EFI | framebuffer console (3); serial to verify | AHCI (5) | PS/2 keyboard and mouse (5) | firmware framebuffer (5) | virtio net if offered, to verify (6) |
| Real x86 PC | framebuffer console (3) | USB storage to boot, then AHCI or NVMe (6) | USB (6), PS/2 where present (5) | firmware framebuffer (5) | one wired card (6) |

To verify for VirtualBox before phase 5: that its AHCI controller offers MSI, that the pointing device can be PS/2, how the EFI screen mode is set, and how a raw image is attached.

## Per-architecture notes
- x86_64: local and I/O APIC from the MADT.
- riscv64: PLIC and memory-mapped virtio from the device tree. The PLIC takes no message-signaled interrupts.
- aarch64: GICv3 from the device tree.

## Build order
- Phase 0 and 1: device tree reader and static ACPI lookup (findings R2).
- Phase 2: device model, platform bus, PCI, virtio core with both transports, virtio block, console and random.
- Phase 3: virtio display and input for the framebuffer console.
- Phase 4: runtime loading, AArch64 controllers.
- Phase 5: AHCI, PS/2, firmware framebuffer.
- Phase 6: xHCI and USB classes, NVMe, virtio net and sound, the real network card, ACPI interpreter.

## Test plan
- Host: ACPI and device tree parsers with fuzzing (Q8); virtqueue against a fake device; PS/2 and USB report decoding.
- In-kernel: bind and unbind a fake driver on a fake bus.
- Boot (H17): one QEMU test per driver: block read-back, keys typed through the QEMU monitor, a screen dump against a reference; AHCI, NVMe and xHCI on `q35`.
- Manual checklist: VirtualBox and the real PC.

## Risks and open questions
- H3, H5, H6, D9: with static tables only, x86 cannot route PCI interrupt pins. If VirtualBox's AHCI lacks MSI, phase 5 needs polling or an early interpreter.
- B10, H2, H9: the phase 3 framebuffer console needs a display and a keyboard, but virtio display and input have no phase and RISC-V has no firmware framebuffer. Phase 3 assumed.
- H16, H2, findings R7: virtio random in phase 2 would supply early entropy; H2 names only block and console.
- P8, H8, H3, H16: the arch contract lacks port I/O, device memory mapping, an MSI vector op, memory barriers and a CPU random op.
- M1: some controllers need memory below 4 GiB; the allocator has no such flag.
- H3, H4: MSI on AArch64 needs GIC parts H4 does not mention.
- H3, P6: our RISC-V firmware assigns no PCI addresses, so PCI there means the kernel must.
- K1, H1: what loads the module for an unmatched device is undecided.
- H5, H7, H11: the interpreter is not chosen, hub support is unspecified, and nothing places the audio mixer.
- D9, H12: the real machine is unnamed; it sets the phase 6 driver list.
- R8, P5: Hyper-V's UEFI machines may offer only paravirtual devices, and VMware's device set is unknown; both to verify.
