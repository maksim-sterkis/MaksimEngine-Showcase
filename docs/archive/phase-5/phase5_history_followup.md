# Phase 5 history and sampling follow-up — 2026-11-01

The subsequent [rough-surface target experiment](phase5_target_proxy.md) passed material/regional reference checks but was removed after inconclusive original-module timing comparisons. Production retains this checkpoint's full-BRDF target and recursive history. The later tests add arbitrary-target CPU coverage, material-level GPU audits and forced validation of the optimized Release binary.

## Recursive history

The earlier GPU feedback brightening failure did not reproduce with the current Basic shader and full reference/motion suite. A read-only estimator review found no demonstrated Basic normalization defect. The prior tests did have gaps: Basic recursive feedback lacked CPU coverage, and a single GPU sequence with 64-frame uncertainty blocks did not establish independent observations.

CPU tests now exercise both history policies with Basic and ray-traced correction, static/changing support and visibility, unequal counts and history caps. Statistics use 2,048 independent trajectories and four time windows. Aggregate and time-window gates are mandatory for all cases; tolerances were not widened. CPU feedback has high variance in the tiny six-pixel fixture, reinforcing that correlated frames must not be counted as independent samples.

A new hidden GPU audit compares 16 independently seeded trajectories per policy against an independent 4096-spp reference. Each trajectory warms 64 frames and measures 256; seeds are paired across policies but distinct across trajectories. Whole-image, early/late and nonempty 4x4 tile RGB means are checked using between-trajectory standard errors. Geometry, camera, transforms, LOD, materials, lights and shadows are fixed. This is offline correctness analysis at 320x180 internal resolution, not a denoiser or performance benchmark.

Both few- and many-light fixtures passed. In the many-light fixture, final-spatial feedback aggregate RGB means were within approximately 0.13% of the reference. This is stronger evidence for the tested fixtures, not a proof of general dynamic unbiasedness or a mathematical identification of the old failure's cause.

Final spatial reservoirs are again retained as the default next-frame history, restoring repeated spatial propagation. No history copy is needed: final output already occupies the history buffer. The former temporal-only policy remains selectable in F5 and through the test-only `VKE_RESTIR_TEMPORAL_HISTORY_ONLY` override. A policy change invalidates incompatible history. The jitter audit reads the temporal intermediate rather than interpreting final spatial diagnostics as donor motion.

The feedback-enabled Basic full reference/motion/resize suite passed. The ray-traced reference suite and stationary Native/real-DLAA jitter suite passed under Debug validation as well, including cached-emission test mode. DLAA donor drift remained approximately 0.000529 rows per eight-frame cycle. Recorded Debug runs have no Vulkan validation/hazard messages. The production light sampling budget remains eight initial candidates, twenty-frame history, one regular Basic spatial proposal and fresh final visibility.

## Sampling-efficiency evidence

The three-policy many-light audit also compares conventional MC1. This matches the at-most-one final-shadow budget, **not total GPU cost**. Means for all methods pass the same reference gates.

| Policy | Raw per-frame RGB MSE | 256-frame-average RGB MSE | Raw luminance MSE | 256-frame-average luminance MSE |
| --- | ---: | ---: | ---: | ---: |
| Temporal-only history | 0.00219311 | 0.000110620 | 0.000395229 | 0.0000189456 |
| Final-spatial feedback | 0.00242272 | 0.0000405363 | 0.000552093 | 0.0000118223 |
| Conventional MC1 | 0.0156101 | 0.0000647767 | 0.0113988 | 0.0000472394 |

Feedback's raw RGB MSE is roughly 10% higher than temporal-only history, but its 256-frame-average MSE is about 63% lower. Against MC1, it has approximately 6.4x lower raw RGB error, 20.6x lower raw luminance error and 37% lower averaged RGB error in this fixture. The averages are validation artifacts; ordinary Native output does not average radiance.

MC8 still has lower per-frame RGB error in the existing reference fixtures. It evaluates up to eight final shadow samples rather than one. Thus the new comparison establishes useful sample efficiency, but does not establish a quality-per-millisecond advantage over MC8, denoised quality or the engine's 200 FPS goal. Raw stochastic noise and correlation remain visible considerations.

## First performance candidate

An emission-cache experiment stores immutable emissive RGB and the global texture slot in unused light-record components, retaining the 112-byte stride and all surface/material identities. World preparation explicitly transforms XYZ with homogeneous W=1 and preserves metadata. Frame-dependent intensity remains separate, including previous-frame values.

The original candidate avoids a dependent material-table lookup but rereads the light record for cached fields. A second candidate carries the fields through existing light reconstruction. Neither reduces proposal PDFs, BRDF, texture sampling, shadows or sample counts. **Both were rejected and removed from normal rendering** after performance measurements; their source patch and shader snapshot remain only in ignored build output.

CPU cache/affine/snapshot tests pass. The first candidate passed Debug reference/motion suites both off and on, with all 144 sampled stored-HDR hashes matching bitwise. These hashes cover stored HDR values, not proof of every intermediate FP32 operation. All cached GPU light factors and texture slots are also checked against registered materials.

The first candidate's two time-balanced Release sweeps averaged 3.112 ms initial/temporal with caching off versus 3.294 ms on, and 6.999 versus 7.192 ms total GPU time: approximately 5.8% and 2.8% slower respectively. Each sweep used eight cases, all 68 instances/128 triangle lights fixed at 3440x1440 DLAA, 600 measured frames per case, 1800 initial warm-up frames and 180 on subsequent cases. Timing runs use no per-frame image/buffer readbacks.

The direct-fetch revision was essentially flat in its first sweep (7.084 ms off versus 7.107 on), but that run is excluded from adoption decisions because CPU tests overlapped its warm-up. Its clean repeat measured 7.895 ms off versus 7.819 on, only about 1% better, with changing GPU clocks. This does not establish a reliable gain, so it was not adopted. No cache format changes, extra light-record loads or larger private light-sample structs remain in production shaders.

## Sustained thermal limit

Live driver diagnostics during the sustained benchmark explicitly reported **SW Thermal Slowdown: Active**. At one sample, graphics clocks were 1275 MHz, GPU temperature 81.94 C and instantaneous power about 258 W against a 350 W limit; software power capping and hardware slowdown were inactive. A later sample reported 1080 MHz at 82.97 C and 247 W, again with software thermal limiting active and software power limiting inactive. This is stronger evidence than the earlier observation of clock drift alone.

[NVIDIA's nvidia-smi documentation](https://docs.nvidia.com/deploy/nvidia-smi/index.html) identifies this event reason as software thermal clock capping. The readings do not identify a cooling fault, memory-junction temperature or how much FPS cooling changes would recover. They do not explain every view-dependent shader cost. No fan, clock or power settings were changed. Sustained thermal limiting complicates small A/B gains and prevents claiming that a short cold-run FPS is a sustained minimum.

Raw telemetry is in `build/restir-cache-thermal-diagnostic.txt` and `build/restir-cache-thermal.csv`. Cache timing logs are `restir-emission-cache-ab-{first,repeat}.log` and `restir-emission-cache-v2-{first,repeat}.log`. The experiment patch is `build/restir-emission-cache-experiment.patch`; its Debug executable and matching loose shaders are archived in `build/restir-cache-experiment-snapshot/` as development evidence, not a standalone runnable distribution.

## Clean production-policy checkpoint

After removing both cache candidates, all eleven Release CPU/offline suites and the final Basic reference/motion/resize GPU suite pass. The fused deferred shader returns to the exact original material-table module hash (`1FCDF0F5030EBD91BD41E863948C6BFFFE71FE0522DDBBDEC8D30CE87B2F1C4A`). No failed cache branch, format or extra light-sample fields remain in normal rendering.

A final eight-case full-resolution history A/B retains all 68 instances, 128 triangle lights, DLAA, eight candidates and fresh shadows. Every accepted case has 600 focused frames and 600 asynchronous samples, with balanced policy ordering and no simultaneous CPU suites. Mean case medians were 6.753 ms total GPU / 3.007 ms initial-temporal for temporal-only history and 6.786 / 3.039 ms for final-spatial feedback. Loop FPS case averages were 133.5 and 132.9. The approximately 0.5% total-GPU difference does not establish a reliable performance change under clock drift; restoration is a data-flow/sampling improvement, not an FPS claim. Log: `build/restir-history-policy-ab-release.log`.

The current build is Release, tests ON and DLSS ON, using the pinned SDK. Final logs are `build/restir-history-final-release-build.log`, `restir-history-final-release-cpu.log` and `restir-history-final-release-gpu.{log,csv}`. Phase 5 remains open for equal-cost quality/performance qualification and the approximately 200 FPS heavy-view goal. The next shader optimization should target repeated material/BRDF importance evaluation with explicit reference and quality checks; small improvements cannot be qualified by comparing cold and thermally limited runs.

## Artifacts and limits

Ignored `build/` artifacts include `restir-feedback-basic-pre.{log,csv}`, `restir-feedback-independent-{few,many}.{log,csv}`, `restir-feedback-three-policy-many.{log,csv}`, `restir-cache-{off,on}-debug.{log,csv}`, `restir-cache-{off,on}-parity.csv`, `restir-feedback-reference-debug.{log,csv}`, `restir-feedback-jitter-debug.log`, `restir-history-debug-cpu.log` and `restir-history-strict-cpu.log`.

The Debug configuration is preserved in `build/CMakeCache.history-debug.txt`; the cache-experiment Debug executable requires its archived matching shader snapshot. The main `build/VK_game_engine.exe` is rebuilt as MinGW Release with tests and DLSS enabled. After rejecting the cache changes, production shader source returns to its original material-table path. Packaging, long memory soak, broader hardware and full benchmark-scene validation remain pending. No GI, denoiser, radiance clamp, visibility reuse, commits, pushes or documentation synchronization is part of this follow-up.
