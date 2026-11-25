# Phase 5: rough-surface importance-target experiment

November 1, 2026. **Decision: not adopted; the proxy and its runtime controls were removed from production.** The original importance target is retained. The independently seeded/material-level audit improvements, arbitrary-target CPU tests, history-policy reset regression and opt-in Release validation remain. This is not a release or Phase 5 completion claim.

## Candidate and estimator contract

The experiment replaces repeated full-BRDF importance evaluation on sufficiently rough, mostly dielectric receivers with a cheaper positive target. It does not replace final shading. Eligibility is roughness >= 0.75 and metallic <= 0.25; other receivers retain the original full-contribution luminance target. The proxy is the luminance of emission multiplied by `albedo * (1-metallic) / pi + 0.25 * F0`, with the same receiver/emitter cosine, area/distance, hemisphere, emissive texture and current/previous emission-scale factors as the original target.

The dielectric F0 component preserves positive support for black diffuse receivers in the eligible material range. Initial, temporal and spatial source/destination evaluations use the same policy. The policy is part of the history key: changing it discards incompatible reservoir weights. Final BRDF evaluation, fresh final shadows, candidate counts, correction mode, light sampling PDF, storage layouts and geometry remain unchanged. This introduces no GI, denoiser, radiance clamp or visibility reuse.

An initial roughness >= 0.5 version increased raw RGB MSE on the two rough colored cube materials by approximately 5.7% and 4.3%, despite less than 0.2% whole-image change. Narrowing eligibility to 0.75 makes those materials use the original target again. Metal receivers also retain the original target.

## Correctness and quality measurements

All eleven Release CPU/offline suites pass. Recursive CPU fixtures additionally use unrelated positive target weights, including positive targets for zero-integrand samples, while retaining physical reference values and the existing gates. Both Basic and ray-traced correction run static and changing-support cases.

Independent GPU audits use a 4096-sample reference, sixteen independently seeded trajectories per policy, 64 warm-up frames and 256 measured frames per trajectory. Full-target and proxy policies have paired random sequences. Whole-image, early/late, 4x4 tile and material mean gates use six independent-trajectory standard errors plus 1% of reference and 0.0001; tolerances were not widened. These tests use hidden windows and raw HDR readback, never performance measurements.

| Scene | Metric | Full target | Proxy, roughness >= 0.75 |
| --- | --- | --- | --- |
| 64 emitters | Raw RGB MSE | 0.00242210 | 0.00242515 |
| 64 emitters | 256-frame-average RGB MSE | 0.0000404855 | 0.0000404630 |
| 64 emitters | Raw luminance MSE | 0.000551815 | 0.000554321 |
| 3 emitters | Raw RGB MSE | 0.00107307 | 0.00107237 |
| 3 emitters | 256-frame-average RGB MSE | 0.0000183481 | 0.0000181682 |
| 3 emitters | Raw luminance MSE | 0.000254538 | 0.000257166 |

All reference gates pass. Many-light raw RGB error changes approximately +0.13%; few-light raw RGB error changes approximately -0.07%. The four fallback receiver materials have identical measured errors under the paired trajectories. Offline averages describe sampling behavior; they are not an engine denoiser. These fixtures do not establish behavior for arbitrary materials or scenes.

A third independent audit uses the same heavy interior camera as the timing tests, with 64 emitters. All whole/time/tile/material mean gates pass with synchronization validation and no validation messages. Raw RGB MSE is 0.00188000 full versus 0.00188061 proxy (+0.032%); 256-frame-average RGB MSE is 0.0000357533 versus 0.0000357073 (-0.13%). Raw luminance MSE is 0.000460951 versus 0.000462714 (+0.38%). Colored/metal fallback material errors again match exactly. Its trajectory CSV uses NaN for regional error quantities that were not measured, rather than repeating whole-image values.

The same optimized Release binary also passes forced-validation Basic and ray-traced reference/motion/mode/topology/area/LOD/resize suites, including target-policy history invalidation in both directions. Synchronization validation was enabled. There are no reported VUID errors or synchronization hazards. The legacy `VK_LAYER_ENABLES` setting produced a deprecation warning; subsequent checks use `VK_LAYER_VALIDATE_SYNC=1`.

## Timing

Two balanced eight-case sweeps hold 68 instances, 64 emitters/128 triangle lights and the heavy interior camera fixed. Output and internal resolution are both 3440x1440 DLAA. Each accepted case has 600 focused frames and 600 asynchronous timestamp samples, after 1800 initial warm-up frames and 180 between cases. Basic correction, eight candidates, one spatial proposal, recursive final-spatial history and fresh shadows are held fixed. Validation, atomic diagnostics and per-frame readbacks are disabled. No CPU suites run concurrently.

| Sweep | Full GPU ms | Proxy GPU ms | Full initial/temporal ms | Proxy initial/temporal ms |
| --- | --- | --- | --- | --- |
| First | 6.77770 | 6.63265 | 2.97600 | 2.96858 |
| Repeat | 7.23390 | 7.01857 | 3.26144 | 3.14214 |

These mean case medians suggest 2.1% and 3.0% less total GPU time relative to the experimental shader's full-target branch. The initial/temporal gain is inconsistent: approximately 0.25% and 3.66%. DLAA time also shifts, so the whole difference cannot be attributed to target math. Driver telemetry reports active software thermal clock limiting: one sample was 1590 MHz at 76.59 C and 315 W, a later sample 1425 MHz at 81.66 C and 262 W. Software power capping was inactive. No hardware settings were changed.

## Artifacts

Ignored build logs: `restir-target-075-{many,few}-validation.log`, matching CSV files, `restir-target-075-{basic,reference}-validation.log`, `restir-target-075-cpu.log` and `restir-proxy-075-ab-{first,repeat}.log`. Initial wider-policy logs use `restir-target-audit-{many,few}` and `restir-proxy-ab-{first,repeat}`. Existing early CSV trajectory rows repeat whole-image error fields for regional rows; only whole-image trajectory errors and explicitly calculated material RGB errors are meaningful there. Aggregate regional luminance fields are unavailable. The harness now writes NaN for unmeasured regional errors.

The original fused initial/temporal shader is independently regenerated under ignored `build/restir-target-original/`; its SHA256 is `1FCDF0F5030EBD91BD41E863948C6BFFFE71FE0522DDBBDEC8D30CE87B2F1C4A`, matching the pre-experiment module. Final adoption must consider this original module, not just the full-target branch of a larger experimental shader.

## Original-module comparison and decision

Thirty-two additional full-resolution cases compare all six original shader modules with the proxy modules. Each executable runs eight 600-frame cases under the same timing requirements above. The launch order is original, proxy, proxy, original. The original runs keep the same current host code and recursive history, isolating shader modules rather than comparing different engine versions.

| Module run | Total GPU ms | Initial/temporal ms | Spatial/final ms | Unchanged DLAA ms | Loop FPS |
| --- | --- | --- | --- | --- | --- |
| Original before | 6.58427 | 2.70323 | 1.58310 | 1.68383 | 140.47 |
| Proxy first | 6.94379 | 2.92595 | 1.46150 | 1.88830 | 133.22 |
| Proxy repeat | 7.56205 | 3.21319 | 1.53434 | 2.08771 | 122.91 |
| Original after | 8.14486 | 3.52499 | 1.60039 | 2.24513 | 114.73 |

The mean of case medians is 7.36456 ms for original modules and 7.25292 ms for proxy modules, only approximately 1.5% favorable to the proxy. This is not a statistically qualified speedup: the unchanged DLAA pass drifts approximately 33% from first to last run, and launch-order balancing cannot remove nonlinear clock drift. End-of-run telemetry continues to report software thermal limiting; those samples do not establish in-frame clock equivalence. No CPU suites or builds ran during these measurements.

Accordingly the prototype is removed rather than retaining extra shader paths and a changed importance distribution for an unproven improvement. This rejects this implementation, not the general use of positive proxy targets. It does not establish a register-pressure or cache diagnosis; hardware counters were not available. Future work should obtain a reliable pass-level improvement against the original modules, with more tightly interleaved comparisons or controlled sustained clock conditions, before changing ordinary sampling.

The six candidate modules remain only in ignored `build/restir-target-proxy/`; the original modules are in `build/restir-target-original/`. Development source is archived under `build/restir-target-experiment-source/` as evidence, not a standalone build or patch to apply wholesale. Module timing logs use `restir-target-modules-{original-before,proxy-first,proxy-repeat,original-after}.log`; end-of-run telemetry is `restir-target-modules-thermal.csv`. The heavy-view quality files are `restir-target-075-interior.csv` and `restir-target-075-interior-validation.log`.

## Final production verification

The cleaned configuration is `MinGW Makefiles`, `CMAKE_BUILD_TYPE=Release`, `VKE_BUILD_TESTS=ON`, `VKE_ENABLE_DLSS=ON`, pinned Streamline 2.14.16 and Vulkan SDK 1.4.350.0 glslc targeting Vulkan 1.3. `cmake --build build -j8` and `ctest --test-dir build --output-on-failure` pass all eleven CPU/offline suites. Final build/CPU logs use `restir-target-clean-final-{build,cpu}.log`.

All six affected production modules match the regenerated original modules byte-for-byte and pass `spirv-val --target-env vulkan1.3`. Separate hidden Basic, ray-traced and Native/DLAA jitter runs pass with `VKE_FORCE_VALIDATION=1` and `VK_LAYER_VALIDATE_SYNC=1`, including switching the spatial-history policy and restoring it. Their logs use `restir-target-clean-final-{basic,reference,jitter}.log`; no validation messages, VUID errors or synchronization hazards are reported. The final jitter cycle remains approximately 0.000529 rows per eight-frame DLAA cycle at 640x360, matching the earlier checkpoint; this is a motion-coordinate check, not proof of noise-free lighting.

After removing the experiment, the retained three-policy heavy-interior audit (temporal-only history, recursive spatial history and conventional MC1) also passes all independent-trajectory, early/late, tile and material mean gates with synchronization validation and no validation messages. Its final log and trajectory CSV are `restir-target-clean-final-feedback.{log,csv}`. Regional error fields that are not actually measured are NaN. CPU tests, hidden readback tests and full-resolution performance probes remain distinct workflows.

The ordinary renderer retains recursive final-spatial history and the original full-contribution luminance target. No GI, denoising, clamp, visibility reuse, asset edits, commits, pushes or documentation synchronization is included. Phase 5's sustained heavy-view performance and equal-cost quality work remain open.
