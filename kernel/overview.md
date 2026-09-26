# Kernel Overview

FreshOS is a microkernel written in Rust (edition 2024, nightly toolchain). It targets aarch64 with UEFI boot. The reference hardware is the Raspberry Pi 4, and QEMU `virt` with HVF on Apple Silicon is the development loop. x86_64 was dropped under decision 0003 in the `freshos` repo.

## Architecture

- **Microkernel:** the kernel handles scheduling, IPC and memory management. Everything else is meant to run as user-mode services.
- **Separate binaries:** every service is its own ELF on the EFI system partition. The kernel starts only `init`; `init` starts the rest from its table and restarts the ones it supervises.
- **Typed message-passing IPC:** tasks communicate through bounded channels, reached through capability handles, and every send is recorded in a trace ring. See [IPC](ipc.md).
- **Kernel heap:** a 1 MiB linked-list allocator, needed by the Rhai scripting engine.

## Current state

Services (`init`, `ping`, `pong`, `pulse`, `fault`) run at **EL0, each in its own address space**, on QEMU `virt` with HVF. Each has its own page table and ASID, pages are never both writable and executable, and the kernel never dereferences a user pointer. Five built-ins (keyboard, compositor, shell, dashboard and MCP bridge) still run at **EL1** inside the kernel; each moves out under its own spec. Input arrives over the serial UART.

The design is `docs/plans/2026-09-26-el0-isolation-design.md` in the `freshos` repo, and its `AGENTS.md` is the detailed map. Nothing has run on a real Pi 4 yet; the checks that need one are in the status block of its `docs/Where-We-Are.md`.

## Key dependencies

| Crate  | Version | Purpose                       |
|--------|---------|-------------------------------|
| `uefi` | 0.33    | UEFI protocol access and boot |
| `rhai` | 1.24    | Embedded scripting engine     |
