# Temporal Rendering Foundation

The Phase 4.1 foundation supplies rendering inputs/history validation and model K DLAA/DLSS. Phase 5 (`v0.3.0`) extended it to multiple opaque instances, emissive area lighting and full ReSTIR DI. Phase 5.1 (in progress) adds demand-driven [DLSS 4.5 Ray Reconstruction](dlss_integration.md#experimental-dlss-45-ray-reconstruction) guides (`rrDiffuseAlbedo`, `rrSpecularAlbedo`, `rrNormalRoughness`, `rrSpecularMotion`), native Vulkan [NVIDIA NRD](nrd_integration.md) guides (`viewZ`, 2.5D motion, `R10G10B10A2` normal/roughness at binding `31`, demodulated diffuse/specular and SIGMA shadow/penumbra), the two-room sandbox with directional sunlight, and output-resolution emitter presentation (`present_emitters.comp.spv`). Custom SVGF remains deferred in favor of NRD. Ordinary Native has no temporal color blending; only the explicit reference accumulates direct-light samples. Standalone TAA and multi-bounce GI are not implemented.

## Frame pipeline

The frame order is geometry visibility/depth, deferred linear HDR lighting and temporal guides, optional NVIDIA or spatial reconstruction, tone mapping/display conversion, then output-resolution ImGui. Native at 100% presents current HDR directly; reduced Native diagnostic resolutions use bilinear reconstruction without previous-color blending. DLAA/DLSS uses SDK temporal reconstruction.

Guide generation is demand-driven: ordinary Native color skips it, active NVIDIA modes need motion and linear depth, selected diagnostic views request their guides, and validity/capture inspection requests the full set. Only consumed guides are cleared; lighting and reconstruction overwrite their complete outputs. Empty scenes explicitly initialize HDR. Guide images remain allocated, and validity inspection waits for compatible previously generated guides. Hi-Z does not depend on these optional guides.

Two frame slots own independent visibility/depth targets, HDR lighting, motion, linear depth, normals, identity, validity, and reconstructed HDR. The previous submitted slot retains raw color and guides. A monotonic submission counter chooses slots independently of acquired swapchain images. Only successful graphics submissions advance camera/object snapshots and jitter; presentation-triggered recreation invalidates history afterward.

Camera matrices are prepared once per frame. Geometry and attribute reconstruction share the jittered projection. Motion and frustum/cone culling use unjittered projections; Hi-Z tests use the exact jittered camera paired with the sampled depth. World-space shading and shadow rays apply full object transforms, with inverse-transpose normal transforms. Culling freeze does not freeze the shading camera or temporal snapshots.

## Input contracts

| Resource | Format | Meaning |
| :--- | :--- | :--- |
| HDR lighting / reconstructed HDR | `R16G16B16A16_SFLOAT` | Linear scene lighting before tone mapping. Alpha 0 marks background, 1 marks lighting, 2 marks display-space diagnostic colors. |
| Motion | `R16G16_SFLOAT` | Previous minus current unjittered surface position, in internal-render pixels. Positive X right, positive Y down. Includes camera and rigid-object motion. |
| Hardware depth | Supported sampled depth format | Reverse Z, near 1 and far/background 0. |
| Linear depth | `R32_SFLOAT` | Positive eye-space distance; background 0. |
| Normal | `R16G16_SFLOAT` | Octahedrally encoded world-space smooth surface normal before normal mapping. |
| Identity | `R32G32B32A32_UINT` | ECS entity, model-local source surface, actual raster LOD from the indirect command, and global material index. Legacy assets synthesize coarse single-material surface ownership. |
| Validity | `R8_UINT` | 1 when the reprojected history surface is compatible; otherwise 0. |
| Diagnostic reference sums | `R32G32B32A32_SFLOAT` | Linear direct-light sum for the optional unjittered Native sandbox reference; separate from reconstruction history. |
| Initial reservoirs | 48-byte std430 buffer records | Selected light identity/sample, candidate count, weight sum/target and ray/error flags. Two internal-extent arrays at set 3 binding 14; current slot is overwritten fully, with no reuse yet. |

`FrameCameraData`, temporal `Snapshot`, and descriptor-backed `FrameGpu` carry current/previous camera and object transforms. `FrameGpu` uses std430 column-major matrices, occupies 704 bytes and remains outside the existing 256-byte draw push constants. The NVIDIA backend converts matrix layout, clip orientation and motion units for the SDK. SDK output is treated as RGB; its alpha is not an engine diagnostic marker.

Native jitter preview is an eight-sample Halton(2,3) sequence centered in render-pixel units, disabled by default. Active DLAA/DLSS automatically enables jitter with phase counts derived from actual SDK dimensions (approximately 8/18/24/32). Given current pixel center `p`, motion `m`, and current/previous jitter `jc`/`jp`, the previous image sample is `p + m + jp - jc`.

The internal target contains only scene content. The output rectangle handles aspect-ratio bars independently; UI always renders at swapchain resolution. Geometry LOD error is measured against output-content height. Jitter-aware frustum guards add one internal pixel on each side.

## History and synchronization

RGBA32 identity was introduced in Chunk 1, with later boulder GPU smoke recorded in [its notes](../archive/phase-5/phase5_chunk1.md). Chunk 2 adds stable entity-based previous-transform matching through the instance SSBO, retaining the 704-byte `FrameGpu`. Its fixed-window suite verifies exact multi-instance surface/material IDs and finite HDR through motion/scene/reconstruction cases under synchronization validation. Detailed per-pixel multi-instance motion/validity accuracy and extended ghosting review remain future coverage.

Scene/model/LOD/light keys conservatively invalidate temporal history when their contracts change. Rigid motion reprojects using the matching entity's current/previous model and normal matrices. The independent reference uses camera, object, light, LOD, dimensions and shading keys, advances only after submission, and resets instead of reprojecting during motion. Two RGBA32f images retain its sums; 16 samples per frame converge to the diagnostic 4096-spp cap. Scene switches retire frames and reset reference, temporal and depth history.

History is rejected for background, invalid projections, out-of-bounds reprojections, entity/surface/LOD/material mismatches, incompatible normals, or depth differences. Depth tolerance is `max(0.01 m, 1% of expected previous eye distance)` and the normal dot-product threshold is 0.9. Expected prior depth and normal account for the previous object transform.

Startup, camera cuts, object/model replacement, target recreation, projection changes, RT LOD changes, jitter mode changes, and lighting/debug mode changes reset history. Ordinary camera/object motion uses reprojection. Raster LOD changes reject the corresponding surface history through identity checks. Call `temporal::reset` for explicit scene or reconstruction-mode changes; camera inputs also accept a cut flag.

The frame-slot and acquired-image fences protect CPU uploads and descriptor/command-buffer reuse. Explicit queue barriers protect guide history, geometry targets, Hi-Z, and shared TLAS rebuilds. TLAS build input is copied from frame-owned staging into a device-local buffer; its build-input reads use the shader-read access class. Normal frames do not wait for device idle. Recreation and test-only readback may wait.

Hi-Z owns depth metadata independently of lighting history. Exactly unchanged views use one geometry pass; camera/object/jitter changes predict against the producing camera, then recover only deferred meshlets against current depth before shading. LOD/model/target incompatibility warms up. F2 retains a paired frozen pyramid and camera/jitter snapshot. See [Hi-Z repair](../file.md) for resource/barrier contracts, exact-reference verification and measured Debug costs.

## F4 performance and F5 diagnostics

F4 is a compact passive performance overlay: FPS/frame time, effective reconstruction mode, input/output dimensions, total GPU time, lighting including RT, reconstruction cost, and the frame-time graph. F3 remains the existing passive HUD. F4 and F5 toggle independently.

F5 opens a separate movable, resizable diagnostics window, initially at the upper right. Only the Reconstruction section starts expanded. Grouped sections retain mode/runtime/fallback information, temporal jitter/scale/guide/reset controls and counters, RT/sun controls, geometry/culling details, and detailed CPU/GPU timings. Unlock the cursor with Esc to interact. Native jitter preview remains unfiltered; active NVIDIA modes control jitter and dimensions automatically. Diagnostic controls remain session-only; reconstruction and presentation settings persist through Settings. Presentation offers Mailbox (default), Immediate (tearing allowed) and VSync (FIFO).

The Detailed timings section reports actual presentation mode, CPU frame wall time and acquire/submit/present calls, tone-map/display GPU cost, and a matched RT-on/off estimate. That estimate includes lighting-path differences and is not isolated ray traversal. Timestamp recording is enabled while either F4 or F5 is open. Reconstruction switches preserve the swapchain and reuse same-size targets; unrelated Settings Apply changes preserve reconstruction history. See the DLSS notes for verification limits and the interrupted display-test run.

## Verification

Configure the existing MinGW Debug build with focused tests, then build and run them:

```powershell
cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug -DVKE_BUILD_TESTS=ON
cmake --build build --target VK_game_engine temporal_math_tests hiz_history_tests -j8
ctest --test-dir build --output-on-failure
```

An opt-in GPU smoke run is compiled only with `VKE_BUILD_TESTS`. Run from the repository root, with process-local environment settings:

```powershell
$previousSmoke = $env:VKE_TEMPORAL_SMOKE
$previousSync = $env:VK_LAYER_VALIDATE_SYNC
try {
    $env:VKE_TEMPORAL_SMOKE = '1'
    $env:VK_LAYER_VALIDATE_SYNC = '1'
    .\build\VK_game_engine.exe *> build/temporal-smoke.log
    if ($LASTEXITCODE -ne 0) { throw 'Temporal GPU smoke failed' }
    if (Select-String -LiteralPath build/temporal-smoke.log -Pattern 'validation layer:|hazard|VUID-') {
        throw 'Vulkan validation reported a problem'
    }
} finally {
    $env:VKE_TEMPORAL_SMOKE = $previousSmoke
    $env:VK_LAYER_VALIDATE_SYNC = $previousSync
}
```

The finite smoke run checks real GPU buffers for finite values, near-zero static motion with jitter enabled, camera/object motion, camera-cut and LOD resets, history recovery, render-scale changes, resize/aspect changes, RT/frozen-culling changes, and HDR lighting above 1.0. It writes native/restored PPM captures into `build/`, leaves persisted settings unchanged, and exits. Test readback deliberately waits for the GPU; normal rendering does not use that path.

Debug CPU tests, extended synchronization-validation smoke and the original 216 exact Hi-Z-on/off depth/HDR frame comparisons passed on the available GeForce RTX 3090. Model K integration adds real reconstruction validation, automated minimize/restore checks and 144 exact source frame comparisons with Quality/K jitter; see its implementation notes for limitations. Development uses Debug only; the extra temporal Release build was removed. Extended interactive F4/F5 and image-quality review, memory-soak testing and remaining integration checks are carried into Phase 5. Closing Phase 4.1 does not mark those checks complete. DLSS 4.5 M/L integration has been removed; model K is fixed. Reflex Off/On/On + Boost works independently of reconstruction, with controls in Settings and details in F5; a current latency benefit remains unverified.

## Phase 5 consumers

Chunk 2 implements opaque instance/model/surface/material lookup, emissive sampling and the small reference scene. All nine CPU/offline tests and the fixed-window GPU suite pass. Rendered captures were inspected, but desktop capture/activation failures prevented direct menu verification. Quantitative lighting/reference comparisons, Cornell, Sponza, imported scene graphs and alpha agreement remain future work.

Chunk 3's initial-only path is superseded by [full ReSTIR DI](restir_di.md): independent temporal/surface/light snapshots, bidirectional identity mapping, temporal/spatial rejection, source-frame correction and final visibility. Physical previous poses now survive lighting/SDK resets; previous depth uses the owning instance. Two TLAS snapshots preserve actual previous geometry. Temporal history retains pre-spatial reservoirs, with explicit compute/transfer barriers and submission/fence alignment across resize. Eleven CPU suites and the long GPU/reference suite validate the new path. RR remains Phase 5.1 and SVGF stays deferred.
