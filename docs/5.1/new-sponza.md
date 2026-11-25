# Intel New Sponza (2022)

Select **Intel New Sponza (2022)** in the Esc menu. The scene starts with the base atrium. The same menu selects six authored camera presets and independently enables Colorful Curtains, Ivy, Trees, and Emissive Candles. **Candle point lights** enables the authored punctual lights when the candle add-on is active. The F5 lighting panel controls the sun, ReSTIR DI, and the alpha-visibility diagnostic.

## Assets and credits

The base scene is Intel's `NewSponza_Main_glTF_003` revision. The scene was commissioned by Frank Meinl and sponsored by Anton Kaplanyan. The add-on crew includes Katica Putica, Cristiano Siqueira, Timothy Heath, Justin Prazen, Sebastian Herholz, Bruce Cherniak, and Anton Kaplanyan. Reference photography is by Katica Putica (Princino.photo, Dubrovnik). Intel's library lists the files under Creative Commons Attribution. Each original credits file accompanies its compiled package and contains the CC BY 4.0 text and the package's usage notice. [Intel GPU Research Samples Library](https://www.intel.com/content/www/us/en/developer/topic-technology/graphics-research/samples.html)

| Package | Intel download ID | Source glTF SHA-256 | Source triangles | Render instances |
| --- | --- | --- | ---: | ---: |
| Base | 830833 | `e04c4c540c74bdddcbd3f590a85c14119bfd5839702246a23b3a737c0cce2400` | 3,747,018 | 115 |
| Curtains | 726650 | `b56968aeb691970b63027ae3ffbb0d77f1d736e7bdace5c70be106d6dcfbb516` | 1,997,366 | 17 |
| Ivy | 726656 | `22039080d6de24569f377210b8ebbe59194b5d44de777679d9d87b69ffda2c1e` | 5,065,558 | 11 |
| Trees | 726662 | `3fef7b677e5d8ae87ac84f2e8a27a32fde9441b82bbcfd0de0a3bc6bf5e41fc9` | 2,023,747 | 6 |
| Candles | 726676 | `285326986de75d90bc9e662ca9a7e325efd04ac6e0d2142ade62878e31d1e453` | 19,540,000 | 10,000 (1 model) |

The imported 10,000 identical candles deduplicate into a single instanced `candle.glb` (`1,954` tris at LOD 0 across `10,000` ECS nodes) and stay instanced in the TLAS by default (reusing one compact L1/L2-resident BLAS saves `346.54 MiB` VRAM and runs faster spatial shading than 115 static spatially-clustered BLASes; `VKE_SPONZA_CLUSTER_CANDLES=1` remains available in test builds for A/B comparison), allowing batched GPU culling/indirect mesh-task dispatch (`>= 8` instances) and opaque mesh-shader specialization (`shader_opaque.mesh.spv`). Meshes share full-resolution KTX2 maps within each package; all 88 compiled textures (`33` `BC7_SRGB`, `28` `BC5_UNORM`, `27` `BC7_UNORM`) are offline pre-transcoded to native GPU block formats (`model_compiler --transcode-ktx2`) for fast multi-threaded disk reads and batched GPU uploads without runtime UASTC transcoding.

The Intel tree foliage and the base dirt decals use glTF `BLEND`. The visibility buffer imports them as `MASK` with cutoffs of 0.5 and 0.05 respectively. Hardware RT BLASes separate opaque geometry (`VK_GEOMETRY_OPAQUE_BIT_KHR`, Geometry 0) from alpha-masked geometry (Geometry 1), so foliage/decals only pay per-triangle alpha tests on intersected non-opaque triangles, while opaque geometry (`RasterClass::OpaqueSingleSided` / `OpaqueDoubleSided`) uses dedicated early-Z visibility fragment shaders (`shader_opaque.frag`). The Alpha visibility agreement view compares foliage hits. Green marks agreement, red marks ray-only hits, and magenta marks mismatches with a masked raster hit. The coplanar dirt decals are excluded from this comparison.

All 2,720,000 authored flame triangles render using a procedural `64×128` incandescent core-to-amber gradient (`candle_flame_emissive-srgb.ktx2` at strength `18.0`). When **Candle point lights** is active (default `on`), the 10,000 authored punctual candle lights (warm orange-amber `[0.05, 0.018, 0.004]`, `2 cm` radius softening `+ 0.0004` m², flame shells excluded from BLAS so enclosed point lights are never self-shadowed) illuminate the atrium via a `16×8×8` 3D spatial light grid (`75%` local `3×3×3`-cell / `25%` global defensive mixture importance sampling) instead of duplicated area-light emitters. The baseline uses a 0.53° sun at azimuth 45°, elevation 70°, and intensity 4 (`1` temporally jittered shadow ray/px for NRD RELAX/REBLUR and DLSS RR, `2` for ReSTIR Native/DLSS SR, and combined solar+candle NRD demodulation). The base glTF's punctual lights have zero intensity.

## Reimport

Extract the Intel glTF package folders and their credits files under one source directory. Keep each folder's glTF, binary buffers, and textures together. From the private workspace root, run the importer with Node.js and a Release model compiler.

```powershell
cmake --build build/release-notests --target model_compiler -j8
node tools/import_sponza.mjs build/sponza-source build/release-notests/model_compiler.exe
```

An optional trailing package list limits the import to `base`, `curtains`, `ivy`, `trees`, or `candles`. The generated assets and original credits live under the Git-ignored `assets/models/sponza/` directory. The engine discovers imported packages at startup and loads each the first time it is enabled. Loaded assets stay resident for the session. Fresh processes provide isolated VRAM results for each configuration.

## Benchmark procedure

Use an optimized Release build with `VKE_BUILD_TESTS=ON` for the benchmark entry point. All testing uses **Force LOD 0** (which is also the engine startup default until Phase 6.1 fixes dynamic LOD temporal/RT/meshlet stability; see [Future work](../planning/future-work.md#force-lod-0-default--dynamic-lod-temporal-stability-deferred-to-phase-61)). The local sweep used the RTX 3090 with driver 621.14, a fixed 1280×720 output, Mailbox, four RIS candidates, Basic ReSTIR DI, LOD 0, matched raster and RT meshes, and diagnostic guides enabled for captures. Every run used `PhysCamera001`, then moved the camera 2 m sideways and turned it 36° over 48 submitted frames. Exclude the first 16 frames from static measurements. The GPU timings cover the submitted work and the presentation compute pass. Wall times also include the host and swapchain work in the hidden probe.

Set `VKE_FIXED_PRESENT_PROBE=1`, `VKE_PROBE_HIDDEN=1`, `VKE_TEST_VSYNC=0`, and `VKE_SPONZA_BENCH` to the required mask. The masks are 0 for base, 1 for curtains, 2 for ivy, 4 for trees, 8 for candles, and 15 for all add-ons. Set `VKE_RECON_MODE` to 0 for Native, 1 for DLAA, 2 for DLSS Quality, 5 for RR DLAA, or 6 for RR Quality. Set `VKE_DENOISER_MODE` to 0 for Off, 1 for RELAX, or 2 for REBLUR. The probe writes per-frame CSVs under `build/sponza-bench/`.

`VKE_SPONZA_CAPTURES=1` saves static and pan checkpoints. `VKE_SPONZA_ALPHA_AUDIT=1` compares primary raster and ray foliage hits and requires the Trees add-on. `VKE_SPONZA_TRANSITIONS=1` tests resizing the window, changing LOD, disabling add-ons, switching camera preset, and returning to Boulder. `VKE_SPONZA_POINTS=1` includes the candle point lights.

## Add-on sweep

Native 1280×720 with ReSTIR DI and denoiser Off. Each timing cell gives the static / pan median in milliseconds. Lighting includes the fused ReSTIR and direct sunlight passes. The CSV also records the constituent ReSTIR timestamps.

| Active packages | LOD 0 triangles | Instances | Active light samples | VRAM (MiB) | Geometry (ms) | Lighting (ms) | GPU frame (ms) |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Base | 3,747,018 | 115 | 8,098 area | 2,566 | 0.164 / 0.186 | 1.535 / 1.681 | 1.807 / 2.002 |
| Base + curtains | 5,744,384 | 132 | 8,098 area | 2,973 | 0.305 / 0.308 | 1.488 / 1.805 | 1.889 / 2.278 |
| Base + ivy | 8,812,576 | 126 | 8,098 area | 3,254 | 0.168 / 0.208 | 1.366 / 1.650 | 1.630 / 1.997 |
| Base + trees | 5,770,765 | 121 | 8,098 area | 3,009 | 0.303 / 0.350 | 1.790 / 2.121 | 2.195 / 2.608 |
| Base + candles + points | 23,287,018 | 10,115 | 8,098 area + 10,000 pt | 2,599 | 1.267 / 1.271 | 1.863 / 2.139 | 3.780 / 4.168 |
| All add-ons + points | 32,373,689 | 10,149 | 8,098 area + 10,000 pt | 4,137 | 1.242 / 1.357 | 2.784 / 3.175 | 4.684 / 5.192 |

The foliage audit matched 72,942 pixels and differed on 1,384 (672 masked raster hits) — 98.1% agreement at the benchmark camera. Cold Release asset loading takes ~1.01 s for Base and ~1.68 s across all five packages combined (`curtains` ~0.18 s, `ivy` ~0.27 s, `trees` ~0.22 s, 10,000-instance `candles` ~0.04 s).

## All-addons + points reconstruction comparison

Each row below uses all add-ons plus all 10,000 candle point lights (`10,149` instances, `8,098` area + `10,000` point lights, `16×8×8` spatial light grid, 4 RIS candidates). For `> 256` lights, NRD RELAX uses 1-spp ReSTIR reuse (`denoising = (0, 1, 0, 0)`) and DLSS Ray Reconstruction uses decorrelated spatial-grid draws (`denoising = (0.85, 2, 8, 0)`).

### 1280×720 output

| Mode | Internal | VRAM (MiB) | Lighting (ms) | NRD (ms) | Reconstruction (ms) | GPU frame (ms) | Wall frame (ms) |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Native | 1280×720 | 4,137 | 2.784 / 3.175 | — | — | 4.684 / 5.192 | 7.674 / 8.314 |
| DLAA + RELAX | 1280×720 | 4,431 | 2.785 / 2.852 | 1.172 / 1.171 | 0.478 / 0.475 | 6.519 / 6.537 | 9.907 / 9.881 |
| DLSS Quality + RELAX | 853×480 | 4,191 | 1.659 / 1.462 | 0.560 / 0.558 | 0.463 / 0.462 | 5.028 / 4.611 | 8.567 / 7.953 |
| RR DLAA | 1280×720 | 4,871 | 7.326 / 6.616 | — | 3.852 / 3.908 | 13.100 / 12.471 | 16.635 / 16.083 |
| RR Quality | 853×480 | 4,607 | 2.810 / 2.954 | — | 1.662 / 1.875 | 6.497 / 6.701 | 9.784 / 10.003 |

### 3440×1440 ultrawide output (`VKE_PROBE_BORDERLESS=1`, post-Wave 4)

The initial `1280×720` and early `3440×1440` baselines were superseded by O(1) Vose Global + `3×3×3` Local Alias Tables, `DEVICE_LOCAL` instance/indirect buffers, static Hi-Z instance culling + TLAS caching, compacted `FORCE_OPAQUE` BLASes, single-dispatch `restir_spatial_one.comp`, and temporally decorrelated dense-light RIS shading (`1` ray/px) for `>256` lights. Full steady-state `3440×1440` Mailbox & Immediate timings and per-pass breakdowns are recorded in [Integration validation](integration-validation.md#1-intel-new-sponza-3440x1440-all-add-ons--10k-point-lights-lod-0):

| Mode | Internal | VRAM (MiB) | `gpu − recon` (ms) | NRD (ms) | Reconstruction (ms) | GPU frame (ms / FPS) | Wall frame (ms / FPS) |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| **Native (ReSTIR DI)** | 3440×1440 | 5,145 | 13.684 / 14.496 | — | 0.001 / 0.006 | 13.685 (73.1) / 14.502 (69.0) | 15.275 (65.5) / 15.880 (63.0) |
| **DLSS Quality + RELAX** | 2293×960 | 4,901 | 8.982 / 9.097 | 3.072 / 3.172 | 1.627 / 1.682 | 10.609 (94.3) / 10.779 (92.8) | 12.267 (81.5) / 12.381 (80.8) |
| **DLSS Quality + REBLUR** | 2293×960 | 4,879 | 8.695 / 8.431 | 2.692 / 2.572 | 1.697 / 1.732 | 10.392 (96.2) / 10.163 (98.4) | 12.322 (81.2) / 11.722 (85.3) |
| **RR Quality** | 2293×960 | 5,330 | 5.883 / 6.149 | — | 9.602 / 9.618 | 15.485 (64.6) / 15.767 (63.4) | 17.248 (58.0) / 17.441 (57.3) |
| **RR Balanced (Mailbox)** | 1995×835 | 5,066 | 5.095 / 5.130 | — | 7.668 / 7.701 | 12.763 (78.4) / 12.831 (77.9) | 14.508 (68.9) / 14.494 (69.0) |
| **RR Balanced (Immediate)** | 1995×835 | 5,066 | 5.207 / 5.187 | — | 7.726 / 7.756 | 12.933 (77.3) / 12.943 (77.3) | 13.724 (72.9) / 13.975 (71.6) |
| **RR Performance** | 1720×720 | 4,841 | 4.179 / 3.951 | — | 5.833 / 5.914 | 10.012 (99.9) / 9.865 (101.4) | 11.779 (84.9) / 11.452 (87.3) |

Geometry averages `1.263–1.790 ms`, `cull + TLAS` takes `0.014–0.017 ms`, and presentation compute takes `0.099–0.153 ms`. Side-by-side PNG captures at `3440×1440` (`RR Balanced`, `DLSS Quality + RELAX`, `DLSS Quality + REBLUR`) confirm smooth warm candle illumination, sunlit foliage, sharp textures, and zero RELAX fireflies, REBLUR boiling, or floor darkening.

## Verification

`two_room_smoke`, `cornell_box_audit`, `sandbox_gpu_smoke`, `restir_full_smoke`, `temporal_smoke`, `hiz_verify`, `nrd_smoke`, `nrd_audit`, `reflex_smoke`, and `sponza` (`VKE_SPONZA_BENCH=15` with `POINTS`, `ALPHA_AUDIT` `97.6%` agreement, and `TRANSITIONS`) all passed with forced Vulkan + synchronization validation (`VKE_FORCE_VALIDATION=1 VK_LAYER_VALIDATE_SYNC=1`) and zero engine-owned errors. Full results, `VKE_MEMORY_SOAK=1` (`0.00 MiB` VRAM delta over 3 cycles), and `VKE_REFLEX_BENCH=1` are documented in [Integration validation](integration-validation.md).
