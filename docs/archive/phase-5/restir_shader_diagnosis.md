# Initial/temporal shader diagnosis — 2026-10-31

The subsequent [rough-surface target experiment](phase5_target_proxy.md) records independent material/heavy-view quality checks and 32 original-module comparison cases. Its small apparent gain remained inconclusive under thermal clock drift; the original full-BRDF target was restored.

The [November 1 follow-up](phase5_history_followup.md) records the subsequent history tests, two cache experiments and thermal-limit evidence. The cache candidates did not yield a reliable speedup and were removed; the diagnosis below remains a historical checkpoint.

This follows the [controlled light-count sweep](restir_light_count_sweep.md). It adds diagnostic evidence, not a production performance change. The installed tools include glslc/SPIR-V validation, but no NVIDIA Nsight Graphics/Compute/Systems profiler was found in PATH or the NVIDIA Corporation program directory. Consequently these are controlled shader ablations, not hardware-counter measurements of cache misses, occupancy or register spills.

## Method

The existing fixed-geometry sweep has a short shader-diagnostic mode: counts `1,64,64,1`, each with 120 warm-up and 360 measured frames, full 3440x1440 DLAA, Basic correction, eight initial candidates, both production fusions, shadows and a locked interior camera. All 68 instances and geometry/motion keys remain unchanged. All 28 accepted cases had 360/360 focused frames and 360 asynchronous GPU samples.

Seven separate processes ran in this order: baseline, no temporal, baseline, fixed initial light, constant initial target, compiler-optimized SPIR-V, baseline. The baselines bracket the experiments because clock drift affects timing. No per-frame image or buffer captures were performed. These short benchmarks do not establish sustained FPS.

Shader copies and variants live only in ignored `build/restir-shader-diagnostics/`. Test builds may choose an alternative fused deferred module with `VKE_PROFILE_DEFERRED_SHADER` only while `VKE_RESTIR_PROFILE` is enabled. Production shader files are not overwritten. The diagnostic baseline's SHA-256 matches the production fused shader exactly: `1FCDF0F5030EBD91BD41E863948C6BFFFE71FE0522DDBBDEC8D30CE87B2F1C4A`.

Variants intentionally alter sampling or reuse and must not be used as performance fixes:

- **No temporal:** return the initial reservoir directly. Mark temporal acceptance only to keep the subsequent spatial kernel on its ordinary one-proposal path, avoiding the separate eight-proposal disocclusion boost as a confound. This removes temporal reuse work and changes the estimator's history.
- **Fixed initial light:** evaluate all eight initial candidates against light index zero, retaining their random triangle positions and the full target function. Temporal reuse remains enabled. The light-selection distribution/PDF is deliberately inconsistent; output is not a correctness result. This changes both memory coherence and which geometric/BRDF branches execute, so it cannot isolate cache behavior from arithmetic/divergence.
- **Constant initial target:** retain the initial RIS loop with target one, allowing the compiler to remove initial light reconstruction and BRDF/material evaluation. Temporal target evaluation remains real. This changes the target and resulting reservoir distribution; it is not a substitute for correctly changing the target consistently through the entire estimator.
- **Optimized SPIR-V:** compile the unchanged source with glslc `-O`. This is the only experiment intended to preserve the algorithm, but it showed no reliable performance win and was not adopted.

## Results

Numbers are fused material/initial/temporal GPU milliseconds, retaining both samples per count. Baselines drift across processes; subtracting arbitrary rows does not yield an exact cost decomposition.

| Variant | One emitter | 64 emitters |
| --- | ---: | ---: |
| Baseline before | 1.88–1.94 | 2.68–2.68 |
| No temporal | 1.12–1.23 | 1.55–1.58 |
| Baseline middle | 1.99–2.10 | 2.96–2.98 |
| Fixed initial light | 2.07–2.16 | 2.12–2.14 |
| Constant initial target | 1.87–1.92 | 2.11–2.12 |
| Compiler `-O` | 2.22–2.33 | 3.28–3.30 |
| Baseline after | 2.19–2.19 | 2.89–3.25 |

Observed implications:

1. Initial generation itself retains count-dependent cost with temporal work removed. Temporal reuse alone does not explain the penalty.
2. Removing initial light/target evaluation reduces the penalty substantially. Forcing coherent initial light selection almost eliminates it, even though all 64 emitters and the full target function remain available to temporal reuse. Varied light access and target evaluation are the next optimization targets.
3. Temporal work is also substantial: bypassing it reduces the fused pass by roughly 0.8–1.4 ms relative to surrounding normal observations. This is an approximate sensitivity result, not an additive attribution or a shippable saving.
4. glslc `-O` produced a smaller module (102,132 vs 145,032 bytes), but did not produce a reliable timing improvement. Smaller SPIR-V does not demonstrate cheaper GPU machine code.

Full target evaluation reconstructs the sampled triangle point/UV, reads emission/material data and evaluates the BRDF. Eight initial candidates are followed by temporal target evaluations. There is no per-pixel loop over the entire light list. The ablations support investigating this repeated work and its access/coherence patterns; they do **not** prove the shader is bandwidth-bound, identify an individual cache, or measure register pressure. Changing the light subset also changes directions, support and branch execution.

DLAA medians drifted from about 1.34 to 1.77 ms across baseline observations. The earlier sweep recorded changing GPU clocks. Exact percentages should not be inferred from different processes, and neither these tests nor the previous telemetry diagnose a specific throttling mechanism. No clocks, fan settings or power limits were changed.

## Outcome and next work

Production shaders and estimator settings remain unchanged. No GI, denoiser, visibility reuse or reduced sampling budget was introduced. The test-only shader selection and four-case harness remain available for reproduction.

The next justified experiment is to reduce repeated target-evaluation work and improve light/material data access while preserving the actual proposal distribution, PDFs, BRDF and fresh final shadows. Compact/cached light attributes or explicit reuse of per-surface invariants are candidates, not established wins. A profiler capture would still be needed to distinguish memory stalls from instruction/register costs. Any production candidate requires repeated matched benchmarks and reference/motion/jitter regression checks before claiming a gain.

## Verification and artifacts

- Five diagnostic SPIR-V modules compiled and validated for Vulkan 1.3 with scalar block layout.
- Release engine build passed; all eleven CPU/offline suites passed (`build/restir-shader-diagnostic-cpu.log`).
- All seven four-case GPU timestamp runs passed their benchmark invariants. Release validation layers were disabled; these runs are not Vulkan synchronization-validation or image-correctness evidence.
- `build/restir-shader-diagnostic-results.csv` records all 28 cases, including CPU and GPU timings. Raw logs are `build/restir-diag-{baseline-before,no-temporal,baseline-middle,fixed-initial-light,constant-target,optimized-spirv,baseline-after}.log`.
- Reproduction uses the sweep environment from the preceding report, plus `VKE_PROFILE_INTERIOR=1`, `VKE_PROFILE_SHADER_DIAGNOSTIC=1`, `VKE_PROFILE_VARIANT=<label>` and `VKE_PROFILE_DEFERRED_SHADER=build/restir-shader-diagnostics/<variant>/deferred.spv`. Ignored generated source copies record the exact edits; no diagnostic module is selected by ordinary startup.
- Production fused shader hash and tracked boulder assets remain unchanged. No commits, pushes, PowerShell scripts or documentation synchronization were performed.
