```mermaid
graph TD
    %% Step 1
    subgraph Step 1: Bootstrapping
        Main1[main.cpp: Default Cam 3,1,0] -->|vfs::initialize| VFS[VFS: engine.pak / Disk Fallback]
        Main1 -->|engine::init| Engine[engine.cpp]
        Engine --> Win[window: GLFW]
        Engine --> Dev[device: Vulkan 1.3 + Mesh Shaders + RT Extensions]
        Engine --> Swap[swapchain: VisBuffer R32G32_UINT + Depth]
        Engine --> Pipe[pipeline: Graphics & Compute Pipelines]
    end

    %% Step 2 & 3
    subgraph Step 2 & 3: Asset Load & Bindless Sync
        Main1 -->|Requests Model| Pool[asset_pool]
        Pool -->|model::load_glb| Mod[model.cpp: 6-Level Meshlet LODs]
        Mod -->|Build Discrete BLAS| BLAS[(Multi-LOD BLAS: lodBlas 0..5)]
        Mod -->|Embedded KTX2 Textures| Tex[texture.cpp: Generate Mipmaps]
        Tex -->|Uploads to GPU| VRAM[(GPU VRAM: SSBOs & Textures)]
        
        Pool -->|build_materials_ssbo| Desc[Global Descriptor Set: Bindless]
        Pool -->|Pack Materials| SSBO[(Material & Meshlet SSBOs)]
    end

    %% Step 4
    subgraph Step 4: Logic Update
        Inp[input: Key T Shadows / Keys 0-6 LOD / F2 Freeze / Sun Controls] -->|Updates| Cam[camera: View/Proj & Frustum Planes]
        ECS[ecs] -->|Transforms & Bounding Boxes| Instances[Upload Instance SSBO]
    end

    %% Step 5
    subgraph Step 5: Rendering Pass
        %% Phase A
        subgraph Phase A: Tier-1 GPU Pre-Cull & TLAS Build
            Instances --> CullComp[shaders/cull.comp]
            Cam --> CullComp
            CullComp -->|Frustum & Hi-Z Test| CullDecide{Visible?}
            CullDecide -->|Yes: Sub-Pixel Error <= 1.0px| PickLOD[Select LOD 0-5]
            PickLOD -->|Dynamic Pointer Swap: Matching BLAS| TLAS[raytracing: Build Dynamic TLAS]
            TLAS --> Desc
            PickLOD -->|Populate Draw Command| IndBuffer[(Indirect Draw SSBO)]
        end

        %% Phase B
        subgraph Phase B: Visibility Geometry Pass
            IndBuffer -->|vkCmdDrawMeshTasksIndirectEXT| TaskSh[shaders/shader.task]
            TaskSh -->|Sub-mesh Frustum/Cone/Hi-Z| MeshSh[shaders/shader.mesh]
            MeshSh -->|Compact Primitives| FragSh[shaders/shader.frag]
            FragSh -->|Write 64-bit Payload| VisBuf[(Visibility Buffer R32G32_UINT)]
        end

        %% Phase C & D
        subgraph Phase C & D: Hi-Z & Deferred Resolve with RT Shadows
            VisBuf --> DefComp[shaders/deferred.comp]
            Desc -.->|Bindless SSBOs & TLAS| DefComp
            DefComp -->|Inline Ray Query: Shadow Rays| RayQ[rayQueryProceedEXT: RT Shadows]
            RayQ --> ShadedFrame[Shaded Frame: Cook-Torrance + RT Shadows]
        end

        ShadedFrame --> UI[imgui: F3 HUD & F4 Frame Time Telemetry]
        UI --> Present[engine.cpp: Submit & Present to Screen]
    end
```
