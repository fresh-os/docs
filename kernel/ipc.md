# IPC Design

Tasks communicate through typed message passing over bounded channels. There is no shared-memory IPC.

## Channels and handles

A channel holds up to 16 messages. Each message is a type tag and 32 bytes of inline payload, and the kernel stamps the sender's identity on it, so no task can forge another.

Tasks never see channel numbers. They hold **handles**, slots in a per-task table that name a channel and the rights over it (`SEND`, `RECV`). `init` creates channels and grants handles when it starts a service. Each channel has exactly one receiving process: a `RECV` right moves to the child it is granted to and returns to `init` when that child exits, and messages sent in the meantime wait for the next receiver.

## Sending and blocking

A send to a full channel fails with `Full` rather than blocking, so a producer can't stall on a stuck consumer. A receive on an empty channel blocks, optionally with a deadline.

## Direct hand-off

A send to a task already waiting in `recv` delivers straight into that task's buffer and runs it next, instead of waiting for the next timer tick. If that receiver then blocks while the sender is still ready, the sender runs next (sender-return). The 1 ms tick still rotates between tasks, so fairness holds at tick granularity. Hand-off took the ping/pong round trip from about 10.8 ms to tens of microseconds on QEMU with HVF.

## Drivers as IPC services

Input drivers are meant to be EL0 services that read hardware and publish events over channels; any task that needs input holds a handle to the relevant channel. This keeps hardware interaction out of the kernel and makes a driver crash non-fatal. The keyboard driver is still an EL1 built-in until it moves out.
