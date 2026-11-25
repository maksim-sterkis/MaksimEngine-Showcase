# Ray Reconstruction performance investigation

November 22, 2026. Native 3440×1440 RR on this RTX 3090 spends approximately 18 ms executing the NVIDIA SDK's reconstruction workload. GPU timestamps and an Nsight Graphics trace agree. The 2–3 ms goal and 5 ms ceiling remain unmet.

## Measurement conditions

The fixed, visible, focused `Sandbox64Interior` fixture uses 64 emitters, four ReSTIR candidates, Basic spatial reuse with one neighbor, retained temporal history, and the production RR final-shading settings. Camera and scene transforms remain fixed. RR averages four final-lighting estimates per pixel; ordinary DLAA uses one. Total frame and lighting timings therefore compare the current mode defaults rather than identical lighting workloads.

The test-enabled MinGW Release build uses `-O3 -DNDEBUG`, no Vulkan validation and no atomic diagnostic counters. Each uninstrumented run warms up for 1,800 frames and reports medians over 600 measured frames, all focused. Output stays 3440×1440. RTX 3090 driver is 621.14, Streamline is 2.14.16 and NGX is 310.9.16. The signed RR DLL hashes match the ordinary Release build.

| Mode | Input | Reconstruction ms | Lighting ms | Total GPU ms |
| --- | --- | ---: | ---: | ---: |
| RR DLAA, preset F | 3440×1440 | 18.493 | 4.911 | 24.076 |
| Ordinary DLAA, preset K | 3440×1440 | 1.422 | 2.878 | 4.826 |
| RR Quality, preset F | 2293×960 | 8.626 | 2.233 | 11.488 |
| RR Balanced, preset F | 1995×835 | 7.509 | 1.868 | 9.872 |
| RR Performance, preset F | 1720×720 | 5.996 | 1.463 | 7.949 |
| RR Ultra Performance, preset F | 1147×480 | 3.353 | 0.600 | 4.373 |

Quality uses 44.4% of the native input pixels and 46.6% of the RR time. This supports a large input-resolution-dependent SDK workload. Quality is an explicit upscaling mode and does not meet the native DLAA requirement.

An explicit native preset comparison, in execution order, produced these direct SDK medians:

| Requested RR preset | SDK evaluation ms | End-of-run clock MHz | Sampled windows |
| --- | ---: | ---: | ---: |
| F, first | 18.3706 | 1575 | 56 |
| E | 19.7601 | 1575 | 60 |
| D | 19.9178 | 1485 | 61 |
| F, repeat | 18.8903 | 1530 | 59 |

All four runs retained native input/output and 600 focused timing samples. D and E were diagnostic requests, with no change to the normal F selection. None produced a low-cost native path in these conditions. GPU clock variation across runs limits precise model ranking; the large absolute cost is not explained solely by introducing preset F. These tests measure timing, not comparative D/E image quality.

## Nsight evidence

Nsight Graphics 2026.4 captured 32 frames beginning at frame 2,000, with Ampere GA10x Top-Level Triage counters and Real-Time Shader Profiler enabled. GPU clocks were explicitly left **unaltered**. Analysis used the native `ngfx-mcp.exe` tools.

Every frame contains exactly one `nv.ngx.dlssd.Evaluate` marker. Its duration is 17.608–19.026 ms, with mean 18.206 ms and median 18.087 ms. Each evaluation contains 50 NVIDIA CUDA kernels, represented by 18 grouped action ranges, plus a grouped pair of buffer fills.

| Component within the RR marker | Mean ms |
| --- | ---: |
| NVIDIA kernel intervals | 18.189869 |
| Buffer-fill intervals | 0.002230 |
| Time outside those action intervals | 0.013994 |

Kernel intervals account for 99.91% of the marker, but can include GPU context preemption. Context residency independently places the engine's Vulkan context on the GPU for 98.39% of the combined RR marker time. Other-process residency overlaps RR by 0.078 ms per evaluation on average. These figures do not support a large CPU submission, copy or inter-kernel gap as the explanation for the measured cost.

In the first RR marker, 99.998% of the sampled warps are attributed to `NVIDIA SDK`. The four largest individually visible kernels take 2.742, 2.439, 2.078 and 1.641 ms, together occupying 47.98% of that RR marker. They are part of a single evaluation, rather than repeated full-frame reconstruction.

Scoped counters for that marker show:

| Counter | Result |
| --- | ---: |
| Tensor pipe active | 65.39% |
| VRAM throughput | 19.55% |
| L2 throughput | 32.35% |
| L2 sector hit rate | 89.81% |
| Active threads per warp | 31.77 / 32 |
| Compute warp occupancy | 19.61% |
| GPC frequency | 1551.87 MHz |

Four additional RR markers sampled across the capture show tensor activity of 65.60–66.87% and VRAM throughput of 19.25–19.46%. The four largest kernels show tensor activity of 64.27–86.42% while VRAM throughput is 16.09–24.18%.

The shader samples' largest categories are Math Pipe Throttle (35.04%), Long Scoreboard (18.49%) and Wait (17.36%). These are warp-state sample proportions, not additive percentages of wall time. Together with the counters, they indicate a compute-heavy tensor inference workload with dependency and memory-latency costs, rather than saturated external-memory bandwidth. Nsight reports zero MiB of demoted VRAM.

The protected NVIDIA shaders expose no source or instruction-level performance counters in this trace. These results cannot identify a particular neural-network layer, arithmetic precision, or an SDK kernel change that would achieve 5 ms.

## Clock telemetry and integration checks

During a repeated uninstrumented F run, GPU clocks measured 1560–1575 MHz via engine NVAPI/NVML telemetry and concurrent `nvidia-smi` polling. That run measured 18.3706 ms directly between the SDK evaluation timestamps and 18.3721 ms for the complete reconstruction slot. Engine work outside the SDK therefore accounts for only about 0.0015 ms in those medians. Even linear clock scaling to higher boost clocks would not bridge the gap to a 5 ms budget; native RR DLAA cost is bound by the SDK tensor inference workload itself.

Source and trace checks confirm one selected reconstruction feature per frame, native-sized RR DLAA input/output, fixed preset F, HDR enabled, alpha upscaling disabled, and packed normal/roughness. The selected RR feature replaces the ordinary SR evaluation. No simultaneous ordinary DLAA pass or steady-state feature recreation was found.

## Reproduction and saved evidence

Configure a Release build with `VKE_BUILD_TESTS=ON`, then set these process environment variables and launch its `VK_game_engine.exe` from the repository root:

```text
VKE_FIXED_PRESENT_PROBE=1
VKE_PROBE_BORDERLESS=1
VKE_PROBE_FOCUS=1
VKE_RESTIR_PROFILE=1
VKE_PROFILE_VIEW_MATRIX=1
VKE_PROFILE_TRACE_INTERIOR=1
VKE_PROFILE_TRACE_CANDIDATES=4
VKE_PROFILE_GPU_LABELS=1
VKE_PROFILE_VIEW_MODE=5
VKE_RECON_MODE=5
```

Mode 1 measures ordinary DLAA; modes 6–9 measure RR Quality, Balanced, Performance, and Ultra Performance. The fixture verifies effective mode, focus, dimensions and timing availability, and exits after the measurement. The log includes `sdkEvaluationMs` and hardware sample counts. Optional `VKE_PROFILE_RR_PRESET=D/E/F` requests an explicit model comparison only in this test build and fixed fixture. It never selects a model in response to failure. Invalid values stop the test. The normal build continues requesting F. NVIDIA documents the available RR presets in the pinned [SDK header](https://github.com/NVIDIA-RTX/Streamline/blob/v2.14.16/include/sl_dlss_d.h).

For Nsight, use `GPU Trace Profiler`, start after 2,000 frames, limit to 32 frames, select `Ampere GA10x` / `Top-Level Triage`, enable Real-Time Shader Profiler and explicitly choose `--set-gpu-clocks unaltered`. Capture/export pauses invalidate the instrumented run's elapsed FPS as a normal-play measurement. The uninstrumented table above is the throughput reference. Some periodic counter samples were merged due to high sampling rate, so the scoped counters are approximate.

Ignored local evidence is under `build/phase51-rr-nsight/`:

- `VK_game_engine_2026_11_22_18_36_28.ngfx-gputrace` and `cli.log` hold the capture and launch/export log.
- `uninstrumented-mode5.log`, `uninstrumented-mode1.log` and `uninstrumented-mode6.log` hold the initial focused runs.
- `preset-*.log` hold explicit native model comparisons and direct SDK telemetry.
- `rr-summary.json`, `rr-aggregate.json`, `rr-range-metrics.json`, `rr-repeat-metrics.json`, `rr-kernel-metrics.json`, `rr-sdk-profile.json`, `context-summary.json` and `capture-settings.json` hold native Nsight query results and calculated summaries.

The measured bottleneck is the SDK's native-resolution RR inference workload on this GPU/runtime. This investigation found no engine-side scheduling or buffer-preparation change that recovers the roughly 13 ms needed to reach the 5 ms ceiling. Lower input resolution changes the effective DLSS mode; another preset changes the selected model. Neither alters native preset F cost directly (use RR Quality/Balanced/Performance or independent Reconstruction scale presets instead).

## Memory footprint

Fresh-process `VKE_RR_MEMORY_AUDIT=1` runs on the same 3440×1440 `Sandbox64Interior` fixture (60 warm-up frames, `vkDeviceWaitIdle`, forced `VK_EXT_memory_budget` and Windows working-set queries) measured:

| Mode | Input | Process VRAM (MiB) | Temporal Targets (MiB) | RR Guides (MiB) | ReSTIR Buffers (MiB) | RAM Working Set (MiB) |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| Native | 3440×1440 | 2002.14 | 614.81 | 0.00 | 907.15 | 363.20 |
| DLAA, preset K | 3440×1440 | 2330.58 | 614.81 | 0.00 | 907.15 | 379.55 |
| DLSS Quality, preset K | 2293×960 | 1449.33 | 318.50 | 0.00 | 403.18 | 379.45 |
| RR DLAA, preset F | 3440×1440 | 3626.58 | 897.00 | 282.19 | 907.15 | 385.65 |
| RR Quality, preset F | 2293×960 | 2210.33 | 444.12 | 125.62 | 403.18 | 384.79 |
| RR Balanced, preset F | 1995×835 | 1946.27 | 357.34 | 95.70 | 305.14 | 385.82 |
| RR Performance, preset F | 1720×720 | 1724.14 | 284.91 | 70.59 | 226.88 | 385.71 |
| RR Ultra Performance, preset F | 1147×480 | 1404.45 | 171.62 | 31.50 | 100.93 | 385.56 |

At native 3440×1440, `RR DLAA` uses `1,296.00 MiB` more process VRAM than ordinary `DLAA` (`282.19 MiB` in engine-allocated RR guide images plus `~1,013.81 MiB` in Streamline/NGX RR workspace and model allocations). In an in-process switchback check (`VKE_RR_MEMORY_SWITCHBACK=1`), transitioning from `RR DLAA` (`3626.58 MiB`) to `Native` (`2549.95 MiB` after 30 frames) immediately freed all `282.19 MiB` of RR guide targets and reclaimed `1,076.63 MiB` total via `slFreeResources`, leaving `547.81 MiB` of persistent Streamline/NGX runtime allocations until process exit.

To reproduce a single mode: set `VKE_FIXED_PRESENT_PROBE=1`, `VKE_PROBE_HIDDEN=1`, `VKE_PROBE_BORDERLESS=1`, `VKE_RR_MEMORY_AUDIT=1`, and `VKE_RECON_MODE=<0..9>` (optionally adding `VKE_RR_MEMORY_SWITCHBACK=1`).
