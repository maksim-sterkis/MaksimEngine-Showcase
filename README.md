# VK Game Engine

[![Vulkan](https://img.shields.io/badge/Vulkan-1.3-ED1B24?style=for-the-badge&logo=vulkan&logoColor=white)](https://www.vulkan.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows%20x64-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases)
[![C++20](https://img.shields.io/badge/C%2B%2B-20-00599C?style=for-the-badge&logo=c%2B%2B&logoColor=white)](https://en.cppreference.com/w/cpp/20)
[![Pipeline](https://img.shields.io/badge/Pipeline-Mesh%20Shaders%20%2B%20VisBuffer-orange?style=for-the-badge)](docs/file.md)
[![Release](https://img.shields.io/badge/Release-v0.1.1-8A2BE2?style=for-the-badge&logo=github)](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases)

A Vulkan 1.3 experimental rendering engine written in C++20, implementing a modern GPU-driven pipeline with mesh shaders, visibility buffer shading, discrete meshlet LODs, and hierarchical-Z occlusion culling.

## Features

- **Visibility Buffer Shading**: Decouples geometry rasterization from material evaluation. The geometry pass rasterizes 64-bit IDs `(meshletIndex, primitiveID)` to an `R32G32_UINT` attachment. A fullscreen compute pass (`shaders/deferred.comp`) reconstructs screen-space barycentric coordinates and evaluates PBR materials only on visible pixels, avoiding redundant fragment shading on occluded surfaces.
- **Meshlet Pipeline**: Geometry is partitioned into meshlets (up to 64 vertices, 124 triangles) using `meshoptimizer` and rendered via Task and Mesh shaders (`VK_EXT_mesh_shader`).
- **Two-Phase GPU Culling & Indirect Dispatch**:
  - *Instance Pre-Cull (`shaders/cull.comp`)*: Evaluates bounding box frustum culling and Hi-Z occlusion tests, calculates rigorous sub-pixel screen-space geometric surface error ($\le 1.0\text{ px}$ budget) to select the optimal discrete LOD step, and writes indirect draw commands (`VkDrawMeshTasksIndirectCommandEXT`).
  - *Meshlet Culling (`shaders/shader.task`)*: Evaluates meshlet bounding spheres against the view frustum, cone backface orientation, and conservative Hi-Z occlusion. Features metric linear-depth testing (`mesh_linear_dist > hiz_linear_dist + 0.25m`), viewport perimeter guards, and bounded mip downsampling to reliably prevent false-positive culling on concave craters and screen edges.
- **Hardware Ray-Traced Shadows & Dynamic TLAS Pointer Swapping**: Exact directional shadows powered by physical GPU RT cores via `VK_KHR_acceleration_structure` and `VK_KHR_ray_query`. Integrates inline ray queries directly into the Visibility Buffer deferred resolve pass (`shaders/deferred.comp`), completely bypassing primary ray traversal and firing coherent shadow rays exclusively from visible surface pixels with normal-offset bias. Features dynamic per-frame TLAS pointer swapping that synchronizes the active BLAS geometry (`lodBlas[0..5]`) to the rasterized mesh LOD, preventing self-shadowing acne and geometric mismatch.
- **Interactive Controls & Hotkeys**:
  - Hotkey `T` toggles hardware ray-traced shadows on and off at runtime; F4 HUD exposes interactive sun azimuth, elevation, and intensity controls.
  - Hotkeys `7`, `8`, `9` toggle Frustum, Cone, and Hi-Z culling stages independently at runtime with live atomic counters displayed on the HUD.
  - Hotkeys `0` (Auto) and `1`–`6` (Force LOD 0–5) provide instant runtime LOD inspection.
  - Hotkey `F1` / `M` toggles debug meshlet cluster coloring; `F2` freezes the culling camera for free-flight inspection.
  - Starting camera initializes at `(3.0, 1.0, 0.0)` aimed directly at the model's true 3D bounding box center.
- **6-Level Discrete Meshlet LODs with Sub-Pixel Error**: Generates 6 progressive simplification steps (100%, 50%, 25%, 12.5%, 6.25%, 3.125%) with cumulative 3D geometric surface deviation in meters (`meshopt_simplifyScale`) embedded in the header to guarantee sub-pixel geometric accuracy across distances.
- **Bindless Descriptors**: Uses Vulkan descriptor indexing (`VK_EXT_descriptor_indexing`) with unbounded descriptor arrays for textures, SSBOs, and Top-Level Acceleration Structures (TLAS), indexing resources directly via PushConstants without per-draw binding changes.
- **Offline Asset Compiler**: Standalone tool (`tools/model_compiler.cpp`) using `fastgltf` and `tinyobjloader` to process `.gltf` and `.obj` assets into binary `.glb` files with embedded meshlet streams, discrete LOD levels, and pre-compressed KTX2 textures (BC7 for albedo/normals, BC5 for roughness/metallic) packed directly into GLB `BufferView` chunks.
- **Virtual File System (VFS)**: Packages SPIR-V shaders, models, and textures into a single archive (`engine.pak`) accessed via memory-mapping on Windows, with automatic disk fallback for loose files during development.
- **PBR Shading**: Evaluates Cook-Torrance BRDF (Albedo, Normal, Metallic, Roughness) with material parameters fetched from a global SSBO and modulated by ray-traced shadow occlusion.
- **Hardware Texture Mipmapping & Analytical Derivatives**: Closed-form screen-space barycentric derivatives (`dUVdx`, `dUVdy`) drive hardware `textureGrad` sampling for smooth, zero-pop anisotropic mipmap transitions.
- **Non-Intrusive Telemetry HUD & Profiler**: Click-through passive F3 HUD and F4 performance telemetry dashboards with a stable 5ms peak time-slot Frame Time (ms) graph with dynamic ceiling (+10% headroom), real-time `Low` / `Avg` / `Max` frame times, sub-pixel error metrics, and active RT BLAS diagnostics.
- **Scalar Block Layout**: Uses `GL_EXT_scalar_block_layout` so C++ structs and shader storage buffers share matching memory layout without manual alignment padding.

## Roadmap & Upcoming Features

1. **ReSTIR DI / GI**: Reservoir-based spatiotemporal importance resampling for direct and indirect lighting.
2. **Temporal Denoising (SVGF)**: Spatiotemporal variance-guided filtering to filter stochastic ray-traced lighting passes.

Check out the [full roadmap](docs/roadmap.md) for detailed progress and upcoming milestones.

## Code Architecture

For a complete breakdown of every directory, component, and the GPU pipeline flow, see the [File Architecture Documentation](docs/file.md).

> **Visualizing the Engine:** We also have an interactive Mermaid.js diagram showing exactly how data moves from CPU to GPU. Check out the **[Pipeline Flowchart](docs/flow.md)**. 
> 
> *Note: When viewing the chart on GitHub, be sure to click the **Fullscreen button** (the expanding arrows icon in the top right of the diagram box) so you can easily read the nodes and use the view controls.*

## Images
<img width="1487" height="1069" alt="Screenshot 2026-10-16 205928" src="https://github.com/user-attachments/assets/138fc5b0-ec4a-4b61-a595-2b71d36fdbca" />


## Release Downloads

Standalone, zero-dependency portable release archives (Windows 64-bit) are available on the [GitHub Releases](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases) page. Simply download, extract, and launch `VK_game_engine.exe`.

## Credits & Dependencies
- [Vulkan](https://www.vulkan.org/)
- [fastgltf](https://github.com/spnda/fastgltf)
- [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader)
- [meshoptimizer](https://github.com/zeux/meshoptimizer)
- [stb_image](https://github.com/nothings/stb)
- [GLFW](https://www.glfw.org/)
- [GLM](https://github.com/g-truc/glm)
- [Dear ImGui](https://github.com/ocornut/imgui)
