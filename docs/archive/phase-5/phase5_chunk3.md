# Phase 5 Chunk 3: Initial Direct-Light Reservoirs

This is the historical initial-only milestone. [Full ReSTIR DI](../../rendering/restir_di.md) now supplies temporal/spatial reuse, source-frame correction and fresh final shading; its completed mode replaces initial-only sampling in the main menu. The results/defaults below describe Chunk 3, not the current default.

Chunk 3 implements the initial candidate/reservoir stage of ReSTIR DI on `phase-5`. There is **no temporal or spatial reuse** yet. Conventional sampling remains the sandbox default; F5 can select Initial RIS. The boulder keeps its directional-light path. Phase 5 remains in progress, RR remains Phase 5.1 and SVGF stays deferred.

## Sampling contract

The implementation follows weighted reservoir sampling and initial resampled importance sampling described in the [ReSTIR paper](https://research.nvidia.com/labs/rtr/publication/bitterli2020spatiotemporal/) and the [RTXDI integration sequence](https://github.com/NVIDIA-RTX/RTXDI/blob/main/Doc/Integration.md). It is an engine implementation in GLSL/C++; no RTXDI library or new runtime dependency is imported.

For a light point `y`, `q(y)` is the existing uniform triangle-selection/world-area density. `f(y)` is RGB incident direct-light contribution in area measure, including BRDF, emission and geometry but excluding visibility. The positive scalar target is `t(y) = luminance(f(y))`. Each candidate contributes weight `t/q`. Weighted streaming selects one candidate, retaining the raw sum `S`, target `t_selected` and proposal count `M`. Zero-target candidates still increment `M`. The final estimate is:

```text
W = S / (M * t_selected)
L = visibility(selected) * f(selected) * W
```

Candidate generation never traces shadows. A nonempty valid reservoir traces at most one finite shadow segment during final shading; shadows-off traces none. Occlusion does not discard the selected reservoir or alter its normalization. All-zero targets produce zero sampled direct lighting, while visible emission is added separately once. Invalid/nonfinite weights mark the reservoir as an error and suppress its estimate; the diagnostic view shows magenta. Contributions are not clamped to hide errors.

The conventional path still averages independent visible contributions. The 4096-spp reference always uses that path, even when Initial RIS is selected. Neither reservoirs nor reference sums are supplied by DLAA/DLSS history. Reservoirs are regenerated every frame; no sample survives scene/light/camera/LOD/resolution changes through reuse in this chunk.

## Layout, lifetime and controls

`restir_math.hpp` defines the CPU math and asserted 48-byte std430 reservoir layout. `restir_initial.glsl` implements streaming and final visibility within deferred lighting, using the shared surface/material/light helpers. Each record contains:

| Field | Meaning |
| --- | --- |
| `identity: uvec4` | Selected light's stable entity, source surface, triangle and global material |
| `selected: uvec4` | Current light index, candidate count M, shadow-segment attempts, valid/error flags |
| `data: vec4` | Barycentric y/z, raw weight sum, selected area-measure target |

The scene header grows from 48 to 64 bytes, adding a `uvec4` for method, candidate budget, reservoir diagnostic and a reserved word. All shaders share `scene_data.glsl`. Visibility remains 64 bits, push constants 256 bytes, temporal `FrameGpu` 704 bytes, and other model/material/instance/light layouts remain unchanged.

Set 3 binding 14 is a storage buffer for the current submission slot's reservoir array. Two arrays are allocated with render targets, independently of SDK resources and reference sums, and released/recreated with the internal extent. Extent multiplication and device storage-buffer range are checked. At 1280x720, each array is 44,236,800 bytes (about 42.2 MiB); both total about 84.4 MiB. This straightforward full-precision format is intentionally uncompressed. A later reuse implementation must budget memory and explicitly map previous lights; the stored current light index alone is not persistent identity.

Deferred dispatch overwrites every pixel, including empty records for background, conventional/reference modes and the boulder. The empty-scene path clears the buffer explicitly. Existing queue barriers order slot reuse; acquired-image fences protect descriptor updates. There is no production CPU image/buffer readback or per-frame buffer allocation. The 48-byte write per pixel is part of the measured lighting cost.

F5's Lighting Sandbox adds Conventional MC / Initial RIS selection, 1–64 RIS candidates (default eight), and selected-light, candidate-count, log-weight and shadow-segment views. RIS controls are disabled for conventional sampling. Reservoir views bypass SDK reconstruction and restore the requested mode when closed. Conventional samples/pixel remain separate. Reference requires Native, unjittered plain lighting with reservoir diagnostics off. Method, candidate and diagnostic changes enter the conservative temporal/reference keys. These are session-only controls.

The chunk also corrects deferred shading's remaining first-object transform uses: current motion projection, world normal, tangent, curved shadow origin, scale and view direction now use the visible instance. The fixed-window test independently reconstructs world/local positions from GPU depth and checks per-instance motion against CPU reprojection during deterministic object motion.

## Verification

Development remains MinGW Makefiles Debug, `VKE_BUILD_TESTS=ON`, `VKE_ENABLE_DLSS=ON`, pinned Streamline 2.14.16. The engine and affected shaders build; affected SPIR-V validates with Vulkan 1.3/scalar-block-layout. No assets, SDK binaries or existing boulder data are changed by this chunk. No commits, pushes, PowerShell scripts or showcase synchronization are performed.

Ten CTest suites pass. New `restir_initial` CPU coverage includes empty/zero targets, weighted replacement frequency, correct M, stable selected identity, unequal proposal probabilities, HDR RGB integrands, independent final visibility, candidate budgets 1/2/8/64, numeric failures and extent-size overflow. Statistical estimates are checked against an independently summed discrete integral with six-standard-error bounds plus a small numerical margin. Existing temporal, reconstruction, Hi-Z, meshlet, compiler and asset tests remain enabled.

`VKE_RESTIR_GPU_SMOKE=1` with `VKE_FIXED_PRESENT_PROBE=1`, `VK_LAYER_VALIDATE_SYNC=1` and `VKE_RECON_MODE=0` runs the finite new suite. It opens one fixed 1280x720 window and uses 320x180 internal Native rendering for the quantitative comparisons. No resize/minimize or desktop display-mode sweep is performed. It covers:

- Three/64 emitters, each with shadows enabled and disabled.
- A 4096-spp conventional reference, then 128 independent frames of MC8 and 128 of RIS8 for each case. Receiver-only RGB means exclude background and emissive surfaces; per-frame standard errors and single-frame pixel MSE are recorded.
- Raw reservoir identity, bounds, barycentrics, candidate counts, positive finite normalization, zero/error state and at-most-one shadow segment.
- Identical one-candidate MC/RIS proposals, zero emission, reference/conventional buffer clearing, object/light motion, per-instance numeric motion, DLAA/Quality evaluation, diagnostics, scene switches and internal extent recreation.

The earlier fixed-window sandbox suite also runs Initial RIS to check exact Hi-Z-on/off HDR parity, same-size reconstruction target reuse, boulder fast/full-guide parity, ownership and reference completion/reset. Test readbacks deliberately wait for the GPU and are absent in production.

Logs and quantitative results are under ignored `build/`: `phase5-chunk3-gpu-final.log`, `phase5-chunk3-regression-gpu.log`, `phase5-chunk3-final-build.log` and `restir-comparison.csv` (the recorded run is also preserved as `phase5-chunk3-comparison-verified.csv`). The CSV records RGB means/reference/error bars, pixel MSE and median combined-lighting, light-preparation and total GPU milliseconds. These Debug/synchronization-validation measurements use full diagnostic guides and are correctness/cost observations, not optimized throughput benchmarks. Sampling, material work and final visibility share one dispatch; shadow-on/off differences are an estimate of path cost, not isolated ray traversal.

## Recorded results

On 2026-10-29, both finite GPU suites passed on RTX 3090 / NVIDIA driver 621.01 with no Vulkan validation or synchronization-hazard messages. The final independent motion check matched 29,784 covered pixels, including 618 moving pixels, within 0.04 internal pixels. Reservoir diagnostic colors also match their stored state, bypass SDK reconstruction while active and restore it when closed; a stale sandbox view does not bypass boulder reconstruction. The four comparisons' maximum RGB mean difference from reference was 0.121%; the acceptance bound is six measured standard errors plus 1.5% reference magnitude and a 0.0001 numerical margin. Finite-reference uncertainty is not zero. The saved few-light reference and 128-frame MC/RIS means were inspected.

| Scene / shadows | MC8 lighting ms | RIS8 lighting ms | MC8 single-frame MSE | RIS8 single-frame MSE |
| --- | ---: | ---: | ---: | ---: |
| Three emitters / on | 0.191488 | 0.111616 | 0.000319552 | 0.00109865 |
| Three emitters / off | 0.072704 | 0.077824 | 0.000309506 | 0.00102831 |
| 64 emitters / on | 0.158720 | 0.122880 | 0.00196220 | 0.00367495 |
| 64 emitters / off | 0.110592 | 0.117760 | 0.00193443 | 0.00348235 |

These are one Debug/validation run at 320x180 internal resolution, with candidates=8, MC samples=8 and Native output. Reference excludes denoising/reconstruction from the estimator comparison. RIS spends less combined-lighting time with shadows in these cases, but has higher per-frame MSE. Shadows-off RIS costs slightly more than MC here. Cached light preparation is typically zero to about 0.001 ms; total render timing and error bars are in the CSV. GPU clocks and concurrent desktop GPU load were not controlled. Differences between rows/runs are not isolated hardware-ray timings or a production-speed guarantee. The conventional method therefore remains the default until reuse is implemented and evaluated.

Initial selection reduces visibility rays but does not guarantee lower noise than averaging all candidates. Recorded noise and cost must be evaluated together. Temporal reuse, previous/current light mapping, spatial reuse, reuse rejection/bias correction, denoising and distribution remain future work. Direct desktop menu interaction and extended image-quality review retain the previously documented capture-tool limitation; rendered GPU captures and numeric checks do not claim successful desktop automation.
