# Boot Flow

FreshOS boots as a UEFI application on aarch64 and ends with `init` running at EL0.

## Stages

### 1. While UEFI is still running

- Enable FP/SIMD (`CPACR_EL1`).
- Pick the largest graphics mode up to 1920×1200 and keep its framebuffer.
- Read every `*.ELF` in `\EFI\FreshOS\` into memory.
- Take the memory map, then exit boot services.

### 2. Kernel bring-up

- Install the exception vectors and bring up the GIC (v3 on QEMU, v2 on the Pi 4, chosen by the board layer).
- Start the frame allocator from the memory map, then the kernel heap.
- Set up paging: detect and enable PAN where the CPU has it, set the EL0-facing system registers to known values, and flush the TLB before any user address space exists.
- Create the well-known IPC channels.

### 3. Start `init`

The kernel requires only `INIT.ELF`. Without it, it prints `INIT.ELF missing from \EFI\FreshOS — nothing to run` and halts. Otherwise it builds `init`'s address space, grants its handles, and starts the scheduler.

### 4. Scheduler

A 1 ms virtual-timer tick drives preemption, and tasks also switch on the way out of a syscall (see [IPC](ipc.md)). `init` then starts every other service from its table.
