# VK Game Engine

A modern Vulkan game engine written in C++20, designed with next-generation rendering techniques in mind.

## Features

- **Bindless Architecture**: Utilizes Vulkan 1.2 Descriptor Indexing (`VK_EXT_descriptor_indexing`) to drastically reduce CPU overhead during draw calls. An unbounded array of 100,000 descriptors allows for virtually limitless textures and materials mapped directly via PushConstants.
- **Offline Asset Compiler**: Custom asset pipeline utilizing `fastgltf` and `tinyobjloader` to process `.obj` and `.gltf` source files into optimized, single-binary `.glb` payloads. Incorporates topological Pos+UV pre-welding to eliminate redundant normal-split locks without disturbing UV chart seams.
- **Embedded Textures & Meshlets**: The compiler natively reads raw PBR texture files (JPEGs/PNGs) and packages them dynamically into the `.glb` buffers, and uses `meshoptimizer` to partition geometry into optimized Meshlets (max 64 vertices, 124 triangles) across 5 discrete LOD levels.
- **5-Level Discrete Meshlet LODs**: Generates 5 progressive simplification levels (100%, 50%, 25%, 12.5%, 6.25%) using strict UV-preserving decimation (zero texture seam distortion or contour artifacts). Stores exact 3D triangle edge length metadata to drive runtime screen-space pixel projection metrics.
- **Visibility Buffer Architecture**: Decouples geometry rasterization from heavy material evaluation. The raster pass writes a compact 64-bit ID `(meshletIndex, primitiveID)` to an `R32G32_UINT` target using `VK_KHR_fragment_shader_barycentric`. Material shading runs in a fullscreen compute pass (`shaders/deferred.comp`) with analytical 2D screen-space barycentric reconstruction, achieving absolute zero material overdraw.
- **Two-Tier GPU Culling & Indirect Dispatch**: 
  - **Tier 1 (Instance Pre-Cull)**: Compute shader (`shaders/cull.comp`) evaluates 6-plane frustum tests and conservative multi-mip Hi-Z occlusion tests on instance bounding boxes, calculates projected screen-space triangle pixel size to dynamically select the LOD level, and writes `VkDrawMeshTasksIndirectCommandEXT` directly into GPU indirect buffers.
  - **Tier 2 (Meshlet Sub-Mesh Cull)**: Task shaders (`shaders/shader.task`) execute sub-mesh frustum, cone backface, and Hi-Z occlusion culling per meshlet, emitting surviving meshlets to the Mesh Shader (`shaders/shader.mesh`).
- **Dynamic Asset Pool**: Robust texture and model pooling system preventing duplicate GPU uploads and seamlessly switching between raw JPEG/PNG loading (using `stb_image`) and compressed formats.
- **PBR Materials**: Complete physical based rendering foundation with Cook-Torrance BRDF (Albedo, Normal, Metallic, Roughness) via SSBOs.
- **Hardware Texture Mipmapping**: Generates full mip chains via `vkCmdBlitImage`. Samples trilinear mipmaps in deferred compute using dynamic screen-space texel footprint estimation clamped to the active mesh LOD level for seamless distance transitions.
- **Perfect Memory Packing**: Uses `GL_EXT_scalar_block_layout` to map C++ structs exactly to GPU memory without any padding overhead.

## Roadmap & Upcoming Features

1. **Hardware Ray Tracing**: Leverage `VK_KHR_ray_tracing_pipeline` (RT cores) for precise shadows, reflections, and ambient occlusion using BLAS/TLAS acceleration structures built from meshlet LODs.
2. **ReSTIR DI / GI**: State-of-the-art reservoir spatiotemporal importance resampling for real-time direct and global illumination.
3. **Temporal Denoising (SVGF)**: Spatiotemporal variance-guided filtering to denoise stochastic ray-traced lighting passes.

Check out the [full roadmap](roadmap.md) for detailed progress and upcoming milestones.

## Code Architecture

For a complete breakdown of every directory, component, and the GPU pipeline flow, see the [File Architecture Documentation](file.md).

> **Visualizing the Engine:** We also have an interactive Mermaid.js diagram showing exactly how data moves from CPU to GPU. Check out the **[Pipeline Flowchart](flow.md)**. 
> 
> *Note: When viewing the chart on GitHub, be sure to click the **Fullscreen button** (the expanding arrows icon in the top right of the diagram box) so you can easily read the nodes and use the view controls.*

## Images
<img width="2251" height="1185" alt="Screenshot 2026-07-09 175616" src="https://github.com/user-attachments/assets/51e2e0f5-ec02-458e-b2f3-72a4e18ddf5e" />

## Build Instructions

### Prerequisites
- **CMake** 3.20+
- **C++20** compliant compiler (GCC/MinGW, MSVC, or Clang)
- **Vulkan SDK** 1.2+

### Building

The project uses CMake to fetch dependencies (like `fastgltf`, `glfw`, `glm`, `imgui`) automatically.

```bash
mkdir build
cd build
cmake ..
cmake --build .
```

### Running

To run the offline model compiler to package your assets (put source models in `assets/models/obj`):
```bash
./build/model_compiler.exe
```

To run the engine itself:
```bash
./build/VK_game_engine.exe
```

## Credits & Dependencies
- [Vulkan](https://www.vulkan.org/)
- [fastgltf](https://github.com/spnda/fastgltf)
- [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader)
- [meshoptimizer](https://github.com/zeux/meshoptimizer)
- [stb_image](https://github.com/nothings/stb)
- [GLFW](https://www.glfw.org/)
- [GLM](https://github.com/g-truc/glm)
- [Dear ImGui](https://github.com/ocornut/imgui)

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.