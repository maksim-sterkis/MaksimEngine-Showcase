# Hi-Z repair and Debug build cleanup

Implemented and verified October 25, 2026 on the available GeForce RTX 3090. This work restores moving-view occlusion culling in the single-model boulder renderer. Later model K DLAA/DLSS and Native guide-generation cleanup are documented in [DLAA/DLSS integration](../file.md#verification-and-limits). Phase 4.1 is now closed under the revised [roadmap](../roadmap.md); unfinished validation and general scene support moved to Phase 5, alongside emissive area lighting, ReSTIR DI and SVGF.

## Depth ownership and frame order

`occlusion::DepthHistory` records the producing submission, jittered view-projection matrix, render extent, model transform, entity/model identity and raster LOD. Previous and frozen metadata advance only after graphics submission succeeds. Lighting/RT/debug history resets do not discard compatible depth. Resource recreation and model/LOD incompatibility require a warm-up frame.

The normal paths are:

1. **Hi-Z off or warm-up:** current frustum/cone tests and one visibility/depth geometry pass. Warm-up produces a pyramid for subsequent frames; ordinary Hi-Z-off rendering skips pyramid construction.
2. **Exactly unchanged camera, jitter, transform, LOD, identity and extent:** one geometry pass tests the matching previous submitted pyramid. No reject-list preparation, recovery setup or recovery draw executes.
3. **Compatible depth with changed camera/jitter/transform:** current frustum/cone tests run first. Prediction projects each current meshlet bound through the camera/jitter that actually produced the previous pyramid. Predicted-visible meshlets render immediately; only predicted-occluded meshlets enter the recovery list. Build current depth, recheck that compact list and render newly visible meshlets using attachment LOAD operations. Shade the completed visibility buffer once. Refresh the final pyramid only when the GPU recovery count is nonzero.

Prediction is a scheduling hint; its occlusion result alone never removes moving-view geometry. Each meshlet rasterizes at most once. This is the reject-list variant of [NVIDIA's two-pass culling approach](https://github.com/nvpro-samples/vk_lod_clusters/blob/main/docs/raster_twopass_culling.md), without a full-scene depth prepass.

Whole-object compute culling performs frustum rejection and writes the selected LOD into the indirect command. CPU scene preparation selects that LOD once for rasterization, temporal identity and RT selection. Recovery uses the same command and saved meshlet IDs; whole-object Hi-Z cannot hide candidates before recovery.

## Resources, conservative tests and diagnostics

Two submission slots own recovery storage and mapped depth constants. Physical list capacity covers all meshlets across the model's six LODs. If an entry cannot be recorded, the task shader renders it immediately. Prediction and recovery use distinct descriptor sets, updated after the acquired-image fence and before command recording binds either set.

The 256-byte draw push-constant block replaces unused `hasTexture` with `geometryPhase`. Work-buffer offsets are 16 bytes for the mesh-task indirect command, 32 bytes for sixteen padded compute indirect commands, and 288 bytes for the reject list. A small compute setup generates recovery and final-pyramid dispatch arguments. Empty work uses zero indirect group counts. Normal rendering adds no CPU readback or device-idle wait.

Projected bounds enclose all eight corners of the meshlet's local bounding-sphere box. Near-plane crossings, incomplete viewport footprints and degenerate bounds fail open. The nearest bound depth is compared against the minimum of **every** overlapped pyramid texel in reverse Z. Reduction includes every intersected source texel, overlapping boundary texels when dimensions are odd rather than losing the last row/column. Explicit barriers cover depth writes, pyramid reads/writes, reject-list writes, indirect arguments, resumed attachment loads and final shading.

Inactive task lanes neither read bounds/list entries nor emit meshlets or update counters. Evaluated, frustum-rejected, cone-rejected, finally Hi-Z-rejected and rasterized counts partition the candidates exactly. Predicted/deferred and recovered counters are separate; the second pass never increments evaluated counts again. F5 Geometry and culling displays the current Hi-Z mode and deferred/recovered counts; F4 is the compact performance overlay.

`9` remains the Hi-Z toggle. F2 deliberately freezes frustum/cone/occlusion diagnostics while the render camera continues moving. Its captured pyramid retains the matching jittered camera and geometry metadata. Resize, model/LOD or object-transform incompatibility causes a new current-view capture; unfreezing resumes live culling.

## Verification results

- Debug focused CPU tests: `temporal_math` and `hiz_history` passed. History contracts include exact subpixel changes, transform movement, lighting independence, LOD/model changes, resize and invalid depth.
- Hi-Z-on/off reference runs: **216 frame pairs** matched hardware D32 depth and R16G16B16A16 HDR **byte-for-byte**, establishing identical visible coverage and shaded output for the sampled frames.
- 72 pairs used a **3440×1440 borderless output** throughout, all six LODs, 100%/75%/50% scales, stationary views, movement, abrupt turns/disocclusion and transformed objects. The remaining 144 pairs included odd 1001×777 output and 751×583 / 501×389 render targets, toggling and camera cuts.
- GPU pyramid readbacks matched CPU-calculated minimum reductions at every mip for 3440×1440, 1001×777, 751×583 and 501×389 inputs.
- Odd-target run sampled 805 final Hi-Z rejections, 559 deferred candidates, 282 recoveries and 217 overflow fallbacks. Native-output run sampled 440 rejections, 141 deferred candidates, 40 recoveries and 252 overflow fallbacks. Overflow was deliberately injected with a logical capacity of two; actual allocated capacity remained 1634. Coverage/HDR still matched exactly.
- Sampled stationary paths had zero deferred/recovered candidates; code dispatches the recovery setup/draw only for the prediction phase. Native motion benchmark observed up to 84 deferred and 28 recovered meshlets in a frame; jitter observed up to 76 deferred and two recovered.
- Extended temporal GPU smoke passed Hi-Z toggling, F2 freeze/unfreeze, captured jitter/camera pairing and recapture after LOD, resize and transform changes. Lighting-history reset preserved usable depth.
- Khronos synchronization validation reported no messages in the reference runs, extended smoke or a separate 1608-frame benchmark with two frames in flight and no test-image readbacks. Image-reference/pyramid inspection waits are confined to opt-in tests.

These checks cover the boulder asset and available RTX 3090. They do not establish correctness for future multi-model scenes or other GPU drivers. Later DLAA/DLSS verification added automated minimize/restore coverage and 144 exact Quality/K source depth/HDR comparisons; see [its verification notes](../file.md#verification-and-limits). Extended interactive F4/F5 review and general scene validation remain Phase 5 work.

## Matched Debug measurements

Same RTX 3090, 3440×1440 borderless output, LOD 0, 100% scale, default lighting/RT settings and camera paths. Validation layers enabled; synchronization validation disabled for **both** timing runs. Each case warms up for 100 frames, then collects 300 samples. Jitter is enabled only in the jitter case. Numbers are medians in milliseconds.

| View | GPU frame before → after | Geometry before → after | Hi-Z before → after | Wall frame before → after | Final Hi-Z rejects before → after |
| :--- | ---: | ---: | ---: | ---: | ---: |
| Rock, stationary | 0.7142 → 0.7301 | 0.0605 → 0.0636 | 0.0379 → 0.0502 | 1.3299 → 1.3661 | 77 → 76 |
| Looking away | 0.5099 → 0.5270 | 0.0324 → 0.0335 | 0.0358 → 0.0492 | 1.0265 → 1.0892 | 0 → 0 |
| Continuous turns | 0.7117 → 0.8013 | 0.0596 → 0.0812 | 0.0379 → 0.1044 | 1.3173 → 1.5474 | 0 → 39 |
| Stationary + jitter | 0.7154 → 0.7559 | 0.0595 → 0.0774 | 0.0379 → 0.0584 | 1.2975 → 1.5127 | 0 → 76 |

The repair restores correct culling during movement/jitter, but **does not improve frame time on this small scene**. Motion costs approximately 0.090 ms more GPU time (12.6%); stationary costs approximately 0.016 ms more (2.2%). Complete-footprint/odd-dimension reduction is more expensive, and moving views add setup, current-depth construction and selective recovery. Current Hi-Z-off GPU medians were 0.6766 / 0.4764 / 0.6748 / 0.6775 ms respectively: this asset is too small for the saved meshlets to repay the pyramid/recovery cost.

GPU frame timestamps span recorded rendering work, excluding presentation. Geometry includes both geometry passes and recovery setup; Hi-Z includes pyramid scopes and final indirect setup. Frame and stage medians are collected independently and are not additive. Wall times include CPU/driver/validation/presentation effects and are more variable. A per-image query pool is read only after its owning fence, without `VK_QUERY_RESULT_WAIT_BIT`; timestamps are opt-in and absent from ordinary development rendering.

Raw results: `build/hiz-bench-before.csv`, `build/hiz-bench-after.csv`, `build/hiz-bench-off.csv`. Synchronization-validation timings in `build/hiz-bench-validate.csv` are a separate run and excluded from the comparison. This Debug comparison makes no claim about the Phase 4 v0.2.0 Release distribution's FPS.

## Reproduction and build housekeeping

`build/` remains Debug with `VKE_BUILD_TESTS=ON`. Build `VK_game_engine`, `temporal_math_tests` and `hiz_history_tests`, then run `ctest --test-dir build --output-on-failure`.

With test support enabled, launch from the repository root using process-local environment variables:

| Variable | Values / behavior |
| :--- | :--- |
| `VKE_TEMPORAL_SMOKE` | `1`: finite extended temporal/F2/toggle smoke; writes a log and native/restored PPM captures when redirected. |
| `VKE_HIZ_VERIFY` | Run `off` then `on` for odd-target reference comparison; run `native-off` then `native-on` for native borderless output. Reference binaries and logs belong under ignored `build/`. A mismatch writes `.actual` and fails. |
| `VKE_HIZ_BENCH` | `after`, `off`, or `validate`: finite timestamp benchmark writing the matching CSV. `before` is reserved for the captured pre-repair baseline; rerunning it on repaired code would overwrite that baseline. |
| `VK_LAYER_VALIDATE_SYNC` | `1` for correctness/validation runs; `0` for matched timing runs. Restore each process environment afterward. |

The only removed build was `C:\Users\maksi\VK_game_engine\build-temporal-release`. Before native PowerShell literal-path recursive deletion, its resolved path was checked against that exact ordinary directory inside the workspace, with no reparse point and no tracked files. Pre-existing `build-release/`, `dist/MaksimEngine/` and `dist/MaksimEngine-v0.2.0-Portable.zip` were preserved. No new Release was configured, built or packaged; no commits, pushes, PowerShell scripts or documentation synchronization were performed.
