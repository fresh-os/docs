# Decision: aarch64 Port

## Context

FreshOS runs on x86_64/UEFI. An aarch64 port opens the door to Apple Silicon hardware and broadens the architecture abstractions in the kernel.

## Plan

The port runs in three phases on **QEMU virt** with **HVF** acceleration (Apple Silicon host).

### Phase 1 — Serial boot

Bring-up to a UART prompt. Set up the GICv2 timer for scheduling.

**Gotcha:** the timer is the virtual timer, PPI INTID 27. (The non-secure physical timer is INTID 30.) The GICC base address on QEMU virt is not at the address some tutorials assume; read it from the device tree.

### Phase 2 — Paging, context switch, IPC

Port the page table, CR3-equivalent (TTBR0/TTBR1) context switch, and IPC layer. The typed message-passing design is architecture-independent; only the context switch and syscall entry stubs need new assembly.

### Phase 3 — Display

Two options considered:

| Option     | Pros                                    | Cons                          |
|------------|-----------------------------------------|-------------------------------|
| virtio-gpu | Proper GPU protocol, resolution control | Complex driver, needs virtio  |
| ramfb      | Trivial post-ExitBootServices setup     | Fixed resolution, no 3D      |

**Decision:** start with **ramfb** for initial bring-up. It is simpler to get pixels on screen after ExitBootServices, and the graphical shell does not need GPU acceleration yet.

## Consequences

- Kernel gains a `arch/` split for platform-specific code.
- Assembly stubs (syscall entry, context switch) are duplicated per architecture.
- Phase 1 can land without touching the existing x86_64 code path.
