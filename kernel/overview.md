# Kernel Overview

FreshOS is a microkernel written in Rust (edition 2024, nightly toolchain). It targets aarch64 with UEFI boot. The reference hardware is the Raspberry Pi 4, and QEMU `virt` with HVF on Apple Silicon is the development loop. x86_64 was dropped under decision 0003 in the `freshos` repo.

## Architecture

- **Microkernel:** the kernel handles scheduling, IPC and memory management. Everything else is meant to run as user-mode services.
- **External services:** `init`, `pong`, `pulse` and `fault` are ELF files that the kernel loads from the EFI system partition. `init` supervises them and restarts them when they exit.
- **Typed message-passing IPC:** tasks communicate through bounded channels, and every send is recorded in a trace ring. See [IPC](ipc.md).
- **Kernel heap:** a 1 MiB linked-list allocator, needed by the Rhai scripting engine.

## Current state

The desktop (compositor, shell and dashboard) runs at **EL1** on QEMU, because HVF traps the `tlbi` instructions that per-task page tables need. Only the `fault` service runs at EL0, and it shares a page table with everything else, so tasks are not yet isolated. Per-task EL0 isolation is planned for the Raspberry Pi 4. Input arrives over the serial UART.

## Key dependencies

| Crate  | Version | Purpose                       |
|--------|---------|-------------------------------|
| `uefi` | 0.33    | UEFI protocol access and boot |
| `rhai` | 1.24    | Embedded scripting engine     |
