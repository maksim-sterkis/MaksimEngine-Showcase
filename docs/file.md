# File Architecture and Connections

This document describes the implemented boulder and Phase 5 sandbox (**Phase 5 completed in `v0.3.0`, 2026-11-14**), including temporal/spatial ReSTIR DI (48-byte packed cached surfaces, 32-byte reservoirs, emission payload cache) and fresh final visibility. Recursive history, sustained 3440x1440 Mailbox throughput, equal-cost multi-view audits, motion/jitter checks, forced synchronization validation, and a clean `VKE_BUILD_TESTS=OFF` Release package are verified on RTX 3090. Lighting is direct only, with no GI or denoising; DLSS Ray Reconstruction, Cornell/Sponza scenes and clean-machine checks follow in Phase 5.1, while SVGF stays deferred. See [verification and limits](#verification-and-limits).

## `demo/` (Interactive Demo)

- **`demo/main.cpp`** initializes the engine, loads the boulder, creates its ECS entity and supplies update, culling and drawing callbacks. The camera starts at `(3.0, 1.0, 0.0)` aimed at the model's bounding-box center.
- Runtime controls include `F1` / `M` meshlet colors, `F2` paired culling/depth freeze, `0`–`6` automatic/manual LOD, `7`–`9` frustum/cone/Hi-Z toggles, `T` RT shadows and `Y` RT visible-LOD matching.
- F3 is the passive HUD. F4 is the compact performance overlay with reconstruction mode/dimensions, key GPU timings and frame-time graph. F5 is the movable diagnostics window for reconstruction/temporal controls, Reflex, RT/sun, geometry/culling and detailed timings.
- Esc releases the cursor and opens the pause/settings UI. Settings persists window/presentation mode, aspect ratio, reconstruction mode, Reflex mode, UI scale and sensitivity. Debug culling/lighting/temporal controls remain session-only.
- **`demo/sandbox.hpp` / `.cpp`** preload the engine-authored room, shared cubes, metal and three/64 emitters. The session Esc selector switches scenes safely; Boulder remains the startup default. F5 exposes sample counts, motion, identity/emission diagnostics and the unjittered Native 4096-spp reference. Opaque instances retain stable entities, per-instance LOD and current/previous transforms. MLOD v2 supplies source-surface/material ownership; the unchanged legacy boulder uses a coarse single-material mapping.

## `src/core/` (Platform, Input and Settings)

- **`window.hpp` / `.cpp`** wrap GLFW window creation, windowed/borderless changes, resize callbacks and the Vulkan surface.
- **`input.hpp` / `.cpp`** track keyboard/mouse input and emit click flash markers for latency instrumentation.
- **`camera.hpp` / `.cpp`** update camera motion, orientation, matrices and frustum planes. Shading uses the render camera independently of frozen culling state.
- **`settings.hpp` / `.cpp`** load/save `settings.ini`. Requested reconstruction and Reflex choices are distinct from effective runtime state.
- **`presentation_mode.hpp`** defines Mailbox (default), explicit Immediate and VSync (FIFO), with supported-mode fallback and legacy VSync migration handled by settings.
- **`reflex_settings.hpp`** defines Off, On and On + Boost; the first-launch request is On.
- **`vfs.hpp` / `.cpp`** access `engine.pak` through Windows memory mapping, expose byte spans and fall back to loose files during development.

## `src/renderer/` (Vulkan and Frame Rendering)

- **`device.hpp` / `.cpp`** create the Vulkan instance/device/queues and command pool, prioritize discrete GPUs, and enable mesh shaders, descriptor indexing, scalar block layout, anisotropic filtering, buffer device addresses and hardware acceleration structures/ray queries. The NVIDIA integration also negotiates SDK requirements and optional low-latency/presentation-ID support.
- **`vulkan_hooks.hpp`** exposes Vulkan dispatch pointers routed through Streamline when the backend is active, with native entry points otherwise.
- **`swapchain.hpp` / `.cpp`** select the effective presentation mode, own output images/UI framebuffers, two frame-owned visibility/depth targets and Hi-Z resources, and manage image/frame fences, acquisition, submission and presentation. Scene render extent and output viewport are separate. Presentation changes recreate the swapchain; reconstruction changes can reconfigure scene targets without doing so.
- **`engine.hpp` / `.cpp`** orchestrate initialization, frame configuration, CPU timing, geometry/culling, TLAS builds, HDR lighting, reconstruction, display conversion and UI. Successful submissions commit temporal and Hi-Z history; frame slots are independent of acquired swapchain-image indices.
- **`pipeline.hpp` / `.cpp`** create visibility graphics, object-cull, Hi-Z, light-preparation and deferred-lighting pipelines. Draw push constants remain 256 bytes. The global set contains materials, textures, TLAS and an immutable buffer-address model table; frame bindings 9/10/11 provide instances and local/world lights.
- **`occlusion.hpp` / `.cpp`** own independent previous/frozen depth metadata, recovery lists, GPU indirect setup and conservative history compatibility. Unchanged views use one geometry pass. Compatible moving/jittered views predict against previous depth and recover deferred meshlets against current depth before lighting.
- **`temporal.hpp` / `.cpp`** own two sets of HDR/guide/resolved targets and RGBA32f diagnostic-reference sums, snapshots, validation, descriptor updates and reconstruction/display pipelines. Ordinary Native presents current HDR without temporal blending; only the explicit sandbox reference accumulates lighting samples.
- **`temporal_math.hpp`** defines matrix/transform packing, jitter, reprojection and history compatibility helpers. Descriptor-backed `FrameGpu` is 704 bytes.
- **`restir_math.hpp`** retains the 48-byte legacy initial-only test path at binding 14; inactive modes bind zeroed stubs. **`restir_full_math.hpp`** defines 32-byte GPU reservoirs, 48-byte cached surfaces, mapping and source-aware normalization. **`shaders/restir_packing.h`** shares integer metadata constants between C++/GLSL; sample/weight floats retain all 32 bits. **`restir_full.hpp` / `.cpp`** own snapshots, scratch, maps and the 320-byte BDA context at binding 15. Scalar Basic reuse and fused material/temporal/spatial/shading variants reduce traffic; reference/ablation kernels remain available. Final spatial output is retained directly as history; temporal-only comparison uses copy-free rotation. See [current verification and costs](#verification-and-limits).
- **`reconstruction.hpp` / `.cpp`** dynamically load signed Streamline components, negotiate NVIDIA support, configure fixed model K DLAA/DLSS, tag inputs/evaluate reconstruction and handle Native fallback. The same backend implements independent Reflex modes, scheduling, frame markers and input-thread PCL pings. The current loader creates/loads the signature-verified `vke_sl_interposer.dll` copy as a Mailbox pacing workaround. See [integration and verification limits](file.md#verification-and-limits).
- **`reconstruction_settings.hpp`** defines Native, DLAA, Quality, Balanced and Performance plus SDK matrix/motion/jitter/mip-bias helpers. Removed M/L settings are ignored.
- **`raytracing.hpp` / `.cpp`** build discrete static BLAS for six model LODs and update the scene TLAS from ECS transforms. RT can follow the selected raster LOD or use LOD 0.
- **`gpu_timing.hpp` / `.cpp`** and **`gpu_timing_math.hpp`** provide fence-protected, demand-driven timestamps and timing calculations. CPU frame wall time, GPU rendering cost and driver latency reports are distinct metrics.
- **`imgui.hpp` / `.cpp`** initialize Dear ImGui's Vulkan backend and record output-resolution UI. The demo defines the actual HUD, diagnostics and settings contents.

## `src/scene/` (Assets and ECS)

- **`asset_pool.hpp` / `.cpp`** register models/textures, remap global material/texture slots and upload immutable material and buffer-address model tables once. After remapping, emitter templates cache emission and the global emissive texture slot for ReSTIR proposals.
- **`model.hpp` / `.cpp`** parse GLB, validate shared payloads before geometry allocation, upload geometry and build per-LOD BLAS. Ownership at model set 0 binding 5 is also addressable through the shared table. The 64-byte material includes HDR emission and its texture. LOD-0 emissive triangles supply local light templates with stable surface/triangle/material identity. Ambiguous legacy multi-material payloads require recompilation; nonopaque materials are unsupported.
- **`scene.hpp` / `.cpp`** own frame-safe instance/source/world-light buffers, capacity reservation, previous transforms and successful-submission reference accounting. **`scene_math.hpp`** shares visibility packing, transform validation, sampling and reference helpers with CPU tests. Scene GPU layouts have explicit size assertions.
- **`meshlet_payload.hpp`** shares versioned CPU serialization, validation and ownership remapping; **`meshlet_builder.hpp`** supplies per-primitive, border-locked CPU LOD/meshlet generation to the compiler and tests.
- **`texture.hpp` / `.cpp`** decode standard images with stb_image, upload embedded KTX2 BC7/BC5 data using libktx, generate mipmaps and create anisotropic samplers.
- **`ecs.hpp` / `.cpp`** manage entity IDs and Transform/MeshRenderer components.

## `shaders/` (GPU Programs)

- **`cull.comp`** performs object frustum pre-culling and output-content-height-based screen-space-error LOD selection, then writes mesh-task indirect commands. Whole-object Hi-Z rejection is bypassed to preserve recovery candidates.
- **`shader.task`** performs valid-lane meshlet frustum/cone tests and complete-footprint conservative reverse-Z occlusion. Prediction appends potentially hidden meshlets to a bounded recovery list; overflow renders immediately. Recovery tests that list against current depth.
- **`shader.mesh`** emits geometry with the current jittered render projection and forwards meshlet/primitive IDs.
- **`shader.frag`** writes meshlet index and packed instance/triangle index to the unchanged 64-bit `R32G32_UINT` visibility target. The instance identifies its model and ownership through the global table.
- **`hiz.comp`** builds minimum-reduced reverse-Z pyramids with complete coverage for odd dimensions.
- **`occlusion_setup.comp`** generates GPU indirect arguments for recovery and conditional final-pyramid work; **`occlusion_data.glsl`** shares the associated constants/work-buffer layout.
- **`deferred.comp`** reconstructs attributes and analytical UV derivatives per instance, resolves ownership/materials and writes HDR plus demanded guides. Boulder retains directional sun/ambient; sandbox evaluates one-sided emissive triangle samples and finite shadow rays without ambient fill. The optional reference writes RGBA32f sample sums and their HDR mean.
- **`scene_data.glsl`**, **`surface_access.glsl`**, **`material_eval.glsl`** and **`light_sampling.glsl`** share scene/model layouts, triangle access, PBR/emission, reconstructable samples and visibility helpers for ReSTIR DI. **`prepare_lights.comp`** updates cached world-space triangle vertices, normals and areas when source transforms change.
- **`restir_initial.glsl`** retains a legacy diagnostic path. **`restir_full.glsl`** and **`restir_packing.h`** define shared storage/math; **`restir_temporal_reuse.glsl`** and **`restir_final.glsl`** share temporal and shading functions. **`restir_spatial_one.comp`** handles accepted Basic history; **`restir_spatial.comp`** handles general/disocclusion reuse. Separate temporal/final kernels remain for reference modes and ablations. Current/previous TLAS use frame bindings 12/13. Independent recursive-history checks pass on the tested fixtures; broader sampling-efficiency qualification remains open.
- **`temporal_frame.glsl`** and **`temporal_images.glsl`** define temporal constants and image bindings.
- **`reconstruct.comp`** provides Native spatial reconstruction for reduced diagnostic scales; it does not implement TAA.
- **`present.comp`** converts HDR to display output, handles the output viewport/aspect bars and diagnostic presentation, and preserves the configured background. Active NVIDIA reconstruction runs before this stage.

## Build, Offline Tools and Verification

- **`CMakeLists.txt`** configures the engine, shaders, asset tools and opt-in focused tests.
- **`cmake/Streamline.cmake`** pins Streamline 2.14.16, verifies/caches the SDK, supports `VKE_ENABLE_DLSS=OFF` native-only builds, and stages NVIDIA DLLs/notices beside the executable.
- **`tools/model_compiler.cpp`** produces MLOD v2 GLBs with primitive ownership, six independent LODs and maximum primitive error. Explicit glTF inputs receive ground only with `--demo-ground`; directory-mode boulder compilation retains it. Emission strength/textures are preserved; conflicting color/data/normal image uses are split before KTX compression. Full scene graphs and alpha remain pending. **`tools/sandbox_assets.cpp`** deterministically generates the sandbox sources for the `sandbox_assets` target.
- **`tools/texture_compiler.cpp`** performs offline texture conversion.
- **`tools/pak_compiler.cpp`** packages shaders, models and textures into the indexed `engine.pak` archive.
- **`package_release.ps1`** configures a `VKE_BUILD_TESTS=OFF` Release build, stages/strips the executable, packs assets/shaders into `engine.pak`, and produces a portable ZIP with NVIDIA Streamline/DLSS/Reflex DLLs, notices, and Microsoft VC++ runtime DLLs. The `v0.3.0` Phase 5 portable package is built and verified locally on RTX 3090; clean-machine non-development verification remains Phase 5.1 work.
- **`tests/`** contains eleven CPU/offline suites and opt-in GPU reference, smoke, jitter and profiling probes. The light-count harness holds geometry fixed; shader-diagnostic mode selects ignored experimental modules only when explicitly requested. Normal startup uses production modules. Release supplies performance observations; Debug and forced-validation runs supply synchronization-validation evidence. See [verification and limits](#verification-and-limits).

## Verification and Limits

**As of 2026-11-14 (`v0.3.0`), Phase 5 is complete.** Core lighting, emission payload caching, 48-byte packed cached surfaces, recursive history, equal-cost multi-view audits, full-resolution motion/jitter checks, forced synchronization validation, sustained 3440x1440 Mailbox throughput measurements, and `VKE_BUILD_TESTS=OFF` Release packaging all pass on the RTX 3090 baseline. Benchmarks run in MinGW Release with validation layers disabled; Debug / `VKE_FORCE_VALIDATION=1` runs provide Vulkan synchronization-validation evidence. The pinned runtime is Streamline 2.14.16 with model K DLAA/DLSS; RR, Frame Generation, standalone TAA, GI and production denoising belong to Phase 5.1 / future work.

### Current checks

- **Clean Release & offline suites (`2026-11-14`):** All eleven CPU/offline `ctest` suites pass (`build/nsight-release`), all 15 SPIR-V shaders pass `spirv-val --target-env vulkan1.3`, and the `VKE_BUILD_TESTS=OFF` Release build (`VK_game_engine.exe` + `pak_compiler.exe`) compiles cleanly with zero unexpected external MinGW DLL dependencies (`23` files packed into `26.17 MB` `engine.pak`).
- **Equal-cost multi-view audits (`2026-11-14`, 48B packed build, `3440x1440`):** `MC7` (`2.61 ms` lighting) and `MC8` (`2.87 ms`) bracket `ReSTIR4` (`2.68 ms`) in `Sandbox64Interior` Native Mailbox. Across 16 independent trajectories × 256 frames vs. `MC8` (all region/tile/material gates `gate=1`), `ReSTIR4` achieves **37.2% lower raw RGB MSE** (`0.00171758` vs. `0.00273713`) and **77.9% lower raw luminance MSE** (`0.000378462` vs. `0.00171114`) in `Sandbox64Interior`, and **42.3% lower raw luminance MSE** (`0.000915713` vs. `0.00158707`) in `Sandbox64Exterior`, while `MC8` leads in 3-emitter `Sandbox3Interior` where 8 samples exceed the 6 light triangles.
- **Full-res motion, jitter & forced synchronization validation (`2026-11-14`, `ReSTIR4`):** `3440x1440` `Motion quality` (`ReSTIR4` vs. `ReSTIR8`, including `7,016` revealed disoccluded receiver pixels), `3440x1440` `Jitter smoke`, and `Full DI` Basic smoke under `VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT` all pass with zero validation/sync-hazard errors.
- Earlier November 1–7 development checks also established 16-trajectory `SpatialFeedback` vs. `MC1` efficiency (`~6.4x`/`20.6x` raw RGB/luminance MSE reduction at equal final-shadow budget) and debug/reference regression coverage. Raw stochastic noise remains in un-denoised Native views; aggregate agreement does not prove zero local error on arbitrary scenes.

### History and memory

Final spatial reservoirs become next-frame ReSTIR history by default. The earlier brightening failure did not reproduce with the current Basic implementation and full GPU suite. New independent-trajectory CPU/GPU checks pass without widening reference tolerances; they distinguish correlated noise from systematic mean drift. This does not identify a mathematical cause for the historical failure or prove every dynamic case unbiased. F5 retains a temporal-only history comparison. Final output already occupies its history buffer; only the comparison policy rotates temporal scratch ownership. Neither requires a full-resolution copy.

Full-DI storage uses `2 * 48 + 3 * 32 = 192` bytes per internal pixel plus small mapping, light and context allocations. The sandbox allocation totals 177,070,848 bytes at 1280x720 and 951,214,848 bytes at 3440x1440. Other geometry/HDR/guide/reference/SDK targets are additional. Inactive legacy initial-only paths bind small stubs. Allocation failure is reported and falls back to conventional sampling.

Temporal guides use previous-minus-current unjittered pixel motion, reverse-Z depth and linear HDR. Surface identity uses an RGBA32 unsigned image containing entity, surface, raster LOD and global material. Static poses avoid previous-position reconstruction; moving poses use existing transform matrices. Sandbox CPU round trips are bounded by the tests; extreme scale/large-world precision is not established. Physical poses, ReSTIR snapshots and Hi-Z depth history have separate reset/validity contracts.

### Performance and diagnosis

Historical Nov 1–5 3440x1440 DLAA Release measurements on the RTX 3090 reached approximately 200 FPS starting view, 168 FPS three-emitter interior and 142–145 FPS 64-emitter interior before packed surface/BSDF optimizations. Following Phase 5's 48B packed surface + 16B `MaterialUniform` refactor (`v0.3.0`), sustained Mailbox benchmarks on the same RTX 3090 reached `174.3–174.5 FPS` DLAA, `250.9–251.1 FPS` DLSS Quality, and `267.4–268.3 FPS` Native (`276.892 FPS` in the Nov 14 equal-cost calibration) in `Sandbox64Interior`, and `>320 FPS` DLAA in the Boulder showcase. F4/F5 request timestamps and output CSV logging, while global per-pixel atomic counters are separately opt-in.

A thirty-case light-count sweep kept all 68 instances, geometry, transforms and settings fixed. The interior initial/material/temporal pass rose from about 1.9–2.2 ms at one active emitter to 2.9–3.1 ms at 64, while spatial/final work stayed around 1.5–1.7 ms. Every accepted case had 360/360 focused frames and 360 asynchronous GPU samples. Total illuminating power changed with count; disabled emitters retained visible emission in this deliberately diagnostic scene.

Twenty-eight shader-ablation cases implicated varied light access and target evaluation, with substantial temporal cost as well. These test-only modules live in ignored build output and are selected only by explicit profile flags. They change sampling/reuse and are not production optimizations. A glslc `-O` trial showed no reliable gain; the production shader remained unchanged. No installed hardware shader profiler was found, so cache stalls, occupancy and register pressure were not directly measured.

Repeated-run telemetry recorded changing graphics clocks, roughly 1545 to 1365 MHz during the confirmation sweep, with GPU temperatures around 78–81 C. This establishes clock variation, not a particular throttling cause. Timestamp-only performance probes have no per-frame image/buffer captures or additional device-idle waits; readback correctness tests must not be interpreted as interactive FPS measurements.

November 1 sustained testing supplies a more specific driver observation: `SW Thermal Slowdown` was active at approximately 82–83 C, including 1275 and 1080 MHz samples, while software power capping was inactive. This confirms thermal clock limiting during those samples, without identifying a cooling fault or unreported memory temperature. No hardware settings were changed. An eight-case balanced history-policy comparison measured 6.753 ms total GPU for temporal-only history versus 6.786 ms for final-spatial feedback, roughly 0.5% apart amid drift; no reliable performance difference is claimed.

Two exact-emission cache candidates passed sampled HDR parity/reference checks but failed to demonstrate a reliable speedup. The first was slower; the second was approximately flat to 1% better under clock drift. Both were removed from normal rendering. Production light records and material-table evaluation remain unchanged; ignored patches/logs retain the experiment. The cleaned Release build and all eleven CPU suites pass, and its final Basic GPU reference/motion/resize suite passes with recursive history enabled.

A subsequent rough-dielectric importance-target prototype passed independent few/many-light and heavy-interior regional/material reference checks. Its apparent 2–3% GPU-time gain against its own full-target branch narrowed to about 1.5% against original shader modules, amid a 33% drift in unchanged DLAA cost. No reliable gain was established, so the prototype and runtime controls were removed. Production retains the original full-BRDF luminance target. Material-level audits, arbitrary-positive-target CPU tests and opt-in validation of the optimized Release binary remain; no denoiser, GI or reduced shadow/sample budget was introduced.

### Remaining integration work

Phase 5 is closed as `v0.3.0` (2026-11-14). Phase 5.1 carries RR evaluation, Cornell/Sponza integration, extended image-quality and memory-soak review, current-loader Reflex latency measurements, broader hardware coverage and clean-machine/RR packaging verification. No denoiser is required to close Phase 5.

Historical reconstruction/Reflex runs verified SDK calls and rendering, not a latency benefit. Their canonical interposer filename and approximately 200 FPS pacing ceiling predate the current signed-copy loader. The current backend uses a signature-verified `vke_sl_interposer.dll` copy; those older results do not qualify this workaround or prove a current throughput/latency improvement.

## How They Connect

1. **Initialize:** VFS, NVIDIA backend, window/device, swapchain/pipelines and frame resources are prepared. Streamline initialization precedes Vulkan instance/device creation.
2. **Preload scenes:** boulder and sandbox assets enter GPU storage; immutable model/material tables and six-LOD BLAS support shared instances. Reserve scene/light/recovery capacity before rendering.
3. **Schedule and sample input:** wait for the reusable frame slot, obtain a runtime frame token, apply Reflex options/sleep, mark simulation start and poll GLFW input. Update camera/object/settings state and mark simulation end.
4. **Acquire/configure:** apply render-target changes, acquire a swapchain image and retire its previous use, update descriptors and prepare temporal/Hi-Z constants. Acquisition can still wait after input sampling.
5. **Cull and rasterize:** select each instance's LOD, update the TLAS and prepare changed lights. Write visibility/depth; when prediction is active, recover each instance's deferred meshlets against current depth before shading.
6. **Shade:** resolve instance/model/ownership, evaluate boulder sun or sandbox emissive lighting, and write HDR. Generate only demanded guides; optionally accumulate the Native diagnostic reference.
7. **Reconstruct and display:** active DLAA/DLSS evaluates HDR/depth/motion inputs. Native at 100% uses current HDR directly; reduced Native diagnostics use spatial reconstruction. Tone mapping/display conversion precedes output-resolution F3/F4/F5 and menu rendering.
8. **Finalize and submit:** refresh Hi-Z as required, capture paired frozen data when requested, submit/present with latency markers and supported frame IDs, then commit successful-submission history. Presentation recreation invalidates the affected histories.

See the [pipeline flowchart](flow.md) for data flow and the [roadmap](roadmap.md) for current milestones and open work.
