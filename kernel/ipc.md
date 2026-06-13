# IPC Design

Tasks communicate through typed message-passing over bounded channels. There is no shared memory IPC.

## Channels

Each channel has a fixed capacity. Senders block when the channel is full; receivers block when it is empty. This backpressure keeps memory usage predictable and prevents runaway producers.

## Blocking

When a task has nothing to receive, the kernel uses an atomic `cli`/`hlt` sleep pattern: interrupts are disabled, the halt instruction is issued, and the next interrupt wakes the core and re-enables scheduling. This avoids busy-waiting while remaining responsive.

## Typed messages

Messages carry a type tag so the receiver can dispatch without parsing opaque byte buffers. The type set is fixed at compile time.

## Drivers as IPC services

Keyboard and mouse input are handled by ring 3 services, not kernel-mode drivers. These services read hardware (via port I/O granted to their address space) and publish events over IPC channels. Any task that needs input subscribes to the relevant channel.

This keeps hardware interaction out of the kernel and makes driver crashes non-fatal to the system.
