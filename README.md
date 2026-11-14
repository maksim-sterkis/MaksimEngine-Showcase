# VK Game Engine

<img src="docs/images/boulder-v0.2.1.png" alt="MaksimEngine - Boulder with ray-traced shadows" width="100%" />

[![Vulkan](https://img.shields.io/badge/Vulkan-1.3-ED1B24?style=for-the-badge&logo=vulkan&logoColor=white)](https://www.vulkan.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows%20x64-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases)
[![C++20](https://img.shields.io/badge/C%2B%2B-20-00599C?style=for-the-badge&logo=c%2B%2B&logoColor=white)](https://en.cppreference.com/w/cpp/20)
[![Pipeline](https://img.shields.io/badge/Pipeline-Mesh%20Shaders%20%2B%20VisBuffer-orange?style=for-the-badge)](docs/file.md)
[![Release](https://img.shields.io/badge/Release-v0.3.0-8A2BE2?style=for-the-badge&logo=github)](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases/tag/v0.3.0)

A Vulkan 1.3 experimental rendering engine written in C++20, implementing a modern GPU-driven pipeline with mesh shaders, visibility buffer shading, discrete meshlet LODs, and hierarchical-Z occlusion culling.

## Features

- **Visibility Buffer Shading**: Decouples geometry rasterization from material evaluation. The geometry pass writes a meshlet index and packed instance/triangle index into a 64-bit `R32G32_UINT` attachment. Compute shading reconstructs barycentrics and UV gradients, then resolves the model, surface and global material through ownership tables. Opaque instances share models while retaining independent transforms and LODs.
- **Meshlet Pipeline**: Geometry is partitioned into meshlets (up to 64 vertices, 124 triangles) using `meshoptimizer` and rendered via Task and Mesh shaders (`VK_EXT_mesh_shader`).
- **GPU Culling & Indirect Dispatch**:
  - *Instance Pre-Cull (`shaders/cull.comp`)*: Evaluates bounding box frustum culling, calculates sub-pixel screen-space geometric surface error ($\le 1.0\text{ px}$ budget) to select the discrete LOD step, and writes indirect draw commands (`VkDrawMeshTasksIndirectCommandEXT`). Whole-object Hi-Z rejection is bypassed so it cannot discard meshlets before recovery.
  - *Meshlet Culling (`shaders/shader.task`)*: Evaluates frustum, cone backface and conservative reverse-Z Hi-Z tests. Unchanged views use one geometry pass; moving/jittered views predict against previous depth and recover deferred meshlets against current depth before shading. Odd-size pyramids retain complete source coverage, and F2 freezes paired depth/camera metadata. See [rendering architecture](docs/file.md).
- **Hardware Ray-Traced Shadows & Dynamic TLAS Pointer Swapping**: Exact directional shadows powered by physical GPU RT cores via `VK_KHR_acceleration_structure` and `VK_KHR_ray_query`. Integrates inline ray queries directly into the Visibility Buffer deferred resolve pass (`shaders/deferred.comp`), completely bypassing primary ray traversal and firing coherent shadow rays exclusively from visible surface pixels with normal-offset bias. Features dynamic per-frame TLAS pointer swapping that synchronizes the active BLAS geometry (`lodBlas[0..5]`) to the rasterized mesh LOD by default (with runtime toggle `Y` to switch between active visible LOD and ground-truth LOD 0), preventing self-shadowing acne and geometric mismatch.
- **Interactive Controls & Hotkeys**:
  - Hotkey `T` toggles hardware ray-traced shadows on and off; Hotkey `Y` toggles ray tracing against the active visible LOD (default) vs. ground-truth LOD 0; F5 diagnostics exposes interactive sun azimuth, elevation, intensity, and RT LOD mode controls.
  - Hotkeys `7`, `8`, `9` toggle Frustum, Cone, and Hi-Z culling stages independently at runtime with live atomic counters displayed on the HUD.
  - Hotkeys `0` (Auto) and `1`–`6` (Force LOD 0–5) provide instant runtime LOD inspection.
  - Hotkey `F1` / `M` toggles debug meshlet cluster coloring; `F2` freezes the culling camera for free-flight inspection.
  - Starting camera initializes at `(3.0, 1.0, 0.0)` aimed directly at the model's true 3D bounding box center.
- **Six Discrete Meshlet LODs**: MLOD v2 simplifies each source primitive independently toward progressively lower triangle budgets, preserving surface/material boundaries and static demo ground. Locked boundaries can limit reduction. Each LOD records the maximum primitive error; runtime selection uses a one-pixel projected-error budget.
- **Bindless Descriptors**: Uses Vulkan descriptor indexing for texture arrays, a global material SSBO and a scene TLAS descriptor. Shaders index material/textures without per-texture draw bindings; geometry still uses per-model descriptor sets.
- **Offline Asset Compiler**: Standalone tool (`tools/model_compiler.cpp`) using `fastgltf` and `tinyobjloader` to process `.gltf` and `.obj` assets into binary `.glb` files with embedded meshlet streams, discrete LOD levels, and pre-compressed KTX2 textures (BC7 for albedo/normals, BC5 for roughness/metallic) packed directly into GLB `BufferView` chunks.
- **Virtual File System (VFS)**: Packages SPIR-V shaders, models, and textures into a single archive (`engine.pak`) accessed via memory-mapping on Windows, with automatic disk fallback for loose files during development.
- **PBR and Emissive Direct Lighting**: Cook-Torrance materials use albedo, normals, metallic and roughness. The boulder retains directional lighting; the sandbox uses one-sided emissive triangle lights, visible emission and finite shadow rays. Conventional sampling and ReSTIR DI are selectable, with an independent 4096-spp direct-light reference.
- **Temporal HDR Foundation & DLAA/DLSS**: Separate internal/output resolutions, linear HDR lighting, camera/object motion vectors and history validation support fixed model K DLAA and DLSS Quality/Balanced/Performance through Streamline 2.14.16. Native is the first-launch rendering default and has no temporal color accumulation. Missing or unsupported NVIDIA reconstruction falls back to Native. See [rendering architecture and verification limits](docs/file.md).
- **Reflex & Presentation Controls**: Independent NVIDIA Reflex Off / On / On + Boost, with effective status and driver timing in F5. Settings offers Mailbox (default), Immediate and VSync (FIFO). Reflex operation has been exercised; a latency improvement on the current scene has not been established.
- **Hardware Texture Mipmapping & Analytical Derivatives**: Closed-form screen-space barycentric derivatives (`dUVdx`, `dUVdy`) drive hardware `textureGrad` sampling for smooth, zero-pop anisotropic mipmap transitions.
- **Non-Intrusive Telemetry HUD & Profiler**: Passive F3 HUD, compact F4 performance overlay with a stable 5ms peak time-slot frame-time graph, reconstruction mode/dimensions and key GPU timings, and an independent F5 inspector for temporal, RT/lighting, culling, Reflex and detailed timing diagnostics. CPU frame time, GPU time and driver-reported latency are labeled separately.
- **Scalar Block Layout**: Uses `GL_EXT_scalar_block_layout` so C++ structs and shader storage buffers share matching memory layout without manual alignment padding.

## Roadmap & Upcoming Features

**Phase 5 is complete (`v0.3.0`, November 14, 2026).** Phase 4.1's temporal, DLAA/DLSS and Reflex foundation is retained, and Phase 5 ships opaque instances, emissive triangle lighting, initial RIS candidates, temporal/spatial ReSTIR DI reuse, fresh final visibility, emission payload caching and 48-byte packed cached surfaces. See [implementation and verification limits](docs/file.md#verification-and-limits).

Phase 5 covers:

1. **Scene materials and emissive area lighting**: Correct per-surface material lookup, emissive materials, emission payload caching in light records, efficient light sampling and reusable lighting functions for ReSTIR DI.
2. **Controlled lighting sandbox and reference**: Few/many-light presets (`3` and `64` emitters), opaque objects, dynamic motion cases and an unjittered Native 4096-spp direct-light reference.
3. **ReSTIR DI**: Basic real-time correction (`4` initial candidates, `1` spatial proposal, scalar stable-history and fused initial/temporal/spatial/shading kernels) and a ray-traced reference policy. Final spatial reservoirs feed next-frame history by default; a temporal-only history comparison and Conventional MC remain selectable in F5.
4. **Validation & sustained benchmarks**:
   - All 11 CPU/offline `ctest` suites, all 15 `spirv-val` Vulkan 1.3 validations, forced Vulkan synchronization validation (`ReSTIR4` Basic Full DI), full-res `3440x1440` Jitter regression, and full-res `3440x1440` dynamic/disocclusion Motion Quality audits (`ReSTIR4` vs. `ReSTIR8`, including `7,016` revealed receiver pixels) pass.
   - **Sustained `3440x1440` Mailbox benchmarks (RTX 3090, Nov 12, 30 runs / 226,201 frames)**: `Sandbox64Interior` averaged **174.3–174.5 FPS DLAA** (`4.82–4.84 ms` GPU), **250.9–251.1 FPS DLSS Quality** (`3.07 ms` GPU), and **267.4–268.3 FPS Native** (`2.85 ms` GPU), while `Boulder` exceeded **320 FPS DLAA** (`322.9–341.8 FPS`).
   - **Equal-cost multi-view audits (48B packed build, `3440x1440`)**: `MC7` (`2.61 ms` lighting) and `MC8` (`2.87 ms`) bracket `ReSTIR4` (`2.68 ms`). Across 16 independent trajectories × 256 frames vs. `MC8`, `ReSTIR4` delivers **37.2% lower raw RGB MSE** and **77.9% lower raw luminance MSE** in `Sandbox64Interior`, and **42.3% lower raw luminance MSE** in `Sandbox64Exterior` (while `MC8` leads in the 3-emitter interior where sample count exceeds triangle count).

This is direct lighting and shadows only: no GI, bounced light, indirect color bleeding or production denoiser. Boulder remains the startup scene; the Esc selector opens Lighting Sandbox, where F5 exposes ReSTIR DI, Conventional MC, telemetry and diagnostics. **Phase 5.1** is next: DLSS 4.5 Ray Reconstruction evaluation, Cornell Box, Crytek Sponza, multi-GPU/clean-machine verification and denoiser selection (SVGF stays deferred).

Check out the [full roadmap](docs/roadmap.md) for detailed progress and upcoming milestones.

## Code Architecture

For a complete breakdown of every directory, component, and the GPU pipeline flow, see the [File Architecture Documentation](docs/file.md).

> **Visualizing the Engine:** We also have an interactive Mermaid.js diagram showing exactly how data moves from CPU to GPU. Check out the **[Pipeline Flowchart](docs/flow.md)**.
>
> *Note: When viewing the chart on GitHub, be sure to click the **Fullscreen button** (the expanding arrows icon in the top right of the diagram box) so you can easily read the nodes and use the view controls.*

## Release Downloads

The latest packaged release is [**MaksimEngine v0.3.0**](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases/tag/v0.3.0), dated November 14, 2026, for Windows 10/11 64-bit. Download [MaksimEngine-v0.3.0-Portable.zip](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases/download/v0.3.0/MaksimEngine-v0.3.0-Portable.zip), extract it into a writable folder keeping all files and folders together, and launch `VK_game_engine.exe`.

The v0.3.0 portable build (`VKE_BUILD_TESTS=OFF`, `NDEBUG`, stripped Release binary + `engine.pak`) includes:
- Startup **Boulder** scene with meshlet LODs, conservative Hi-Z culling, and hardware ray-traced directional shadows.
- **Esc scene selector → Lighting Sandbox** (`3` and `64` emissive lights, `ReSTIR DI` and `Conventional MC`, dynamic object/light motion toggles, `4096-spp` reference capture, and F3/F4/F5 diagnostics with live CPU/GPU/VRAM telemetry).
- NVIDIA **model K DLAA / DLSS Super Resolution** (`Streamline 2.14.16` + Microsoft VC++ runtime DLLs + NVIDIA notices) and **NVIDIA Reflex** Off / On / On + Boost. Native is the first-launch default; DLAA/DLSS require a supported NVIDIA RTX GPU and compatible driver (non-RTX systems fall back cleanly to Native).

Local RTX 3090 verification covers Release (`VKE_BUILD_TESTS=OFF`) compilation with zero external MinGW dependencies, `engine.pak` archive creation (`23` files: `15` SPIR-V shaders, `7` GLBs, `1` KTX2), and offline/GPU validation suites on the same codebase. Clean-machine verification on non-development Windows systems and Phase 5.1 Ray Reconstruction / Cornell / Sponza content remain pending.

## Credits & Dependencies
- [Vulkan](https://www.vulkan.org/)
- [fastgltf](https://github.com/spnda/fastgltf)
- [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader)
- [meshoptimizer](https://github.com/zeux/meshoptimizer)
- [stb_image](https://github.com/nothings/stb)
- [GLFW](https://www.glfw.org/)
- [GLM](https://github.com/g-truc/glm)
- [Dear ImGui](https://github.com/ocornut/imgui)
- [NVIDIA Streamline](https://github.com/NVIDIA-RTX/Streamline) (DLAA/DLSS, Reflex and PC Latency Stats)
