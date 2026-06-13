# Performance Targets

## Targets

| Metric            | Target | Status     |
|-------------------|--------|------------|
| Frame time        | As low and stable as possible | Instrumented on `aarch64` dashboard |
| Input-to-photon   | <5 ms  | Instrumented on `aarch64` dashboard |
| IPC latency       | <1 us  | Instrumented via ping/pong probe on `aarch64` |
| Scheduler wake    | As low and bounded as possible | Instrumented on `aarch64` dashboard |
| Scheduler tick    | 1000 Hz | Active     |

## Definitions

- **Input-to-photon** — time from a hardware input event (keystroke, mouse movement) to a visible change on the framebuffer. Covers the full path: driver IPC, task processing, and pixel write.
- **Frame time** — compositor loop duration from frame start through display present.
- **IPC latency** — time from send to receive for a single typed message on an unbuffered channel, measured between two ring 3 tasks.
- **Scheduler wake** — time from a blocked receiver becoming ready to that task actually running again.
- **Scheduler tick** — timer interrupt frequency that drives preemptive scheduling. Currently set to 1000 Hz (1 ms quantum).

## Measurement plan

The Apple Silicon/HVF path now exposes live coarse metrics in the dashboard and compositor overlay, plus a compositor phase split for background/content redraw, window composition, chrome, and present. Record representative values and methodology in the [development log](../log.md) once measured on hardware.
