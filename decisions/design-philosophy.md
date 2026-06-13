# Decision: Design Philosophy

## "Understandable magic"

FreshOS should feel like magic to use, but the architecture must remain comprehensible to a single person. If a subsystem cannot be understood by reading its source in one sitting, it is too complex.

## Anti-features

These are things FreshOS will never ship:

- **No browser engine** — the web platform is unbounded complexity.
- **No monolithic applications** — large apps belong in user-space compositions, not the base system.
- **No backward compatibility** — the system is free to break its own interfaces between releases.
- **No telemetry** — the OS does not phone home. Ever.
- **No cloud dependency** — every feature works offline, always.

## Drivers as ring 3 services

Drivers run as unprivileged user-mode tasks, not kernel modules. They communicate with the kernel and other tasks through [typed IPC](../kernel/ipc.md). A driver crash does not take down the kernel.

## Rust edition 2024 patterns

Edition 2024 changes several unsafe ergonomics that affect OS code:

- `core::ptr::addr_of_mut!` replaces raw mutable pointer casts for statics.
- `#[unsafe(no_mangle)]` replaces `#[no_mangle]` — the attribute itself must be marked unsafe.
- `unsafe {}` blocks are required in more contexts; the compiler rejects implicit unsafe in places edition 2021 allowed.

These are good changes for an OS project. They make every unsafe boundary explicit and greppable.
