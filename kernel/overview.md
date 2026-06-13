# Kernel Overview

FreshOS is a microkernel written in Rust (edition 2024, nightly toolchain). The primary target is x86_64 with UEFI boot. An aarch64 port is [in progress](../decisions/aarch64-port.md).

## Architecture

- **Microkernel** — the kernel handles scheduling, IPC, and memory management. Everything else (drivers, filesystem, shell) runs as ring 3 user-mode tasks.
- **Per-process page tables** — each task has its own address space, switched via CR3 on context switch.
- **Typed message-passing IPC** — tasks communicate through bounded channels. See [IPC](ipc.md).
- **No post-boot heap** — after initialisation completes, the kernel uses stack-based buffers and static mutation only. No dynamic allocator runs during normal operation.

## Current state

Seven concurrent tasks running, including a graphical shell. Keyboard and mouse input arrive through ring 3 IPC services, not kernel-mode drivers.

## Key dependencies

| Crate    | Version | Purpose                        |
|----------|---------|--------------------------------|
| `uefi`   | 0.33    | UEFI protocol access and boot  |
| `x86_64` | 0.15    | GDT, IDT, paging structures    |
| `rhai`   | 1.24    | Embedded scripting engine       |
