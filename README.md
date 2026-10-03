# VK Game Engine

[![Vulkan](https://img.shields.io/badge/Vulkan-1.3-ED1B24?style=for-the-badge&logo=vulkan&logoColor=white)](https://www.vulkan.org/)
[![Platform](https://img.shields.io/badge/Platform-Windows%20x64-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases)
[![C++20](https://img.shields.io/badge/C%2B%2B-20-00599C?style=for-the-badge&logo=c%2B%2B&logoColor=white)](https://en.cppreference.com/w/cpp/20)
[![Pipeline](https://img.shields.io/badge/Pipeline-Mesh%20Shaders%20%2B%20VisBuffer-orange?style=for-the-badge)](docs/file.md)
[![Release](https://img.shields.io/badge/Release-v0.1.0-8A2BE2?style=for-the-badge&logo=github)](https://github.com/maksim-sterkis/MaksimEngine-Showcase/releases)

A Vulkan 1.3 experimental rendering engine written in C++20, implementing a modern GPU-driven pipeline with mesh shaders, visibility buffer shading, discrete meshlet LODs, and hierarchical-Z occlusion culling.

## Features

- **Visibility Buffer Shading**: Decouples geometry rasterization from material evaluation. The geometry pass rasterizes 64-bit IDs `(meshletIndex, primitiveID)` to an `R32G32_UINT` attachment. A fullscreen compute pass (`shaders/deferred.comp`) reconstructs screen-space barycentric coordinates and evaluates PBR materials only on visible pixels, avoiding redundant fragment shading on occluded surfaces.
- **Meshlet Pipeline**: Geometry is partitioned into meshlets (up to 64 vertices, 124 triangles) using `meshoptimizer` and rendered via Task and Mesh shaders (`VK_EXT_mesh_shader`).
- **Two-Phase GPU Culling & Indirect Dispatch**:
  - *Instance Pre-Cull (`shaders/cull.comp`)*: Evaluates bounding box frustum culling and Hi-Z occlusion tests, selects the active LOD step based on projected screen-space triangle dimensions, and writes indirect draw commands (`VkDrawMeshTasksIndirectCommandEXT`).
  - *Meshlet Culling (`shaders/shader.task`)*: Evaluates meshlet bounding spheres against the view frustum, cone backface orientation, and conservative Hi-Z occlusion before emitting surviving meshlets to the mesh shader.
- **5-Level Discrete Meshlet LODs**: Generates 5 progressive simplification steps (100%, 50%, 25%, 12.5%, 6.25%) with embedded 3D triangle edge length metadata to drive runtime screen-space pixel projection metrics.
- **Bindless Descriptors**: Uses Vulkan descriptor indexing (`VK_EXT_descriptor_indexing`) with unbounded descriptor arrays for textures and SSBOs, indexing materials and resources directly via PushConstants without per-draw binding changes.
- **Offline Asset Compiler**: Standalone tool (`tools/model_compiler.cpp`) using `fastgltf` and `tinyobjloader` to process `.gltf` and `.obj` assets into `.glb` files with embedded meshlet streams, LOD levels, and textures. Welds normal-split vertices sharing UV coordinates prior to simplification to preserve texture chart seams.
- **Virtual File System (VFS)**: Packages SPIR-V shaders, models, and textures into a single archive (`engine.pak`) accessed via memory-mapping on Windows, with automatic disk fallback for loose files during development.
- **PBR Shading**: Evaluates Cook-Torrance BRDF (Albedo, Normal, Metallic, Roughness) with material parameters fetched from a global SSBO.
- **Hardware Texture Mipmapping**: Generates mipmaps via `vkCmdBlitImage`. Samples trilinear mip levels in deferred compute using screen-space texel footprint estimation clamped to the active mesh LOD level.
- **Scalar Block Layout**: Uses `GL_EXT_scalar_block_layout` so C++ structs and shader storage buffers share matching memory layout without manual alignment padding.

## Roadmap & Upcoming Features

1. **Hardware Ray Tracing**: Ray-traced shadows, ambient occlusion, and reflections using `VK_KHR_ray_tracing_pipeline` with BLAS/TLAS structures built from mesh geometry.
2. **ReSTIR DI / GI**: Reservoir-based spatiotemporal importance resampling for direct and indirect lighting.
3. **Temporal Denoising (SVGF)**: Spatiotemporal variance-guided filtering to filter stochastic ray-traced lighting passes.

Check out the [full roadmap](docs/roadmap.md) for detailed progress and upcoming milestones.

## Code Architecture

For a complete breakdown of every directory, component, and the GPU pipeline flow, see the [File Architecture Documentation](docs/file.md).

> **Visualizing the Engine:** We also have an interactive Mermaid.js diagram showing exactly how data moves from CPU to GPU. Check out the **[Pipeline Flowchart](docs/flow.md)**. 
> 
> *Note: When viewing the chart on GitHub, be sure to click the **Fullscreen button** (the expanding arrows icon in the top right of the diagram box) so you can easily read the nodes and use the view controls.*

## Images
<img src="https://github.com/user-attachments/assets/51e2e0f5-ec02-458e-b2f3-72a4e18ddf5e" alt="VK Game Engine - Meshlet LOD & Visibility Buffer Tech Demo" width="100%" />

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