# Vulkan Engine Roadmap

This plan outlines the next major architectural upgrades for the engine, focusing on bindless textures, PBR materials, advanced culling, hardware ray tracing, and next-gen lighting.

Current status (2026-10-31): Phase 5 core is implemented, with estimator/history validation and performance still open. Phase 5.1 is pending. Phases 0–4 describe historical milestones; [file architecture](file.md) and the [pipeline flowchart](flow.md) describe current behavior.

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

- **Status: CORE IMPLEMENTED; VALIDATION AND PERFORMANCE OPEN.** Initial candidates, temporal reuse, spatial reuse and fresh final visibility run on the small opaque sandbox. Passing reference means and regression tests do not close the history audit or establish a sampling-efficiency advantage. See [architecture and verification](file.md#verification-and-limits).
- **Goal**: Establish surface/material ownership, opaque instances, emissive direct lighting, an independent reference and independently tested ReSTIR DI.
- **Scope**: Direct illumination and shadows. No GI, bounced light or indirect color bleeding. Denoising, full Cornell/Sponza integration and distribution belong to Phase 5.1.

### Implemented

1. **Baseline and surface ownership**: MLOD v2 retains source primitive/material ownership through six per-primitive LODs with locked boundaries and maximum primitive error. Static demo ground is retained. Payload validation precedes GPU allocation; ambiguous legacy multi-material payloads require rebuilding. The checked-in boulder remains unchanged. Temporal identity is entity/surface/raster-LOD/material in an RGBA32 unsigned image.
2. **Opaque instances and emission**: Shared buffer-address model tables, per-instance transforms/LOD/culling, motion and current/previous TLAS support the sandbox. Emissive factors, strength and textures feed one-sided triangle lights with stable identities. World-space light data is updated when changed. General imported scene graphs and alpha handling remain pending.
3. **Controlled scene and reference**: Room, shared cubes, rough metal and three/64 emitters are preloaded. Boulder is the startup default; Esc switches scenes and F5 exposes motion, sample and identity controls. Conventional MC and an unjittered Native 4096-spp direct-light reference remain independent of ReSTIR and display reconstruction.
4. **ReSTIR DI stages**: Initial RIS, light-index mapping, current/previous surface/light/geometry snapshots, temporal rejection, spatial reuse and final shading are implemented. Basic real-time correction uses unoccluded source targets; the ray-traced reference policy evaluates source visibility. Both retain fresh final shadow queries. No visibility cache or radiance filter is enabled.
5. **Traffic and execution improvements**: GPU reservoirs are 32 bytes and cached surfaces 80 bytes, preserving full-width lighting floats. Scalar stable-history reuse, separate disocclusion work and fused Basic passes reduce traffic. Buffer ownership rotation retains pre-spatial temporal history without a GPU copy. Full-DI allocations at 3440x1440 are 1,268,245,248 bytes, excluding other renderer/SDK targets.
6. **Recorded verification**: Eleven CPU/offline suites pass in MinGW Debug and Release. Debug synchronization-validation Basic/reference suites cover reuse, source correction, motion, lights, topology, cuts, LOD, area changes, modes and resizing. Release functional GPU checks also pass. The stationary Native/DLAA jitter regression measures approximately 0.000529 rows of donor drift per eight-frame DLAA cycle; it does not prove noise-free output. The current-loader boulder baseline is a later checkpoint, not a retroactive pre-change GPU baseline.

### Current performance evidence

- Short RTX 3090 Release observations at full **3440x1440 DLAA**, eight candidates, one Basic spatial proposal, shadows and F4/F5 reached approximately **200 FPS starting view**, **168 FPS three-emitter interior**, and **142–145 FPS 64-emitter interior**. Longer normal-shader repetitions across views/clocks ranged roughly **124–200 FPS**. These are development observations, not sustained minima or hardware-independent expectations.
- A controlled sweep held all 68 instances, materials and transforms fixed while sampling 1, 3, 16, 32 or 64 emitters. Thirty accepted cases recorded 360/360 focused frames and 360 asynchronous GPU samples each. Initial/material/temporal cost increased from roughly 1.9–2.2 ms at one emitter to 2.9–3.1 ms at 64 in the interior runs; spatial/final work stayed much flatter.
- Twenty-eight subsequent shader-diagnostic cases narrowed the count penalty to varied light access and target evaluation, with temporal work also significant. Coherent-light and constant-target variants deliberately change sampling and were not adopted. Compiler-only optimization showed no reliable win. No installed hardware shader profiler was available, so cache stalls and register pressure remain hypotheses rather than measured diagnoses.
- GPU clock drift was observed during repeated runs. Neither clocks nor power/fan settings were modified. Readback correctness probes and timestamp-only performance runs are separate; tiny correctness resolutions and per-frame readback timings are not ultrawide performance evidence.

### Required before closure

1. **Resolve the estimator/history audit**: Current history retains pre-spatial temporal reservoirs after a recursive spatial-feedback test exposed unstable brightening. This workaround loses repeated spatial propagation. Correct recursive reuse needs validation; simply reconnecting the buffers is insufficient.
2. **Demonstrate useful sampling efficiency**: In the tested small-scene reference fixtures, MC8 currently has lower per-frame image error than full ReSTIR8. Mean lighting agrees within the tested gates, but a convincing quality-per-cost advantage remains unestablished. Compare raw mean, noise and temporal response at matched cost/quality.
3. **Qualify sustained performance and temporal behavior**: The requested approximately 200 FPS heavier-view target is unmet. Optimize the demonstrated initial/temporal cost, repeat representative worst views and distinguish shader cost, GPU clock variation and scheduling waits. Raw stochastic noise remains; no denoiser may conceal estimator errors.

**Completion checkpoint**: Correct basic light sampling and staged ReSTIR DI validation on the small scene, with the history audit resolved or rigorously justified, reference comparisons and recorded GPU costs. The engine's 200 FPS goal remains explicit performance work. Denoising, full benchmark scenes and distribution are not prerequisites for this phase.

**Reference**: [NVIDIA ReSTIR DI integration and light-data requirements](https://github.com/NVIDIA-RTX/RTXDI/blob/main/Doc/Integration.md).

## Phase 5.1 (DLSS Ray Reconstruction, Benchmark Scenes & Integration Validation)
- **Status**: **PENDING**
- **Goal**: Evaluate DLSS 4.5 Ray Reconstruction as the primary RTX denoising/reconstruction path for Phase 5 lighting, integrate the full benchmark scenes, and complete outstanding validation and distribution work.
- **Why**: Tests neural denoising before investing in a custom SVGF implementation, then evaluates the completed lighting pipeline on representative scenes and supported rendering paths.
- **Scope**: Builds on validated Phase 5 ReSTIR DI. Custom SVGF is deferred pending the RR evaluation and a decision on portable denoising; multi-bounce global illumination and color bleeding remain outside scope.
- **Tasks**:
  1. **DLSS Ray Reconstruction Inputs & Integration**:
     - Target DLSS 4.5 Ray Reconstruction, verifying the compatible SDK, signed runtime/model version and driver requirements actually used. Existing model K DLAA/DLSS support does not establish RR capability or input correctness.
     - Supply noisy linear-HDR color before tone mapping and final denoising, depth, dense camera/object motion vectors, diffuse and specular albedo, normals/roughness, and the appropriate specular motion-vector or hit-distance inputs and matrices. Define valid inputs for background, emissive-only surfaces and paths without traced reflections according to the integration requirements.
     - Extend the demand-driven temporal guide generation for RR and expose input/debug views. Validate dimensions, spaces, exposure conventions, resource lifetimes, barriers and history resets.
     - Implement feature/runtime capability checks, signed dependency staging, an RR toggle and explicit fallback reporting. Keep ordinary Native and existing DLAA/DLSS operation available when RR is disabled or unavailable.
     - Treat active RR as the combined denoising/reconstruction evaluation that replaces ordinary DLSS SR evaluation. Do not run SVGF or a second DLSS SR resolve over the same lighting by default; keep ReSTIR reservoirs independent of RR history.
  2. **RR Evaluation & Denoiser Decision**:
     - Start on the Phase 5 test scene and compare raw ReSTIR lighting, the high-sample reference and RR output for lighting accuracy, noise, material detail, ghosting and response to moving lights, geometry and disocclusion.
     - Measure RR GPU cost and memory use on the available RTX 3090; do not assume a benefit for this direct-lighting workload from existing DLSS SR measurements or path-traced game results.
     - Decide whether to retain RR as the primary RTX path based on measured quality and cost. Record the behavior and quality limits of RR-off and non-RTX paths; preserving those paths does not supply denoising automatically.
     - Keep custom SVGF deferred. If clean low-sample lighting is required without RR, explicitly select and schedule a portable denoiser, such as SVGF or another suitable implementation, rather than making both denoisers prerequisites for RR evaluation.
  3. **Cornell Box Scene**:
     - **Status**: **PENDING**.
     - Build a GLB from Cornell's measured diffuse two-block reference, preserving geometry, scale, camera placement and source data. Document the conversion of measured spectral reflectance/emission to the engine's RGB representation.
     - Implement the red/green walls, diffuse neutral surfaces, interior blocks and rectangular ceiling emitter using the shared material/light system.
     - Compare direct area lighting, ReSTIR DI and RR against the direct-only high-sample reference with no artificial ambient fill. The measured geometry/material data does not make this direct-only output equivalent to Cornell's full-illumination photographs; indirect illumination and color bleeding remain future work.
  4. **Crytek Sponza Scene**:
     - **Status**: **PENDING**.
     - Import through the model compiler and texture pipeline, preserving scene transforms, multiple meshes and materials, textures, and alpha-masked surfaces.
     - Extend and validate the Phase 5 surface/material and temporal support across the full scene, including instancing, raster/ray-query alpha agreement and history rejection at material and object boundaries.
     - Record the exact asset version, source, licensing terms, and attribution with the scene assets.
  5. **Scene Selection & Completed-Pipeline Benchmarks**:
     - **Status**: **PARTIAL**. Boulder/sandbox selection is implemented; scripted scene transitions pass GPU smoke. Direct menu interaction remains unverified because desktop capture/activation failed. Cornell, Sponza and completed-pipeline benchmarks remain pending.
     - Add an Esc-menu scene selector as the boulder, test scene, Cornell box and Crytek Sponza become available, with camera presets, safe resource transitions and explicit temporal/Hi-Z, ReSTIR and RR invalidation on scene changes.
     - Compare Native, DLAA, DLSS SR and supported RR configurations for image stability, ghosting, GPU/frame time and memory use during motion, disocclusion, resizing and LOD transitions. Record the lighting sample budget and active denoising path for each comparison.
     - Establish an optimized build configuration for production performance measurements while retaining Debug/synchronization-validation checks for correctness. Measure light preparation, ReSTIR, any denoising/reconstruction, and presentation costs separately using matched scene and camera conditions.
  6. **Remaining Integration Validation**:
     - **Status**: **PENDING**. Extended human image-quality/diagnostic review, a quantitative memory soak and other GPU/driver combinations remain unverified.
     - Complete the outstanding Phase 4.1 display/fallback and diagnostic review, then repeat relevant coverage with the new scenes, lighting, RR and resource transitions. Preserve the distinction between historical boulder checks and validation of the completed pipeline.
     - Quantify memory behavior over extended operation, scene/mode switching and resizing, and record tested GPU/driver combinations and remaining coverage limits.
     - Measure Reflex Off/On/On + Boost using the current runtime-loading path and distinguish average PC latency from GPU render time.
  7. **Distribution & Clean-Machine Verification**:
     - Debug correctness and Release performance builds are available. The published v0.2.1 boulder package already includes NVIDIA components, Microsoft runtime DLLs and NVIDIA notices, with a recorded RTX 3090 launch/DLAA check. Revalidate packaging and clean-machine behavior for the newer Phase 5/5.1 pipeline, adding RR dependencies if retained. Runtime DLLs remain beside the executable, outside `engine.pak`; current local Release builds do not establish the health of a new distribution.
     - Verify a future packaged build on a clean machine, including missing-runtime/unsupported-feature fallback and the selected RR-off behavior. Existing Phase 4 Release artifacts do not validate the Phase 4.1 runtime or the new lighting/RR paths.
- **Completion Checkpoint**: A documented RR quality/performance decision, explicit fallback/portable-denoising policy, integrated benchmark scenes, completed integration checks with stated coverage limits, and a verified portable distribution.
- **References**:
  - [Streamline DLSS Ray Reconstruction integration guide](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS_RR.md)
  - [DLSS 4.5 Ray Reconstruction announcement](https://www.nvidia.com/en-us/geforce/news/gamescom-2026-nvidia-geforce-rtx-dlss-4-5-announcements/)
  - [Cornell box reference data](https://bowers.cornell.edu/computer-graphics/data)
  - [Crytek Sponza glTF asset and attribution](https://github.com/KhronosGroup/glTF-Sample-Assets/blob/main/Models/Sponza/README.md)
