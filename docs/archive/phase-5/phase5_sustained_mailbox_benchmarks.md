# Phase 5 sustained Mailbox benchmarks

November 12, 2026 · Phase 5 item 2

The packed ReSTIR DI build completed **30 focused runs across all five views**, totaling **226,201 measured frames** and **12.5 minutes of measurement windows**. The 64-light interior averaged **174.3–174.5 FPS in DLAA**, **250.9–251.1 FPS in DLSS Quality**, and **267.4–268.3 FPS in Native**. A separate 64-frame Nsight GPU trace of the DLAA interior was saved and exported.

## Test setup

- RTX 3090, NVIDIA driver 621.14, Intel Core i7-12700, and the optimized `build/nsight-release/VK_game_engine.exe` with 48-byte ReSTIR surfaces.
- The engine used its normal startup path and applied the saved borderless settings. All three sweeps logged Vulkan Mailbox, Reflex On, and an engine frame limit of `0`. The first restarted process was checked live and had `vke_sl_interposer.dll` loaded.
- Output was 3440×1440 in every case. DLAA and Native rendered at 3440×1440. DLSS Quality rendered at 2293×960. The sandbox used four initial ReSTIR candidates, Basic correction, one spatial proposal, recursive history, and fresh final shadows. Cameras, objects, and lights were fixed. F4/F5 timing overlays were enabled and per-pixel counters were off.
- Each mode ran the five views in forward and reverse order. DLAA and Quality used 30-second measurement windows. Native used 15-second windows. The harness excluded the 1,800-frame initial warm-up and the 360-frame warm-ups between cases. The windows were time-limited, so each heavy DLAA repeat captured about 5,230 frames.
- All accepted runs had full focus and one asynchronous GPU timing sample per measured frame. One Quality attempt and three Native attempts lost focus and were retried. FPS is measured engine-loop throughput. GPU times are per-run medians unless marked p99. [Benchmark harness](../../../tests/reconstruction_view_profile.hpp) · [DLAA log](../../../build/phase5-sustained-mailbox-20261112/dlaa.log) · [Quality log](../../../build/phase5-sustained-mailbox-20261112/quality.log) · [Native log](../../../build/phase5-sustained-mailbox-20261112/native.log)

## All five views

Ranges span the two accepted run averages or medians for each view and mode.

| View | DLAA FPS | DLAA GPU ms | DLSS Quality FPS | Quality GPU ms | Native FPS | Native GPU ms |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Boulder | 322.9–341.8 | 2.35–2.48 | 372.3–373.6 | 2.10–2.11 | 906.7–909.5 | 0.75–0.76 |
| Sandbox exterior, 3 lights | 234.5–241.8 | 3.41–3.50 | 306.3–308.9 | 2.60–2.60 | 344.3–441.2 | 1.71–1.73 |
| Sandbox interior, 3 lights | 195.3–198.7 | 4.40–4.50 | 280.9–281.6 | 2.87–2.87 | 297.9–312.2 | 2.59–2.60 |
| Sandbox exterior, 64 lights | 212.9–214.9 | 3.63–3.66 | 275.9–276.3 | 2.73–2.74 | 371.7–382.7 | 1.84–1.84 |
| Sandbox interior, 64 lights | **174.3–174.5** | **4.82–4.84** | **250.9–251.1** | **3.07–3.07** | **267.4–268.3** | **2.85–2.85** |

[DLAA CSV](../../../build/phase5-sustained-mailbox-20261112/dlaa.csv) · [Quality CSV](../../../build/phase5-sustained-mailbox-20261112/quality.csv) · [Native CSV](../../../build/phase5-sustained-mailbox-20261112/native.csv)

## 64-light interior and thermal behavior

| Mode | Frames, two runs | Lighting median | Reconstruction median | GPU p99 | Loop p99 | Thermal slowdown samples |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| DLAA | 10,463 | 2.64 ms | 1.46–1.47 ms | 5.79 ms | 7.39–7.41 ms | 238/238 |
| DLSS Quality | 15,061 | 1.18–1.19 ms | 1.39–1.40 ms | 3.67 ms | 5.39–5.41 ms | 239/239 |
| Native | 8,035 | 2.48 ms | — | 3.53–3.54 ms | 5.49–5.53 ms | 119/119 |

- The heavy DLAA view stayed below **5.0 ms median GPU time** in both 30-second runs. Its loop rate averaged about **174 FPS**, and GPU p99 was **5.79 ms**. The earlier **194.5–194.7 FPS** runs measured 1,800 frames each, and the **198.6 FPS** run measured 600 frames with a cooler end-of-run hotspot reading of 97.75 °C. [Earlier packed-build benchmark](../../roadmap.md#closure-summary-2026-11-14-v030) · [Sustained DLAA CSV](../../../build/phase5-sustained-mailbox-20261112/dlaa.csv)
- The heavy DLAA runs peaked at **105.0–105.3 °C hotspot**. Their end-of-run readings were **1,635–1,650 MHz**, about **329 W**, and **74.8–74.9 °C core**. Across all modes, the highest sampled hotspot was **105.5 °C** and the software thermal slowdown flag was set in **2,812 of 2,977** sampled readings. The contribution of cooling to the FPS difference and the current condition of the previously repaired fan remain unverified. [DLAA CSV](../../../build/phase5-sustained-mailbox-20261112/dlaa.csv) · [Quality CSV](../../../build/phase5-sustained-mailbox-20261112/quality.csv) · [Native CSV](../../../build/phase5-sustained-mailbox-20261112/native.csv)
- The 3-light Native exterior had the widest repeat gap, **344.3 vs. 441.2 FPS**, despite similar median GPU times of **1.71 vs. 1.73 ms**. Loop p99 was **16.08 vs. 3.10 ms**. The recorded timing shows a loop-side difference and leaves the source of those delays undetermined. [Native CSV](../../../build/phase5-sustained-mailbox-20261112/native.csv)

## Mailbox and the apparent 200 FPS ceiling

The initial launch used `VKE_FIXED_PRESENT_PROBE` and `VKE_PROBE_BORDERLESS`. The harness rejected the Boulder DLAA window because only 56 of 5,893 frames were focused over roughly 30 seconds. The normal startup path applies the saved settings and recreates the swapchain. With that path and Mailbox active, Boulder averaged **341.8 FPS** on the first DLAA run and **322.9 FPS** on the reverse run. The cause of the apparent ceiling in the rejected attempt remains undetermined. [Initial attempt log](../../../build/phase5-sustained-packed-20261112/dlaa.log) · [Restarted DLAA log](../../../build/phase5-sustained-mailbox-20261112/dlaa.log) · [Signed Streamline workaround history](../../rendering/dlss_integration.md#historical-canonical-name-200-fps-investigation)

## Nsight GPU trace

Nsight Graphics 2026.4.0 captured 64 frames of the 64-light interior at 3440×1440 DLAA with four candidates and unaltered GPU clocks. The selected metric set exported pass timings with the Real-Time Shader Profiler disabled. Instruction and stall attribution remain unverified for the packed build. Use the uninstrumented CSVs for throughput comparisons. [Trace file](../../../build/phase5-sustained-mailbox-20261112/nsight/VK_game_engine_2026_11_12_20_18_36.ngfx-gputrace) · [Capture configuration](../../../build/phase5-sustained-mailbox-20261112/nsight/BASE_UNLOCKED/REPRO_INFO.xls)

| GPU scope | Median | Min–max |
| --- | ---: | ---: |
| GPU frame time | 5.387 ms | 5.303–5.713 ms |
| ReSTIR resolve, initial sampling, and temporal reuse | 1.547 ms | 1.523–1.594 ms |
| ReSTIR spatial classification and fast shading | 0.872 ms | 0.869–0.877 ms |
| ReSTIR spatial reuse and shading | 0.179 ms | 0.178–0.183 ms |
| DLAA evaluation | 1.410 ms | 1.403–1.436 ms |

The fused resolve, initial sampling, and temporal reuse dispatch was the largest labeled ReSTIR scope. DLAA evaluation was close behind. The two spatial scopes together accounted for about 1.05 ms based on their medians. [Frame export](../../../build/phase5-sustained-mailbox-20261112/nsight/BASE_UNLOCKED/FRAME.xls) · [Scope export](../../../build/phase5-sustained-mailbox-20261112/nsight/BASE_UNLOCKED/D3DPERF_EVENTS.xls)

## Display incident

After the first trace was saved, a second capture started for shader-level sampling. The monitor briefly went dark and remained dim while that capture was running. The game and profiler were stopped before the second trace exported, and the GPU returned to idle. The dimming persisted after the user power-cycled the monitor. The cause remains unresolved. [Second capture log](../../../build/phase5-sustained-mailbox-20261112/nsight-shader.log)

Item 2 is complete for the requested run lengths. Broader scene and stress-motion qualification and equal-GPU-cost sampling comparisons remain Phase 5 work. [Phase 5 roadmap](../../roadmap.md#phase-5-emissive-lighting-foundation--restir-di)
