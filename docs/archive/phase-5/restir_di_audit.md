# ReSTIR DI audit

Historical checkpoint: the [2026-11-01 follow-up](phase5_history_followup.md) supersedes the current-history/performance status below. Final-spatial history is restored after stronger checks, and sustained thermal clock limiting is now directly observed. The dated audit remains a record of the earlier implementation.

Date: 2026-10-30. Scope: the Phase 5 implementation entering this audit, NVIDIA's published RTXDI guidance, and the recorded RTX 3090 measurements. This is a review of raw direct-light sampling. No RR, SVGF, radiance filter, reservoir outlier filter or other denoiser is enabled by this audit.

The sampling stages exist, but the current renderer is not a finished, tuned ReSTIR DI integration. Reference-mean checks establish useful correctness evidence; they do not establish acceptable noise, temporal stability, recursive reuse or ultrawide performance. **The requested approximately 200 FPS with DLAA at 3440x1440 has not been achieved.**

## Frame budget

200 FPS allows **5 ms for the entire frame**. The reported earlier 280 FPS boulder result corresponds to 3.57 ms, but that scene's directional-light workload differs from the new area-light scene. In the recorded three-emitter interior view, initial lighting takes about 3.1 ms, temporal reuse 2.9 ms and DLAA 1.5 ms before spatial reuse or final shading. Spatial reuse adds roughly 9.5–12.6 ms. This requires substantial sampling/data-flow changes, not a claim that a single setting or CPU Release build will restore the old frame rate. Existing measurements use MinGW Debug; a matched optimized benchmark is still required.

## NVIDIA comparison

These are the current **RTXDI library defaults**, not universal recommendations or a sample preset guaranteed to suit this scene. NVIDIA describes regular spatial counts around 1–3 as typically sufficient in converged regions. [Default parameters](https://github.com/NVIDIA-RTX/RTXDI-Library/blob/main/Source/ReSTIRDI.cpp), [noise and bias guidance](https://github.com/NVIDIA-RTX/RTXDI/blob/main/Doc/NoiseAndBias.md).

| Setting/data flow | Audited starting implementation | NVIDIA library |
| --- | --- | --- |
| Initial local candidates | 8 | 8 |
| Temporal history limit | 20 | 20 |
| Temporal/spatial correction | Ray-traced source correction | Basic |
| Regular spatial work | Four accepted donors, up to 64 trials | One proposal |
| Disocclusion count / radius | 8 / 16 pixels | 8 / 32 pixels |
| Temporal permutation | Absent | Enabled |
| Stored reservoir | 64 bytes | Packed 24 bytes |
| Next-frame history | Pre-spatial temporal result | Final shading/spatial result |

Eight initial candidates and twenty-frame history are reasonable starting values. More important differences are correction cost, retained history and storage. Basic correction assumes visibility rather than tracing every source; Pairwise is another supported alternative. These have accuracy tradeoffs and require explicit reference comparisons. Removing normalization is not an equivalent optimization. [Correction definitions and packing](https://github.com/NVIDIA-RTX/RTXDI-Library/blob/main/Include/Rtxdi/RtxdiParameters.h), [parameter requirements](https://github.com/NVIDIA-RTX/RTXDI-Library/blob/main/Include/Rtxdi/DI/ReSTIRDIParameters.h).

## Concrete costs and history limits

- The [spatial pass](../../../shaders/restir_spatial.comp) repeatedly gathers surface records and evaluates full material targets, then traces correction segments. Shadows-off still costs about 9.8 ms in the three-emitter interior case: removing rays alone will not solve its entire cost. Exact entity/surface/LOD/material equality also restricts sharing more than NVIDIA's geometric/material compatibility tests.
- Full DI allocates about 1.90 GB at ultrawide. The [legacy initial-only arrays](../../../src/renderer/temporal.cpp) add another 475.5 MB even when unused. The [deferred pass](../../../shaders/deferred.comp) clears legacy reservoirs on every pixel and clears full-DI records before overwriting visible records. The [temporal history copy](../../../src/renderer/restir_full.cpp) moves 317 MB each frame, approximately 634 MB of read/write traffic. Packing and removing unused work are warranted; three reservoir arrays alone are not a bug, because the current NVIDIA context also uses three. [NVIDIA array count](https://github.com/NVIDIA-RTX/RTXDI-Library/blob/main/Include/Rtxdi/DI/ReSTIRDI.h).
- Our pre-spatial history was a workaround for a failing recursive-feedback test. It is **not a general ReSTIR safety rule**. It prevents repeated spatial propagation across frames, losing a key convergence mechanism. NVIDIA normally advances final spatial/shading reservoirs into history. Correct recursive normalization and correlation handling need validation before restoring that flow; merely reconnecting the buffers is insufficient.

## Verification corrections and next work

The final 64-emitter timing log has unfocused cases: stage 1 records 118/180 focused samples and stage 2 records 71/180. Exclude these from controlled paired comparisons and rerun them. Its stages 0 and 7 are fully focused, as are the three-emitter cases. Recorded passes and image comparisons remain useful, but are not a performance qualification.

Next work should fix demonstrated reprojection drift, validate recursive reuse, expose and compare correction policies, measure bounded spatial work, eliminate legacy allocations/clears and the history copy, then pack persistent data. Keep accurate diagnostic/reference modes and measure raw mean, variance, motion response, resource use and the complete 5 ms budget. Visibility shortcuts require prior invisible samples to be discarded; that precondition must be implemented and tested before enabling a shortcut. No filtering is an implicit substitute for these fixes.

## Changes and verification in this audit

**No GI path was found.** The sandbox replaces the legacy sun/ambient result with direct area-light response plus visible emission. Authored warm, blue and red emitters illuminate the floor/wall directly. No secondary surface radiance or bounced-light color is evaluated. ReSTIR DI samples direct illumination, including its diffuse/specular response and shadows; it is not only a shadow mask.

**The upward-moving lighting pattern has a demonstrated cause.** Deterministic nearest history sampling under our eight-phase Halton jitter chooses vertical donor offsets `+1,0,+1,0,0,+1,0,0`: retained patterns move upward three pixels each cycle despite stationary geometry. The temporal shader now uses independent stochastic rounding and accepts a compatible sampled center before any fallback search. This fixes directional quantization bias; it does not denoise the image or guarantee that all boiling/smearing is gone. NVIDIA also randomizes temporal reprojection when permutation sampling is disabled. [NVIDIA temporal resampling](https://github.com/NVIDIA-RTX/RTXDI-Library/blob/main/Include/Rtxdi/DI/TemporalResampling.hlsli).

The new CPU regression reproduces the old drift and integrates 65,536 random strata per phase to verify unbiased rounding. A finite **640x360 correctness probe** measures actual pre-spatial GPU donor displacement over 64 stationary Native frames and 64 real DLAA frames, after separate warmups. Native remains zero; the legacy negative control gives exactly **3 rows/cycle**; the fix gives **0.000529 rows/cycle**. Geometry motion, finite weights and active SDK evaluation pass, without Vulkan validation messages. Logs: `build/restir-jitter-{legacy,stochastic}.log`.

The attempted 3440x1440 variant was a poor interactive test setup: full-buffer readbacks and device waits every measured frame made its displayed frame rate unusable. It aborted on the stationary-motion assertion and is **not a passing full-resolution verification or a performance result**. No engine process remained afterward. This readback work is absent from ordinary rendering. The expensive variant is now explicitly opt-in through `VKE_RESTIR_JITTER_FULL_RES=1`; it must not be used as an interactive FPS benchmark. No further visible tests were launched in this pass.

All eleven CPU/offline suites pass again, including an added recursive six-pixel fixture with 2,048 independent 160-frame trajectories, changing support/visibility, unequal counts and history caps. Production temporal-only history has per-pixel reference assertions. Spatial feedback remains an exploratory comparison: this fixture found no statistically significant counterexample, but it does not prove that reconnecting the GPU feedback path is safe. Four affected shaders validate offline. Existing assets remain unchanged; no denoiser, boiling/outlier filter, RR or SVGF was added. The updated engine is built, but **the 200 FPS target and full-resolution temporal-quality validation remain open**.
