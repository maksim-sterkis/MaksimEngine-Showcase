# DLAA and DLSS: model K baseline

## Supported scope

Model K (DLSS 4) is implemented for DLAA, DLSS Quality, Balanced and Performance. Native is the first-launch rendering default. Settings retain the requested rendering mode separately from the effective rendering mode, and Apply/Accept/Cancel retain their existing behavior. F4 reports effective mode, dimensions and key GPU timings; F5 holds runtime/fallback details and interactive diagnostics. Presentation defaults to Mailbox; explicit settings and legacy VSync choices are retained.

The unused DLSS 4.5 M/L integration was removed by design: no advanced model selector, M/L presets, model setting, model fallback branches or M/L test cases remain. K is fixed for DLAA and DLSS Super Resolution. Ray Reconstruction uses preset F. Legacy INI `dlssModel` keys are ignored, so a previously saved M/L choice no longer forces Native. SDK binaries remain unmodified; the bundle itself may contain other models.

Phase 4.1 closed with model K DLAA and Super Resolution. Phase 5 added the opaque scene and emissive foundation and ReSTIR DI (`v0.3.0`). Phase 5.1 (`v0.3.1`, 2026-11-25) completed DLSS 4.5 Ray Reconstruction evaluation below, native Vulkan [NVIDIA NRD](nrd_integration.md) (`RELAX` and `REBLUR`), the two-room sandbox with directional sunlight, Cornell Box, Intel New Sponza (2022), output-resolution emitter resolve, the Win32 Mailbox presentation pacing fix, independent DLSS reconstruction scale (`Off (100%)`..`50%`) + EASU/RCAS spatial upscaling (`Task 8`), and the `v0.3.1` portable distribution. Standalone TAA, Frame Generation, dynamic resolution, and multi-bounce GI remain future work. Custom SVGF is deferred in favor of NRD. [Roadmap](../roadmap.md)

## Experimental DLSS 4.5 Ray Reconstruction

Build the Windows x64 engine with the pinned Streamline SDK, then launch `./build/VK_game_engine.exe` from the repository root. The engine loads the signed `sl.dlss_d.dll` and `nvngx_dlssd.dll` staged next to the executable. Select **Ray Reconstruction DLAA**, **Ray Reconstruction Quality**, **Ray Reconstruction Balanced**, **Ray Reconstruction Performance**, or **Ray Reconstruction Ultra Performance** in Settings and apply the change. Settings reports the reason if the runtime or GPU is unavailable. When an RR mode is selected and the runtime, configuration, or evaluation fails, the engine stops with the error. For an explicit process override, `VKE_RECON_MODE=0` selects Native, `1`–`5` select model K DLAA through Ultra Performance, and `6`–`10` select the corresponding RR mode (`RR DLAA` through `RR Ultra Performance`).

RR requests preset F and consumes pre-tonemap ReSTIR or conventional lighting, reverse-Z depth, and unjittered motion. The deferred shader writes linear diffuse and specular albedo plus world-space shading normals and roughness. Direct specular highlights use the primary surface motion in a separate guide image. The current renderer has no traced reflections. Background pixels use unit diffuse reflectance, zero specular reflectance and motion, and a unit normal with roughness 1. RR runs once per rendered frame in the reconstruction slot. Raw diagnostics intentionally bypass reconstruction. [NVIDIA's Streamline RR guide](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS_RR.md)

Initial GPU smoke ran 240 evaluations per mode on an RTX 3090 with driver 621.14 and Streamline 2.14.16 / NGX 310.9.16. The moving ReSTIR sandbox produced finite, nonempty output in DLAA (1280×720 input) and Quality (853×480 input), both at 1280×720 output. The NGX logs reported preset F. Vulkan validation flagged first-frame synchronization and layout errors on NGX-owned images in both modes. The write-after-write report matches an open upstream issue. Non-development clean-machine packaging verification is tracked in [future work](../planning/future-work.md#clean-machine--multi-gpudriver-verification). [Streamline issue #126](https://github.com/NVIDIA-RTX/Streamline/issues/126)

### 3440×1440 timing and denoising audit

The RTX 3090 Release test used a hidden 3440×1440 Mailbox window, a fixed interior sandbox camera, three emitters, and ReSTIR DI with four candidates. Ordinary DLAA used 3.95–4.66 ms of GPU frame time. RR DLAA used 19.12–23.22 ms with 16.31–19.91 ms in reconstruction. RR Quality rendered internally at 2293×960 and used 9.40–10.52 ms per GPU frame. Each range spans the static-light and moving-light runs with 120 measured frames per mode. These are GPU timestamps under the test conditions.

Previously, moving an emitter reset RR history on all 120 measured frames. The engine now updates the light data while retaining RR history through continuous emitter motion. Scene, lighting settings, and camera cuts still invalidate history. Both RR modes completed the moving-light retest with zero resets. RR uses at least 32 jitter phases. [NVIDIA's RR integration checklist](https://github.com/NVIDIA-RTX/Streamline/blob/main/plugins/streamline-agent-skills/skills/integrate-dlss-rr/references/integration-checklist.md)

Over eight static-light frames, linear HDR receiver RMSE against a 1,024-spp reference was 0.0384 for raw ReSTIR and 0.0061 for RR DLAA. This checks static receiver accuracy. Moving-edge and changing-shadow image quality remain open. Run the audit in a Release test build with `VKE_RR_AUDIT=1`, `VKE_FIXED_PRESENT_PROBE=1`, `VKE_PROBE_HIDDEN=1`, and `VKE_PROBE_BORDERLESS=1`. Set `VKE_RR_AUDIT_MOVE_LIGHT=1` for the moving-light timing and history regression.

### Stationary soft shadows and SDK timing

The native-resolution sandbox audit used 352,668 receiver pixels in partial shadow. The mask includes pixels with 15–85% of their unoccluded luminance and reference luminance above 0.015. All light and object transforms and the camera stayed fixed. Dark pixels in the table are at least 25% below the 1,024-spp reference. Error was measured across four images after warm-up. Timings are medians over 120 GPU frames.

| Lighting input to RR DLAA | Total GPU ms | SDK evaluation ms | RR penumbra luminance RMSE | Dark pixels after RR |
| --- | ---: | ---: | ---: | ---: |
| ReSTIR with recursive spatial history | 19.15 | 16.24 | 0.01170 | 3.88% |
| ReSTIR with pre-spatial temporal history | 19.16 | 16.35 | 0.01530 | 8.65% |
| ReSTIR spatial reuse, temporal reuse disabled | 27.23 | 17.57 | 0.00382 | 0.10% |
| Four fresh conventional samples | 19.32 | 16.98 | 0.00363 | 0.10% |

The previous default reproduced visible stationary speckle in the RR result. Disabling ReSTIR temporal reuse kept the noisy input error similar while reducing the RR error and dark-pixel rate. This is consistent with NVIDIA's observation that RR can interpret correlated input noise as signal. Every SDK evaluation median exceeded the 5 ms ceiling for native 3440×1440 on RTX 3090. [NVIDIA RTXDI decorrelation notes](https://github.com/NVIDIA-RTX/RTXDI/blob/main/Doc/RestirPT.md#decorrelation-for-dlss-ray-reconstruction)

### ReSTIR final-shading decorrelation

The first correction selected an independent current-frame RIS estimate with probability 0.5 and the resampled ReSTIR estimate otherwise. It reduced the largest dark speckles, while low-angle views still exposed grain and wrinkles on the ground.

At 3440×1440, the production default passed the stationary penumbra regression with both three and 64 emitters. The three-emitter scene had 0.096% dark pixels and luminance RMSE 0.00387, compared with 3.89% and 0.01170 for the earlier recursive-history control. Mean penumbra luminance was 1.14% below the reference. With 64 emitters, dark pixels were 0.154%, RMSE was 0.00554, and mean luminance was 0.97% above the reference. All camera, object, and emitter transforms were fixed. SDK evaluation still took 16.40 ms and 16.73 ms respectively.

### Ground shadows across camera angles

The current RR path averages four normalized final-lighting estimates per pixel. Each estimate selects fresh RIS with probability 0.75 and the resampled ReSTIR estimate otherwise, independently of the lighting values. Each estimate traces current visibility, and reservoir history retains the resampled estimate. Other reconstruction modes use one final estimate.

Ground-specific tests select only the room's upward-facing floor and partially shadowed pixels. Four fixed 3440×1440 views cover a low frontal camera, low cameras from either side, and the exterior. A 1,024-spp reference anchors the comparisons. Across those views, RR ground-shadow luminance RMSE was 0.00258–0.00391, the dark-pixel rate was 0.055–0.233%, and mean luminance differed by less than 0.61%. The RR floor crops were visually inspected, including the regions with the largest 32×32-pixel error. Frontal and exterior retests also passed after a camera pan, including spatial-gradient and local-error limits. The additional final estimates raised lighting cost by about 1 ms in the low-angle tests. Native RR SDK evaluation remains above the 5 ms target.

Use `VKE_RR_SHADOW_AUDIT=1` with the fixed, hidden and borderless probe settings above and without the other audit flag. The audit compares zero, 25%, 50%, 75%, and 100% fresh-sample probabilities. Add `VKE_RR_SHADOW_VERIFY=1` to verify the production default against limits of 0.5% dark pixels, RMSE below 0.006, and mean luminance within 2% of the reference. `VKE_RR_GROUND_ONLY=1` adds floor coverage and spatial-gradient/local-error gates. `VKE_RR_AUDIT_VIEW=1/2/3/4` selects frontal/right/left/exterior fixtures. `VKE_RR_AUDIT_PAN=1` pans the camera before it settles at the reference view. `VKE_RR_AUDIT_MANY=1` selects 64 emitters. The PNG crops under `build/phase51-rr-shadow-*.png` show the raw input, RR output and reference from left to right.

### Nsight performance investigation

The November 22 focused Release profile measured RR DLAA at 18.493 ms for reconstruction and 24.076 ms for the GPU frame at native 3440×1440. A 32-frame Nsight Graphics trace places the cost in one NVIDIA RR evaluation per frame, containing 50 CUDA kernels. Scoped counters indicate a compute-heavy tensor workload, with about 65% tensor activity and 20% VRAM throughput. Direct SDK timestamps agree with the reconstruction slot. The native 5 ms ceiling remains unmet. See the [RR performance investigation](rr_performance.md) for measured comparisons, counter interpretation, clock telemetry, and reproduction commands.

### Guide debug views

The F5 `Temporal view` selector exposes the five RR guide buffers alongside the existing temporal views: `RR diffuse albedo`, `RR specular albedo`, `RR shading normals`, `RR roughness`, and `RR specular motion` (`debugView` indices `5`–`9`). Selecting any RR guide view in a non-RR mode allocates the RR guide images on demand and frees them when returning to ordinary views.

Run `VKE_RR_GUIDE_SMOKE=1` with `VKE_FIXED_PRESENT_PROBE=1` and `VKE_PROBE_HIDDEN=1` to verify guide allocation, background contracts, unit shading normals, roughness bounds, specular-motion parity, and guide-buffer release on return to Native (`VKE_RR_GUIDE_SMOKE_NATIVE_ONLY=1` with `VKE_FORCE_VALIDATION=1` and `VK_LAYER_VALIDATE_SYNC=1` passed with zero validation or synchronization errors).

### Dynamic-scene quality audit

`VKE_RR_DYNAMIC_AUDIT=1` replays a 64-frame camera + moving-cube + moving-emitter trajectory at 3440×1440 (including a cube snap at frame `32` revealing `474,553` disoccluded pixels and emitter motion across frames `32`–`47` followed by 16 stationary frames) against 1,024-spp references in both the 3-emitter and 64-emitter (`VKE_RR_AUDIT_MANY=1`) fixtures:

- **History continuity**: Ordinary `DLAA` resets temporal history `15` times during emitter motion (`frames 32`–`47`), whereas `RR DLAA`, `RR Quality`, and `RR Performance` all complete with `0` history resets.
- **Disocclusion & ghosting**: On the disocclusion step (`frame 32`), `RR DLAA` cuts disoccluded luminance RMSE from `0.03064` (`DLAA`, `9.68%` dark holes, `8.40%` ghosting) to `0.00851` (`0.10%` dark holes, `0.041%` ghosting) in the 3-emitter scene, and from `0.02503` (`12.23%` dark, `6.83%` ghost) to `0.00677` (`0.12%` dark, `0.027%` ghost) in the 64-emitter scene. Visual inspection of the `frame 32` crop shows mild low-frequency blotching in newly revealed pixels that clears by `frame 33` (`lumRMSE` `0.00475` / `0.00369`). `RR Quality` (`0.00793` / `0.00617` at `f=32`) and `RR Performance` (`0.00731` / `0.00564`) track closely.
- **Moving edges & changing shadows**: Across moving object boundaries (`20,618`–`26,999` px), `DLAA` luminance RMSE peaks at `0.03447` (`3`-emitter `f=36`, `6.70%` ghosting) and `0.04125` (`64`-emitter `f=36`, `8.35%` ghosting), while `RR DLAA` stays at `0.00922`–`0.01775` (`<= 1.94%` ghosting) and `0.00745`–`0.00879` (`<= 0.72%` ghosting). On changing-shadow pixels (`f=36`, `f=40`), `RR DLAA` achieves `0.00658`/`0.00647` (3-emitter) and `0.00656`/`0.00564` (64-emitter) with `<= 0.49%` dark holes and `<= 0.044%` ghosting.

### VRAM footprint and denoiser decision

Fresh-process `VKE_RR_MEMORY_AUDIT=1` runs at 3440×1440 (`Sandbox64Interior`, `VK_EXT_memory_budget`) measured `2002.14 MiB` process VRAM in `Native`, `2330.58 MiB` in ordinary `DLAA`, `1449.33 MiB` in `DLSS Quality`, `3626.58 MiB` in `RR DLAA` (`282.19 MiB` engine RR guides + `~1,014 MiB` Streamline/NGX overhead above `DLAA`), `2210.33 MiB` in `RR Quality` (`125.62 MiB` RR guides), and `1724.14 MiB` in `RR Performance` (`70.59 MiB` RR guides). Switching from `RR DLAA` back to `Native` frees all `282.19 MiB` of engine RR guides and reclaims `1,076.63 MiB` total (`-> 2549.95 MiB`), leaving `547.81 MiB` of persistent Streamline/NGX allocations until shutdown. See the [RR performance and memory report](rr_performance.md#memory-footprint) for all modes.

**Phase 5.1 Denoiser Decision**: Retain DLSS 4.5 RR as an optional neural RTX path (practical primarily in upscaled RR modes such as Quality/Balanced/Performance, since native 3440×1440 RR DLAA costs `~24.1 ms` GPU and `+1.30 GiB` VRAM over ordinary DLAA). Keep `Native` and model K `DLAA`/`DLSS SR` as RR-off / non-RTX paths (which do not perform lighting denoising on their own). Phase 5.1 Task 2 provides **NVIDIA Real-Time Denoisers (NRD — `RELAX` and `REBLUR`)** as the lightweight analytical compute denoiser for non-RR modes (mutually exclusive with DLSS RR), while custom hand-written SVGF remains deferred. See [NRD integration & benchmarks](nrd_integration.md).

## Dependency and Vulkan integration

`cmake/Streamline.cmake` pins the official Windows x64 [Streamline 2.14.16 SDK archive](https://github.com/NVIDIA-RTX/Streamline/releases/tag/v2.14.16). Its published SHA-256 is `32041e51db0cab709a271b72a352281b690c2abbd669a827809401a5dd23956b`. The archive/extracted SDK live under ignored `.cache/streamline/2.14.16/`; cached configurations verify the checksum and reuse it without downloading again. Other dependencies' disconnected-fetch configuration is unchanged.

`VKE_ENABLE_DLSS=ON` is the build default. `VKE_STREAMLINE_SDK_DIR` can supply an existing SDK root containing `include/sl.h`. `VKE_ENABLE_DLSS=OFF` compiles native operation without fetching or loading NVIDIA dependencies. Final development configuration is MinGW Debug with focused tests enabled.

The build stages signed production `sl.interposer.dll`, `sl.common.dll`, `sl.dlss.dll`, `sl.dlss_d.dll`, `sl.reflex.dll`, `sl.pcl.dll`, `nvngx_dlss.dll`, `nvngx_dlssd.dll` and `NvLowLatencyVk.dll` beside the executable, with notices in `build/nvidia-notices/`. These remain ignored and outside `engine.pak`. No MSVC import libraries are linked. The backend verifies signatures of its core DLLs and optional RR/Reflex/PCL runtimes, and dynamically loads Streamline from the executable directory.

The current loader creates or refreshes a copy of `sl.interposer.dll` named `vke_sl_interposer.dll` based on file existence/timestamps, verifies its signature and loads that copy. The executable directory must permit creating/updating this file. The original staged DLL remains required, and NVIDIA binary contents are not patched. This is the implemented Mailbox pacing workaround; the historical canonical-name runs below do not measure its current performance or Reflex latency benefit.

The portable packaging script (`package_release.ps1`) stages the stripped executable, `engine.pak`, NVIDIA Streamline/DLSS/RR/Reflex DLLs + notices (`nvidia-notices/`, including `nrd-license.txt`), and required Microsoft VC++ runtime DLLs (`MaksimEngine-v0.3.1-Portable.zip`). Verification on non-development hardware is tracked in [future work](../planning/future-work.md#clean-machine--multi-gpudriver-verification).

Manual Vulkan integration initializes Streamline before instance/device creation, queries feature requirements and adapter support, and routes instance/device creation, physical-device enumeration, surface/swapchain lifecycle, acquire, presentation and device-idle hooks. Routing physical-device enumeration is necessary for Streamline's instance association. Private-data/timeline features support its tracking; optional low-latency Vulkan support now serves the explicit Reflex integration described below; Frame Generation remains disabled. Unsupported adapter setup returns to native Vulkan creation.

One viewport/history context and two frame-owned input/output resource sets use frame-based tagging. The presentation hook services Streamline cleanup. Resizing or rendering-mode changes retire outstanding work before freeing SDK resources. The pinned bundle still logs three unsupported command-tracking-hook warnings at startup; these are distinct from Vulkan validation messages.

In the refreshed portable build, Ray Reconstruction initialization or evaluation failures during interactive play fall back to Native. Settings shows the fallback reason. Fatal errors write `crash.log` beside the executable. [Release refresh validation](../5.1/integration-validation.md#v031-portable-release-refresh)

## Rendering contract

SDK reconstruction runs after completed visibility shading, before tone mapping and output-resolution ImGui. It consumes linear FP16 HDR, sampled reverse-Z hardware depth, previous-minus-current unjittered pixel motion including camera/object motion, and fixed exposure 1. Automatic exposure is disabled; pre-exposure and exposure scale are 1. Output is RGB HDR; presentation does not rely on SDK alpha. The background is converted to linear HDR to preserve the configured displayed clear color through tone mapping.

The adapter packs matrices for the SDK, converts Vulkan clip Y orientation, supplies unjittered matrices, normalizes motion by exact input dimensions and supplies pixel jitter. DLAA uses native dimensions. DLSS uses SDK-selected exact dimensions: at 3440x1440 output the tested inputs are 2293x960 Quality, 1995x835 Balanced and 1720x720 Performance. Jitter phases derive from actual dimensions (8/18/24/32 here), and the engine advances jitter/history only after successful submission.

Automatic texture sampling applies `log2(renderWidth / outputWidth) - 1` bias while reconstruction is active; forced texture-mip diagnostics are preserved. Raw diagnostic views bypass reconstruction and reset SDK history upon return. Native jitter preview/manual scales remain available; active DLAA/DLSS controls these inputs. Hi-Z keeps its independent producing-camera/depth metadata.

Explicit barriers make input writes available to reconstruction and output writes available to presentation; engine pipeline/descriptor state is rebound afterward. Missing/untrusted DLLs, unsupported capability or SDK configuration failure leave effective Native with a visible reason. Evaluation failure discards the unsubmitted frame, consumes its acquire semaphore and recreates Native at a safe boundary, without presenting stale output or advancing engine history for discarded work. This exceptional path may wait; normal frames add no blocking readback/idle wait.

Native at 100% presents directly from scene HDR. Reduced Native diagnostic scales retain spatial reconstruction. The descriptor-backed `FrameGpu` is 704 bytes; draw push constants remain within their existing 256-byte limit.

### Native work, timing and configuration cleanup

Ordinary Native color now generates no temporal guides. DLAA/DLSS generates motion and linear depth (the latter supports clear-background composition); Native motion/depth/normal views generate their selected guides, and validity inspection or opt-in capture generates all guides. Descriptor-compatible images remain allocated for fast switching. Only consumed guides are cleared each frame. Full-frame lighting/reconstruction outputs are overwritten rather than cleared first; empty scenes explicitly clear HDR. Hi-Z continues using independent hardware depth and history. Enabling validity inspection rejects history until compatible guide data is available.

Rendering-mode, internal-scale and raw-diagnostic changes retire graphics fences and reconfigure rendering without replacing the presentation swapchain. Same-size switches can reuse scene and temporal targets when their required guide and reference images match. When render dimensions change, the engine releases ReSTIR buffers before allocating new targets. Configuration changes that rebuild targets also retire NRD and spatial resources first. Changed internal dimensions recreate scene targets; unchanged geometry resources retain Hi-Z history. Actual window/presentation changes still use swapchain recreation. Applying unrelated settings, such as sensitivity or UI scale, does not retry a failed backend or reset reconstruction.

F5 Detailed timings shows the actual Immediate/Mailbox/FIFO presentation mode, CPU frame wall time, time inside acquire/fence and submit/present calls, remaining CPU wall time, total GPU time, lighting including RT shadows, reconstruction, and tone-map/display GPU scopes. CPU wall time is not CPU utilization. GPU queries remain fence-protected and opt-in. A rolling RT-on/off lighting estimate requires at least 16 settled samples in each state at matching camera/object/lighting/LOD/resolution inputs. Moving the view or changing those inputs invalidates the comparison. The estimate includes other lighting-path differences when shadows are toggled and does not isolate pure ray traversal.

Cleanup verification: all three CPU suites and the Debug build passed. The extended temporal/Hi-Z synchronization-validation smoke passed. An additional mode matrix completed Native and all four K cases, including stationary image comparisons and mode/diagnostic/resize transitions, without Vulkan validation messages; it was stopped during the later deferred-model fallback cases after desktop flicker occurred on multi-mode transitions. It is an interrupted run, not a full-matrix pass. Additional resource-reuse and Native fast/full-guide image-parity assertions were compiled afterward but not executed. Display tests and after-change performance measurements were not restarted. No speedup is claimed from this cleanup yet.

The gray-bar workaround is unchanged by this cleanup: presentation restores the configured flat clear color where a 5x5 source-depth neighborhood contains no geometry, preserving reconstructed silhouette pixels nearby. The original cause of the displayed bands remains unproven; intermediate HDR captures alone did not expose them. Settings offers Mailbox, Immediate and VSync (FIFO). Mailbox is the default and falls back to FIFO if unavailable; Immediate is selected only explicitly and may tear. Unsupported Immediate falls back to Mailbox, then FIFO. The temporary switch to Immediate bypassed the observed 200 FPS limit but did not diagnose it. The later small-window statistics-HUD experiment temporarily ran uncapped, but did not establish a reliable fix for normal borderless operation.

## Verification on the RTX 3090

Verified with MinGW Debug, Streamline 2.14.16 / NGX 310.9.16 and NVIDIA driver 616.92:

- All three focused CPU suites passed: temporal math, Hi-Z history and reconstruction. Added coverage includes settings persistence/defaults/malformed input, jitter phases, matrix packing/reprojection, motion scaling, mip bias and clear-color round trips.
- Real DLAA evaluation passed the compatibility checkpoint. Independent K runs in all four NVIDIA modes reported no Vulkan validation messages.
- The historical 13-case synchronization-validation matrix passed with 464 real K evaluations and 13 stationary image comparisons. Native plus four K modes evaluate normally; eight then-deferred M/L requests exercised Native fallback. Those eight cases are now removed; the current K-only matrix has five cases. Cases cover movement, abrupt turns, object transforms, cuts, all six LODs, F2, Hi-Z toggling, diagnostic bypass, odd 1001x777 resizing, aspect bars, return to 3440x1440 borderless and two frames in flight. Native/DLAA cases also exercise minimize/restore. Stationary RGB comparisons require finite/nonempty output and mean absolute change below 0.005.
- With Quality/K and automatic jitter, 144 sampled source hardware-depth/HDR frame pairs matched Hi-Z-off references byte-for-byte, including all six LODs, movement, reveals, transforms and odd SDK input dimensions. Every sampled pyramid mip matched CPU minimum reduction. The run recorded 883 final rejects, 281 recoveries and 256 deliberately injected overflow fallbacks, with no Vulkan validation messages. Continuous SDK jitter correctly uses prediction/recovery rather than the unchanged-view fast path.
- Missing `sl.dlss.dll`, simulated unsupported capability and a native-only build each passed native temporal/F2 smoke. An injected evaluation failure after three successful SDK evaluations recovered to unjittered Native with no repeated evaluation attempts or validation errors. The temporarily moved DLL was restored.
- Cached reconfiguration reused the SDK. Test logs, captures and CSVs remain under ignored `build/`; principal logs are `dlss-mode-matrix-final.log`, `dlss-k-hiz-{off,on}.log`, `dlss-failure-fallback.log`, `dlss-missing-runtime.log`, `dlss-native-only-smoke.log` and `dlss-unsupported-fallback.log`.
- Explicit VSync-on presentation checks selected FIFO on the 100 Hz primary display and measured 99.986 FPS Native / 99.998 FPS DLAA/K over 180 frames after 60 warm-up frames. Off requests select Mailbox on this device. The test UI now mirrors the renderer request; first-launch settings default to off. Loading saved settings recreates presentation in windowed mode too, closing a path where an on checkbox could disagree with the active swapchain. This establishes correct current behavior without isolating the earlier 200 FPS Mailbox cap. Logs: `build/presentation-on-mode{0,1}.log`.

Historical boulder automated image checks and basic capture inspection passed. Extended human inspection for subtle ghosting, texture instability and moving edges, quantitative memory-growth testing and other GPU/driver combinations remain unverified in Phase 5.1. Repeated resize/mode-switch exercises alone do not prove a memory plateau. Phase 4.1 closure reflects revised scope, not additional verification; these historical results do not validate the new sandbox.

## Historical Debug cost

These measurements predate the current signed-copy loader. They remain the recorded cost of the stated configuration, not a current post-workaround benchmark.

One sequential run used 3440x1440 borderless output, LOD 0, default lighting/RT settings and identical stationary/sinusoidal-turn camera paths. Validation layers were enabled, synchronization validation was not enabled for timing. The test requests VSync off (Mailbox where supported); this is not a strict frame limiter. Native is unjittered and NVIDIA modes use their required automatic jitter. Each case warms up for 100 frames and targets 300 measured frames; retired query pools leave 297 GPU samples in most cases, 300 in the last case. Numbers are medians in milliseconds.

| Mode, K | Stationary GPU | Moving GPU | Stationary reconstruction | Stationary wall | Moving wall |
| :--- | ---: | ---: | ---: | ---: | ---: |
| Native | 0.756 | 0.974 | 0 | 7.852 | 7.274 |
| DLAA | 2.019 | 2.079 | 1.294 | 6.112 | 6.145 |
| Quality | 1.714 | 1.777 | 1.270 | 6.339 | 6.719 |
| Balanced | 1.669 | 1.711 | 1.267 | 5.886 | 6.320 |
| Performance | 1.640 | 1.684 | 1.266 | 6.635 | 6.689 |

Reconstruction costs more GPU time than the reduced lighting workload saves in this small scene. GPU scopes use fence-protected timestamp queries without WAIT; ordinary rendering records them only when F4, F5 or a benchmark requests them. Wall time includes CPU, validation, driver and presentation pacing, varies substantially, and does not establish an uncapped throughput gain. All comparison modes here load the same SDK backend. These numbers do not compare against the historical Phase 4 Release distribution. Raw stage timings are in `build/reconstruction-benchmark.csv`.

## Reproduction

Configure/build the existing directory, without making a Release:

```text
cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug -DVKE_BUILD_TESTS=ON -DVKE_ENABLE_DLSS=ON
cmake --build build --target VK_game_engine reconstruction_tests temporal_math_tests hiz_history_tests -j8
ctest --test-dir build --output-on-failure
```

From the repository root, process-local `VKE_DLSS_CHECKPOINT=1` runs the finite DLAA checkpoint; `VKE_DLSS_SMOKE=1` runs the matrix. Use `VK_LAYER_VALIDATE_SYNC=1` for correctness checks. `VKE_DLSS_BENCH=1` runs the K timing cases, sequentially without another GPU test. `VKE_RECON_MODE=2` with `VKE_HIZ_VERIFY=off` then `on` verifies Quality/K source parity. Restore environment variables after testing. Test image readbacks may wait; normal rendering does not use those readbacks.

For finite presentation checks, combine `VKE_DLSS_CHECKPOINT=1`, `VKE_PRESENT_SMOKE=1`, `VKE_TEST_VSYNC=1` and `VKE_RECON_MODE=0` (Native) or `1` (DLAA). This measures loop FPS after warm-up and logs the primary display refresh and chosen presentation mode. It leaves persisted settings unchanged.

No commits, pushes, documentation synchronization, PowerShell scripts, new Release builds or changes to existing distributions were performed.

## Performance/diagnostics window split

F4 now contains only FPS/frame time, active mode and dimensions, total/lighting/reconstruction GPU timings and the small frame-time graph. F5 independently toggles a movable, resizable inspector at the upper right. Its collapsible sections retain Reconstruction, Temporal inspection, Ray tracing and lighting, Geometry and culling, and Detailed timings. Only Reconstruction starts expanded. This UI-only split built cleanly in Debug before subsequent Phase 5 display and validation checks. The background-composition workaround and renderer behavior were not changed by the split.

## Historical canonical-name 200 FPS investigation

This section records the earlier runtime-loading state. The current loader uses the signed `vke_sl_interposer.dll` copy described above; the FPS results here belong to their stated historical configurations.

A fixed 1280x720 decorated window isolated the limit without resize, minimize/restore or window-mode transitions. With the NVIDIA statistics HUD visible, canonical `sl.interposer.dll` reproduced roughly 200 FPS while focused, with about 4 ms inside image acquisition. The same Mailbox window was uncapped unfocused. Native operation without Streamline remained uncapped while focused, including when the same Vulkan extensions/features were enabled. DXGI loading/factory enumeration, NVAPI initialization and matching the SDK timer-resolution request did not reproduce the limit. A separate acquisition fence did not remove it.

A diagnostic load-only test reproduced 200 FPS without calling slInit or evaluating DLSS. Loading a byte-identical, still-signed copy under another basename avoided it. That implicated external recognition of the canonical module name rather than scene or reconstruction work. At that checkpoint the alias was diagnostic only; the current loader now uses a signed copy under `vke_sl_interposer.dll`. No NVIDIA binary was patched.

Hiding NVIDIA's statistics HUD with Alt+R with the ORIGINAL DLL name and Mailbox preserved allowed the focused control to measure 1445 FPS Native. Final Debug synchronization-validation checks with the canonical runtime measured 1369 FPS Native and 726 FPS DLAA, with all 180 measured frames focused; DLAA performed 240 real evaluations. Neither log contained Vulkan validation messages. These small-window results establish removal of the 200 FPS lock under the tested overlay state; they are not claimed 3440x1440 performance measurements. The result suggests an interaction with external module recognition, though the 200 FPS limit still recurred intermittently with the statistics HUD hidden. Alt+R hides statistics; it does not disable all NVIDIA overlay or recording features.

Hiding the statistics HUD was not established as a reliable fix, and retaining the NVIDIA overlay remains a requirement. At the end of that earlier investigation the loader returned to canonical filenames and removed the experimental probes. The current loader subsequently adopted the signed-copy workaround; it preserves the requested presentation mode. Earlier feature masks, init-stage probes and acquire-fence experiments are absent, and no global driver-profile settings were changed by that investigation.

Reproduction support: with test builds, combine `VKE_FIXED_PRESENT_PROBE=1`, `VKE_PRESENT_SMOKE=1`, `VKE_PROBE_FOCUS=1`, and `VKE_RECON_MODE=0` (Native) or `1` (DLAA). This creates one fixed 1280x720 window, bypasses persisted window settings and rejects display-changing suites. `VKE_TEST_VSYNC` can explicitly select FIFO for a control; omission keeps Mailbox. The report includes effective mode, real evaluation count and focused sample count; do not interpret an unfocused run as a focused performance result. Restore process-local variables afterward. Logs are `build/pacing-hud-hidden-mode0.log` and `build/pacing-hud-hidden-mode1.log`; the original-name post-Alt+R control is `build/pacing-after-alt-r.log`.

## Presentation setting

Settings now provides a single Presentation selector: Mailbox (tear-free), Immediate (tearing allowed), and VSync (FIFO). It uses the existing Apply/Accept/Cancel flow and persists `presentationMode` (0/1/2 respectively). The first-launch default is Mailbox. Older INI files migrate `vsync=0` to Mailbox and `vsync=1` to FIFO; an explicit `presentationMode` takes precedence regardless of key order. New saves also write derived `vsync` for legacy readers. Invalid mode values fall back to Mailbox.

Applying a presentation change recreates the swapchain live without requiring a restart. F5 Detailed timings shows requested and actual presentation modes; unsupported Immediate falls back to Mailbox and then FIFO, while Mailbox never silently falls back to a tearing mode. In Phase 5.1, Win32 NVIDIA Mailbox swapchains unconditionally apply `VK_FULL_SCREEN_EXCLUSIVE_DISALLOWED_EXT` (`tears = 0`) and wait on the current frame's `inFlightFences` right before `vkQueuePresentKHR` (`syncBeforeMailboxPresent`), eliminating VBlank stalls (`0` stalls on fixed Mailbox in `VK_EXT_present_timing` audits). A separate 200 FPS cap reported under some Win32 Mailbox configurations remains deferred work. [Pacing verification](../roadmap.md#win32-mailbox-presentation-pacing) and [future work](../planning/future-work.md#win32-mailbox-200-fps-cap).

The Debug build and focused CPU checks validate persistence, legacy migration, invalid values and supported-mode fallback. No engine/display test was launched for this settings change.

## Historical Mailbox image-count experiment and Reflex verification (2026-10-26)

These measurements used the canonical interposer filename before the current signed-copy workaround. They establish behavior of that configuration, not current post-workaround FPS or latency.

The existing surface minimum-plus-one policy produced three swapchain images (reported minimum 2, maximum 8). A single fixed 3440x1440 borderless window compared requested/actual counts 3, 4, 5, then 3, with Mailbox, two frames in flight, identical scene/camera/lighting/RT settings, and the original signed NVIDIA runtime. Each case warmed up for 120 frames, then measured 599 intervals; all 599 samples were focused. No NVIDIA overlay setting was changed.

| Actual images | Native FPS | Native acquire/fence ms | DLAA FPS | DLAA acquire/fence ms |
| --- | ---: | ---: | ---: | ---: |
| 3 | 199.74 | 4.405 | 200.08 | 4.089 |
| 4 | 200.27 | 4.356 | 200.24 | 4.052 |
| 5 | 200.10 | 4.320 | 199.84 | 4.063 |
| 3, repeated | 199.76 | 4.333 | 199.87 | 4.083 |

More images did not remove the lock. The production image-count policy remains unchanged. Logs: `build/mailbox-native-sweep.log` and `build/mailbox-dlaa-sweep.log`. CPU acquisition stalls are separate from GPU rendering cost; this result does not identify the underlying driver/compositor/overlay cause.

Settings adds NVIDIA Reflex Off / On / On + Boost, default On when supported, with the existing Apply/Accept/Cancel behavior. Requested mode persists as `reflexMode`; effective mode and fallback reason are separate. Reflex works with Native as well as every DLAA/DLSS mode. F5 has a collapsed Reflex section with status, scheduling sleep, and driver-reported simulation-to-GPU-complete time when available. That duration excludes full input-to-display latency.

The shared dynamically loaded backend requests the signed Reflex and PCL plugins. It calls Reflex sleep once per loop before ordinary input polling, including with Reflex Off; emits simulation, queue-submit and present markers using one opaque frame token; observes the application's input-thread Windows message pump so PCL pings are marked even when they arrive during GLFW polling; and emits click flash markers. The message hook is restricted to the application input thread and is removed before SDK shutdown. SDK frame IDs track loop attempts independently of successfully submitted temporal/Hi-Z history. `frameLimitUs=0` explicitly disables the application's Reflex FPS limiter. On supported devices, `VK_NV_low_latency2` and feature-checked `VK_KHR_present_id` associate submissions and presentation with that same ID. Existing Streamline swapchain hooks enable/retire the latency context.

Frame-slot fences are waited before Reflex schedules input sampling. The redundant second image-fence wait at submission is removed; acquisition still retires each acquired image before command-buffer reuse. Missing or failed Reflex keeps rendering available, selects effective Off and explains why. No extra per-frame CPU readback or idle wait was added; the Reflex sleep is deliberate scheduling, and latency-state queries are sampled every 120 frames.

The fixed-window Reflex smoke exercised all 15 rendering/Reflex combinations, movement, Hi-Z toggling and two frames in flight under synchronization validation. It asserted one sleep per loop (including Off), six simulation/submit/present markers per completed frame, increasing SDK frame IDs across mode changes, and real evaluations in every NVIDIA mode. All cases passed, all measured frames stayed focused, and no Vulkan validation messages were reported. Mailbox remained approximately 199-201 FPS in all combinations. GPU costs were approximately 0.54-0.59 ms Native, 2.12-2.18 ms DLAA, 1.88-1.90 ms Quality, 1.84-1.86 ms Balanced and 1.78-1.81 ms Performance. No latency reduction is established by this presentation-limited scene; driver samples are not an end-to-end latency benchmark. NVIDIA's external Reflex certification utilities were not installed or run.

The first matrix exposed a telemetry ordering issue after moving the frame fence: its acquire field measured only the early fence. That field is not used as acquisition evidence; the ordering was corrected and rechecked afterward. The earlier image-count sweep above measured full acquisition correctly. The final run recorded full acquire/fence means of about 4.18-4.24 ms Native and 3.66-3.84 ms in NVIDIA modes. Final log: `build/reflex-mode-matrix-final.log`; injected PCL pings were received by the input-thread hook.

Fallback checks: temporarily removing `sl.reflex.dll` still produced 240 real DLAA evaluations with effective Reflex Off; the DLL was restored. A native-only Debug configuration compiled and ran with zero NVIDIA calls. The existing build directory was restored to DLSS-enabled Debug afterward. CPU tests cover Reflex defaults/persistence/malformed input, presentation choices, and ignoring obsolete M/L settings.

Current source audit: the signed-copy loader is present; earlier DXGI/NvAPI/timer probes, feature masks, initialization-stage probes and acquire-fence experiments are absent. Required signature checks, Vulkan lifecycle hooks, presentation cleanup and fence-protected timestamps remain. Fixed-window image-count and Reflex probes are compiled only with `VKE_BUILD_TESTS`; they do not change normal startup, settings or rendering. To run them on the current loader, combine `VKE_FIXED_PRESENT_PROBE=1`, `VKE_PROBE_BORDERLESS=1`, `VKE_PROBE_FOCUS=1` with either `VKE_SWAPCHAIN_SWEEP=1` plus `VKE_RECON_MODE=0/1`, or `VKE_REFLEX_SMOKE=1`. Use `VK_LAYER_VALIDATE_SYNC=1` for synchronization checks. These probes use a fixed window and do not execute the older resize/minimize suites. New results must be labeled with the loader state rather than treated as reproductions of the old capped configuration.

## Current latency interpretation and remaining work

The current sandbox supports [full ReSTIR DI](restir_di.md), including temporal/spatial reuse, retained previous TLAS/lights/surfaces and independent history. Native/DLAA/Quality and diagnostic bypass/restoration are tested together with reconstruction/internal/output-size changes. Physical pose history survives SDK lighting resets. These checks do not add RR, GI, denoising or a Reflex latency claim; the Chunk 3 initial-only paragraph below is historical.

Phase 5 Chunk 3 adds initial direct-light reservoirs with independent per-pixel storage. Its fixed-window suite passes real DLAA and DLSS Quality evaluation, raw reservoir views bypass SDK reconstruction correctly, and closing them restores the requested mode. Numeric per-instance motion is checked against CPU reprojection. These are correctness checks, not new Reflex/latency measurements or full-mode image-quality acceptance; see [Chunk 3](../archive/phase-5/phase5_chunk3.md).

F5's driver report is simulation-start to GPU-complete time, sampled every 120 loops. NVIDIA's Alt+R Average PC Latency (`PCL (av)`) includes input sampling, simulation-to-present and present-to-displayed time; it is a different metric and is not full mouse-to-photon latency. CPU/GPU frame timings are separate again. See [NVIDIA's PC latency definition](https://developer.nvidia.com/blog/understanding-and-measuring-pc-latency/).

Uncontrolled overlay observations (~20 ms Reflex Off vs. ~22 ms On in early checks) do not replace controlled end-to-end benchmarks (`simToGpuMs` in Phase 5.1 `VKE_REFLEX_BENCH=1` sweeps dropped from `25.48 ms` to `19.24 ms`). Frame-slot fences are waited before Reflex/input sampling, while swapchain-image acquisition still occurs after input has been sampled.

Native fast/full-guide HDR parity and same-size Native/DLAA reuse assertions pass in the fixed-window suite. Full ReSTIR DI is now implemented and GPU verified on RTX 3090 with Native/DLAA/Quality; see its current verification notes above. This is not a latency or optimized throughput result. Phase 5.1 retains controlled Reflex comparisons, extended quality review, memory soak, broader hardware, RR and NVIDIA packaging. Direct desktop UI automation retains the previous capture-tool limitation. Match mode, presentation, focus, camera and overlay conditions for latency measurements; SVGF stays deferred.
