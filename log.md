# Development Log

## 2026-04-14

Brought the x86 target up on QEMU's q35 machine with `virtio-gpu-pci`, so the same desktop that runs on aarch64/HVF now also runs on x86 — the display path goes through virtio under the hood (via OVMF's GOP driver).

ACPI/PCI/virtio-GPU scaffolding already in the tree needed three real fixes to boot cleanly:

- ACPI XSDT entries sit after a 36-byte header, so the u64 array is 4-byte-aligned. Walking it with an aligned `read_volatile` panicked on the first dereference; switched RSDT and XSDT walks to `read_unaligned`.
- Virtio PCI capabilities use byte-sized fields at arbitrary offsets. The existing `read_u16(cap_ptr + 3) >> 8` trick was reading the BAR byte instead of `cfg_type`, so every vendor cap was being parsed as if it were the wrong type. All PCI config reads now go through `read_unaligned`, and `cfg_type` / `cap_next` are read as `u8` directly.
- Virtio-gpu-pci on q35 places its 64-bit BAR at `0xC000004000` — **above** the kernel's 4 GiB identity map and in a **different PML4 slot** (slot 1) than the one `paging::init()` populated. Added `paging::map_mmio_1gib` to identity-map a 1 GiB supervisor MMIO window on demand, and extended `create_user_page_table` to mirror every populated kernel PDPT slot (and every non-zero PML4 slot) into each user PML4 so syscalls can touch MMIO on the caller's CR3.

`SYS_PRESENT_RECT` now lands the kernel-owned virtio-GPU handle behind a syscall. The x86 compositor calls `user_present_rect(x, y, w, h)` after drawing. Today it's a no-op because `set_virtio_gpu` is never called; once the driver takes the scanout over, the same call site transfers the dirty rect and flushes without a copy.

Taking the scanout over from OVMF is **deferred**. `device reset` + `resource_create_2d` + `attach_backing` + `set_scanout` all succeed on the wire, but the display stays on the pre-reset frame. Wants a dedicated session with virtio spec reading or QMP/trace-level debugging to see what the device is actually doing. The GOP path through OVMF's virtio resource is the working default in the meantime.

Live verification on `x86_64 + q35 + virtio-gpu-pci`:

- boots cleanly through ACPI/PCI discovery, virtio-GPU BAR mapping, scheduler start
- 7 tasks running as before (kbd, mouse, compositor, shell, dashboard, chiptune)
- desktop, menu bar, taskbar, and workspace pills render at 1280×800
- no page faults, no panics

Drafted the roadmap for the next structural step — extracting the menu bar, taskbar, and stats overlay out of the compositor into their own ring-3 tasks with double-buffered surfaces and event-driven redraws. Recorded in [Chrome as Services](decisions/chrome-as-services.md). Phase 1 (taskbar) is the smallest element and will validate the commit-surface IPC protocol before the harder chrome elements follow.

## 2026-04-05

Seeded the project wiki with initial architecture documentation covering:

- Kernel overview, boot flow, IPC design, performance targets
- aarch64 port plan and phased approach
- Design philosophy and anti-features

Seven concurrent tasks running with a graphical shell on x86_64/UEFI. The aarch64 port is next.

## 2026-04-12

Promoted Apple Silicon / `aarch64` + `HVF` to the primary demo path and added coarse live metrics on the ARM dashboard/compositor path:

- frame time
- present time
- input-to-photon
- IPC round-trip
- scheduler wake latency

First coarse serial-captured measurements from the live ARM boot on QEMU `virt` + `HVF`:

- frame time: roughly **450-520 ms**
- present time: roughly **0.7-6 ms** with occasional larger spikes
- IPC round-trip: roughly **6.1 ms**
- scheduler wake: roughly **0.99 ms** with occasional ~5 ms spikes
- input-to-photon: roughly **1.2-1.7 s** for injected serial keystrokes

Boot note:

- The earlier `Image type X64 can't be loaded on AARCH64 UEFI system.` line from edk2 was misleading firmware noise during boot-option probing, not an actual FreshOS load failure. The AArch64 EFI image still boots correctly on the `HVF` path.

Compositor phase breakdown from the same live ARM boot:

- background/content redraw: roughly **355-425 ms**
- window frames + blits: roughly **19-37 ms**
- menu/stats/taskbar chrome: roughly **49-68 ms**
- display present: roughly **0.7-6 ms**

The split stayed broadly the same after injecting shell input, with input-to-photon samples around **1.3-1.8 s**.

Interpretation:

- The dominant cost is still software frame production, not display present.
- The biggest single cost is the background/content redraw pass, not the window blits.
- `virtio-gpu` present is comparatively cheap in this path.
- Wake latency is close to the 1 ms scheduler quantum in the common case.
- The measured input-to-photon figure is bad, but consistent with the current frame time.

Follow-up after caching the desktop layer, throttling dashboard refresh, and making the compositor redraw only dirty regions:

- steady compositor frame time: roughly **43-48 ms**
- background/content redraw: roughly **6.8-12 ms**
- window frames + blits: roughly **13-18 ms**
- menu/stats/taskbar chrome: roughly **12.2 ms**
- display present: roughly **0.4-5.6 ms**
- input-to-photon: roughly **81.7 ms** for an injected shell keystroke

Interpretation:

- The architectural change worked. The compositor is no longer dominated by full-scene background redraw.
- The current cost is now split much more evenly across cached background restore, window composition, chrome draw, and present.
- FreshOS is still not at the intended latency target, but it moved from "seconds" to "tens of milliseconds" without changing the fundamental UI model.

Follow-up after adding per-surface damage tracking, cached window decorations, and active-window sub-rect copies:

- idle compositor work on the ARM/HVF path now sits around **12.3 ms** per update, with `bg=0`, `win=0`, and almost all cost in the menu/stats refresh path
- injected shell keystroke input-to-photon fell to roughly **8.2 ms**
- the shell damage path was small enough that the coarse `windows_us` metric rounded to **0 us** in sampled frames

Interpretation:

- The shell edit path is now close to the stated latency ambition.
- The remaining steady-state cost is mostly chrome redraw and present for the always-visible overlay, not content composition.
- The next bottleneck is the compositor's own HUD cadence, not full-window repainting.

Follow-up after decoupling the compositor HUD cadence from the offscreen dashboard surface and logging compositor redraw rate directly:

- idle compositor redraw rate on the shell workspace now sits at **1 redraw/second**, with **0 content redraws/second** in the steady state
- the sampled idle redraw is now chrome-only, typically around **19-25 ms** in the `ui` phase, with `bg=0` and `win=0`
- an injected shell keystroke still measured at roughly **7.4 ms** input-to-photon
- the compositor reported **3 redraws** in the second containing that keystroke, with only **1 content redraw**

Interpretation:

- The remaining idle churn was not the HUD itself; it was the compositor reacting to periodic dashboard commits while that window was inactive behind the shell.
- Ignoring inactive dashboard surface commits fixed the steady-state redraw problem without hurting the interactive shell path.
- The next cost to attack is the 1 Hz chrome pass itself if lower idle overhead matters, but it is no longer a blocker for perceived interactivity.

Follow-up after caching the menu bar and stats overlay base layers so only dynamic values redraw:

- idle shell-workspace redraw rate remains **1 redraw/second**, with **0 content redraws/second**
- the idle chrome pass now typically sits around **0.4-0.7 ms** in `ui`, with occasional low-single-digit millisecond spikes
- the previous ~**19-25 ms** idle redraw cost is gone; larger remaining frame spikes on this path are now dominated by `present`, not chrome
- an injected shell keystroke still measured at roughly **8.6 ms** input-to-photon

Interpretation:

- The expensive part of the old 1 Hz chrome pass was repeatedly redrawing static AA text and panel backgrounds, not just presenting the rects.
- Caching the static HUD layers moved the compositor into a state where the shell workspace is mostly idle between real content changes.
- The next performance ceiling on the ARM/HVF path is increasingly the display/present side and QEMU jitter rather than software chrome drawing.

Follow-up after introducing an ESP-loaded external `init` ELF on the ARM/HVF path:

- the kernel now reads `\EFI\FreshOS\INIT.ELF` before `ExitBootServices`, relocates it into RAM, and starts it as the first non-idle task
- the external `init` process now launches the built-in keyboard, compositor, shell, dashboard, and IPC probe services through a tiny kernel ABI rather than having `main.rs` spawn them directly
- live serial output confirms the handoff:
  - `Boot init: ... bytes from ESP`
  - `Init ELF loaded: base=... entry=...`
  - `[init] starting services`
  - `[init] services launched`
- injected shell input remained in the same range at roughly **9.1 ms** input-to-photon on the live ARM boot

Interpretation:

- FreshOS now has a real external bootstrap binary, not just a statically linked launch policy hidden in the kernel entry point.
- This is not yet a full userspace process model: `init` is external, but the services it launches are still built into the kernel image.
- The next honest step toward a working OS is to replace one of those built-ins with an externally loaded ELF service using the same path.

Follow-up after replacing the built-in IPC probe `pong` task with an ESP-loaded external `PONG.ELF` on the ARM/HVF path:

- the kernel now reads `\EFI\FreshOS\PONG.ELF` before `ExitBootServices`, relocates it into RAM, marks it executable, and registers its entry point for `init` to launch
- `SERVICE_PONG` now prefers the external ELF path and falls back to the built-in probe task only if `PONG.ELF` is absent or fails to load
- live serial output confirms the external service path is active:
  - `Boot pong: ... bytes from ESP`
  - `Pong ELF loaded: base=... entry=...`
  - `[probe] pong ext`
- the ping side kept running over the existing real IPC channels, with RTT samples still in the same coarse range on this path
- an injected shell keystroke after the service swap still produced a photon sample of roughly **16.0 ms**
- `run-arm.sh` now uses `rustup run nightly cargo ...` explicitly so the freestanding `aarch64-unknown-none` binaries build correctly from the normal demo script

Interpretation:

- FreshOS now has one service that is genuinely loaded and launched as a separate binary, not just an external bootstrap binary that immediately starts built-in functions.
- This is still an EL1 accelerated prototype path under HVF, so it is not the final isolation story, but the boot, relocation, ABI, and service-start policy are now proving the right shape.
- The next meaningful OS milestone is to do the same thing for a more consequential service, ideally one that exercises filesystem or restart/supervision behavior rather than a benchmark loop.

Follow-up after adding a supervised external `PULSE.ELF` service and real task exit/restart on the ARM/HVF path:

- the ARM task scheduler now supports reclaiming and reusing exited task slots instead of leaving `SYS_EXIT` tasks parked forever
- the kernel service ABI now exposes `exit_now`, and `init` periodically reconciles a supervised `pulse` service through the existing `spawn_builtin` interface
- the kernel now preloads `\EFI\FreshOS\PULSE.ELF` and `SERVICE_PULSE` prefers that external entry point with a built-in fallback
- live serial output confirms the lifecycle loop:
  - `Pulse ELF loaded: base=... entry=...`
  - `[pulse] ext start`
  - `[pulse] ext exit`
  - `task 8 exited`
  - `[init] service pulse exited (task 8)`
  - a fresh `task 8 @ ... stack ...` line immediately after, proving restart
- after fixing stack reclamation, the restarted service reused the same stack range (`0x40b65000..0x40b69000`) across repeated exits instead of walking upward through memory

Interpretation:

- FreshOS now has the first honest supervision loop: an external service can start, exit, and be restarted by `init` without rebooting the whole system.
- This is still cooperative exit, not crash containment; synchronous exceptions still panic the kernel on this ARM/HVF path.
- The next meaningful step is to move from “restart after explicit exit” to “restart after contained fault,” or to apply the same supervision path to a more consequential service such as a disk-backed userspace binary.

Follow-up after adding contained restart for synchronous ARM task faults and a supervised external `FAULT.ELF`:

- the aarch64 exception vector now routes current-EL synchronous exceptions through a containment path instead of treating every scheduled-task fault as a whole-kernel panic
- a new external `FAULT.ELF` service now starts, logs a couple of beats, executes `brk #0`, and is then terminated and made restartable through the same `init` supervision path as `pulse`
- live serial output confirmed the contained-fault loop:
  - `Fault ELF loaded: base=... entry=...`
  - `[fault] ext start`
  - `[fault] ext crash`
  - `*** TASK FAULT ***`
  - `Task: 9`
  - `Action: terminate faulting task`
  - `[init] service fault exited (task 9)`
  - a fresh `task 9 @ ...` line after that, proving restart without reboot
- the rest of the system stayed alive while this was happening:
  - IPC RTT samples continued
  - compositor metrics continued
  - an injected shell keystroke still measured roughly **9.4 ms** input-to-photon during the crash/restart loop

Interpretation:

- FreshOS now has both sides of the lifecycle story on the ARM/HVF path: supervised restart after explicit exit and supervised restart after a contained synchronous task fault.
- This is still not full privilege isolation. Tasks are still running in EL1 on this accelerated path, so containment is currently “scheduler task containment” rather than final hardware-enforced userspace isolation.
- The next honest milestone is no longer lifecycle wiring; it is either disk-backed external services or true fault containment with real EL0 separation.

## 2026-04-13

Follow-up after pushing one external service onto a real EL0 path under aarch64/HVF:

- `FAULT.ELF` now runs as a real EL0 task using the existing AArch64 `SVC` path for `yield` and debug logging
- the lower-EL synchronous exception vector now contains and terminates non-SVC EL0 faults instead of panicking the whole kernel
- the fault service is loaded into a dedicated **2 MiB-aligned** reserved region before the scheduler starts, with a pre-granted user stack in the same region
- task stack reclamation is now deferred until the kernel has actually switched off the dying task's stack, avoiding reuse of live stack memory during restart

Live serial output on QEMU `virt` + `HVF` confirmed the full loop:

- `Fault ELF loaded: base=0x40400000 ... ustack=0x405fc000 region=0x40400000..0x40600000`
- `[init] starting services`
- `[fault] el0 start`
- `[fault] ext beat 1`
- `[fault] ext beat 2`
- `[fault] el0 crash`
- `*** EL0 TASK FAULT ***`
- `[init] service fault exited (task 9)`
- a fresh `task 9 @ 0x40411c6c, ustack 0x405fc000, kstack ...` line after that, repeated across multiple restart cycles without taking `init`, `pong`, or the compositor down

Live behavior during that loop:

- external `init` stayed alive
- external `pong` stayed alive
- external `pulse` kept being restarted
- IPC RTT samples kept streaming, typically around **7.1 ms**
- compositor metrics kept streaming while the EL0 fault service repeatedly crashed and restarted

Interpretation:

- FreshOS now has a real EL0 proof path on Apple Silicon: userspace entry, `SVC` syscalls, lower-EL fault containment, and supervised restart are all working together on the ARM/HVF route.
- The dedicated 2 MiB region matters. Earlier attempts that granted EL0 access inside the same shared L2 block as `init` caused collateral instruction-abort faults; isolating the EL0 service into its own block removed that regression.
- This is still **shared `TTBR0`**, not per-process address-space isolation. The Apple Silicon path now proves EL0 execution and restart mechanics, but not the final microkernel memory-isolation story.

Follow-up after starting Milestone 1's generic service-runtime work:

- the kernel now owns a single service descriptor table and exports both service enumeration and service status to `init`
- service status now carries:
  - current task id
  - running state
  - restart count
  - exit count
  - last exit reason
- `init` now discovers services from the descriptor table instead of hardcoding its own service id list, and supervised restarts are driven by observed exits rather than by blind periodic spawn calls
- the first bring-up exposed a boot bug on the ARM path: the memory map snapshot for the frame allocator was being taken **before** the ESP service binaries were loaded, so those later UEFI pool allocations could still be recycled as "free" frames
- moving the memory-map capture to after the ESP file loads fixed the regression and restored clean external-`init` boot

Live serial output after that fix confirmed:

- external `init` still launched the desktop services through the generic runtime path
- repeated `pulse` clean exits still produced `[init] service pulse exited (..., reason=clean)` and restarts
- repeated EL0 `fault` crashes still produced `[init] service fault exited (..., reason=fault)` and restarts
- the rest of the system stayed alive while this happened

Interpretation:

- FreshOS now has the first reusable service-runtime slice instead of a special-case `init` loop.
- The boot ordering fix matters beyond this milestone: once service binaries are treated as first-class boot inputs, the allocator and memory-map snapshot have to agree on what firmware memory is still in use.

Follow-up after adding the first shell-facing service control surface:

- the ARM shell now has a real command path instead of only echoing typed characters
- it supports:
  - `services`
  - `ps`
  - `restart <name>`
  - `help`
- shell command output is mirrored to serial so the same path can be verified in headless QEMU runs
- the command implementation is backed by the generic service runtime:
  - service enumeration
  - live service status snapshots
  - manual restart/start requests for named services

Live serial verification on `aarch64 + HVF` confirmed:

- `services` printed the current service table with running state, task id, restart count, exit count, and last exit reason
- `restart pulse` reported `pulse already running`
- the desktop, compositor, `pong`, and the supervised `pulse`/`fault` services all stayed alive while those shell commands ran

Interpretation:

- FreshOS now has the first user-visible service inspection/control path, even though the shell itself is still a built-in task.
- This is enough to support the next obvious step: a real `ps`/`services` workflow in userland once the shell is externalized, using the same runtime model instead of inventing a second one.
