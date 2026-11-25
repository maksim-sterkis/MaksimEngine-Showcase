# Phase 5 benchmarks, RR and GI handoff

November 6 continuation: read [the Nsight and cost-quality follow-up](phase5_nsight_followup.md) for the newer checkpoint. It records the profiling setup, layout experiments, sampled full-resolution static and motion quality evidence and the four-candidate default. The November 1 settings and measurements below are historical.

The four-candidate default passed independent few-/many-light reference checks, Native/DLAA jitter checks, functional motion/resize cases and a full-resolution quantitative motion audit. All eleven CPU/offline suites passed. The Release executable is `build/nsight-release/VK_game_engine.exe`; the main `build/` cache is Debug. Sustained DLAA performance remains open, with measured thermal clock drift. The user resumed work after the previous approximately 10%-remaining stopping point; consult the follow-up for the current optimization checkpoint.

The retained November 6 emission cache reuses existing light-record lanes and removes a dependent ReSTIR material read. It preserves all 144 sampled HDR hashes and passes both Basic and ray-corrected Release GPU suites. A 6,000-frame within-process DLAA comparison supports approximately 4% lower lighting time and 2.4% lower total GPU time on the heavy fixture; these gains are separate from the earlier four-candidate budget improvement. Sequential sustained runs still fall from 161 to 123 FPS as graphics clocks decline under thermal limiting. A second selected-light reuse experiment was rejected after failing to establish a useful gain. Both normal Debug and profiling Release executables were rebuilt to match the shared production shaders.

Next work: confirm the actual cooling arrangement (NVML reports 0% fan speed under load; no hardware settings were changed), establish representative clocks for sustained measurements, and target the remaining fused initial/temporal and neighbor-surface/shadow costs. A compact cached-surface representation is a possible experiment, not an implemented or validated improvement. Keep the independent reference and motion gates, preserve the full-contribution target and avoid claiming a universal scene-quality proof. Phase 5 remains open; the 200 FPS DLAA target is not achieved. Do not start RR/GI or a release as a substitute for closing this performance work.

Checkpoint: November 1, 2026. Workspace: `C:\Users\maksi\VK_game_engine`.

## Read this first

The latest user request was to create this handoff file with `handoff` in its name, but not name it `handoff.md`. No GI/RR implementation, Nsight installation, commit, push or documentation synchronization was requested by that instruction.

The user is discussing performance and future lighting. Preserve the distinction between **measured results**, **engineering targets** and **speculative estimates**. In particular, do not carry forward a prediction that GI plus native RR will deliver 50 FPS: that came from unmeasured assumptions and was explicitly qualified afterward.

Follow `AGENTS.md` and [DOCS_SYNC_GUIDE.md](../../DOCS_SYNC_GUIDE.md). Never commit or push without a new explicit request. Never execute `.ps1` scripts without an explicit request. Keep private code/shaders/assets out of `showcase/`; do not sync this handoff automatically.

## Repository and build state

- Branch: **`phase-5`**, without the `codex/` prefix. The user explicitly requested this branch name.
- HEAD: **`68cbcc8`**, `docs: correct v0.2.1 release and packaging status`.
- There is substantial intentional uncommitted work. Do not reset, restore or overwrite it wholesale.
- `showcase/` was clean at this checkpoint. Tracked boulder assets and `settings.ini` were unchanged by the experiments and benchmarks.
- Some files restored after experiments still appear modified in `git status` despite an empty content diff. Inspect `git diff` rather than assuming every status entry contains a change.
- Main executable: [build/VK_game_engine.exe](../../../build/VK_game_engine.exe).
- Configuration: MinGW Makefiles, Release, `VKE_BUILD_TESTS=ON`, `VKE_ENABLE_DLSS=ON`, pinned Streamline 2.14.16, model K ordinary DLAA/DLSS. Vulkan SDK 1.4.350.0 glslc, shader target Vulkan 1.3.
- The latest executable includes the view/mode benchmark harness. Ordinary startup remains outside that opt-in path. All finite benchmark engine processes exited.

Normal build commands, from the workspace root:

```text
cmake --build build --target VK_game_engine -j8
ctest --test-dir build --output-on-failure
```

Do not casually reconfigure the main cache to Debug and leave it there. The earlier Debug cache snapshot is in ignored build output.

## Phase 5 assessment and implemented behavior

**Core ReSTIR DI is implemented and passes the tested correctness fixtures. Phase 5 remains in progress.** The roughly 200 FPS heavy-view DLAA goal and a convincing quality advantage at equal GPU cost remain open. No claim that every optimization has been exhausted is justified.

Implemented foundations include primitive/material ownership, opaque instances, emissive triangle lights, initial reservoirs, temporal/spatial reuse and fresh final visibility. Current lighting is direct only. There is **no GI, lighting denoiser, RR, SVGF, radiance clamp or visibility reuse**. Ordinary DLAA/DLSS has temporal reconstruction, which is distinct from adding a lighting denoiser. The boulder retains its directional-sun renderer.

Production sandbox settings used for the recent measurements: Basic correction, eight initial candidates, twenty-frame-equivalent history, one spatial proposal, radius 16, scalar/fused paths enabled, and fresh final shadows. Final spatial reservoirs again feed next-frame history by default. F5 offers a temporal-only comparison. Policy changes invalidate incompatible history, and buffer ownership avoids a full-resolution history copy.

The restored history passed stricter CPU and independent seeded GPU trajectory checks. In the many-light fixture, it has approximately 6.4x lower raw RGB MSE and 20.6x lower luminance MSE than MC1 at the same at-most-one final-shadow budget. **That is not an equal-GPU-cost comparison.** MC8 still wins some raw per-frame RGB error comparisons.

See [history follow-up](phase5_history_followup.md) and [current roadmap](../../roadmap.md).

## Rejected optimization experiments

Two exact-emission cache candidates failed to establish a reliable speedup and were removed. A subsequent rough-dielectric importance-target proxy also passed quality/reference checks but was removed after inconclusive original-module performance comparisons.

The proxy appeared 2-3% faster than the full-target branch of its own larger shader. Comparing all six original modules against the candidate narrowed its favorable aggregate to about 1.5%, amid major timing drift. Original-module GPU time changed from 6.58 to 8.14 ms across the run sequence, while unchanged DLAA time rose from 1.68 to 2.25 ms. This does not establish a reliable proxy improvement or a hardware-counter diagnosis.

Production retains the original full-contribution luminance target. The six modules matched regenerated originals byte-for-byte and passed SPIR-V validation. The original fused initial/temporal SHA256 is `1FCDF0F5030EBD91BD41E863948C6BFFFE71FE0522DDBBDEC8D30CE87B2F1C4A`.

Retained improvements include arbitrary-positive-target CPU fixtures, material-level GPU audit regions, independent trajectory/time-window gates, history-policy reset checks, and test-only forced validation of the optimized Release binary. Candidate source/modules remain only in ignored build output, not ordinary shader paths. Do not reintroduce an experiment merely because its archived code exists.

Details: [target-proxy report](phase5_target_proxy.md).

## Measured reconstruction benchmarks

All numbers below are **actual measured FPS**, not inferred from inverse GPU timestamps. Output is 3440x1440 on the RTX 3090. F4 and F5 were drawn. Reflex On, no frame limit, Mailbox presentation with V-Sync-off request. Automatic LOD, static cameras/objects/lights, validation and atomic diagnostics disabled. Timing uses asynchronous timestamps without correctness readbacks. No builds or CPU suites overlapped timing.

Each cell is the rounded range of **two 600-frame average FPS measurements**, not instantaneous minimum/maximum FPS. Every case had 600/600 focused frames and 600 asynchronous samples. First warm-up: 1800 frames; later warm-ups: 360 frames. Repeating in reverse exposes order/thermal effects but does not eliminate them.

### First four-mode matrix: 40 accepted cases

| View | DLAA | DLSS Quality | Balanced | Performance |
| --- | ---: | ---: | ---: | ---: |
| Boulder | 290-370 | 339-429 | 347-436 | 362-453 |
| Sandbox exterior, 3 lights | 188-223 | 259-316 | 284-342 | 305-359 |
| Sandbox interior, 3 lights | 149-167 | 235-267 | 259-298 | 290-325 |
| Sandbox exterior, 64 lights | 175-189 | 246-267 | 269-289 | 294-308 |
| Sandbox interior, 64 lights | 132-138 | 211-218 | 240-244 | 272-274 |

Actual internal resolutions: DLAA 3440x1440, Quality 2293x960, Balanced 1995x835, Performance 1720x720. Runtime assertions verify active/effective reconstruction and scaled dimensions. These measurements show a substantial DLSS throughput benefit; they do not constitute a full visual-quality review.

### Later paired Native/DLAA matrix: 20 accepted cases

| View | Native | DLAA |
| --- | ---: | ---: |
| Boulder | 827-945 | 304-368 |
| Sandbox exterior, 3 lights | 341-367 | 196-222 |
| Sandbox interior, 3 lights | 225-241 | 155-168 |
| Sandbox exterior, 64 lights | 289-306 | 179-188 |
| Sandbox interior, 64 lights | 185-192 | 137-139 |

An additional earlier Native-only matrix had 10 accepted cases. Its heavy interior measured 198.156 and 203.330 FPS; do not blend those with the paired run into a precise DLAA overhead estimate.

Native is truly 3440x1440 internally, with NVIDIA reconstruction inactive and camera jitter disabled. Native also removes reconstruction-guide demand and changes Hi-Z recovery work. Therefore the entire Native/DLAA frame difference is not just one SDK dispatch.

Paired Native **lighting GPU time**:

| Sandbox view | Lighting ms |
| --- | ---: |
| Exterior, 3 lights | 2.13-2.30 |
| Interior, 3 lights | 3.45-3.71 |
| Exterior, 64 lights | 2.56-2.69 |
| Interior, 64 lights | 4.42-4.55 |

Heavy paired total GPU time: Native 4.78771/4.92803 ms; DLAA 6.70166/6.79530 ms. DLAA evaluation itself: 1.55088/1.58720 ms. Lighting is roughly 92% of the Native GPU frame there.

**Lighting timing includes deferred surface/material resolution, candidates, reuse, final shading and shadows. It is not isolated incremental ReSTIR overhead.** A matched conventional-lighting baseline is needed to quantify that overhead. The small boulder scene is not that baseline. These numbers are plausible absolute costs but do not demonstrate competitive optimization.

Sandbox exterior camera: (6,3,8), looking at (0,1,0). Interior: (0,1.7,3), yaw -90, pitch -10. Boulder: ordinary start at (3,1,0). Three-/64-light scene configurations contain different normal geometry; these matrices are not the earlier geometry-preserving light-count ablation.

## Thermal drift and interpretation

Driver telemetry explicitly reported `SW Thermal Slowdown: Active` during sustained measurements. This is evidence of thermal clock limiting, not a diagnosis of a fan/cooling fault or memory-junction temperature.

In-run samples from the four-mode matrix included 1725 MHz at 67.94 C (limiting inactive), then 1590 MHz at 77.72 C, 1395 MHz at 80.75 C and 1335 MHz at 81.62 C (limiting active). A paired Native/DLAA sample was 1515 MHz at 79.69 C, limiting active. Earlier long experiments also reached 1275/1080 MHz at roughly 82-83 C. Fan, power and clock settings were not modified.

A 210 MHz, 54.03 C, 9.35 W reading after the Native-only run was **idle telemetry**. Never use it as an in-frame benchmark clock. Most telemetry is sparse and cannot normalize every case's timing.

Avoid interpreting a few-percent shader gain across drifting runs as established. Use tighter interleaving and hardware profiling; keep sustained performance separate from brief cooler-run observations.

## GI and RR discussion: critical corrections

The user asked how future GI and RR would integrate, their cost, and whether at least 100 FPS is plausible. These were discussions, not implementation authorization.

### ReSTIR GI

A proposed first scope is one diffuse indirect bounce: trace from the visible surface, evaluate the secondary hit's material and direct lighting, form a separate GI reservoir, apply GI-specific temporal/spatial reuse and reconnection visibility, then combine indirect and direct estimators without double-counting paths. Existing BLAS/TLAS, primary identity and motion/history infrastructure can help, but secondary ray-hit material evaluation and GI reservoir/history data are new work. Do not reuse the DI reservoir format unchanged or assume offscreen hits can simply read screen-space DI.

An unmeasured **3-6 ms additional GI allowance** was discussed for full-resolution, one-path-per-pixel, modest-reuse, one-bounce diffuse lighting on the small opaque scene. This is a low-confidence engineering guess, not a benchmark or a bound. Multiple bounces, complex materials and an early unoptimized implementation can cost more. Half-width/half-height GI was discussed as a possible later tradeoff, not an implemented feature.

At a measured Native baseline near 190 FPS (about 5.3 ms/frame), adding hypothetical GI costs of 3/4/6 ms yields about 120/108/88 FPS **without RR**. Thus 100 FPS for DI plus limited GI is plausible, not guaranteed.

### Ray Reconstruction 4.5

RR is intended as the combined denoising/reconstruction evaluation replacing ordinary SR/DLAA evaluation, not an additional pass followed by another SR pass. We currently have no lighting denoiser to remove for extra savings. Keep ReSTIR sample history independent of RR reconstruction history.

The official NVIDIA RR integration guide downloaded during research is [the August 2026 PDF](../../../build/dlss-rr-integration-guide-estimate.pdf). Public source: [NVIDIA/DLSS RR Integration Guide](https://github.com/NVIDIA/DLSS/blob/main/doc/DLSS-RR%20Integration%20Guide.pdf).

- Page 11, Table 3, Preset F/Default, RTX 3080 Ti: 3.97 ms at 2560x1440 output and 8.75 ms at 3840x2160 output.
- **These timing tables explicitly measure Performance mode.** They do not measure native/DLAA mode, and there is no RTX 3090 row.
- Multiplying 3.97 by 3440/2560 gives roughly 5.3 ms, but only as an output-scaled **Performance-mode proxy on comparable hardware**. It cannot establish native-resolution RR cost.
- Page 13 confirms RR supports DLAA mode. Native-mode support does not establish its execution time. Page 25 describes the DLAA preset with matching input/output sizes.
- Do not automatically multiply Performance cost by four either: the distribution of input-/output-resolution work is not established by the table.
- NVIDIA's claim of similar RR 4.5 performance compares it with previous RR models, not ordinary model K SR/DLAA.

The agent initially floated a 5-8 ms native RR estimate based too heavily on Performance-mode data. **The user caught the mode mismatch; that estimate was withdrawn.** The agent later supplied an even broader **6-12 ms native RR planning allowance**, explicitly unmeasured. That later range also lacks a native timing anchor; it must not become a measured fact, a reliable prediction or a statement that native RR necessarily costs that much.

An illustrative midpoint of 8 ms RR + 4 ms GI + 0.3 ms extra guides, replacing 1.57 ms DLAA in a roughly 7.25 ms current frame, produced about 18 ms / 56 FPS. **The user was alarmed by this. The subsequent clarification was that this is speculative arithmetic including RR, not evidence that DI plus GI alone will run at 50 FPS.** Do not repeat 50-55 FPS as the expected project outcome.

Correct accounting is `future frame ~= current frame - current AA/SR evaluation + added GI + total RR evaluation + additional input preparation`, with qualifications for scheduling, jitter/guide changes and thermal conditions. The unknown inputs need measurement.

### External ReSTIR DI comparison

The [2021 optimized DI paper](https://cwyman.org/papers/hpg21_rearchitectingReSTIR.pdf), Table 1 / section 8, reports RTX 3090 direct-lighting costs of roughly 1.0-5.1 ms at 1080p across different scenes, with standard settings including 32 candidates and two visibility rays per pixel. It excludes G-buffer creation, denoising and post-processing. The pixel ratio to 3440x1440 is 2.3889. Extrapolated costs of roughly 2.4-12.2 ms are illustrative lighting-only estimates, not measured ultrawide full-engine FPS and not an equal-quality comparison with our renderer.

## Nsight and sensible next work

Nsight was researched but **no Nsight profiling captures have been made**. The relevant app is Nsight Graphics; its `ngfx.exe` CLI can launch a target, capture automatically and export metrics with `--auto-export`. Ordinary profiling need not involve clicking the user's screen. Installation, performance-counter access and export contents still need verification on this machine. Do not promise every investigation is fully headless or install software implicitly.

Official references: [GPU Trace CLI](https://docs.nvidia.com/nsight-graphics/UserGuide/gpu-trace-overview.html#generate-gpu-trace-capture-from-a-command-line), [Shader Profiler](https://docs.nvidia.com/nsight-graphics/UserGuide/shader-profiler.html), [ReSTIR GI integration](https://github.com/NVIDIA-RTX/RTXDI/blob/main/Doc/RestirGI.md), [Streamline RR integration](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS_RR.md).

Subject to the next user instruction, useful work would be a minimal, correctly tagged RR probe measuring **actual native and upscaled modes separately** on the 3090 before forecasting RR+GI FPS. Verify the compatible signed runtime/model, capability checks and required inputs. Separately, hardware-profile the initial/temporal and spatial/shading costs to distinguish memory stalls, register/occupancy limits, ray work and scheduling. These remain hypotheses until measured. Inspect profiler clock-control defaults and record conditions.

GI is not currently implemented and remains outside the documented Phase 5.1 scope. Phase 5.1 currently covers RR evaluation and benchmark-scene/integration work. Do not silently expand it into GI implementation.

## Verification and artifacts

Last renderer-cleanup checkpoint: all eleven CPU/offline suites passed; six affected SPIR-V modules passed validation; hidden Basic, ray-traced reference, Native/DLAA jitter and independent three-policy heavy-interior feedback audits passed. Forced Release validation used `VKE_FORCE_VALIDATION=1` and modern `VK_LAYER_VALIDATE_SYNC=1`, without reported validation messages/hazards. Jitter cycle donor drift remained about 0.000529 rows at 640x360. This does not prove noise-free output or general unbiasedness.

Those correctness suites preceded the subsequent test-only view benchmark extensions. The later harness was rebuilt successfully and exercised through 40 four-mode + 10 Native-only + 20 paired Native/DLAA cases. Do not describe the CPU suites as freshly rerun after that extension. The extension changes opt-in test behavior, not ordinary renderer behavior.

Important uncommitted code:

- [restir_full.hpp](../../../src/renderer/restir_full.hpp) and [restir_full.cpp](../../../src/renderer/restir_full.cpp): restored spatial-feedback history and optional comparison.
- [device.hpp](../../../src/renderer/device.hpp), [main.cpp](../../../demo/main.cpp): opt-in Release validation and audit entry points.
- [restir_feedback_gpu_smoke.hpp](../../../tests/restir_feedback_gpu_smoke.hpp): independent trajectory/material/region audits; NaN for unavailable regional error metrics.
- [restir_full_tests.cpp](../../../tests/restir_full_tests.cpp), [restir_full_gpu_smoke.hpp](../../../tests/restir_full_gpu_smoke.hpp), [restir_jitter_gpu_smoke.hpp](../../../tests/restir_jitter_gpu_smoke.hpp): stronger regressions.
- [restir_profile.hpp](../../../tests/restir_profile.hpp) and new [reconstruction_view_profile.hpp](../../../tests/reconstruction_view_profile.hpp): finite timestamp-only benchmarks.

Ignored build artifacts are local evidence and may not exist in a fresh clone:

- [Four-mode report](../../../build/reconstruction-view-matrix.md), `reconstruction-view-matrix.{csv,log}`, build log and thermal CSV.
- [Native/paired report](../../../build/reconstruction-view-native.md), `reconstruction-view-native.{csv,log}` and `reconstruction-view-native-dlaa.{csv,log}`, corresponding build logs and thermal CSVs.
- `restir-target-clean-final-{build,cpu,basic,reference,jitter,feedback}.log`, comparison/trajectory CSVs.
- Rejected proxy archives: `restir-target-original/`, `restir-target-proxy/`, `restir-target-experiment-source/`; original-module A/B logs and thermal CSV. Earlier cache archives also remain ignored.
- Official RR guide and page-11 rendered table image in `build/`.

Benchmark opt-ins: `VKE_RESTIR_PROFILE=1`, `VKE_PROFILE_VIEW_MATRIX=1`; add **either** `VKE_PROFILE_NATIVE_ONLY=1` or `VKE_PROFILE_NATIVE_DLAA_AB=1` for those modes. Shared flags: `VKE_FIXED_PRESENT_PROBE=1`, `VKE_PROBE_BORDERLESS=1`, `VKE_PROBE_FOCUS=1`, `VKE_REFLEX_MODE=1`, `VKE_TEST_VSYNC=0`. Clear validation and other suite flags for performance runs. Clear Native-specific flags for the original four-mode matrix.

## User preferences and boundaries

- The user wants actual implementation/results and concise milestone paragraphs with findings, not repeated permission questions or unsupported reassurance.
- Performance testing should use a real visible 3440x1440 window. Do not open tiny visible readback tests that appear to run at less than 1 FPS. Expensive correctness readbacks stay hidden.
- Do not open an IDE or take over the mouse unnecessarily. GPU launches were authorized for the earlier benchmark tasks; the handoff request itself requires no new launch.
- No lighting denoiser should be added implicitly. RR is a future explicit denoising/reconstruction decision; GI is real indirect lighting and must not be confused with DI shadows.
- Respect pauses for gaming. Prior usage percentages/reset windows are stale; check live account limits if relevant rather than repeating old numbers. The earlier 10%-remaining threshold was later described by the user as a warning, with permission to exceed on that occasion.
- The requested roughly 200 FPS heavy-view DLAA goal is a stretch target, not a proven hardware entitlement. At roughly 135 FPS, reaching 200 requires approximately 32% less overall frame time. Around 100 FPS for limited DI+GI remains a plausible target whose feasibility is unmeasured.

No commits, pushes, docs synchronization, PowerShell scripts or new renderer implementation are part of creating this handoff.
