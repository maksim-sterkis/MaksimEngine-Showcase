# File Architecture and Connections

This document explains **every single file** in the `VK_game_engine` project and how they interact to form a modern GPU-driven Vulkan engine.

## `demo/` (Interactive Tech Demo Entry Point)
- **`demo/main.cpp`**: The primary executable entry point. Initializes the engine via `engine::init()`, loads assets (`asset_pool`), creates ECS entities (`MeshRenderer`, `Transform`), manages camera inputs, provides ImGui debug controls (freeze culling camera, manual LOD 0–4 overrides, meshlet debug coloring), updates frame logic, and drives the `draw_scene` rendering loop.

## `src/` (Core Engine Framework)

### `src/core/` (Platform, Windowing, Input & VFS)
- **`src/core/window.hpp` / `.cpp`**: An abstraction over the GLFW library. Creates the OS window, handles fullscreen/borderless toggling, registers resize callbacks, and tracks the window surface.
- **`src/core/input.hpp` / `.cpp`**: GLFW input system wrapper. Tracks keyboard key states (including F2 for freeze culling camera and F3 for debug UI) and mouse cursor movement, allowing `main.cpp` and `camera.cpp` to respond smoothly to user input.
- **`src/core/vfs.hpp` / `.cpp`**: Virtual File System. Implements memory-mapped binary archive streaming (`engine.pak`) via Windows `CreateFileMapping` / `MapViewOfFile` with zero-copy binary slice access (`std::span`), accompanied by transparent fallback to loose disk files when running in developer mode.
- **`src/core/camera.hpp` / `.cpp`**: Manages the 3D camera. Uses the `input` system to move around (WASD/Mouse), extracts 6 frustum planes, and calculates the View and Projection matrices sent to GPU shaders via PushConstants.

### `src/renderer/` (Vulkan Setup & GPU Pipeline)
- **`src/renderer/device.hpp` / `.cpp`**: Handles low-level GPU initialization. Creates the Vulkan `VkInstance`, queries and scores all available `VkPhysicalDevice` candidates (prioritizing dedicated discrete GPUs over integrated ones), creates the logical `VkDevice`, sets up queue families, creates the Command Pool, and enables critical modern device features:
  - `VK_EXT_descriptor_indexing` (Bindless textures and unbounded arrays)
  - `VK_EXT_mesh_shader` (Task and Mesh shaders)
  - `VK_EXT_scalar_block_layout` (Scalar block layout matching C++ structs to GPU buffers without manual alignment padding)
- **`src/renderer/swapchain.hpp` / `.cpp`**: Manages the Vulkan Swapchain (the array of images presented to the screen). Handles querying surface capabilities, selecting present modes (Mailbox/Vsync), recreating the swapchain on resize, and allocating the **`VK_FORMAT_R32G32_UINT` Visibility Buffer** image alongside the Depth Buffer.
- **`src/renderer/engine.hpp` / `.cpp`**: The central orchestrator for the engine framework. It bundles `Device`, `Window`, and `Swapchain`. Manages frame synchronization (Fences and Semaphores), per-frame staging buffers, GPU indirect buffers (`VkDrawMeshTasksIndirectCommandEXT`), GPU instance buffers (`InstanceDataSSBO`), Hi-Z depth pyramid downsampling, fullscreen compute dispatches (`cull.comp`, `deferred.comp`), and presentation.
- **`src/renderer/pipeline.hpp` / `.cpp`**: Defines the Graphics and Compute Pipelines:
  - **Visibility Graphics Pipeline**: Compiles `shader.task`, `shader.mesh`, and `shader.frag`, rendering directly to the 64-bit Visibility Buffer.
  - **Cull Compute Pipeline**: Compiles `cull.comp` for object-level frustum, Hi-Z, and LOD selection.
  - **Deferred Resolve Compute Pipeline**: Compiles `deferred.comp` for fullscreen Visibility Buffer decoding and PBR evaluation.
  - Configures the **Bindless Global Descriptor Set Layout** (Binding 0: SSBO array, Binding 1: texture sampler array) and defines `PushConstantData`.
- **`src/renderer/imgui.hpp` / `.cpp`**: Integration of Dear ImGui with Vulkan. Sets up the ImGui context, allocates dedicated descriptor pools, and handles rendering real-time performance stats (active LOD, meshlet counters, triangle counts, freeze toggles).

### `src/scene/` (Scene, Assets & Resources)
- **`src/scene/asset_pool.hpp` / `.cpp`**: Central asset registry holding loaded `ModelData` and `TextureData`. Compiles the global `MaterialSSBO` and registers texture samplers into the Global Descriptor Set array.
- **`src/scene/model.hpp` / `.cpp`**: Uses `fastgltf` to parse binary `.glb` files streamed through the VFS. Parses multi-LOD meshlet headers ("MLOD"), stores up to 8 discrete LOD levels with individual meshlet counts, offsets, and average 3D triangle edge lengths $L_{\text{tri}}$, and allocates GPU SSBO buffers for vertices, meshlets, vertex indices, and triangle indices.
- **`src/scene/texture.hpp` / `.cpp`**: Responsible for decoding images into VRAM. Decodes standard formats (PNG/JPG) using `stb_image`, generates complete trilinear mipmap chains down to $1\times 1$ via `vkCmdBlitImage`, and creates `VkSampler` objects with anisotropic filtering.
- **`src/scene/ecs.hpp` / `.cpp`**: A custom, lightweight Entity Component System (ECS). Uses `entt`-style sparse sets to manage `Entity` IDs and their attached components (`Transform`, `MeshRenderer`).

## `shaders/` (GPU Programs)
- **`shaders/cull.comp`**: The Tier-1 GPU Compute Pre-Cull shader. Evaluates 6-plane frustum culling and conservative multi-mip Hi-Z occlusion tests on instance bounding boxes. Calculates screen-space projected triangle pixel dimensions to select the optimal discrete LOD (LOD 0 to 4), writes `VkDrawMeshTasksIndirectCommandEXT` directly into an indirect SSBO, and populates draw counts.
- **`shaders/shader.task`**: The Tier-2 Vulkan Task Shader. Evaluates sub-mesh meshlet bounding spheres against the camera frustum, tests for cone backface culling, and performs Hi-Z occlusion culling per meshlet. Dispatches surviving meshlets to `shader.mesh` and updates atomic debug stats.
- **`shaders/shader.mesh`**: The Vulkan Mesh Shader. Reads packed meshlet vertex and triangle indices from GPU SSBOs, emits minimal vertex positions (`gl_Position`), and outputs `outMeshletIndex` and per-primitive IDs (`gl_PrimitiveID`) directly to the rasterizer.
- **`shaders/shader.frag`**: The Visibility Buffer fragment shader. Writes the 64-bit compact ID payload `uvec2((instanceId << 20) | inMeshletIndex, inPrimitiveID)` directly into the `R32G32_UINT` Visibility Buffer.
- **`shaders/deferred.comp`**: The Fullscreen Visibility Buffer Resolve Compute Shader. Decodes pixel IDs from `inVisibility`, loads the 3 triangle vertices from Bindless SSBOs, analytically reconstructs 2D screen-space barycentric coordinates in NDC, calculates perspective-correct attribute weights, evaluates texel footprint, and samples Bindless PBR textures with hardware trilinear mipmapping without redundant fragment shading on occluded surfaces.

## `tools/` (Offline Asset Pipeline & Packaging)
- **`tools/model_compiler.cpp`**: Standalone offline asset pipeline. Reads raw `.gltf` and `.obj` files:
  1. Deduplicates normal-split duplicate vertices using `meshopt_generateVertexRemapMulti` on `(Position, UV)` while strictly preserving genuine UV chart boundaries.
  2. Generates 5 discrete progressive LOD levels (100%, 50%, 25%, 12.5%, 6.25%) using `meshopt_simplify` with locked UV seams to preserve texture chart boundaries and silhouette contours.
  3. Calculates exact 3D triangle edge length metadata $L_{\text{tri}}$ for each LOD.
  4. Partitions each LOD into optimized Meshlets (max 64 vertices, 124 triangles) using `meshopt_buildMeshlets`.
  5. Remaps vertex indices back to original vertex buffers via `remapToOriginal`.
  6. Packages the geometry, embedded textures, and multi-LOD meshlet streams into a single, dense `.glb` binary payload.
- **`tools/texture_compiler.cpp`**: Helper tool used for batch conversions of raw texture directories into `KTX2` format.
- **`tools/pak_compiler.cpp`**: Asset packaging tool that bundles compiled SPIR-V shaders, GLB models, and textures into an indexed, contiguous memory-mapped binary archive (`engine.pak`) for distribution.
- **`package_release.ps1`**: Master packaging automation script. Reconfigures CMake with `-DCMAKE_BUILD_TYPE=Release` (defining `NDEBUG` to strip validation layers), builds optimized targets with static MinGW runtime linking, compiles `engine.pak`, strips symbols, and packages a zero-dependency portable `.zip`.

---

## How They Connect (The Complete Pipeline Flow)

1. **Bootstrapping**: `demo/main.cpp` calls `engine::init()`, which initializes `window`, `device`, `swapchain`, and `pipeline` in order, setting up both graphics and compute pipelines and allocating the Visibility Buffer and Depth targets.
2. **Asset Loading**: `demo/main.cpp` requests models through `asset_pool`. `model::load_glb()` loads the multi-LOD meshlet payload, pushes vertex/meshlet SSBOs to VRAM, extracts embedded textures, and `texture.cpp` generates trilinear mipmap chains.
3. **Bindless Sync**: `asset_pool::build_materials_ssbo()` compiles all material properties into the global `MaterialSSBO` and registers all texture views/samplers into the single unbounded Global Descriptor Set.
4. **Logic Update**: In the frame loop, `input` processes keyboard/mouse events, `camera` computes the View/Projection matrices and frustum planes, and active ECS entities update their `Transform` components.
5. **Rendering Pass**:
   - **Phase A (Tier-1 Compute Pre-Cull)**:
     - `engine.cpp` uploads instance bounding boxes and transforms into `instanceBuffers`.
     - `cull.comp` runs: tests instance AABBs against frustum planes and previous-frame Hi-Z depth pyramid, computes projected triangle pixel size to select LOD level (0–4), and writes `VkDrawMeshTasksIndirectCommandEXT` directly into `indirectBuffers`.
     - A pipeline barrier synchronizes compute writes to indirect draw reads (`VK_ACCESS_INDIRECT_COMMAND_READ_BIT`).
   - **Phase B (Visibility Geometry Pass)**:
     - `engine.cpp` begins the Visibility renderpass targeting the `R32G32_UINT` Visibility Buffer and Depth Buffer.
     - Issues `vkCmdDrawMeshTasksIndirectEXT` driven by the GPU indirect buffer.
     - `shader.task` performs sub-mesh frustum, backface cone, and Hi-Z culling, emitting surviving meshlets.
     - `shader.mesh` emits screen vertices and primitive indices; `shader.frag` writes 64-bit `(instanceId | meshletIndex, primitiveID)` payloads.
   - **Phase C (Hi-Z Pyramid Downsample)**:
     - The depth buffer is downsampled across a multi-level mip pyramid using compute/blit shaders to serve conservative occlusion culling for subsequent frames.
   - **Phase D (Fullscreen Deferred Visibility Resolve)**:
     - A pipeline barrier transitions the Visibility Buffer to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`.
     - `deferred.comp` runs: samples each pixel's 64-bit ID, loads triangle vertices from the global SSBOs, reconstructs 2D NDC barycentrics with perspective correction, evaluates texel footprint, and writes shaded PBR color to the swapchain image without redundant fragment shading on occluded surfaces.
   - **Phase E (Debug Overlay & Presentation)**:
     - `imgui` renders the debug HUD (FPS, active LOD, meshlet culling counts, freeze camera toggle, manual LOD overrides).
     - `engine.cpp` submits the command buffer and presents the swapchain image to the display.
