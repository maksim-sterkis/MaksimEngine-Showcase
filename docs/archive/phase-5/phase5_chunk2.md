# Phase 5 Chunk 2: Multi-Instance Lighting Sandbox

This records Chunk 2's implementation and verification. [Chunk 3](phase5_chunk3.md) adds initial reservoirs, grows the scene header to 64 bytes, corrects remaining per-instance transform uses and supplies quantitative mean/noise/cost comparisons. Its current results supersede the corresponding pending items below.

Implemented on `phase-5`, initially with CPU/build-only verification on 2026-10-29. The user subsequently authorized engine launches: the fixed-window GPU suite now passes under synchronization validation on RTX 3090 / driver 621.01, and rendered captures were inspected. Desktop UI inspection remains limited by capture/activation failures described below. No performance benchmark was performed. Phase 5 remains in progress; ReSTIR DI is next, RR remains Phase 5.1 and SVGF remains deferred.

## Scene and controls

The boulder remains the startup scene. A session-only Esc-menu selector switches between Boulder and Lighting Sandbox after retiring frame work. Both scenes are preloaded, with stable ECS entity IDs and camera presets. Switching scenes resets temporal, Hi-Z and reference history.

The engine-authored sandbox contains a room, two instances of a multi-material cube, rough metal and either three or 64 rectangular textured emitters (six or 128 triangle lights). It is a direct-lighting test scene, not the measured Cornell box. There is no indirect illumination, artificial ambient fill, denoising or reservoir reuse. Boulder retains its directional sun and ambient behavior.

F5's Lighting Sandbox section controls the emitter preset, 1/4/16/64 samples per pixel (default four), emission intensity, area-light shadows, object/light motion, selected moving light, identity/emission views and reference accumulation. Existing LOD, culling, reconstruction and RT controls remain available. Emissive models stay at LOD 0 so light topology is stable. Assets are opaque; mirrored/singular transforms, alpha handling and general imported node hierarchies are outside this milestone.

## Interfaces and history

Visibility stays `R32G32_UINT`: word zero is model-local meshlet index, word one packs a dense instance index above seven triangle-index bits. Triangle index 127 is reserved by the cleared background sentinel; meshlets retain the 124-triangle limit. Stable history identity remains `RGBA32_UINT` with `(entity, surface, raster LOD, global material)`. Dense instance indices may change; ECS entity identity governs previous-transform matching.

The global set adds binding 3 for an immutable model table. Each 64-byte record stores five `uvec2` buffer addresses for meshlets, meshlet vertices, meshlet triangles, attributes and ownership, plus counts/flags. Shaders use `GL_EXT_buffer_reference_uvec2`. Model set 0 binding 5 retains the ownership buffer created in Chunk 1; deferred shading now reaches it through the shared table. Draw push constants remain 256 bytes and temporal `FrameGpu` remains 704 bytes.

Frame set 2 bindings 9/10/11 hold the scene header/instance table, local light sources and prepared world-space lights. The header is 48 bytes; instances are 352 bytes; light records are 112 bytes, with compile-time size assertions. Each instance supplies current/previous model and normal matrices, a producing-camera occlusion matrix, model/LOD identity and its recovery-list range. Raster and RT select one LOD per instance unless the existing explicit RT LOD-0 override is active. Offscreen objects remain in the TLAS.

Acquired-image fences protect mapped scene uploads and descriptor reuse. Frame slots retain temporal/reference images independently of swapchain images. Light sources and transformed lights are cached per acquired-image resource and refreshed when their key changes; a compute barrier makes preparation visible to lighting. Capacity is reserved before rendering and can grow through the explicit retired-resource path. Capacity errors fail explicitly rather than silently truncating objects or lights. Normal frame rendering adds no CPU image readback.

Previous transforms commit only after successful submission. Scene/model/LOD/light changes conservatively invalidate relevant temporal history; ordinary object motion uses the matching entity's previous transform. Hi-Z keeps independent depth history, uses per-instance recovery regions against the shared pyramid, and invalidates frozen data when objects move. Scene switches deliberately wait for frame fences before changing the active list.

## Materials, sampling and reference

The 64-byte material record adds an emissive texture index and linear-HDR emission factor, including imported emissive strength. The compiler separates color, linear-data and normal texture uses even when one source image serves several roles. Color/emission use sRGB encoding, data remains linear, and normals retain the normal-map compression convention. Visible emission and triangle-light evaluation use the same texture/material values. Legacy boulder UV correction is flagged per model; authored v2 sandbox UVs are used directly.

`surface_access.glsl`, `material_eval.glsl` and `light_sampling.glsl` share triangle access, PBR evaluation, reconstructable light samples, emission and visibility helpers. Sampling chooses a triangle uniformly and a point uniformly over its world-space area, with area density `1 / (triangleCount * area)`. The estimator uses receiver BRDF and cosine, emitter cosine, squared distance and the explicit density. Emission is one-sided. Finite shadow segments exclude their endpoints using geometric-normal bias. No contribution clamp, power distribution or light/BRDF MIS is implemented.

The diagnostic reference accumulates linear sums in two `RGBA32_SFLOAT` images, with 16 new samples per submitted frame up to 4096 spp. HDR output is their running mean. It uses the same direct-light estimator as the low-sample path, so it is a sampling reference rather than an independent GPU implementation. An independent CPU quadrature test checks the basic area estimator. Reference operation requires Native, no jitter and the plain lighting view. Camera, object, light, LOD, dimensions, scene or relevant shading changes restart it; failed/skipped submissions do not advance its sample count. Ordinary Native rendering still has no temporal color blending.

## Assets and verification

`tools/sandbox_assets.cpp` generates deterministic private glTF/bin inputs and the emission bitmap under `assets/models/sandbox/`; the existing model compiler produces six self-contained MLOD v2 sandbox GLBs. The `sandbox_assets` build target integrates generation/compilation. All pre-existing boulder assets remain unchanged. Malformed/role-conflict test fixtures are generated under ignored `build/`.

Exact development configuration: `MinGW Makefiles`, `CMAKE_BUILD_TYPE=Debug`, `C:/mingw64/bin/g++.exe`, `VKE_BUILD_TESTS=ON`, `VKE_ENABLE_DLSS=ON`, pinned Streamline 2.14.16 from `.cache/streamline/2.14.16/sdk`. Engine and model compiler build successfully; the final engine build log is `build/phase5-chunk2-final-build.log`.

All nine CTest cases passed: `temporal_math`, `hiz_history`, `reconstruction`, `meshlet_surfaces`, `scene_lighting`, `sandbox_assets`, `sandbox_texture_roles`, `legacy_boulder_payload` and `compiler_surface_fixtures`. Coverage includes visibility packing, capacities, transform/history matching, reference reset/sample accounting, area sampling versus independent rectangular quadrature, generated opaque emissive assets, shared-image texture-role splitting, and retained Chunk 1 payload/ownership coverage. `build/Testing/Temporary/LastTest.log` records the results.

Affected shaders compile, and offline `spirv-val --target-env vulkan1.3 --scalar-block-layout` passes for deferred lighting, light preparation, task/mesh/fragment visibility, occlusion setup, object culling, presentation and spatial reconstruction. `git diff --check` passes. These checks establish CPU/build and SPIR-V validity; they do not establish correct Vulkan synchronization, rendered appearance, convergence or performance.

`tests/sandbox_gpu_smoke.hpp` is compiled only with focused tests and is not registered with CTest. It requires `VKE_SANDBOX_GPU_SMOKE=1` and `VKE_FIXED_PRESENT_PROBE=1`; use `VK_LAYER_VALIDATE_SYNC=1` for correctness testing. The suite opens one fixed 1280x720 window. After explicit launch authorization, its 448-frame scripted run passed boulder fast/full-guide HDR parity, sandbox instance/surface/material readbacks, same-size Native/DLAA target reuse, exact Hi-Z-on/off HDR parity, 64 emitters/128 triangles, object/light motion, real DLAA and DLSS Quality evaluation, scene switching and reference completion/reset.

The first run exposed missing `VK_IMAGE_USAGE_TRANSFER_SRC_BIT` on visibility images used by the test readback. That flag was added; the repeated suite exits successfully with no `validation layer:`, `VUID-` or `SYNC-HAZARD` messages. The log is `build/phase5-chunk2-gpu-smoke-clean.log`; build log is `build/phase5-chunk2-gpu-fix-build.log`. The three previously recorded Streamline unsupported-hook warnings remain distinct from Vulkan validation errors. All nine CPU/offline tests pass again after the fix.

GPU readbacks produce `build/sandbox-{boulder,few,many,reference}.ppm` (PNG copies were made for inspection). The captures show intact boulder geometry/materials, colored direct area illumination, occluders/shadows and a much smoother completed reference. Four-spp output remains visibly noisy by design. The reference case uses reduced Native internal resolution; its upscaled output has coarse silhouettes and is not a full-resolution image-quality result. Capture dimensions now follow the selected HDR source correctly.

Separate interactive launches were attempted. The desktop tool listed the engine but captured wallpaper and failed with `failed to activate captured window`, including after refreshing the window binding. Those launches were stopped. Therefore the GPU suite and rendered captures establish rendering/readback behavior; successful direct menu interaction, end-to-end desktop presentation quality and prolonged visual inspection are not claimed.

Remaining acceptance includes quantitative low-sample/reference mean comparisons, matched lighting/preparation/visibility costs, direct menu interaction and extended motion/image-quality review. Memory soak, other hardware/drivers and full benchmark scenes remain later validation. No commits, pushes, documentation synchronization or PowerShell scripts were performed.
