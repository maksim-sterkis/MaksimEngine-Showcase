# File Architecture and Connections

This document explains **every single file** in the `VK_game_engine` project and how they interact to form a modern GPU-driven Vulkan engine.

## `demo/` (Interactive Tech Demo Entry Point)
- **`demo/main.cpp`**: The primary executable entry point. Initializes the engine via `engine::init()`, loads assets (`asset_pool`), creates ECS entities (`MeshRenderer`, `Transform`), manages camera inputs (starting at `(3.0, 1.0, 0.0)` aimed dynamically at the target model's 3D bounding box center, with smooth `0.5` step speed scrolling in range `[0.5, 20.0]`), provides ImGui debug controls (freeze culling camera `F2`, manual LOD overrides via keys `0`–`6`, runtime culling stage toggles via keys `7`–`9`, RT shadow toggle `T`, meshlet debug coloring `F1`/`M`), manages the F4 performance telemetry HUD (frozen 5ms peak time-slot frame time graph with dynamic ceiling, `Low`/`Avg`/`Max` frame times, sub-pixel error readout, live RT BLAS diagnostics, sun azimuth/elevation controls, and tooltip suppression), updates frame logic, and drives the `draw_scene` and `cull_scene` rendering loops.

## `src/` (Core Engine Framework)

### `src/core/` (Platform, Windowing, Input & VFS)
- **`src/core/window.hpp` / `.cpp`**: An abstraction over the GLFW library. Creates the OS window, handles fullscreen/borderless toggling, registers resize callbacks, and tracks the window surface.
- **`src/core/input.hpp` / `.cpp`**: GLFW input system wrapper. Tracks keyboard key states (including F2 for freeze culling camera, F3 for debug HUD, and T for RT shadows) and mouse cursor movement, allowing `main.cpp` and `camera.cpp` to respond smoothly to user input.
- **`src/core/vfs.hpp` / `.cpp`**: Virtual File System. Implements memory-mapped binary archive streaming (`engine.pak`) via Windows `CreateFileMapping` / `MapViewOfFile` with zero-copy binary slice access (`std::span`), accompanied by transparent fallback to loose disk files when running in developer mode.
- **`src/core/settings.hpp` / `.cpp`**: Configuration and player preference persistence system. Implements INI serialization and deserialization (`settings.ini`), managing display mode, VSync, aspect ratio, meshlet debug colors, freeze culling, manual LOD overrides, and mouse sensitivity.
- **`src/core/camera.hpp` / `.cpp`**: Manages the 3D camera. Uses the `input` system to move around (WASD/Mouse), extracts 6 frustum planes, and calculates the View and Projection matrices sent to GPU shaders via PushConstants.

### `src/renderer/` (Vulkan Setup & GPU Pipeline)
- **`src/renderer/device.hpp` / `.cpp`**: Handles low-level GPU initialization. Creates the Vulkan `VkInstance`, queries and scores all available `VkPhysicalDevice` candidates (prioritizing dedicated discrete GPUs over integrated ones), creates the logical `VkDevice`, sets up queue families, creates the Command Pool, and enables critical modern device features:
  - `VK_EXT_descriptor_indexing` (Bindless textures and unbounded arrays)
  - `VK_EXT_mesh_shader` (Task and Mesh shaders)
  - `VK_EXT_scalar_block_layout` (Scalar block layout matching C++ structs to GPU buffers without manual alignment padding)
  - `samplerAnisotropy` (16x anisotropic filtering across all texture samplers)
  - `VK_KHR_acceleration_structure`, `VK_KHR_ray_query`, `VK_KHR_deferred_host_operations`, and `bufferDeviceAddress = VK_TRUE` (Hardware ray tracing)
- **`src/renderer/swapchain.hpp` / `.cpp`**: Manages the Vulkan Swapchain (the array of images presented to the screen). Handles querying surface capabilities, selecting present modes (Mailbox/Vsync), recreating the swapchain on resize, and allocating the **`VK_FORMAT_R32G32_UINT` Visibility Buffer** image alongside the Depth Buffer.
- **`src/renderer/engine.hpp` / `.cpp`**: The central orchestrator for the engine framework. It bundles `Device`, `Window`, and `Swapchain`. Manages frame synchronization (Fences and Semaphores), per-frame staging buffers, GPU indirect buffers (`VkDrawMeshTasksIndirectCommandEXT`), GPU instance buffers (`InstanceDataSSBO`), Hi-Z depth pyramid downsampling, fullscreen compute dispatches (`cull.comp`, `deferred.comp`), and presentation.
- **`src/renderer/pipeline.hpp` / `.cpp`**: Defines the Graphics and Compute Pipelines:
  - **Visibility Graphics Pipeline**: Compiles `shader.task`, `shader.mesh`, and `shader.frag`, rendering directly to the 64-bit Visibility Buffer.
  - **Cull Compute Pipeline**: Compiles `cull.comp` for object-level frustum, Hi-Z, and sub-pixel error LOD selection.
  - **Deferred Resolve Compute Pipeline**: Compiles `deferred.comp` for fullscreen Visibility Buffer decoding, inline ray query hardware ray tracing, and PBR evaluation.
  - Configures the **Bindless Global Descriptor Set Layout** (Binding 0: SSBO array, Binding 1: texture sampler array, Binding 2: Top-Level Acceleration Structure) and defines `PushConstantData`.
- **`src/renderer/raytracing.hpp` / `.cpp`**: Hardware ray tracing acceleration structures wrapper. Manages Bottom-Level Acceleration Structures (BLAS) for static meshes built with `VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR` across all discrete LOD levels (`lodBlas[0..5]`) with zero vertex buffer duplication, and dynamic Top-Level Acceleration Structures (TLAS) built each frame from ECS instance transforms, supporting dynamic per-frame BLAS pointer swapping to match the active rasterized LOD.
- **`src/renderer/imgui.hpp` / `.cpp`**: Integration of Dear ImGui with Vulkan. Sets up the ImGui context, allocates dedicated descriptor pools, and handles rendering real-time performance stats (passive F3 HUD, F4 performance telemetry graphs with 5ms peak intervals and tooltip suppression, ESC pause menu, and persistent settings dialog).

### `src/scene/` (Scene, Assets & Resources)
- **`src/scene/asset_pool.hpp` / `.cpp`**: Central asset registry holding loaded `ModelData` and `TextureData`. Compiles the global `MaterialSSBO`, registers texture samplers into the Global Descriptor Set array, and exposes model retrieval for geometric calculations.
- **`src/scene/model.hpp` / `.cpp`**: Uses `fastgltf` to parse binary `.glb` files streamed through the VFS. Parses multi-LOD meshlet headers ("MLOD"), stores up to 8 discrete LOD levels with individual meshlet counts, offsets, and cumulative 3D geometric surface deviation in meters, extracts discrete index buffers for building `lodBlas[0..5]`, and guards against double-free when destroying aliased BLAS handles.
- **`src/scene/texture.hpp` / `.cpp`**: Responsible for decoding images into VRAM. Decodes standard formats (PNG/JPG) using `stb_image` and streams pre-compressed KTX2 textures (BC7/BC5) directly from in-memory GLB buffer chunks using `libktx` (`ktxTexture2_CreateFromMemory`, `ktxTexture2_IterateLevels`), generates complete trilinear mipmap chains down to $1\times 1$ via `vkCmdBlitImage`, and creates `VkSampler` objects with 16x anisotropic filtering.
- **`src/scene/ecs.hpp` / `.cpp`**: A custom, lightweight Entity Component System (ECS). Uses `entt`-style sparse sets to manage `Entity` IDs and their attached components (`Transform`, `MeshRenderer`).

## `shaders/` (GPU Programs)
- **`shaders/cull.comp`**: The Tier-1 GPU Compute Pre-Cull shader. Evaluates 6-plane frustum culling and conservative multi-mip Hi-Z occlusion tests on instance bounding boxes. Calculates sub-pixel screen-space projected geometric surface deviation ($\le 1.0\text{ px}$ budget) to select the optimal discrete LOD (LOD 0 to 5), writes `VkDrawMeshTasksIndirectCommandEXT` directly into an indirect SSBO, and populates draw counts.
- **`shaders/shader.task`**: The Tier-2 Vulkan Task Shader. Evaluates sub-mesh meshlet bounding spheres against the camera frustum (toggleable via key `7`), tests for cone backface culling (toggleable via key `8`), and performs robust Hi-Z occlusion culling (toggleable via key `9`) using linear metric eye-distance testing (`mesh_linear_dist > hiz_linear_dist + 0.25m`), viewport perimeter guards, and bounded mip downsampling to mitigate false-positive culling on concave craters or screen edges. Dispatches surviving meshlets to `shader.mesh` and updates atomic debug stats.
- **`shaders/shader.mesh`**: The Vulkan Mesh Shader. Reads packed meshlet vertex and triangle indices from GPU SSBOs, emits minimal vertex positions (`gl_Position`), and outputs `outMeshletIndex` and per-primitive IDs (`gl_PrimitiveID`) directly to the rasterizer.
- **`shaders/shader.frag`**: The Visibility Buffer fragment shader. Writes the 64-bit compact ID payload `uvec2((instanceId << 20) | inMeshletIndex, inPrimitiveID)` directly into the `R32G32_UINT` Visibility Buffer.
- **`shaders/deferred.comp`**: The Fullscreen Visibility Buffer Resolve Compute Shader. Decodes pixel IDs from `inVisibility`, loads the 3 triangle vertices from Bindless SSBOs, analytically reconstructs 2D screen-space barycentric coordinates in NDC, evaluates closed-form analytical screen-space UV derivatives ($\text{dUVdx}, \text{dUVdy}$), samples Bindless PBR textures using hardware `textureGrad` (anisotropic + trilinear) without artificial LOD clamping, and executes hardware inline ray queries (`rayQueryInitializeEXT`, `rayQueryProceedEXT`) against the scene TLAS to evaluate contact-accurate directional shadows with normal-offset bias.

## `tools/` (Offline Asset Pipeline & Packaging)
- **`tools/model_compiler.cpp`**: Standalone offline asset pipeline. Reads raw `.gltf` and `.obj` files:
  1. Deduplicates normal-split duplicate vertices using `meshopt_generateVertexRemapMulti` on `(Position, UV)` while strictly preserving genuine UV chart boundaries.
  2. Generates 6 discrete progressive LOD levels (100%, 50%, 25%, 12.5%, 6.25%, 3.125%) using `meshopt_simplifyWithAttributes` with UV streams and `meshopt_SimplifyLockBorder` to preserve seamless chart boundaries and silhouette contours.
  3. Computes true cumulative 3D geometric surface deviation in meters via `meshopt_simplifyScale` and records it into multi-LOD level headers.
  4. Optimizes meshlet clusters using `meshopt_optimizeMeshlet` to maximize vertex cache locality.
  5. Calculates conservative bounding spheres and cone cutoffs with safety margins to avoid false-positive culling on simplified silhouettes.
  6. Compresses raw textures into KTX2 (BC7 for albedo/normal, BC5 for metallic-roughness) and embeds them directly into binary `.glb` `BufferView` chunks.
  7. Supports CLI parameters (`--force`, explicit input and output file paths) for flexible developer compilation.
- **`tools/texture_compiler.cpp`**: Helper tool used for batch conversions of raw texture directories into `KTX2` format.
- **`tools/pak_compiler.cpp`**: Asset packaging tool that bundles compiled SPIR-V shaders, GLB models, and textures into an indexed, contiguous memory-mapped binary archive (`engine.pak`) for distribution.
- **`package_release.ps1`**: Master packaging automation script. Reconfigures CMake with `-DCMAKE_BUILD_TYPE=Release` (defining `NDEBUG` to strip validation layers), builds optimized targets with static MinGW runtime linking, compiles `engine.pak`, strips symbols, and packages a zero-dependency portable `.zip`.

---

## How They Connect (The Complete Pipeline Flow)

1. **Bootstrapping**: `demo/main.cpp` calls `engine::init()`, which initializes `window`, `device`, `swapchain`, and `pipeline` in order, setting up both graphics and compute pipelines, enabling ray tracing extensions (`VK_KHR_acceleration_structure`, `VK_KHR_ray_query`, `VK_KHR_deferred_host_operations`), and allocating the Visibility Buffer and Depth targets.
2. **Asset Loading**: `demo/main.cpp` requests models through `asset_pool`. `model::load_glb()` loads the multi-LOD meshlet payload, pushes vertex/meshlet SSBOs to VRAM with BDA flags, builds discrete Bottom-Level Acceleration Structures (`lodBlas[0..5]`) across all 6 LOD levels on the GPU with zero vertex duplication, streams pre-compressed KTX2 textures directly from in-memory GLB chunks, and `texture.cpp` generates trilinear mipmap chains.
3. **Bindless Sync**: `asset_pool::build_materials_ssbo()` compiles all material properties into the global `MaterialSSBO`, registers all texture views/samplers into the single unbounded Global Descriptor Set, and prepares the TLAS acceleration structure descriptor.
4. **Logic Update**: In the frame loop, `input` processes keyboard/mouse events (including hotkey `T` for RT shadows and keys `0`–`6` for LOD selection), `camera` computes the View/Projection matrices and frustum planes, and active ECS entities update their `Transform` components.
5. **Rendering Pass**:
   - **Phase A (Tier-1 Compute Pre-Cull & TLAS Build)**:
     - `engine.cpp` uploads instance bounding boxes and transforms into `instanceBuffers`.
     - `cull.comp` runs: tests instance AABBs against frustum planes and previous-frame Hi-Z depth pyramid, computes projected sub-pixel geometric surface deviation ($\le 1.0\text{ px}$ budget) to select the optimal discrete LOD level (0–5), and writes `VkDrawMeshTasksIndirectCommandEXT` directly into `indirectBuffers`.
     - `cull_scene` executes dynamic TLAS pointer swapping: updates the instance's active BLAS device address to match the evaluated LOD (or manual override `0`–`6`).
     - `raytracing::build_tlas()` records `vkCmdBuildAccelerationStructuresKHR` guarded by an acceleration structure memory barrier.
     - A pipeline barrier synchronizes compute writes to indirect draw reads (`VK_ACCESS_INDIRECT_COMMAND_READ_BIT`).
   - **Phase B (Visibility Geometry Pass)**:
     - `engine.cpp` begins the Visibility renderpass targeting the `R32G32_UINT` Visibility Buffer and Depth Buffer.
     - Issues `vkCmdDrawMeshTasksIndirectEXT` driven by the GPU indirect buffer.
     - `shader.task` performs sub-mesh frustum, backface cone, and Hi-Z culling, emitting surviving meshlets.
     - `shader.mesh` emits screen vertices and primitive indices; `shader.frag` writes 64-bit `(instanceId | meshletIndex, primitiveID)` payloads.
   - **Phase C (Hi-Z Pyramid Downsample)**:
     - The depth buffer is downsampled across a multi-level mip pyramid using compute/blit shaders to serve conservative occlusion culling for subsequent frames.
   - **Phase D (Fullscreen Deferred Visibility Resolve & Hardware Ray Queries)**:
     - A pipeline barrier transitions the Visibility Buffer to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`.
     - `deferred.comp` runs: samples each pixel's 64-bit ID, loads triangle vertices from the global SSBOs, reconstructs 2D NDC barycentrics with perspective correction, evaluates closed-form analytical screen-space UV derivatives ($\text{dUVdx}, \text{dUVdy}$), samples Bindless PBR textures using hardware `textureGrad`, fires hardware inline ray queries via `rayQueryProceedEXT` against the TLAS for direct directional lighting occlusion, and writes shaded Cook-Torrance PBR color to the swapchain image without redundant fragment shading or primary ray traversal overhead.
   - **Phase E (Debug Overlay & Presentation)**:
     - `imgui` renders the passive F3 HUD and F4 performance telemetry graphs (Frame Time history with 5ms peak slots, `Low` / `Avg` / `Max` frame times, sub-pixel error readout, active RT BLAS diagnostics, and sun controls).
     - `engine.cpp` submits the command buffer and presents the swapchain image to the display.
