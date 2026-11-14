# Current Engine Pipeline

This diagram describes the implemented boulder and Phase 5 sandbox pipeline (`v0.3.0`, completed 2026-11-14). ReSTIR DI runs initial candidate sampling, temporal reuse, spatial reuse and fresh final shadows with Basic real-time or ray-traced reference correction, 48-byte packed cached surfaces, 32-byte reservoirs and emission payload caching. See [interfaces and validation](file.md#verification-and-limits). There is no GI or production denoiser; DLSS Ray Reconstruction, Cornell Box, Crytek Sponza and extended integration validation follow in Phase 5.1, while SVGF stays deferred.

```mermaid
flowchart TD
    subgraph Init["Initialization"]
        Main["demo/main.cpp"] --> VFS["VFS: engine.pak or loose files"]
        Main --> Runtime["Streamline initialization or native-only path"]
        Runtime --> Device["Vulkan instance/device and dispatch hooks"]
        Device --> Targets["Swapchain, two geometry slots, HDR and guide targets"]
        Main --> Assets["Preloaded boulder and sandbox GLBs"]
        Assets --> BLAS["Six LOD BLAS and scene TLAS"]
        Assets --> Bindless["Buffer-address model table and global materials/textures"]
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
        Cull --> TLAS["Update all-instance TLAS with selected LOD or LOD 0"]
        Frame --> Lights["Prepare changed world-space triangle lights"]
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
        Visibility --> Deferred["Resolve instance/model/surface; PBR, sun or conventional/initial-RIS area lighting"]
        Lights --> Deferred
        Bindless --> Deferred
        TLAS --> Deferred
        Deferred --> HDR["Linear HDR; optional Native reference sum and mean"]
        Deferred --> TemporalDI["Initial candidates + temporal reuse (Basic fused); previous surface/light/TLAS"]
        TemporalDI --> SpatialDI["Scalar stable-pixel + general disocclusion reuse; Basic/reference correction"]
        TemporalDI -.-> TemporalHistory["Optional temporal-only history comparison"]
        SpatialDI --> FinalDI["Fresh final visibility and direct RGB (Basic fused with spatial)"]
        FinalDI --> HDR
        FinalDI -.-> HistoryDI["Retain final spatial reservoir for next frame by default"]
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
        HistoryDI -.-> Commit
        TemporalHistory -.-> Commit
        Commit -.-> Frame
        Present --> Recreate{"Recreation required?"}
        Recreate -->|"Yes"| Reset["Recreate affected resources and invalidate histories"]
        Reset -.-> Targets
        Recreate -->|"No"| Fence
    end
```

Geometry targets, HDR/guide images and history use submission-owned frame slots; acquired swapchain-image indices protect output images, descriptors and command buffers separately. Normal rendering has no test-image readback or added device-idle wait. Frame-slot/image fences and Reflex scheduling may wait.

Exactly unchanged views use compatible previous depth in one pass. Moving/jittered views recover predicted-occluded meshlets before lighting; overflow fails open by rendering immediately. Hi-Z history is independent of optional temporal guides and lighting resets.

Ordinary Native does not blend previous color. Only the explicit unjittered reference accumulates sample sums to 4096 spp. ReSTIR reuses samples independently of SDK/reference state. Final-spatial history restores repeated propagation; independent trajectory/regional/time-window checks replace reliance on one correlated sequence. Temporal-only history remains an optional comparison. See [verification and limits](file.md#verification-and-limits).

Basic material/initial generation and temporal reuse are fused; scalar stable-pixel and general disocclusion kernels fuse spatial reuse with final shading. Reference modes and ablations retain separate kernels. Current GPU records are 48 bytes per cached surface and 32 bytes per reservoir; retaining final output directly avoids a full-resolution history copy. Diagnostic shader variants and readback suites are opt-in and are absent from ordinary startup.

Importance targets still use full-contribution luminance. A tested rough-surface proxy was removed after failing to establish a reliable speedup against the original modules; final BRDF/shadow behavior and sample budgets remain unchanged. Material and regional reference checks remain part of the hidden offline audit.

See [file architecture](file.md), [verification and limits](file.md#verification-and-limits) and the [Phase 5 roadmap](roadmap.md).
