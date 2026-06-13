# Decision: Chrome as Ring-3 Services

## Context

The compositor task draws every pixel on screen: desktop background, window frames, window contents (via surface blits), **menu bar, taskbar, stats overlay, cursor**. Window contents already come from independent tasks (shell → surface 0, dashboard → surface 1), but the chrome elements are drawn inline inside `user_compositor` in `kernel/src/main.rs` (and the equivalent `compositor_el1` in `kernel/src/arm_tasks.rs`).

This violates the design philosophy: ["no monolithic applications"](design-philosophy.md) and "drivers as ring 3 services" — the chrome is effectively a set of small apps that the compositor is running on behalf of the system.

Concrete problems it creates today:

- A slow or broken chrome path janks the compositor frame.
- The compositor holds state that logically belongs elsewhere (`latency_hist`, `hist_idx`, clock counter).
- The same monolith pattern is duplicated across the x86 and aarch64 compositors.
- Supervision gives us restart-on-crash for `pulse`/`fault` today but not for the UI.

## Decision

Extract each chrome element into its own ring-3 task that owns a dedicated off-screen surface. The compositor becomes a pure compositor: it blits surfaces at fixed positions each frame. It does not draw chrome content.

The three elements in scope:

| Element       | Surface size | Redraws on                                    |
|---------------|--------------|-----------------------------------------------|
| Menu bar      | 1280×28      | clock tick (1 Hz), metrics refresh, task count change |
| Stats overlay | 200×140      | metrics sample (event-driven)                 |
| Taskbar       | 1280×36      | workspace change                              |

Cursor is **out of scope** for this decision — it has low-latency requirements that warrant its own analysis.

### Surface model

Each chrome surface is **double-buffered**. The chrome task draws to its back buffer and sends a `CommitSurface { idx }` IPC message to the compositor. The compositor swaps front/back on its next frame. No tearing.

### Event channels

Two new IPC channels:

- `CH_WORKSPACE` — `WorkspaceChanged { ws: u32 }`. The compositor publishes when the user switches workspace; the taskbar subscribes.
- `CH_METRICS` — `MetricsSample { frame_us, ipc_rtt_us, wake_us, input_to_photon_us, tasks }`. The compositor (or a dedicated metrics publisher) emits periodically; the menu bar and stats overlay subscribe.

### Kernel-side support

One new syscall may be needed: `SYS_METRICS` returning a `MetricsSample` struct. The alternative — per-field syscalls — multiplies ring transitions for data that's always read together. Decision: introduce the struct syscall when Phase 2 begins, not before.

## Plan

Phases land one at a time. Each phase ships on **x86 first**, is verified visually on q35, then is ported to aarch64 as an external ELF service (mirroring the existing init / pong / pulse / fault shape).

### Phase 1 — Taskbar

The smallest element with no live metrics. Validates the surface / IPC / commit protocol before we spend it on anything harder.

- Allocate surface 2 (1280×36×4 double-buffered).
- New task `user_taskbar`: subscribes to `CH_WORKSPACE`, draws pills into its back buffer, sends `CommitSurface { idx: 2 }`.
- Compositor: remove `draw_taskbar` call, add `blit(surface_2, 0, TBAR_Y)` each frame.
- Compositor also publishes on `CH_WORKSPACE` when the user switches workspace.

**Stop here if:** double-buffer coordination or IPC back-pressure is thornier than expected. The taskbar is the cheap experiment; do not continue until the pattern feels right.

### Phase 2 — Menu bar

Depends on `CH_METRICS` and `SYS_METRICS`. Both land in this phase.

- Surface 3 (1280×28×4 double-buffered).
- Task `user_menu_bar`: reads task count + time via syscalls, reads latency via `CH_METRICS`, redraws on 1 Hz tick or metrics change.
- Compositor: remove `draw_menu_bar` call, add blit.

### Phase 3 — Stats overlay

- Surface 4 (200×140×4 double-buffered).
- Task `user_stats`: subscribes to `CH_METRICS`, maintains its own history ring buffer, renders graph.
- Compositor: remove `draw_stats_overlay` call, add blit, drop local `latency_hist` / `hist_idx`.

### Phase 4 — Cursor (stretch, separate decision)

Not part of this record. A cursor-as-service needs its own latency analysis and may want a priority scheduling class.

## Consequences

- Compositor shrinks to: clear, blit windows, blit chrome surfaces, present. Roughly one screenful of code.
- Task count grows from 7 to 9 by end of Phase 3. Scheduler is expected to cope at 1 kHz PIT, but measure at each phase and revisit if input-to-photon regresses.
- `knowledge/kernel/ipc.md` needs to document the two new channels and the `CommitSurface` message type before Phase 1 merges.
- The aarch64 side gains three more ELF services (TASKBAR.ELF, MENU.ELF, STATS.ELF) loaded from the ESP, which exercises the existing external-service path for UI rather than only test services like `pulse` and `fault`.
- The compositor stops being the single point of failure for on-screen output. Each chrome service can crash and be restarted by `init` without taking the UI down.

## Non-goals

- Generic window management, z-order, resizing — resist abstraction. Three fixed-rect surfaces is three instances, not a pattern.
- Virtio-GPU scanout takeover on x86 — separate work, tracked independently.
- Changes to how shell / dashboard surfaces work — they're already external; no refactor needed.
