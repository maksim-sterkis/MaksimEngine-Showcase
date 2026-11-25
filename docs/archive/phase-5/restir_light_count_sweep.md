# Controlled ReSTIR DI light-count comparison — 2026-10-31

This is a diagnostic benchmark, not a rendering optimization. No FPS gain is claimed from the harness change. Ordinary rendering retains the existing emitter list; filtering is enabled only with both `VKE_RESTIR_PROFILE` and `VKE_PROFILE_LIGHT_SWEEP` in test builds.

The subsequent [shader diagnosis](restir_shader_diagnosis.md) uses controlled ablations to narrow the initial/temporal cost to varied light access and target evaluation, with explicit limits on what the measurements establish.

## Controls

All 64 small emitter meshes and four receiver/room instances remain present for every case. Raster geometry, ray-tracing geometry, materials, transforms, camera, LOD and per-emitter intensity remain fixed. Only membership in the sampled triangle-light list changes. A deterministic permutation (`grid index * 37 mod 64`) spreads nested emitter subsets across the grid. The sweep uses 1, 3, 16, 32 and 64 emitters, then repeats in reverse order. Each emitter has two triangle lights.

Inactive emitters retain their visible emissive material but are excluded from illumination sampling. This deliberately diagnostic configuration is not a physically consistent scene. Total illuminating power increases with active count; one small active emitter makes the room dark. No brightness normalization, GI or denoiser is introduced.

Every case warms 120 frames and measures 360 frames. Runtime assertions check 68 instances, unchanged geometry/motion keys, expected triangle-light counts, full 3440x1440 output/internal resolution, active DLAA, shadows, ReSTIR and focus. All accepted cases recorded 360/360 focused frames and 360 asynchronous GPU samples. Eight initial candidates, Basic correction, one spatial proposal, history limit 20 and both production pass fusions stay enabled. F4/F5 are drawn; atomic diagnostics and image/buffer captures are disabled. Timing collection adds no per-frame GPU waits.

Configuration: MinGW Release, tests ON, DLSS ON, pinned Streamline 2.14.16 runtime, RTX 3090, Mailbox, Reflex On. Starting camera is `(6,3,8)`; interior is `(0,1.7,3)`, yaw -90, pitch -10. Reverse repeats check ordering drift; they do not lock GPU clocks.

## Observations

Ranges below retain both forward and reverse results. The confirmation interior is a separate process with the same settings.

| Active emitters | Starting-view FPS | Interior FPS | Confirmation interior FPS |
| ---: | ---: | ---: | ---: |
| 1 | 192–198 | 155–164 | 147–161 |
| 3 | 189–198 | 152–160 | 144–153 |
| 16 | 185–190 | 148–155 | 139–144 |
| 32 | 183–187 | 145–151 | 137–141 |
| 64 | 178–180 | 139–140 | 133–133 |

The interior initial/material/temporal pass rises from **1.88–2.20 ms at one emitter** to **2.86–3.06 ms at 64** across both processes. Spatial/final shading remains approximately **1.53–1.66 ms** across all counts. Light preparation is zero in the settled measurements. The combined culling/build category is about 0.10–0.13 ms. Thus changing emitter geometry is not necessary for the observed scaling penalty, and extra final-shadow traversal is not its main measured location.

The measurements do not yet distinguish light-record locality, material/target-evaluation branches, temporal behavior or register/memory stalls inside the fused initial/temporal pass. Fixed candidate counts do not imply fixed instruction execution or memory cost. A shader profiler or further isolated ablations are needed before selecting an optimization.

CPU work is usually about 0.7–0.8 ms; occasional larger values include Reflex scheduling sleep. Acquisition/fence waits dominate CPU wall time in the heavy cases. CPU and GPU samples are collected asynchronously; these medians must not be added as serial frame stages.

There is material clock/order drift. Confirmation telemetry recorded active-render graphics clocks falling from roughly **1545 to 1365 MHz**, memory at 9751 MHz, GPU utilization about 93–96%, temperature approximately 78–81 C and power approximately 274–305 W. DLAA timing also drifted upward. These readings establish changing clocks, not a specific thermal/power throttling cause. The light-count trend persists in both sweep directions and both interior processes, but exact milliseconds attributable solely to count are not established. No hardware clocks, power limits or fan settings were changed.

## Reproduction and artifacts

Set these environment variables, then run `./build/VK_game_engine.exe` from the repository root:

```text
VKE_FIXED_PRESENT_PROBE=1
VKE_PROBE_BORDERLESS=1
VKE_PROBE_FOCUS=1
VKE_RESTIR_PROFILE=1
VKE_PROFILE_LIGHT_SWEEP=1
VKE_RECON_MODE=2
VKE_REFLEX_MODE=1
```

Add `VKE_PROFILE_INTERIOR=1` for the interior camera. Choose no other profile A/B flag. The finite benchmark closes its full-size window after ten cases and does not save user settings.

Ignored `build/` artifacts:

- `restir-sweep-interior-full.log`, `restir-sweep-default-full.log`, `restir-sweep-interior-confirm.log`: all three ten-case sweeps passed.
- `restir-sweep-results.csv`: thirty accepted case records, with run identifiers.
- `restir-sweep-confirm-gpu.csv`: polled GPU telemetry; startup/shutdown rows are not rendering samples.
- `restir-sweep-cpu.log`: all eleven CPU/offline suites passed.
- `restir-light-sweep-build.log`: successful Release engine build.

The first decorated-window attempt was rejected for a 3440x1421 framebuffer (`restir-sweep-interior.log`) and contributes no accepted result. An initial background telemetry capture produced no flushed rows; only the confirmation telemetry above is used.

Tracked boulder assets remain unchanged. No commits, pushes, PowerShell scripts, documentation synchronization or production sampling changes were made. This narrows the next investigation; it does not complete the Phase 5 performance target.
