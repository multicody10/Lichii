# Emulator Issue Workflow

Use this workflow for every game compatibility, performance, audio, rendering, stability, or hardware regression investigation. The goal is a correct and usable emulator, not a better benchmark number or a green CI badge.

## Definition of done

A change is not a fix merely because it compiles, passes unit tests, raises reported FPS, or removes one log message. A fix is complete only when:

1. The original failure is reproduced or the available evidence is strong enough to identify its root cause.
2. The change addresses that cause, with a focused regression test where practical.
3. The affected game is tested in a real emulator session on the target hardware and build.
4. All relevant dimensions are rechecked: correctness, game speed, frame pacing, audio/video sync, stability, and performance.
5. No new regression is introduced in the comparison scenarios.
6. The result is recorded as verified, partially verified, or unverified. Never describe a compile-only result as a gameplay fix.

The owner currently tests on Windows with an Intel i7-7700, 16 GB RAM and NVIDIA GTX 1070 8 GB. Treat this as the primary target unless the owner supplies a different test machine. Do not assume AMD-specific optimizations help Pascal NVIDIA hardware.

## 1. Establish the baseline

Before changing code:

- Record exact repository commit, build tag, OS, CPU, GPU, driver, Vulkan version, RAM, emulator settings, game title ID, game version/build ID, and launch method.
- Identify the affected game session in `eden_log.txt`. Each `Booting game: <program id>` begins a session; the build is on the first line.
- Record the scene, action, duration, and repeatability. Define a short reproduction route with a clear start and end.
- Capture a baseline log, screenshot, and measured performance sample. Use the live debug server when available; do not start, stop, or take control of the owner's emulator.
- Compare against the latest relevant upstream changes, issues, pull requests, and known forks before implementing a local fix. Read diffs and verify whether a proposed fix actually applies to this codebase.
- Preserve a known-good build and user data. Do not overwrite saves, settings, shader caches, or the owner's working installation.

If the issue cannot be reproduced, state exactly what evidence is missing. Do not invent a root cause.

## 2. Classify symptoms independently

Never collapse these into a single "performance" issue. A game can render at high FPS and still be broken.

| Dimension | What to measure | Common causes to investigate |
|---|---|---|
| Rendering throughput | Presented FPS, GPU time, CPU time, per-thread time | Shader translation, GPU workload, CPU emulation, synchronization stalls |
| Frame pacing | Per-present intervals, variance, long-frame spikes | Queue depth, fences, scheduler stalls, vsync and swap interval |
| Game speed | Guest simulation time against wall clock; known animation/timer rates | Incorrect frame delta, guest clock scaling, timer conversion, frame pacing assumptions |
| Audio/video synchronization | Audio presentation timestamps against video presentation; offset and drift over time | Guest audio clock, resampling, buffering, callback scheduling, underruns, timestamp conversion, video pacing |
| Audio continuity | Callback cadence, queue occupancy, underruns/overruns, silence and discontinuities | Host audio backend, buffer sizing, producer/consumer races, starvation |
| Freezes and hangs | Stall duration, blocked thread, wait dependency, last progress marker | Fence deadlock, lock inversion, unsubmitted work, scheduler starvation, guest wait |
| Shader stutter | First-use versus warm-cache frame times, compile duration | Pipeline compilation, cache invalidation, synchronous translation |
| Rendering correctness | First incorrect pass, inputs, bindings, formats, shader hash | Shader lowering, descriptors, texture cache, synchronization, render state |
| Crashes | Exception, faulting module, native stack, preceding log | Invalid memory, device loss, unsupported instruction, lifetime/race bugs |

For audio/video issues, distinguish:
- **Constant offset:** audio is consistently early or late.
- **Drift:** synchronization gets progressively worse over time.
- **Discontinuity:** sudden jump after a stall, scene transition, or cutscene.
- **Starvation:** gaps, crackles, repeated samples, or silence.
- **Wrong game speed:** both audio and video may be internally synchronized but run too slowly or quickly.

A peak-volume log only proves that audio output is active. It does not measure synchronization, latency, queue depth, or drift.

## 3. Collect evidence before selecting a fix

Use the existing live debug server and diagnostics documented in `AGENTS.md` and `docs/nx2/debug-mcp.md`.

- Start with `status`, `log_tail`, and a screenshot. Record FPS over a defined interval, not a single instantaneous reading.
- For performance, correlate present intervals with CPU/GPU thread timing. Do not infer a CPU bottleneck from total CPU utilization or a GPU bottleneck from utilization alone.
- For rendering defects, capture one affected frame and find the first incorrect output. Correlate the live state with frame-dump shader hashes and RenderDoc event/resource state. A visually improved replacement shader is not proof that the original pipeline is correct.
- For hangs, identify which thread is blocked, what it waits on, and which thread can satisfy that wait. Capture a useful trace or dump before changing wait behavior.
- For audio synchronization, inspect the audio renderer/backend path and guest timing source. If existing logs do not expose enough evidence, add narrowly scoped, opt-in telemetry before changing buffering or clock logic.

### Required audio telemetry when the existing evidence is insufficient

Do not claim an audio fix based only on listening impressions or output peak levels. Add or use diagnostics that can correlate, on a common monotonic timeline:

- guest audio sample position and guest audio timestamp;
- audio callback/presentation timestamp and submitted sample count;
- audio queue/buffer occupancy over time;
- underrun and overrun counts;
- resampler input/output rates and accumulated sample correction;
- video present timestamp/frame index and guest frame or simulation timestamp.

Log at a bounded cadence or expose rolling counters through the debug server. Keep tracing opt-in and low overhead. Report measured A/V offset, drift rate, and discontinuities across a repeatable test interval. Do not add a permanent high-volume per-sample log.

## 4. Form and test a root-cause hypothesis

For each candidate fix, write down:

- observed symptom and exact reproduction;
- evidence supporting the suspected subsystem;
- expected mechanism by which the change resolves it;
- alternative explanations and likely side effects;
- a falsifiable prediction for the next test.

Change one causal variable at a time. Prefer a minimal reversible experiment over a broad rewrite. Use runtime A/B switches only during investigation; remove experiment switches before merging a proven fix. Do not ship game-specific patches without exact title/build/hook guards and tests proving that unsupported builds remain untouched.

Do not tune audio buffer sizes to hide a clock error. Do not change game speed to hide audio drift. Do not raise FPS caps to hide frame pacing problems. Fix the responsible clock, queue, scheduler, or synchronization contract.

## 5. Implement with regression coverage

- Follow `docs/Coding.md`, existing architecture, and upstream conventions.
- Add focused unit or integration tests for timing conversions, clock progression, queue boundaries, fence state transitions, shader lowering, or other deterministic logic.
- Include negative tests for guards and unsupported game/build identifiers.
- Keep diagnostics opt-in, bounded, and safe for normal gameplay.
- Avoid unrelated refactors and dependency changes.
- Run formatting, targeted tests, and the production build/test workflow. Inspect warnings and failures rather than suppressing them.

## 6. Validate in the real emulator

A real game session on the owner's target hardware is required for claims about gameplay behavior. CI cannot establish audio sync, frame pacing, visual correctness, or long-session stability.

Use the same build, game version, settings, scene, route, and duration for before/after comparisons. Test:

1. The exact original reproduction.
2. A second scene or transition likely to exercise the same subsystem.
3. A warm-cache run and, where relevant, a first-use shader run.
4. A sustained session long enough to expose audio drift, queue growth, or delayed hangs.
5. A control game or path to detect broad regressions.

Record separate outcomes for:
- average and 1% low FPS, if available;
- frame-time distribution and long stalls;
- game speed against wall clock;
- A/V offset at start and end, drift, and audio underruns;
- freeze count and maximum stall duration;
- rendering defects and crash status.

For A/V validation, listen to identifiable synchronization events (for example, a cutscene with visible impacts or dialogue) and compare them at multiple points in the run. Listening is useful but should complement timestamp/queue telemetry, not replace it.

If the owner must perform the final gameplay test, provide one clearly named build, exact settings, a short reproduction route, and the specific observations/logs needed. Do not make them test a collection of unrelated builds or vague "try it now" instructions.

## 7. Ship only what the evidence supports

- Run the applicable CI checks and inspect the exact commit's status.
- Package the exact tested commit. Include source revision, build configuration, install instructions, and known limitations.
- Never label a release as fixed when the game has only been compiled or unit-tested.
- Keep experimental builds clearly marked. Preserve a rollback path and known-good release.
- Update the issue/work log with root cause, changed files, tests, hardware, game/build, before/after measurements, and remaining failures.

Use these status labels precisely:

- **Confirmed:** reproduced before, corrected after, and regression checks passed on target hardware.
- **Partially confirmed:** one or more symptoms improved, but other dimensions or scenarios remain unresolved.
- **Build verified:** compiles and automated tests pass; gameplay behavior is not yet validated.
- **Unverified:** evidence is insufficient to establish the result.
- **Regressed:** a previously working behavior became worse; revert or isolate the change before proceeding.

## Current Ocarina of Time priority

For program ID `0400486024DEC000`, keep the user's observed symptoms separate:

- The Sharkiix fork reportedly achieved high FPS on the user's machine, while audio remained desynchronized. This is evidence that throughput and A/V synchronization must be investigated independently, not proof that its audio implementation is the cause.
- The local timing correction is intended to address game speed below target FPS. It is not, by itself, proof of correct audio clocking or A/V synchronization.
- The Vulkan deferred-submission change targets a specific fence deadlock. It does not establish that all freezes are fixed.
- The user's target is a stable, playable emulator, not a high FPS counter. Prioritize correct speed, synchronized audio, consistent pacing, no recurring hangs, and correct rendering. Optimize throughput after correctness is measured, without regressing it.

The next OOT investigation should therefore establish a repeatable baseline on the user's GTX 1070, instrument the guest and host audio clocks/queues, and correlate audio presentation against video presents during both gameplay and cutscenes. Do not make another speculative timing or buffer adjustment until those measurements identify the failing clock or queue.
