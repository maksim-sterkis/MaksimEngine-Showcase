# Vulkan Engine Roadmap

This plan outlines the next major architectural upgrades for the engine, focusing on bindless textures, PBR materials, advanced culling, hardware ray tracing, and next-gen lighting.

**Phase 5 was completed on 2026-11-14 (`v0.3.0`).** The sandbox ReSTIR DI pipeline uses 48-byte cached surfaces, emission payload caching, and scalar/fused execution. Across 30 sustained 3440x1440 Mailbox runs (226,201 frames), `Sandbox64Interior` averaged 174.3–174.5 FPS DLAA, 250.9–251.1 FPS DLSS Quality, and 267.4–268.3 FPS Native (`Boulder` > 320 FPS DLAA). Equal-cost multi-view audits (`ReSTIR4` vs. `MC8` / `MC7`), full-res motion/jitter checks, forced synchronization validation, `11/11` `ctest`, `15/15` `spirv-val`, and a clean `VKE_BUILD_TESTS=OFF` Release + `dist` package all pass. [File architecture](file.md) and [pipeline flowchart](flow.md).

**Phase 5.1 is in progress.** The DLSS 4.5 Ray Reconstruction integration and evaluation, native Vulkan NVIDIA NRD (`RELAX`, `REBLUR`, and `SIGMA`), the two-room Lighting Sandbox, Cornell Box, Intel New Sponza (2022), output-resolution emitter presentation, the Win32 Mailbox pacing fix, and independent DLSS reconstruction resolution with spatial presentation upscaling and live Advisor (`Task 8`) are implemented on the RTX 3090 `3440×1440` baseline. Clean-machine distribution verification and multi-GPU/driver testing (`Task 9`) remain open. [Phase 5.1 results](#phase-51-dlss-ray-reconstruction-nvidia-nrd-denoisers-benchmark-scenes--integration-validation)

## Phase 0 (Asset Pipeline & KTX2 Compression)
- **Status**: **COMPLETED**
- **Goal**: Built an offline model compiler that parses `.gltf` and `.obj` files using `fastgltf` and `tinyobjloader`, compresses raw textures into `KTX2 UASTC` format using `toktx`, and packages them into binary `.glb` files.
- **Why**: Allows the engine to load Poly Haven PBR assets with pre-compressed KTX2/BC7 textures streamed to VRAM, reducing memory footprint.

## Phase 0.5 (Fix GLB Texture Embedding)
- **Status**: **COMPLETED**
- **Goal**: Ensure the textures are properly embedded into the `.glb` binary chunk rather than saving as external URIs.
- **Why**: The debug logs revealed `index=2` (`fastgltf::sources::URI`), which means `fastgltf::Exporter` is trying to write the KTX2 textures out as separate files instead of packing them into the `.glb` binary! This is why `ktxTexture2_CreateFromMemory` is failing—the byte array is empty because the data isn't in the buffer chunk. We will fix `tools/model_compiler.cpp` to properly serialize the textures into a `BufferView` within the GLB so it behaves as a single self-contained file.

## Phase 1 (Bindless Architecture & SSBOs)
- **Status**: **COMPLETED**
- **Goal**: Transition from tight per-draw descriptor binding to global descriptor arrays (Bindless) and SSBO-driven material properties.
- **Why**: Eliminates CPU overhead when drawing thousands of objects. Crucial for upcoming Mesh Shaders and Ray Tracing which require arbitrary access to vertex and material data globally.
- **Tasks Completed**:
  1. Added `VK_EXT_descriptor_indexing` (Vulkan 1.2 Descriptor Indexing) to Device features.
  2. Created a Global Descriptor Set containing unbounded arrays of `sampler2D` and SSBOs for Vertex/Index data.
  3. Modified shaders to accept a `MaterialID` and `MeshID` via PushConstants to index into the global SSBOs/Textures, removing texture bindings from the `vkCmdBindDescriptorSets` loop entirely.

## Phase 2 (PBR Materials)
- **Status**: **COMPLETED**
- **Goal**: Implement physical based rendering (Cook-Torrance BRDF) evaluating Albedo, Normal, Metallic, and Roughness maps.
- **Why**: Required to accurately represent realistic assets like the Poly Haven rock, calculating specular reflections and energy conservation correctly.
- **Tasks Completed**:
  1. Updated Material SSBO to include PBR coefficients and correctly aligned memory using `std140` block limits.
  2. Implemented Schlick-GGX, Smith Geometry, and Fresnel equations in `shader.frag`.
  3. Extracted and mapped multi-texture PBR payloads from GLB models dynamically to the Bindless array.

## Phase 3 (Meshlets & Advanced Culling)
- **Status**: **COMPLETED**
- **Goal**: Replace legacy vertex pipelines with Task/Mesh shaders (`VK_EXT_mesh_shader`) featuring sub-mesh level Frustum, Cone Backface, and Hi-Z Occlusion Culling.
- **Why**: Enables GPU geometry culling at a sub-mesh granularity, evaluating meshlets efficiently in task shaders.
- **Tasks Completed**:
  1. Offline tool extension to partition `ModelData` into meshlets (max 64 vertices, 124 triangles) using `meshoptimizer`.
  2. Create Task and Mesh shaders to evaluate bounding spheres, extract frustum planes, and emit visible meshlets.
  3. Generate and bind a multi-mip Hi-Z depth pyramid for conservative occlusion culling.
  4. Implement SSBO atomic counters to safely track evaluated vs. drawn meshlets without driver performance stalls.

## Phase 3.5 (Visibility Buffer & Pipeline Optimization)
- **Status**: **COMPLETED**
- **Goal**: Transition the engine to a true Visibility Buffer architecture and plug rendering bottlenecks.
- **Why**: Decouples geometry rasterization from heavy material evaluation. By outputting only a 64-bit payload per pixel, material shading runs only on visible pixels, eliminating redundant fragment shading on occluded geometry.
- **Tasks Completed**:
  1. Replaced hardware fragment shader barycentric extension requirement with analytical 2D screen-space barycentric reconstruction in compute, maximizing compatibility across NVIDIA, AMD, and Intel GPUs.
  2. Created `VK_FORMAT_R32G32_UINT` Visibility Buffer render target with proper layout transitions and barriers.
  3. Stripped `shader.mesh` to minimal vertex/primitive payload (`outMeshletIndex`, `perprimitiveEXT outPrimitiveID`), with `shader.frag` writing 64-bit IDs.
  4. Built fullscreen `deferred.comp` pass: fetches 3 vertices from GPU SSBOs, calculates exact 2D screen-space barycentrics in NDC, performs perspective-correct attribute reconstruction, and samples base albedo without redundant fragment shading and with full validation layer compliance.
  5. Built GPU Whole-Object Compute Pre-Culling: added `cull.comp` evaluating 6-plane frustum culling and conservative multi-mip Hi-Z occlusion culling on instance bounding boxes, populating `VkDrawMeshTasksIndirectCommandEXT` into indirect SSBOs and issuing GPU-driven `vkCmdDrawMeshTasksIndirectEXT`.
  6. **6-Level Discrete Mesh LODs & Texture Mipmaps**: 
     - Added offline Pos+UV topological pre-welding to eliminate normal-split vertex locks without touching UV seams.
     - Preserved UV chart boundaries with progressive `meshopt_simplify` across 6 discrete LOD levels (100%, 50%, 25%, 12.5%, 6.25%, 3.125%) to preserve silhouette contours and texture mapping.
     - Calculated exact 3D triangle edge length $L_{\text{tri}}$ and implemented screen-space triangle pixel projection math in `cull.comp` (stepping up LOD when triangles project $< 3.0\text{ px}$).
     - Implemented dynamic texel footprint calculation and hardware trilinear mipmapping in `deferred.comp` with active LOD clamping, F2 camera freeze, and manual F3 ImGui overrides.

## Phase 3.6 (Virtual File System & Release Packaging)
- **Status**: **COMPLETED**
- **Goal**: Implement a Virtual File System (VFS) and an automated production release pipeline.
- **Why**: Eliminates loose shader and asset dependencies, packs game assets into an indexed, single-binary archive (`engine.pak`) for distribution, and enables zero-dependency portable deployment on clean machines.
- **Tasks Completed**:
  1. Designed and built `vfs::initialize()`, `read_file()`, and `get_file_span()` utilizing memory-mapped files (`CreateFileMapping` / `MapViewOfFile`) on Windows with transparent fallback to loose disk files for dev mode.
  2. Implemented `tools/pak_compiler.cpp` to bundle SPIR-V shaders, GLB models, and KTX2/raw textures into contiguous `engine.pak` archives with 128-byte normalized path headers.
  3. Created `package_release.ps1` to automate CMake Release configuration, `-O3` compilation, symbol stripping (`strip -s`), static MinGW runtime linking (`-static -static-libgcc -static-libstdc++`), and portable `.zip` generation.
  4. Added GPU scoring in `src/device.cpp` to automatically prioritize discrete GPUs over integrated graphics on hybrid systems.
  5. Gated validation layers behind `NDEBUG` so packaged release builds run cleanly without requiring the Vulkan SDK or Khronos validation layer runtime.

## Phase 3.7 (Robust Meshlets, Analytical VisBuffer Filtering & Ergonomics)
- **Status**: **COMPLETED**
- **Goal**: Eliminate LOD meshlet boundary tears, remove texture mip-popping with analytical screen-space derivatives, add persistent settings, and overhaul in-game diagnostic telemetry.
- **Why**: Provides artifact-free geometry transitions across all 6 LODs, seamless anisotropic texture filtering without artificial mip-clamping, persistent player preferences, and non-intrusive HUD and profiler tools.
- **Tasks Completed**:
  1. **Seam-Free Meshlet Generation & Simplification**:
     - Upgraded `tools/model_compiler.cpp` to utilize `meshopt_simplifyWithAttributes` preserving UV chart boundaries with attribute weighting and `meshopt_SimplifyLockBorder`.
     - Integrated `meshopt_optimizeMeshlet` across all generated clusters to optimize vertex cache locality and eliminate internal indexing gaps.
     - Added 5% conservative safety margins and loosened cone cutoff limits on simplified silhouettes to prevent boundary meshlets from false-positive culling.
  2. **Analytical Screen-Space UV Derivatives & Zero-Popping Mipmaps**:
     - Completely eliminated artificial discrete `max(mipLevel, float(activeLod))` clamping in `shaders/deferred.comp`.
     - Derived closed-form analytical screen-space partial derivatives $\frac{\partial \text{UV}}{\partial x}$ and $\frac{\partial \text{UV}}{\partial y}$ from 2D barycentric gradients and perspective weights.
     - Implemented hardware anisotropic and trilinear filtering via `textureGrad`, enabling smooth continuous mipmap transitions with zero popping between LOD transitions.
     - Enabled 16x anisotropic filtering in Vulkan device features and samplers.
  3. **Persistent Settings & ESC Pause Menu**:
     - Created `src/core/settings.hpp` / `.cpp` with an INI parser/serializer persisting to `settings.ini`.
     - Built modal ESC Pause Menu (Resume, Settings, Exit to Desktop) with cursor release.
     - Added Settings menu managing Window Mode, VSync, Aspect Ratio Mode, Meshlet Debug Colors, Freeze Culling, Manual LOD Overrides, and Mouse Sensitivity.
  4. **Non-Interactive F3 HUD & Hotkey LOD Switching**:
     - Converted F3 into a click-through, non-blocking telemetry HUD overlay (`ImGuiWindowFlags_NoInputs`) displaying real-time FPS, 1% Lows, latency, active LOD, task/mesh invocations, and % culled.
     - Bound number keys `0` (Auto) and `1`–`6` (LOD 0–5) for instant on-the-fly manual LOD switching without menus.
  5. **F4 Performance Telemetry & Diagnostic Graphs**:
     - Added F4 dashboard featuring rolling history `ImGui::PlotLines` graphs for FPS, frame time latency (ms), and meshlet culling efficiency.
  6. **Robust Occlusion Culling, Edge Guards & Pipeline Diagnostics**:
     - Converted Hi-Z occlusion testing in `shaders/shader.task` from non-linear NDC to real-world metric eye distance (`mesh_linear_dist > hiz_linear_dist + 0.25m`), preventing concave craters from falsely self-occluding against their rims.
     - Implemented viewport perimeter guard (`min_x <= -0.98 || max_x >= 0.98 || min_y <= -0.98 || max_y >= 0.98`) and clamped downsampling mip level (Mip 5 / 32x32) to completely eliminate perimeter popping and coarse mip bleed.
     - Added number hotkeys `7` (Frustum), `8` (Cone), and `9` (Hi-Z) to toggle culling stages in real-time, live-reflected on the F3 HUD alongside atomic culling breakdowns (`[F:%u C:%u H:%u]`).
     - Guarded telemetry counters against `uint64_t` underflows during transient LOD switches, cleanly reporting `100.0%` culled when objects are off-screen.
     - Refined mouse wheel camera speed adjustments to half-unit steps (`0.5`) clamped to `[0.5, 20.0]`.

## Phase 4 (Hardware Ray-Traced Shadows - Ray Queries & Acceleration Structures)
- **Status**: **COMPLETED**
- **Goal**: Implement hardware-accelerated ray-traced shadows using `VK_KHR_acceleration_structure` and `VK_KHR_ray_query` integrated directly into the Visibility Buffer deferred compute pipeline.
- **Why**: Replaces traditional cascaded shadow maps with exact hardware ray-traced directional shadows executed on physical GPU RT cores. Leverages the Visibility Buffer to completely skip primary camera ray traversal, casting coherent shadow rays solely from visible surface pixels in deferred compute using inline ray queries.
- **Tasks Completed**:
  1. **Device & Memory Extensions**:
     - Enabled `VK_KHR_acceleration_structure`, `VK_KHR_ray_query`, `VK_KHR_deferred_host_operations`, and `bufferDeviceAddress = VK_TRUE`.
     - Added `VK_MEMORY_ALLOCATE_DEVICE_ADDRESS_BIT` to buffer allocation in `device::create_buffer` and added `device::get_buffer_device_address`.
     - Appended `VK_BUFFER_USAGE_ACCELERATION_STRUCTURE_BUILD_INPUT_READ_ONLY_BIT_KHR | VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT` to vertex and index buffer allocations in `model::create`.
  2. **Acceleration Structure Infrastructure (`src/renderer/raytracing.hpp` / `.cpp`)**:
     - Built compact Bottom-Level Acceleration Structures (BLAS) for static meshes using LOD 0 vertex positions and indices at asset load time.
     - Implemented dynamic Top-Level Acceleration Structures (TLAS) with persistent host-visible coherent instance buffers, converting ECS entity transforms into 3x4 row-major affine matrices.
     - Synchronized TLAS builds on the main frame command buffer with compute-read pipeline barriers prior to deferred compute resolve.
  3. **Global Descriptor Set Binding**:
     - Added Binding 2 (`VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR`) to `globalSetLayout` with `descriptorBindingAccelerationStructureUpdateAfterBind` support.
     - Bound the active scene TLAS handle to the global descriptor set via `write_tlas_descriptor`.
  4. **Inline Ray Query Shading (`shaders/deferred.comp`)**:
     - Enabled `#extension GL_EXT_ray_query : require` and bound `topLevelAS`.
     - Reconstructed surface world-space coordinates and geometric normals from perspective-correct barycentrics.
     - Cast inline shadow rays via `rayQueryInitializeEXT` / `rayQueryProceedEXT` with normal-offset surface bias to prevent self-shadowing acne.
     - Modulated Cook-Torrance BRDF direct diffuse and specular terms by shadow occlusion while maintaining ambient baseline illumination.
  5. **Multi-LOD RT Acceleration Structures & Dynamic TLAS Pointer Swapping**:
     - Unpacked discrete index buffers for all 6 LOD levels directly from the multi-LOD meshlet stream in `model::load_glb`.
     - Built discrete Bottom-Level Acceleration Structures (`lodBlas[0..5]`) for each discrete LOD at load time with zero vertex buffer duplication.
     - Generalized `raytracing::build_blas` for explicit buffer pairs and extended `InstanceRT` with `VkDeviceAddress blasAddress`.
     - Implemented dynamic per-frame TLAS pointer swapping in `cull_scene` based on real-time screen-space LOD evaluation (or manual overrides `0`–`6`), ensuring ray queries test against the exact geometric silhouette seen on screen with zero visual acne. Added runtime toggle `Y` allowing instant switching between tracing against active visible LOD (default) and ground-truth LOD 0.
  6. **Interactive Controls & Runtime Diagnostics**:
     - Hotkey `T` toggles hardware ray-traced shadows on and off; Hotkey `Y` toggles ray tracing against the active visible LOD (default) vs. ground-truth LOD 0.
     - F4 HUD exposes interactive sun azimuth, elevation, intensity controls, and `Match Visible LOD [Y]` checkbox, live-reporting active RT BLAS level, triangle count, and mode (`Hardware RTX: BLAS L%u (%u tris) [%s]`).
     - F3 HUD live-reports active RT BLAS level and triangle count alongside rasterized mesh LOD (`RT BLAS: L%u (%u tris)`).
     - Dynamic sun azimuth and elevation controls with real-time HUD and F4 performance telemetry status.
  7. **Sub-Pixel Screen-Space Geometric Surface Error Metric**:
     - Replaced empirical triangle edge length heuristic with rigorous sub-pixel screen-space geometric surface error metric.
     - Upgraded `tools/model_compiler.cpp` to calculate true cumulative geometric deviation in meters (`meshopt_simplifyScale` * `resultError`) across all 6 LOD levels.
     - Updated `shaders/cull.comp` and CPU telemetry to compute projected surface deviation:
       $$\text{projectedErrorPixels} = \frac{\text{surfaceError} \times \text{viewportHeight}}{2 \times \text{dist} \times \tan(\text{fov} / 2)}$$
       Transitioning to coarser LODs only when visual surface deviation stays strictly $\le 1.0\text{ px}$.
  8. **Embedded KTX2 Textures in Self-Contained GLB**:
     - Upgraded `tools/model_compiler.cpp` to embed pre-compressed KTX2 BC7/BC5 texture payloads directly into binary `.glb` chunks (`BufferView`) rather than referencing external files.
     - Implemented in-memory KTX2 streaming in `src/scene/texture.cpp` via `ktxTexture2_CreateFromMemory` and `ktxTexture2_IterateLevels`.
     - Added CLI parameter parsing (`--force`, explicit input/output files) to the model compiler for flexible asset iteration.
  9. **F4 Performance Dashboard & Telemetry Stabilization**:
     - Overhauled F4 telemetry into a clean Frame Time (ms) rolling graph sampled in frozen 5ms time slots (2048 slots), eliminating horizontal phase shimmering and frame jitter.
     - Dynamically bound graph ceiling to highest visible frame time (+10% headroom) that immediately recovers once frame spikes scroll off screen.
     - Fixed center-seam phase inversion artifact by passing exact positive widths to `ImGui::PlotLines`.
     - Displayed live `Low`, `Avg`, and `Max` frame times with clarification disclaimer.
     - Suppressed ImGui's built-in 200 Hz flickering hover tooltip and line highlights via `ImGuiItemFlags_Disabled` without altering visual styling.
  10. **Camera Positioning & Dynamic Target Alignment**:
     - Set default camera position to `(3.0f, 1.0f, 0.0f)`.
     - Dynamically calculated starting camera pitch and yaw from the model's true 3D bounding box center (`(aabbMin + aabbMax) * 0.5f`) so the target geometry is centered upon engine boot.

## Phase 4.1 (Temporal Rendering, DLSS & Reflex)
- **Status**: **CLOSED** (implemented foundation retained; all unfinished work moved to Phases 5 and 5.1). Closure reflects the revised scope, not completion of the outstanding validation or distribution checks.
- **Goal**: Establish the temporal rendering foundation, model K DLAA/DLSS Super Resolution, independent Reflex controls, and presentation/performance diagnostics for the boulder demo.
- **Why**: Supports image stability and lower-resolution rendering, and supplies the initial temporal infrastructure for Phase 5 direct lighting and Phase 5.1 denoising/reconstruction. Performance gains must be measured per scene.
- **Implemented Work**:
  1. **Temporal Rendering Foundation**:
     - **Status**: **IMPLEMENTED FOR THE BOULDER DEMO**. Camera jitter, unjittered motion vectors, previous camera/object transforms, full object transforms and inverse-transpose normals are in place.
     - Lighting uses linear HDR targets with independent internal/output dimensions; reconstruction precedes tone mapping and output-resolution UI. History resets and surface-validity checks cover cuts, target/mode changes, disocclusion and LOD changes.
     - Ordinary Native rendering is unjittered and has no temporal color accumulation or standalone TAA resolve. Native jitter/scale previews are session-only diagnostics; active DLAA/DLSS controls jitter and input dimensions automatically.
     - Ordinary Native color skips optional temporal guides; NVIDIA modes and diagnostic views generate the guides they consume. Empty scenes explicitly initialize HDR, and reconstruction switches retain the swapchain and reuse same-size targets. Unrelated Settings Apply changes preserve reconstruction history.
     - **Hi-Z repair implemented**: independent depth history, unchanged-view single pass, predictive/recovery culling for movement and jitter, conservative odd-size pyramids, valid task lanes and paired F2 capture. The original 216 exact depth/HDR comparisons and a further 144 Quality/K comparisons matched Hi-Z-off references.
     - Debug CPU tests, synchronization-validation smoke and automated minimize/restore checks passed on the RTX 3090. See [rendering architecture and verification limits](file.md) for the current pipeline and recorded coverage.
  2. **DLAA & DLSS Super Resolution Integration**:
     - **Status**: **MODEL K IMPLEMENTED** through pinned Streamline 2.14.16 on Windows/MinGW with Debug validation and Release performance builds. Modes are Native, DLAA, DLSS Quality, Balanced and Performance. Native is the first-launch rendering default; K is fixed for all NVIDIA modes.
     - Persistent settings retain requested mode separately from effective mode. Runtime capability checks and missing-runtime/evaluation-failure handling fall back to Native with a visible reason. A native-only build is supported.
     - **DLSS 4.5 M/L integration removed**; obsolete model settings are ignored. Standalone TAA, Frame Generation, Ray Reconstruction and dynamic resolution are outside the implemented scope.
     - Real RTX 3090 evaluations, CPU tests, synchronization-validation checks, source-image comparisons and Native fallback checks passed. Earlier Debug measurements put reconstruction near 1.3 ms on the small boulder scene, exceeding the lighting savings; they do not establish a throughput gain or a speedup from the later cleanup.
     - See [architecture and verification limits](file.md) for runtime staging, recorded checks and remaining validation.
  3. **Reflex, Presentation & Diagnostics**:
     - **Status**: **IMPLEMENTED**. Settings provides independent NVIDIA Reflex Off / On / On + Boost, defaulting to On when supported, for Native and all four NVIDIA reconstruction modes. Effective status/fallback, scheduling sleep, latency markers and driver timing are exposed; the application Reflex FPS limiter is disabled.
     - Presentation is a persistent Mailbox / Immediate / VSync (FIFO) selector with Apply/Accept/Cancel and legacy VSync migration. Mailbox is the default; Immediate is explicit and may tear. Requested and actual presentation modes are reported separately.
     - F4 is a compact passive performance overlay with mode, dimensions, key GPU timings and frame-time graph. F5 holds interactive reconstruction/temporal, Reflex, RT/lighting, geometry/culling and detailed timing diagnostics. The RT-on/off estimate includes lighting-path differences and is not isolated ray-traversal cost.
     - The fixed-window 15-combination reconstruction/Reflex synchronization-validation smoke passed; missing Reflex retained working DLAA, and native-only operation passed. Driver timing is not full input-to-display latency, and no latency reduction is established by these runs.
     - **Mailbox pacing workaround implemented**: the current backend loads a signature-verified copy of Streamline as `vke_sl_interposer.dll` to avoid the canonical-name-triggered throttle. Earlier 3/4/5-image tests and the Reflex matrix remained near 200 FPS at 3440x1440 before this change; they do not measure the current workaround. The normal swapchain image-count policy is retained. Current Reflex latency benefits still need measurement; the [integration notes](file.md#verification-and-limits) distinguish the implemented workaround from the historical canonical-filename results.
- **References**:
  - [NVIDIA DLSS integration overview](https://developer.nvidia.com/rtx/dlss)

## Phase 5 (Emissive Lighting Foundation & ReSTIR DI)

- **Status: COMPLETED (`v0.3.0`, 2026-11-14).** Initial candidates (`4` default), temporal reuse, spatial reuse (`1` Basic proposal), and fresh final visibility run on the opaque lighting sandbox with 48-byte packed cached surfaces and emission payload caching. Final spatial history is retained directly, sustained multi-view Mailbox benchmarks, equal-cost multi-view comparisons (`ReSTIR4` vs. `MC8`), full-resolution motion/jitter/sync-validation checks, and a clean `VKE_BUILD_TESTS=OFF` Release build all pass. See [architecture and verification](file.md#verification-and-limits).
- **Goal**: Establish surface/material ownership, opaque instances, emissive direct lighting, an independent reference and independently tested ReSTIR DI.
- **Scope**: Direct illumination and shadows. No GI, bounced light or indirect color bleeding. Denoising, benchmark integration for Cornell Box and Intel New Sponza, and clean-machine multi-GPU validation belong to Phase 5.1.

### Implemented

1. **Baseline and surface ownership**: MLOD v2 retains source primitive/material ownership through six per-primitive LODs with locked boundaries and maximum primitive error. Static demo ground is retained. Payload validation precedes GPU allocation; ambiguous legacy multi-material payloads require rebuilding. The checked-in boulder remains unchanged. Temporal identity is entity/surface/raster-LOD/material in an RGBA32 unsigned image.
2. **Opaque instances and emission**: Shared buffer-address model tables, per-instance transforms/LOD/culling, motion and current/previous TLAS support the sandbox. Emissive factors, strength and textures feed one-sided triangle lights with stable identities. World-space light data is updated when changed. General imported scene graphs and alpha handling remain pending.
3. **Controlled scene and reference**: Room, shared cubes, rough metal and three/64 emitters are preloaded. Boulder is the startup default; Esc switches scenes and F5 exposes motion, sample and identity controls. Conventional MC and an unjittered Native 4096-spp direct-light reference remain independent of ReSTIR and display reconstruction.
4. **ReSTIR DI stages**: Initial RIS, light-index mapping, current/previous surface/light/geometry snapshots, temporal rejection, spatial reuse and final shading are implemented. Basic real-time correction uses unoccluded source targets; the ray-traced reference policy evaluates source visibility. Both retain fresh final shadow queries. No visibility cache or radiance filter is enabled.
5. **Traffic and execution improvements**. GPU reservoirs are 32 bytes and preserve FP32 targets and weights. The 48-byte cached surfaces keep FP32 position/depth and exact IDs, with packed normals and material values. Scalar stable-history reuse, separate disocclusion work and fused Basic passes reduce traffic. Final spatial output is retained directly as history, and the temporal-only comparison uses copy-free buffer rotation. The sandbox full-DI allocation at 3440x1440 is 951,214,848 bytes, excluding other renderer/SDK targets.
6. **Recorded verification**: Eleven CPU/offline suites pass in MinGW Debug and Release. Debug synchronization-validation Basic/reference suites cover reuse, source correction, motion, lights, topology, cuts, LOD, area changes, modes and resizing. Release functional GPU checks also pass. The stationary Native/DLAA jitter regression measures approximately 0.000529 rows of donor drift per eight-frame DLAA cycle; it does not prove noise-free output. The current-loader boulder baseline is a later checkpoint, not a retroactive pre-change GPU baseline.

### Current performance evidence

- Short RTX 3090 Release observations at full **3440x1440 DLAA**, eight candidates, one Basic spatial proposal, shadows and F4/F5 reached approximately **200 FPS starting view**, **168 FPS three-emitter interior**, and **142–145 FPS 64-emitter interior**. Longer normal-shader repetitions across views/clocks ranged roughly **124–200 FPS**. These are development observations, not sustained minima or hardware-independent expectations.
- A controlled sweep held all 68 instances, materials and transforms fixed while sampling 1, 3, 16, 32 or 64 emitters. Thirty accepted cases recorded 360/360 focused frames and 360 asynchronous GPU samples each. Initial/material/temporal cost increased from roughly 1.9–2.2 ms at one emitter to 2.9–3.1 ms at 64 in the interior runs; spatial/final work stayed much flatter.
- Twenty-eight subsequent shader-diagnostic cases narrowed the count penalty to varied light access and target evaluation, with temporal work also significant. Coherent-light and constant-target variants deliberately change sampling and were not adopted. Compiler-only optimization showed no reliable win. No installed hardware shader profiler was available, so cache stalls and register pressure remain hypotheses rather than measured diagnoses.
- Benchmarks use separate timestamp-only runs with readback probes kept separate from performance measurements.

### Closure Summary (2026-11-14, v0.3.0)

1. **Recursive-history checkpoint passed on the tested scene**: The earlier brightening failure did not reproduce in the current Basic GPU suite. Strict CPU and independent seeded GPU trajectory, regional and time-window checks pass without wider tolerances. Final spatial reservoirs are again the default history source; temporal-only history remains a comparison. Motion, reference, resize and Native/DLAA jitter regressions pass. These results support the current fixtures, not a universal unbiasedness proof or a mathematical identification of the old failure.
2. **Demonstrated useful sampling efficiency at matched cost (`ReSTIR4` vs. `MC7`/`MC8`)**:
   - Earlier November 5 baseline (`ReSTIR8` `4.15 ms` vs. `MC10` `4.18 ms`) was superseded by November 6–7 shader optimizations and November 12 48-byte surface packing.
   - November 14 calibration on the 48-byte packed build (`Sandbox64Interior`, `3440x1440` Native Mailbox, 600 measured frames) measured `ReSTIR4` at `2.67981 ms` lighting (`276.892 FPS`), bracketed tightly by `MC7` (`2.60710 ms`, `282.153 FPS`) and `MC8` (`2.87437 ms`, `263.150 FPS`).
   - Across 16 independent trajectories × 256 frames (`3440x1440` stratified audit, all region/tile/material gates passing): in `Sandbox64Interior`, `ReSTIR4` achieved **37.2% lower raw RGB MSE** (`0.00171758` vs. `0.00273713`), **77.9% lower raw luminance MSE** (`0.000378462` vs. `0.00171114`), and **21.2% lower 256-frame-averaged luminance MSE** (`7.93299e-06` vs. `1.00645e-05`) than `MC8`; in `Sandbox64Exterior`, `ReSTIR4` achieved **42.3% lower raw luminance MSE** (`0.000915713` vs. `0.00158707`), while `MC8` had lower raw RGB MSE (`0.00223663` vs. `0.00368276`) across the bright open floor; in 3-emitter `Sandbox3Interior` (6 triangles), uniform `MC8` outperformed single-sample `ReSTIR4` (`0.000435905` vs. `0.00159357` RGB MSE), confirming the expected few-light vs. many-light crossover.
3. **Qualified sustained performance, temporal/motion checks, and clean `dist` build**:
   - November shader optimizations + November 12 48-byte packed surfaces (`302.34 MiB` saved at `3440x1440`) brought short 600–1,800-frame `Sandbox64Interior` `3440x1440` DLAA runs to `194.5–198.6 FPS` (`4.48–4.57 ms` GPU). Across the November 12 sustained 3440x1440 Mailbox runs (30 runs, 226,201 frames, up to 30 s per run), `Sandbox64Interior` averaged **174.3–174.5 FPS DLAA** (`4.82–4.84 ms` GPU), **250.9–251.1 FPS DLSS Quality** (`3.07 ms` GPU), and **267.4–268.3 FPS Native** (`2.85 ms` GPU), while `Boulder` exceeded **320 FPS DLAA** (`322.9–341.8 FPS`).
   - November 14 verification on the committed 48B build passed `11/11` `ctest`, `15/15` `spirv-val`, `3440x1440` full-res `Motion quality` (`ReSTIR4` vs. `ReSTIR8`, including `7,016` revealed disoccluded pixels), `3440x1440` Jitter regression, forced Vulkan synchronization validation (`ReSTIR4` Basic Full DI), and a clean `VKE_BUILD_TESTS=OFF` Release + `pak_compiler` build (`26.17 MB` `engine.pak`, zero external MinGW DLL dependencies). Cornell Box and Intel New Sponza (2022) integration, DLSS Ray Reconstruction, and extended soak/stress coverage continue in Phase 5.1.

**Completion checkpoint**: Met (`v0.3.0`, 2026-11-14). Basic light sampling, staged ReSTIR DI validation, resolved recursive history, equal-cost multi-view audits, sustained Mailbox benchmarks and clean Release packaging are complete on the RTX 3090 sandbox baseline. Denoising, benchmarks for Cornell Box and Intel New Sponza, and clean-machine multi-GPU validation proceed in Phase 5.1.

**Reference**: [NVIDIA ReSTIR DI integration and light-data requirements](https://github.com/NVIDIA-RTX/RTXDI/blob/main/Doc/Integration.md).

## Phase 5.1 (DLSS Ray Reconstruction, NVIDIA NRD Denoisers, Benchmark Scenes & Integration Validation)
- **Status**: **IN PROGRESS**. Tasks 1–8 are complete on the RTX 3090 `3440×1440` baseline. Task 9 covers clean-machine distribution and broader validation.
- **Goal**: Evaluate DLSS 4.5 Ray Reconstruction, integrate NVIDIA Real-Time Denoisers (NRD: RELAX, REBLUR, SIGMA) for lower-cost / non-RR paths, expand the test scene and build out the Cornell Box and Intel New Sponza (2022) benchmark scenes, and complete integration validation and clean-machine distribution.
- **Why**: Native-resolution RR proved too expensive on the RTX 3090 (~18.5 ms SDK / ~24.1 ms GPU at 3440×1440) to serve as the sole real-time denoiser across all modes and GPUs. Pairing optional RR (for upscaled RTX paths) with analytical NRD compute denoisers covers Native, DLAA, DLSS SR, and portable Vulkan paths without reinventing SVGF from scratch.
- **Scope**: Builds on validated Phase 5 ReSTIR DI. Custom hand-written SVGF remains deferred in favor of NRD; multi-bounce global illumination and color bleeding remain outside scope.
- **Tasks**:
  1. **DLSS Ray Reconstruction Integration, Evaluation & Decision**:
     - **Status**: **COMPLETED**.
     - Integrated DLSS 4.5 Ray Reconstruction (`sl::DLSSDPreset::ePresetF`) via pinned Streamline `2.14.16` / NGX `310.9.16` across DLAA, Quality, Balanced, Performance, and Ultra Performance modes, with explicit capability/signature checks, stop-on-error handling, 32/72 jitter phases, demand-driven RR guides (`rrDiffuseAlbedo`, `rrSpecularAlbedo`, `rrNormalRoughness`, `rrSpecularMotion`), F5 RR guide debug views (`5`–`9`), 4-estimate decorrelated final shading (`75%` fresh RIS probability), and history retention through continuous emitter motion (`0` resets in dynamic audits vs. `15` in DLAA).
     - Verified static/ground shadow quality (3/64-emitter penumbra RMSE `0.00387`/`0.00554`, `0.096%`/`0.154%` dark pixels; 4-angle ground RMSE `0.00258–0.00391`) and dynamic-scene quality (3/64-emitter camera+object+light motion: disoccluded `frame 32` lumRMSE `0.00851`/`0.00677` tightening to `0.00475`/`0.00369` at `frame 33` vs. `0.03064`/`0.02503` DLAA; moving-edge ghosting `<= 1.94%`/`0.72%`; changing-shadow RMSE `0.00564–0.00658`). Native guide views passed forced Vulkan synchronization validation with zero hazards.
     - Quantified RTX 3090 cost and memory (`3440×1440`, `Sandbox64Interior`, 4 candidates): Nsight profiling measured native RR DLAA at `18.493 ms` SDK / `24.076 ms` total GPU (exceeding the `5 ms` ceiling due to 50 CUDA tensor-inference kernels), vs. `8.626 / 11.488 ms` (RR Quality), `7.509 / 9.872 ms` (RR Balanced), `5.996 / 7.949 ms` (RR Performance), and `3.353 / 4.373 ms` (RR Ultra Performance). Process VRAM (`VK_EXT_memory_budget`) was `3626.58 MiB` in RR DLAA (`282.19 MiB` engine RR guides + `~1014 MiB` Streamline/NGX overhead vs. ordinary DLAA's `2330.58 MiB`), `2210.33 MiB` in RR Quality, `1724.14 MiB` in RR Performance, and reclaimed `1076.63 MiB` (`-> 2549.95 MiB`) on switchback to Native.
     - **Denoiser Decision & Policy**: Retain DLSS 4.5 RR as an optional high-quality neural RTX path (best suited to upscaled RR modes where total GPU frame cost is roughly 2–5× lower than native RR DLAA). Keep Native and ordinary model K DLAA/DLSS SR as RR-off / non-RTX paths (explicitly noting they supply no lighting denoising on their own). Adopt **NVIDIA Real-Time Denoisers (NRD)** in Task 2 for lightweight compute denoising across non-RR workloads, keeping custom hand-written SVGF deferred.
  2. **NVIDIA Real-Time Denoisers (NRD: RELAX, REBLUR, SIGMA)**:
     - **Status**: **COMPLETED**.
     - Pinned and statically integrated `NVIDIA-RTX/NRD v4.18.0` (`MathLib v11`, `ShaderMake`, `159` DXC SPIR-V shaders with `NRD_NORMAL_ENCODING=2` `R10G10B10A2_UNORM` and `NRD_ROUGHNESS_ENCODING=1`, MIT notice staged at `nvidia-notices/nrd-license.txt`) with zero extra runtime DLLs via native Vulkan descriptors, barriers, and `nrd_compose.comp` (no NRI).
     - Split direct/ReSTIR lighting into demodulated diffuse and specular radiance alongside `viewZ`, 2.5D `motion`, `R10G10B10A2` normal/roughness (binding `31`), and hit/penumbra distance. RELAX uses four fresh RIS estimates for ReSTIR final shading and guides the diffuse filter with emitter distance. The extra draws stay out of reservoir history, and NRD remains mutually exclusive with DLSS RR.
     - Exposed persistent `Denoiser` settings (`Off`, `NRD RELAX`, `NRD REBLUR`, `NRD SIGMA`), F5 NRD controls and `Temporal view` `10..15` (including `OUT_VALIDATION`), and dedicated `denoiserMs` GPU timestamps.
     - Verified `12/12` CTest, `spirv-val`, and forced Vulkan + sync validation (`VKE_NRD_SMOKE=1`: zero errors across RELAX, RELAX+DLSS Quality, REBLUR, RR exclusivity, and SIGMA). At `3440×1440` (`Sandbox64` ReSTIR 4-cand), **NRD RELAX Native** reaches **34.34 dB** static PSNR (`0.00517` RGB RMSE vs. raw ReSTIR `0.04209`, matching RR DLAA `34.18 dB`) in **9.969 ms** GPU (`5.188 ms` NRD vs. RR DLAA `24.917 ms`), **6.102 ms** GPU in **NRD RELAX + DLSS Quality** (`34.22 dB`, `1784.04 MiB` process VRAM vs. RR DLAA `3649.61 MiB`), `0` resets + **31.82 dB** snap-disocclusion PSNR (`f=32`) vs. RR DLAA `26.36 dB`, and **0.411–0.434 ms** GPU for **NRD SIGMA** on Boulder (`1.309 ms` Native).
  3. **Two-Room Test Scene & Directional Sun**:
     - **Status**: **COMPLETED**. The Esc scene selector offers **Lighting Sandbox (two rooms)** alongside the original benchmark layout. The roofed room has three walls and area emitters. A doorway connects it to a two-wall, roofless courtyard with no overhead emitters. Both rooms share one floor. The courtyard holds three differently sized and rotated instances of the rock-only Boulder mesh in a triangle centered on the floor.
     - **F5 → Lighting Sandbox** exposes **Direct sunlight**, azimuth, elevation, and intensity. Directional shadow rays sample a 0.53° solar disk (0.265° radius). Conventional MC and ReSTIR DI sample the area emitters independently of the sun. The Native reference accumulates both terms and restarts when the sun changes. [NASA Sun Fact Sheet](https://radiojove.gsfc.nasa.gov/education/sun/basics/material/sunfacts.htm)
     - The fixed-window `VKE_TWO_ROOM_SMOKE=1` GPU suite passed on RTX 3090 at 1280×720 with forced Vulkan and synchronization validation. It verified independent area lighting when the sun is off, doorway illumination, roof and wall occlusion, the fused ReSTIR path, and a 10%–90% Boulder penumbra width of 6 px near and 24 px after lifting the rock 2.2 m above its original placement.
  4. **Cornell Box Scene**:
     - **Status**: **COMPLETED**. The Esc scene selector loads the measured two-block GLB and camera. The shared area-light sampler evaluates Lambertian walls and blocks lit by the rectangular ceiling emitter. [Cornell University scene data](https://bowers.cornell.edu/computer-graphics/data)
     - Converted measured reflectance and relative emission to linear sRGB using the CIE 1931 2° observer and D65 for reflectance.
     - Compared MC4, ReSTIR DI, NRD RELAX/REBLUR, and RR DLAA to a 4,096-sample direct-only reference at 1280×720 on the RTX 3090. Verified that disabling the ceiling emission leaves the diffuse receivers dark. RR reproduced the first-frame NGX validation reports documented in Task 1. Cornell's full-illumination photographs include indirect lighting, which is Phase 6 work for the engine.
  5. **Intel New Sponza (2022)**:
     - **Status**: **COMPLETE**. Imported the official base scene and all four add-ons. LOD 0 retains 32,373,689 triangles with all add-ons enabled. The importer preserves node hierarchy and transforms and shares full-resolution KTX2 maps within each package.
     - Trees use alpha-masked foliage in raster and ray queries. The candle add-on retains all flame geometry, samples a subset of flame triangles for direct lighting, and exposes the authored point lights separately. [Asset versions, credits, and import choices](5.1/new-sponza.md)
  6. **Scene Selection & Completed-Pipeline Benchmarks**:
     - **Status**: **COMPLETE on the RTX 3090 baseline**. The Esc selector includes Intel New Sponza, six authored camera presets, and independent add-on switches. The transition probe covers history resets, resizing, LOD changes, preset changes, add-on removal, and switching back to Boulder.
     - The Release benchmark covers the base scene, each add-on, all add-ons, Native, DLAA, DLSS Quality, RELAX, and Ray Reconstruction at matched internal and output resolutions. It records per-pass GPU timings, wall time, geometry and light counts, and process VRAM. [New Sponza benchmark results](5.1/new-sponza.md)
  7. **Remaining Integration Validation**:
     - **Status**: **COMPLETED on the RTX 3090 baseline** ([Integration validation](5.1/integration-validation.md)). Multi-GPU/other-driver matrix and clean-machine distribution verification belong to Task 9.
     - Completed Wave 1–4 Sponza optimization (`3440×1440` `RR Balanced`: `12.763 / 12.831 ms` GPU `78.4 / 77.9 FPS`, `14.508 / 14.494 ms` wall Mailbox / `13.724 ms` Immediate `72.9 FPS`; `DLSS Quality + RELAX` `10.609 ms` GPU `94.3 FPS`; `DLSS Quality + REBLUR` `10.392 ms` GPU `96.2 FPS`; `RR Performance` `10.012 / 9.865 ms` GPU `99.9 / 101.4 FPS`) and resolved dense-light NRD RELAX/REBLUR fireflies, blur, and boiling.
     - Overhauled ESC Pause, tabbed Settings, Exit Confirm, and F5 Diagnostics (all headers collapsed by default) and verified 6 captured UI PNGs alongside all-scene captures (Boulder, Sandbox 3/64, Two-Room, Sponza modes).
     - Passed CTest, `19/19` `spirv-val --target-env vulkan1.3`, clean `VKE_BUILD_TESTS=OFF` Release build (`build/release-notests/VK_game_engine.exe`), forced Vulkan + sync validation across all 10 GPU smoke/audit suites (`0` engine errors), missing-runtime Native fallback (`VKE_DLSS_DISABLE_RUNTIME=1`), `VKE_MAILBOX_CADENCE_AUDIT=1` (`0` stalls on fixed Mailbox), 3-cycle `VKE_MEMORY_SOAK=1` (`0.00 MiB` VRAM delta, `+4.00 MiB` private RAM delta), and `VKE_REFLEX_BENCH=1` `Off`/`On`/`On + Boost` sweeps (`Sandbox64` `simToGpuMs` `25.48 -> 19.24 ms`).
  8. **Independent DLSS Reconstruction Resolution, Spatial Upscaling & Live Settings Advisor**:
     - **Status**: **COMPLETED on the RTX 3090 `3440×1440` baseline** ([Spatial upscaling results](5.1/spatial-upscaling.md)). Note: 4K (`3840×2160`) presentation was not benchmarked because the test system uses a `3440×1440` display.
     - Separated internal `renderExtent`, DLSS `reconstructionExtent` (`temporal::outputExtent`), and `outputViewport.extent`, wiring Streamline optimal-size queries, jitter phases, and `mip_bias` to `reconstructionExtent`. Added percentage presets in **Reconstruction scale** (`Off (100%)`, `85%`, `75%`, `67% (2/3)`, `50%`) alongside **Spatial upscaler** (`EASU`, `Bilinear`), optional **RCAS** sharpness (`0.00–1.00`), live pipeline preview, and a non-hardcoded **`Recommend...` Settings Advisor** (auto-runs a 7-point live scene probe on click with `Re-measure`, reporting FPS-only Quality / Balanced / Fastest picks) in Settings → Upscaling & Denoising with `settings.ini` persistence.
     - Enforced strict zero-overhead `Off (100%)` behavior (`0` bytes VRAM, all spatial Vulkan handles `VK_NULL_HANDLE`, `0` dispatches; `0.00` sharpness also frees the intermediate image and RCAS pipelines without resetting DLSS history), while preserving native-resolution analytical emitter compositing and ImGui UI.
     - Verified `12/12` `ctest` (`reconstruction_tests` covering scale presets + Advisor math), `24/24` engine SPIR-V + `159` NRD SPIR-V (`spirv-val --target-env vulkan1.3`), and `VKE_SPATIAL_SMOKE=1` under `VKE_FORCE_VALIDATION=1` + `VK_LAYER_VALIDATE_SYNC=1` (`0` engine errors). At `3440×1440` Immediate on New Sponza (all 4 add-ons, Force LOD 0), `75% RR Quality → 3440` (`1720×720 → 2580×1080 → 3440×1440`, `EASU + RCAS 0.25`) matched `3440 RR Performance` internal resolution in `11.184 ms` GPU (`89.4 GPU FPS`, `0.446 ms` spatial, `4807.9 MiB` vs. `11.264 ms` / `4861.3 MiB`), and `67% RR Quality → 3440` reached `9.548 ms` GPU (`104.7 GPU FPS`, `4634.7 MiB`) vs. `16.736 ms` (`59.8 GPU FPS`, `5332.4 MiB`) for `3440 RR Quality`.
  9. **Distribution & Clean-Machine Verification**:
     - **Status**: **IN PROGRESS** (`v0.3.1` packaged locally; clean-machine & multi-GPU verification pending).
     - Updated `package_release.ps1` for Phase 5.1 and built `dist/MaksimEngine-v0.3.1-Portable.zip` (`75.66 MB`, SHA-256 `728B08313B4983C1107C62079549F3E2F8C524A9A3B3F0DF26DBC0CD6EDC0766`) from `build/release-notests` (`VKE_BUILD_TESTS=OFF`), bundling stripped `VK_game_engine.exe` (`12.04 MB`, zero external MinGW DLL dependencies), `engine.pak` (`34.53 MB`, `35` files: `24` SPIR-V shaders, `10` GLBs, `1` KTX2), NVIDIA Streamline/DLSS/RR DLLs (`sl.dlss_d.dll`, `nvngx_dlssd.dll`), `nvidia-notices/` (including `nrd-license.txt`), and Microsoft VC++ runtime DLLs.
     - Remaining: verify `dist/MaksimEngine-v0.3.1-Portable.zip` on a clean non-development Windows machine and additional GPU/driver environments (including explicit RR-unavailable error handling and NRD/off modes).
- **Completion Checkpoint**: Integrated NRD compute denoisers, the expanded two-room test scene with a togglable directional sun, Cornell Box and Intel New Sponza with its four independently toggleable add-ons, completed multi-scene/multi-denoiser benchmarks and integration checks with stated coverage limits, benchmarked independent DLSS reconstruction resolution with spatial upscaling and native-resolution UI, and a verified portable distribution.
- **References**:
  - [Streamline DLSS Ray Reconstruction integration guide](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS_RR.md)
  - [NVIDIA Real-Time Denoisers (NRD)](https://github.com/NVIDIAGameWorks/RayTracingDenoiser)
  - [DLSS 4.5 Ray Reconstruction announcement](https://www.nvidia.com/en-us/geforce/news/gamescom-2026-nvidia-geforce-rtx-dlss-4-5-announcements/)
  - [Cornell box reference data](https://bowers.cornell.edu/computer-graphics/data)
  - [Intel New Sponza (2022) base scene and add-ons](https://www.intel.com/content/www/us/en/developer/topic-technology/graphics-research/samples.html)

### Win32 Mailbox presentation pacing

- **Status** — Verified in `VK_EXT_present_timing` audits with `0` VBlank stalls; the separate 200 FPS Mailbox cap reported on some configurations remains deferred in [future-work.md](planning/future-work.md#win32-mailbox-200-fps-cap). Phase 5.1 proceeds without blocking.
- **Root cause**: Two separate issues caused the tear/flicker and pacing failures in borderless `3440×1440`:
  1. Without `VK_FULL_SCREEN_EXCLUSIVE_DISALLOWED_EXT` (including when DXGI Flip Model became active after creating a FIFO swapchain), NVIDIA's Win32 driver could flip with `AllowsTearing=1`, producing mid-refresh tearing and flicker.
  2. Under `VK_FULL_SCREEN_EXCLUSIVE_DISALLOWED_EXT` (`Hardware Composed: Independent Flip`), `vkQueuePresentKHR` schedules a short DXGI Flip-Model present on the shared 3D hardware engine while the next compute/ray-tracing frame (`MAX_FRAMES_IN_FLIGHT = 2`) was already submitted ahead of it, stalling presentation behind GPU rendering work and dropping ~10–45% of VBlanks (`55–92` displayed FPS with `20–40 ms` intervals on workloads running `115–186` app FPS). An earlier workaround also suppressed Mailbox after FIFO recreation and capped throughput at `100` FPS.
- **Implemented fix** (`src/renderer/swapchain.hpp`, `src/renderer/swapchain.cpp`, `demo/main.cpp`):
  - Removed the restart/FIFO-fallback workaround (`mailboxNeedsRestart`, `hasCreatedFifo`, startup default-FSE exception) so live switching between Mailbox, FIFO, and Immediate takes effect immediately without restarting.
  - Applied `VK_FULL_SCREEN_EXCLUSIVE_DISALLOWED_EXT` unconditionally to Win32 NVIDIA Mailbox swapchains (`tears = 0`).
  - Wait on the current frame's `inFlightFences[currentFrame]` right before calling `vkQueuePresentKHR` in Win32 NVIDIA Mailbox mode (`syncBeforeMailboxPresent`), so the DXGI Flip-Model present executes on an idle 3D engine.
- **Controlled `3440×1440` borderless comparison (`VK_EXT_present_timing`, 4-second focused pan per stage on RTX 3090 / 100 Hz display, Reflex On unless noted)**:

  | Workload | Unpaced Mailbox (`app` / `disp` FPS, `p95`, `stalls`) | Fixed Mailbox (`app` / `disp` FPS, `p95`, `stalls`) | FIFO VSync (`app` / `disp` FPS, `stalls`) | Post-FIFO Live Mailbox (`app` / `disp` FPS, `p95`, `stalls`) |
  | :--- | :--- | :--- | :--- | :--- |
  | **Lighting Sandbox 3** (`DLSS Quality` + `NRD RELAX`) | `139.4` / `95.2`, `10.0 ms`, `18` | `130.2` / `100.0`, `10.0 ms`, `0` | `99.8` / `99.0`, `4` | `129.6` / `100.0`, `10.0 ms`, `0` |
  | **Lighting Sandbox 64** (`DLSS Quality` + `NRD RELAX`) | `145.9` / `71.2`, `20.0 ms`, `93` | `118.3` / `100.0`, `10.0 ms`, `0` | `99.6` / `99.7`, `1` | `118.8` / `99.7`, `10.0 ms`, `1` |
  | **Boulder** (`DLAA`) | `204.2` / `98.0`, `10.0 ms`, `6` | `177.2` / `100.0`, `10.0 ms`, `0` | `99.7` / `50.0`*, `196` | `188.8` / `100.0`, `10.0 ms`, `0` |
  | **Boulder** (`RR Balanced`, sub-100 Hz baseline) | `87.2` / `63.2`, `30.0 ms`, `113` | `83.3` / `78.4`, `20.0 ms`, `85` | `70.6` / `64.9`, `130` | `85.8` / `79.2`, `20.0 ms`, `83` |
  | **Boulder** (`RR Balanced`, Reflex Off) | `88.1` / `64.5`, `30.0 ms`, `122` | `87.3` / `81.5`, `20.0 ms`, `73` | `66.7` / `60.7`, `133` | `85.0` / `76.9`, `20.0 ms`, `89` |

  *\*FIFO `VK_EXT_present_timing` on fast `DLAA` frames omitted half of `firstPixelOut` timestamps despite `99.7` app FPS.*
- **Verification**: All three CMake builds (`build/nsight-release`, `build` Debug, `build/release-notests`), `12/12` CTest, and forced Vulkan + synchronization validation (`VKE_FORCE_VALIDATION=1`, `VK_LAYER_VALIDATE_SYNC=1`) across `VKE_PRESENT_SMOKE` (Mailbox: `157.9` loop FPS; FIFO: `100.5` loop FPS), `VKE_REFLEX_SMOKE` (`15/15` mode/Reflex combinations), `VKE_SWAPCHAIN_SWEEP` (`3/4/5/3` images), and `VKE_MAILBOX_CADENCE_AUDIT` (`0` stalls on fixed Mailbox) passed with `0` validation errors and `0` recorded tearing events.

## Phase 6 (ReSTIR GI)

Implement ReSTIR GI. Repeat the four two-room sandbox baseline captures and compare the images with GI off and on.

## Phase 6.1 (Dynamic LOD, RT & Meshlet Temporal Stability)

Resolve dynamic LOD transitions across raster meshlets, RT BLAS geometry, and temporal history (DLSS Ray Reconstruction, NRD, and ReSTIR) so automatic screen-space LOD can be re-enabled without temporal resets or popping. Until Phase 6.1, the engine defaults to Force LOD 0 and all testing uses Force LOD 0 ([Future work](planning/future-work.md#force-lod-0-default--dynamic-lod-temporal-stability-deferred-to-phase-61)).

- **Cluster LOD DAG** — Prototype an offline directed acyclic graph (DAG) of triangle cluster groups using the existing `meshoptimizer` pipeline as a starting point. Regrouping simplified clusters across earlier group boundaries gives the DAG multiple refinement relationships. Select groups by screen-space error and preserve matching boundaries so different parts of a mesh can use different detail levels without cracks. Feed the selected clusters to both raster meshlets and ray tracing in the initial prototype. [NVIDIA’s Vulkan clustered LOD sample and DAG design](https://github.com/nvpro-samples/vk_lod_clusters/blob/main/docs/lod_generation.md)
- **RTX Mega Geometry** — Prototype `VK_NV_cluster_acceleration_structure` as an optional ray-tracing path. Build and cache triangle-cluster acceleration structures (CLAS), then assemble cluster BLAS from the CLAS selected by the DAG. The local RTX 3090 on driver `621.14` reports extension revision 4. Check the feature bit and device limits at startup and use the existing `VK_KHR_acceleration_structure` path when cluster acceleration is unavailable. Verify ray-query hit mapping for materials, alpha-masked foliage, and emissive triangles. [Vulkan extension reference](https://docs.vulkan.org/refpages/latest/refpages/source/VK_NV_cluster_acceleration_structure.html) · [NVIDIA RTX Mega Geometry Cluster LOD sample](https://github.com/NVIDIA-RTX/RTXMG/blob/main/docs/ClusterLOD.md)
- **Validation** — Track temporal surface identity across LOD changes and reject history where the surface has changed. Once the transition fix is ready for validation, replay matched New Sponza camera paths against the Force LOD 0 baseline and look for cracks, popping, and flashing in DLSS Ray Reconstruction, NRD, and ReSTIR. Compare the cluster and existing BLAS paths at matched geometry detail, recording BLAS/TLAS update time, ray-query cost, frame time, and VRAM use on the RTX 3090.
