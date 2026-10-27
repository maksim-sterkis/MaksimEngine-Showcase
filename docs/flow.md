# Current Engine Pipeline

This diagram describes the implemented boulder renderer after Phase 4.1. F4 is the performance overlay and F5 contains interactive diagnostics. Phase 5 scene/material expansion, emissive area lighting, ReSTIR DI and SVGF are planned additions; multi-bounce GI is outside that phase.

```mermaid
flowchart TD
    subgraph Init["Initialization"]
        Main["demo/main.cpp"] --> VFS["VFS: engine.pak or loose files"]
        Main --> Runtime["Streamline initialization or native-only path"]
        Runtime --> Device["Vulkan instance/device and dispatch hooks"]
        Device --> Targets["Swapchain, two geometry slots, HDR and guide targets"]
        Main --> Assets["Boulder GLB: geometry, materials and textures"]
        Assets --> BLAS["Six LOD BLAS and scene TLAS"]
        Assets --> Bindless["Model SSBOs and global materials/textures"]
    end

    subgraph CPU["Frame Scheduling and Input"]
        Targets --> Fence["Wait for reusable frame slot"]
        Fence --> Reflex["Frame token, Reflex options and scheduling sleep"]
        Reflex --> SimStart["Simulation-start marker"]
        SimStart --> Input["Poll input and PCL pings; update camera/object/UI"]
        Input --> SimEnd["Simulation-end marker"]
        SimEnd --> Acquire["Configure rendering; acquire and retire swapchain image"]
        Acquire --> Frame["Current/previous transforms, jitter and depth-history metadata"]
    end

    subgraph Geometry["Culling and Visibility"]
        Frame --> Cull["Object frustum pre-cull and screen-space-error LOD"]
        Cull --> TLAS["Update TLAS with selected visible LOD or LOD 0"]
        BLAS --> TLAS
        Cull --> Path{"Hi-Z path"}
        Path -->|"Off / warm-up / unchanged / frozen"| Single["One geometry pass with applicable culling"]
        Path -->|"Compatible moving or jittered view"| Predict["Predict against previous producing-camera depth"]
        Predict --> Initial["Raster predicted-visible meshlets; queue deferred meshlets"]
        Initial --> CurrentDepth["Build current reverse-Z minimum pyramid"]
        CurrentDepth --> Recover["GPU indirect recovery test and raster with attachment LOAD"]
        Single --> Visibility["Completed visibility IDs and hardware depth"]
        Recover --> Visibility
    end

    subgraph Lighting["HDR Lighting and Reconstruction"]
        Visibility --> Deferred["Deferred attribute reconstruction, PBR and directional-sun shadow queries"]
        Bindless --> Deferred
        TLAS --> Deferred
        Deferred --> HDR["Linear HDR lighting"]
        Deferred --> Guides["Demanded motion/depth/normal/identity/validity guides"]
        HDR --> Mode{"Effective reconstruction"}
        Guides --> SDK
        Mode -->|"DLAA / DLSS K"| SDK["Streamline temporal reconstruction"]
        Mode -->|"Native 100%"| Native["Current HDR directly"]
        Mode -->|"Reduced Native diagnostics"| Spatial["Spatial reconstruction"]
        SDK --> Display["Tone mapping, background and output viewport"]
        Native --> Display
        Spatial --> Display
    end

    subgraph Finish["UI, History and Presentation"]
        Display --> UI["Output-resolution F3 HUD, F4 performance, F5 diagnostics and menus"]
        UI --> FinalDepth["Finalize required Hi-Z / paired F2 capture"]
        Visibility -.-> FinalDepth
        FinalDepth --> Submit["Submit and present markers with supported frame IDs"]
        Submit --> Present["Mailbox / explicit Immediate / FIFO"]
        Submit --> Commit["Commit successful-submission temporal and depth snapshots"]
        Commit -.-> Frame
        Present --> Recreate{"Recreation required?"}
        Recreate -->|"Yes"| Reset["Recreate affected resources and invalidate histories"]
        Reset -.-> Targets
        Recreate -->|"No"| Fence
    end
```

Geometry targets, HDR/guide images and history use submission-owned frame slots; acquired swapchain-image indices protect output images, descriptors and command buffers separately. Normal rendering has no test-image readback or added device-idle wait. Frame-slot/image fences and Reflex scheduling may wait.

Exactly unchanged views use compatible previous depth in one pass. Moving/jittered views recover predicted-occluded meshlets before lighting; overflow fails open by rendering immediately. Hi-Z history is independent of optional temporal guides and lighting resets.

Native color does not accumulate previous color. DLAA/DLSS owns its reconstruction history; future ReSTIR reservoirs and SVGF lighting histories will require their own state and invalidation. The current visibility/material path is still tailored to the boulder.

See [file architecture](file.md), [verification and limits](file.md#verification-and-limits) and the [Phase 5 roadmap](roadmap.md).
