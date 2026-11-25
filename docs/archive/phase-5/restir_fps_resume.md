# ReSTIR FPS resume — 2026-10-31

See the [2026-11-01 follow-up](phase5_history_followup.md) for restored recursive history, independently seeded reference comparisons, rejected cache optimizations and confirmed driver thermal clock limiting. The timings and pre-spatial history policy below describe this older checkpoint.

This follows the [89 FPS checkpoint](restir_fps.md). No GI, lighting denoiser, reservoir outlier filter, RR or SVGF was added. Eight candidates, twenty-frame-equivalent history, one Basic spatial proposal and fresh final shadows remain the real-time policy. Recursive spatial feedback remains separate open work.

The subsequent [controlled light-count sweep](restir_light_count_sweep.md) holds geometry fixed across 1–64 sampled emitters. It confirms scaling in initial/temporal work and records GPU clock drift; the short measurements below are historical checkpoints, not sustained FPS guarantees.

## Implemented

- A dedicated scalar Basic kernel handles pixels with accepted temporal history. A separately dispatched general kernel retains the eight-proposal disocclusion boost. Proposal RNG, geometric/material gates, source counts and normalization are preserved; CPU closed-form cases cover empty/zero-weight sources and unequal counts.
- GPU reservoirs use **32 bytes instead of 64**. Index, count, age, both barycentric coordinates, target and normalized weight retain their full 32 bits. Duplicated light identity is resolved through the stable light table; streaming sums remain private temporaries. Integer diagnostics share reserved flag bits, using common C++/GLSL packing constants and round-trip tests.
- Cached surfaces use **80 bytes instead of 96**. Albedo, normals, material parameters and world position remain FP32. Instance bits share the albedo record; previous position/depth are reconstructed from existing matrices. Static poses use current position/normals exactly, based on a CPU transform-equality flag. Moving/nonuniformly scaled cases retain full previous transforms. CPU checks bound the sandbox round trip to 0.02 mm and 0.005 pixels; large-world/extreme-scale precision is not established.
- Spatial reuse and final shading share a kernel, keeping the freshly selected reservoir/surface in registers. Basic material/initial resolve and temporal reuse also share a kernel, avoiding intermediate initial-reservoir stores and current-surface reloads. Reference modes and ablations retain separate kernels. All final visibility queries remain fresh.
- Stable pixels exit the disocclusion pass using one metadata word. Background output already cleared by deferred resolve is not redundantly rewritten. Post-submission ownership rotation still retains **pre-spatial** history.

Full-DI storage at 3440x1440 falls from **1,902,306,048 to 1,268,245,248 bytes**, saving **634,060,800 bytes**. This excludes existing HDR/geometry/SDK targets; the previous lazy legacy-array saving remains in place.

## Measurements

RTX 3090; pinned Streamline 2.14.16/NGX runtime; Mailbox; full **3440x1440 output and internal resolution**, real DLAA and Reflex On, shadows enabled, F4/F5 drawn, counters disabled. Cameras/objects/lights are locked. Every accepted case warms 60 frames and measures 180, with 180/180 focused frames and 180 asynchronous GPU samples. There are no per-frame image/buffer readbacks in these timing runs.

| Build/view | Observed loop FPS | GPU frame | Lighting |
| --- | ---: | ---: | ---: |
| Debug interior, previous generic path | 87.5 | 10.16 ms | 8.45 ms |
| Debug interior, scalar kernel | 102–103 | 8.56 ms | 6.86 ms |
| Debug interior, compact records | 128–129 | 6.72 ms | 5.00 ms |
| Debug interior, spatial/shade fused | 140–141 | 5.98 ms | 4.24 ms |
| Debug interior, all current optimizations | 155–158 | 5.28 ms | 3.53 ms |
| Release starting view, three emitters | **200.0 / 200.1 repeat** | 4.00–4.01 ms | 2.19 ms |
| Release interior, three emitters | **167.9 / 168.4 repeat** | 5.34–5.35 ms | 3.55–3.56 ms |
| Release interior, 64 emitters | **141.8 / 145.0 repeat** | 6.19–6.22 ms | 4.28–4.30 ms |

Debug runs include synchronization validation. Release disables engine validation, uses the same shader binaries, and improves host-side execution; shader hashes match. The starting-view MC8 comparison also runs around 200 FPS, so the observed loop rate is not an isolated GPU-throughput limit. These short measurements do not guarantee a minimum frame rate across arbitrary views. The approximately 200 FPS target is met in the starting view, **not throughout the heavier interiors**. Remaining interior cost is concentrated in the fused material/temporal pass, spatial/final pass and DLAA.

Logs under `build/`: `restir-scalar-3440.log`, `restir-packed-3440.log`, `restir-fused-3440.log`, `restir-combined-debug-3440.log`, `restir-release-3440.log`, `restir-release-default-3440.log`, `restir-release-many-3440.log`.

## Verification and current build

- All eleven CPU/offline suites pass in Debug and Release, including packing, affine reconstruction/static-pose flags, source ownership, normalization, temporal math and compiler fixtures.
- Debug synchronization-validation runs pass both Basic and accurate ray-traced 4096-spp/512-frame-per-method reference matrices, motion, light-off/on, topology, cuts, LOD, area and resize checks. Basic's largest aggregate receiver RGB mean difference is about **0.207%**, within the existing confidence/numerical gates; this does not prove zero local error or noise-free output.
- Legacy initial-only allocation transitions pass after the compact-format changes. The combined hidden stationary Native/DLAA jitter probe passes with approximately **0.000529 rows per eight-frame cycle**. No Vulkan validation/hazard messages occur in the recorded Debug runs. Six affected SPIR-V modules validate offline.
- The Release functional GPU suite passes reference/reuse, motion, light response, mode, Hi-Z, topology, cuts, LOD, area and resize cases (`build/restir-release-basic-gpu.log`). Its engine validation is disabled; GPU validation evidence comes from the Debug runs, not from an absence of Release messages.

`build/VK_game_engine.exe` is now **Release**, with tests and DLSS support retained. `build/VK_game_engine-debug-validated.exe` preserves the validated Debug executable; both use the matching loose shader files. `build/CMakeCache.Debug.performance.txt` records the previous configuration. To return to Debug, configure `cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug` and rebuild. No release package was produced.

Sources and tracked boulder assets remain intact. No commits, pushes, PowerShell scripts or documentation synchronization were performed. Raw stochastic noise and recursive-history/convergence work remain open.
