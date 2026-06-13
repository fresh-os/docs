# FreshOS — Development Notes

Architecture decisions, kernel notes, and a running development log for [FreshOS](https://github.com/fresh-os/freshos) — a microkernel operating system in Rust.

## Kernel

- [Overview](kernel/overview.md) — architecture, dependencies, current state
- [Boot Flow](kernel/boot-flow.md) — UEFI boot through to the ring 3 context switch
- [IPC](kernel/ipc.md) — typed message-passing and channel design
- [Performance](kernel/performance.md) — latency targets and measurement status

## Decisions

- [aarch64 Port](decisions/aarch64-port.md) — three-phase bring-up plan
- [Chrome as Services](decisions/chrome-as-services.md) — extracting menu/taskbar/stats into ring-3 tasks
- [Design Philosophy](decisions/design-philosophy.md) — anti-features and guiding principles

## Log

- [Development Log](log.md) — chronological progress notes, newest first
