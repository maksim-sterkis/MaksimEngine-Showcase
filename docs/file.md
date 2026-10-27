# File Architecture and Connections

This document describes the current boulder renderer and its main components. Phase 4.1 is closed with the temporal, model K DLAA/DLSS and Reflex foundation implemented. Phase 5 contains general scene/material support, emissive area lighting, Cornell box, Sponza, ReSTIR DI, SVGF and the outstanding integration checks. See the [roadmap](roadmap.md); planned lighting features are not part of the current frame pipeline.

## `demo/` (Interactive Demo)

- **`demo/main.cpp`** initializes the engine, loads the boulder, creates its ECS entity and supplies update, culling and drawing callbacks. The camera starts at `(3.0, 1.0, 0.0)` aimed at the model's bounding-box center.
- Runtime controls include `F1` / `M` meshlet colors, `F2` paired culling/depth freeze, `0`–`6` automatic/manual LOD, `7`–`9` frustum/cone/Hi-Z toggles, `T` RT shadows and `Y` RT visible-LOD matching.
- F3 is the passive HUD. F4 is the compact performance overlay with reconstruction mode/dimensions, key GPU timings and frame-time graph. F5 is the movable diagnostics window for reconstruction/temporal controls, Reflex, RT/sun, geometry/culling and detailed timings.
- Esc releases the cursor and opens the pause/settings UI. Settings persists window/presentation mode, aspect ratio, reconstruction mode, Reflex mode, UI scale and sensitivity. Debug culling/lighting/temporal controls remain session-only.
- The boulder is the only integrated scene. Runtime scene selection and general per-surface material resolution are Phase 5 work; the current draw path selects the model's first submesh material for deferred shading.

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
- **`pipeline.hpp` / `.cpp`** create visibility graphics, object-cull, Hi-Z and deferred-lighting pipelines and descriptor layouts. Draw push constants remain 256 bytes. The global set contains material SSBOs, bindless texture samplers and the TLAS.
- **`occlusion.hpp` / `.cpp`** own independent previous/frozen depth metadata, recovery lists, GPU indirect setup and conservative history compatibility. Unchanged views use one geometry pass. Compatible moving/jittered views predict against previous depth and recover deferred meshlets against current depth before lighting.
- **`temporal.hpp` / `.cpp`** own two sets of HDR/guide/resolved targets, current/previous snapshots, history validation, descriptor updates, spatial reconstruction and display pipelines. Native at full resolution presents current HDR without temporal color accumulation.
- **`temporal_math.hpp`** defines matrix/transform packing, jitter, reprojection and history compatibility helpers. Descriptor-backed `FrameGpu` is 704 bytes.
- **`reconstruction.hpp` / `.cpp`** dynamically load signed Streamline components, negotiate NVIDIA support, configure fixed model K DLAA/DLSS, tag inputs/evaluate reconstruction and handle Native fallback. The same backend implements independent Reflex modes, scheduling, frame markers and input-thread PCL pings. The current loader creates/loads the signature-verified `vke_sl_interposer.dll` copy as a Mailbox pacing workaround. See [integration and verification limits](file.md#verification-and-limits).
- **`reconstruction_settings.hpp`** defines Native, DLAA, Quality, Balanced and Performance plus SDK matrix/motion/jitter/mip-bias helpers. Removed M/L settings are ignored.
- **`raytracing.hpp` / `.cpp`** build discrete static BLAS for six model LODs and update the scene TLAS from ECS transforms. RT can follow the selected raster LOD or use LOD 0.
- **`gpu_timing.hpp` / `.cpp`** and **`gpu_timing_math.hpp`** provide fence-protected, demand-driven timestamps and timing calculations. CPU frame wall time, GPU rendering cost and driver latency reports are distinct metrics.
- **`imgui.hpp` / `.cpp`** initialize Dear ImGui's Vulkan backend and record output-resolution UI. The demo defines the actual HUD, diagnostics and settings contents.

## `src/scene/` (Assets and ECS)

- **`asset_pool.hpp` / `.cpp`** register models/textures, build the material SSBO and populate bindless samplers.
- **`model.hpp` / `.cpp`** parse binary GLB through VFS, upload vertex/index/meshlet data, read multi-LOD payloads and create per-LOD RT index buffers/BLAS. The loader records submesh material indices, but general per-surface selection in the visibility resolve remains unfinished. Emissive material fields and area-light extraction are not implemented.
- **`texture.hpp` / `.cpp`** decode standard images with stb_image, upload embedded KTX2 BC7/BC5 data using libktx, generate mipmaps and create anisotropic samplers.
- **`ecs.hpp` / `.cpp`** manage entity IDs and Transform/MeshRenderer components.

## `shaders/` (GPU Programs)

- **`cull.comp`** performs object frustum pre-culling and output-content-height-based screen-space-error LOD selection, then writes mesh-task indirect commands. Whole-object Hi-Z rejection is bypassed to preserve recovery candidates.
- **`shader.task`** performs valid-lane meshlet frustum/cone tests and complete-footprint conservative reverse-Z occlusion. Prediction appends potentially hidden meshlets to a bounded recovery list; overflow renders immediately. Recovery tests that list against current depth.
- **`shader.mesh`** emits geometry with the current jittered render projection and forwards meshlet/primitive IDs.
- **`shader.frag`** writes `uvec2(meshletIndex, primitiveID)` to the `R32G32_UINT` visibility target. The payload does not yet encode a general instance/material lookup.
- **`hiz.comp`** builds minimum-reduced reverse-Z pyramids with complete coverage for odd dimensions.
- **`occlusion_setup.comp`** generates GPU indirect arguments for recovery and conditional final-pyramid work; **`occlusion_data.glsl`** shares the associated constants/work-buffer layout.
- **`deferred.comp`** reconstructs perspective-correct attributes and analytical UV derivatives, applies full object/normal transforms, evaluates PBR and directional-sun shadow ray queries, and writes linear HDR plus demanded temporal guides. It currently uses a constant ambient term. Emissive area lighting, ReSTIR and SVGF are Phase 5 work.
- **`temporal_frame.glsl`** and **`temporal_images.glsl`** define temporal constants and image bindings.
- **`reconstruct.comp`** provides Native spatial reconstruction for reduced diagnostic scales; it does not implement TAA.
- **`present.comp`** converts HDR to display output, handles the output viewport/aspect bars and diagnostic presentation, and preserves the configured background. Active NVIDIA reconstruction runs before this stage.

## Build, Offline Tools and Verification

- **`CMakeLists.txt`** configures the engine, shaders, asset tools and opt-in focused tests.
- **`cmake/Streamline.cmake`** pins Streamline 2.14.16, verifies/caches the SDK, supports `VKE_ENABLE_DLSS=OFF` native-only builds, and stages NVIDIA DLLs/notices beside the executable.
- **`tools/model_compiler.cpp`** processes glTF/OBJ into GLB with meshlet data, six simplification LODs, cumulative geometric error and embedded KTX2 textures. General multi-material/scene and emissive handling must be completed for the Phase 5 assets.
- **`tools/texture_compiler.cpp`** performs offline texture conversion.
- **`tools/pak_compiler.cpp`** packages shaders, models and textures into the indexed `engine.pak` archive.
- **`package_release.ps1`** configures a Release build, stages/strips the executable, packs assets and produces a portable ZIP, optionally including a MinGW pthread DLL. NVIDIA DLL/notice staging and clean-machine verification remain Phase 5 work. Existing v0.2.0 artifacts belong to Phase 4.
- **`tests/`** contains temporal/reconstruction/Hi-Z CPU suites and opt-in GPU smoke, exact-reference, benchmark and Reflex/presentation probes. See [verification and limits](#verification-and-limits). Current development uses MinGW Debug.

## Verification and Limits

The recorded MinGW Debug checks on the available RTX 3090 include three focused CPU suites, temporal/Hi-Z synchronization-validation smoke, real model K reconstruction and Native fallback checks. The original 216 Hi-Z-on/off source depth/HDR frame pairs and a further 144 Quality/K pairs matched exactly, covering sampled LOD, motion, transform and odd-resolution cases. Automated minimize/restore coverage passed during the NVIDIA integration. These results apply to the boulder and tested configuration, not general multi-model scenes or all drivers.

The historical 15-combination reconstruction/Reflex smoke verified SDK calls, markers and rendering under synchronization validation. It did not establish lower average PC latency. Those runs used the canonical interposer filename before the current signed-copy loader; their approximately 200 FPS presentation limit is not a measurement of the current workaround. Earlier reconstruction measurements put the reconstruction stage near 1.3 ms on this small scene, exceeding its lighting savings; no current throughput improvement is established by those numbers.

Phase 5 carries extended image-quality/F4/F5 review, quantitative memory-soak testing, current-loader Reflex Off/On/On + Boost measurements, broader hardware coverage and portable NVIDIA runtime packaging. Later resource-reuse and Native fast/full-guide parity assertions were compiled but not run; an additional display matrix was interrupted after Native and all model K cases. Closing Phase 4.1 does not mark these outstanding checks complete.

Temporal guides use previous-minus-current unjittered pixel motion, reverse-Z hardware depth and linear HDR before tone mapping. Hi-Z retains its own producing-camera/depth history, while DLAA/DLSS controls reconstruction jitter and input dimensions. Future ReSTIR reservoirs and SVGF lighting history require separate state and rejection rules. See the [roadmap](roadmap.md) for the remaining work.

## How They Connect

1. **Initialize:** VFS, NVIDIA backend, window/device, swapchain/pipelines and frame resources are prepared. Streamline initialization precedes Vulkan instance/device creation.
2. **Load the boulder:** GLB geometry, materials and textures enter GPU storage; six LOD BLAS and the scene TLAS support shadow queries.
3. **Schedule and sample input:** wait for the reusable frame slot, obtain a runtime frame token, apply Reflex options/sleep, mark simulation start and poll GLFW input. Update camera/object/settings state and mark simulation end.
4. **Acquire/configure:** apply render-target changes, acquire a swapchain image and retire its previous use, update descriptors and prepare temporal/Hi-Z constants. Acquisition can still wait after input sampling.
5. **Cull and rasterize:** select LOD, update the TLAS and write visibility/depth. If prediction is active, build current depth and recover deferred meshlets before shading. Unchanged/warm-up/disabled/frozen paths follow their own depth-history rules.
6. **Shade:** reconstruct visible attributes, evaluate directional-sun lighting/shadows and write HDR. Generate only the guides needed by NVIDIA reconstruction or diagnostics; empty scenes explicitly clear HDR.
7. **Reconstruct and display:** active DLAA/DLSS evaluates HDR/depth/motion inputs. Native at 100% uses current HDR directly; reduced Native diagnostics use spatial reconstruction. Tone mapping/display conversion precedes output-resolution F3/F4/F5 and menu rendering.
8. **Finalize and submit:** refresh Hi-Z as required, capture paired frozen data when requested, submit/present with latency markers and supported frame IDs, then commit successful-submission history. Presentation recreation invalidates the affected histories.

See the [pipeline flowchart](flow.md) for the data flow and the [roadmap](roadmap.md) for planned Phase 5 consumers.
