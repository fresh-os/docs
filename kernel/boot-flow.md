# Boot Flow

The boot sequence moves through five stages: UEFI handoff, CPU table setup, paging, kernel init, and the first context switch into ring 3.

## Stages

### 1. UEFI boot

The UEFI firmware loads the kernel as a PE executable. The kernel retrieves the framebuffer, memory map, and ACPI tables before calling `ExitBootServices`.

### 2. GDT and IDT

A Global Descriptor Table is loaded with kernel and user code/data segments. The segment ordering matters: user segments must follow kernel segments in the specific order the `sysret` instruction expects. The IDT is populated with exception and interrupt handlers.

### 3. Paging

Page tables use 2 MiB pages. The kernel identity-maps its own region and creates per-task page tables for user-mode address spaces.

### 4. Context switch to ring 3

- **CR3 swapping** — each task switch writes the target task's PML4 address into CR3.
- **TSS.RSP0** — updated per-task so the CPU knows where to find the kernel stack on privilege transitions.
- **Syscall/sysret path** — configured through three MSRs:
  - `IA32_STAR` — segment selectors for syscall/sysret
  - `IA32_LSTAR` — kernel entry point address
  - `IA32_FMASK` — RFLAGS mask on syscall entry
- Assembly stubs handle register save/restore around the transition.

### 5. Scheduler

Once the first user task launches, the scheduler runs at [100 Hz](performance.md) distributing time across tasks.
