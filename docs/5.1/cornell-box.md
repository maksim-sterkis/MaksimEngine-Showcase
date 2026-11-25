# Cornell Box

Choose **Esc → Scene → Cornell Box** to enter at the measured camera. Reselect the scene after switching away to restore the camera. **F5 → Cornell Box Lighting** exposes the conventional sampler, ReSTIR DI, and the Native 4,096-sample reference. Select NRD or DLSS Ray Reconstruction in the engine's reconstruction settings.

## Geometry and camera

`sandbox_asset_generator` builds the room, both blocks, and one rectangular ceiling emitter. `model_compiler` emits `assets/models/glb/sandbox_cornell_box.glb` for the engine's normal asset path. The original coordinates in millimetres are in `tools/sandbox_assets.cpp`. The model rotates Cornell's coordinates 180° about Y and scales them to metres, preserving the measured skew in the shell and the published positions of the blocks. The left wall is red and the right wall is green. The `130 × 105 mm` emitter replaces a matching opening in the ceiling at `y = 548.8 mm`. [Cornell University scene data](https://bowers.cornell.edu/computer-graphics/data)

The engine-space camera starts at `(-0.278, 0.273, 0.800) m` and looks along `−Z` with `+Y` up. The `35 mm` focal length and `25 × 25 mm` film gate give a square field of view of `2 atan(25 / (2 × 35)) ≈ 39.31°`. On a wide viewport, the center square keeps the measured framing and the view extends horizontally. A tall viewport extends vertically. [Cornell University camera parameters](https://bowers.cornell.edu/computer-graphics/data)

## Spectral conversion

The white, green, and red measurements are stored in `assets/models/sandbox/cornell_reflectance.csv` at Cornell's original `4 nm` steps. The published relative emission samples are `(400 nm, 0)`, `(500 nm, 8)`, `(600 nm, 15.6)`, and `(700 nm, 18.4)`. Cornell gives the emitter a constant diffuse reflectance of `0.78`. [Cornell University reflectance and light source properties](https://bowers.cornell.edu/computer-graphics/data)

Interpolate each Cornell spectrum linearly on a `1 nm` grid from `400` to `700 nm` inclusive. Integrate with the trapezoidal rule and the CIE 1931 2° color matching functions. For a reflectance spectrum `R`, integrate `D65 × R × (x̄, ȳ, z̄)` and divide by the integral of `D65 × ȳ` on the same interval. For emission, integrate `E × (x̄, ȳ, z̄)` and divide by the integral of `ȳ`. This preserves the source's relative magnitude as engine radiance. Absolute luminance and camera exposure are uncalibrated. [CIE 1931 observer](https://doi.org/10.25039/CIE.DS.xvudnb9b) and [CIE D65 illuminant](https://doi.org/10.25039/CIE.DS.hjfjmt59)

Multiply XYZ by the following matrix and store linear sRGB values in glTF.

```text
 3.2404542  -1.5371385  -0.4985314
-0.9692660   1.8760108   0.0415560
 0.0556434  -0.2040259   1.0572252
```

| Material | Linear RGB |
| --- | --- |
| White | `(0.736179, 0.736703, 0.737733)` |
| Red | `(0.492172, 0.049116, 0.051050)` |
| Green | `(0.067920, 0.398722, 0.086265)` |
| Emitter radiance | `(20.726452, 10.861242, 2.772402)` |

The GLB stores emission as `emissiveFactor = (1, 0.524028, 0.133762)` multiplied by `KHR_materials_emissive_strength = 20.726452`. The shared area sampler evaluates Cornell Box surfaces as Lambertian (`albedo / π`). The renderer multiplies RGB reflectance and emission, an approximation to multiplying the underlying spectra. Direct visibility accounts for the block shadows. Indirect illumination and color bleeding from the physical photographs require GI.

## Direct-lighting audit

The test-enabled Release build ran on an RTX 3090 with driver `621.14` at `1280 × 720`. The audit compares one frame after 64 submitted frames per mode against a `4,096`-sample conventional MC reference with ray-traced shadows. It measures linear HDR error across diffuse receivers in the `720 × 720` center crop, including shadowed pixels. The mask excludes the visible emitter and background. The PSNR peak is the brightest reference receiver luminance.

| Mode | Linear RGB RMSE | Luminance PSNR | Mean luminance / reference |
| --- | ---: | ---: | ---: |
| MC4 | 0.00822 | 30.98 dB | 1.0003 |
| ReSTIR DI, 4 candidates | 0.00855 | 30.78 dB | 1.0009 |
| ReSTIR DI + NRD RELAX Native | 0.00391 | 37.21 dB | 0.9897 |
| ReSTIR DI + NRD REBLUR Native | 0.00361 | 38.29 dB | 0.9802 |
| ReSTIR DI + DLSS 4.5 RR DLAA, preset F | 0.01167 | 27.09 dB | 1.0012 |

The audit also checks that the scene registers exactly two emissive triangles, that switching off emission leaves the diffuse receivers dark, and that returning to Boulder clears the Cornell Box geometry and lighting. Vulkan synchronization validation reported the previously documented first-evaluation hazards and layout errors on NGX-owned images during RR. The other four modes completed without validation reports. [Ray Reconstruction validation status](../rendering/dlss_integration.md#experimental-dlss-45-ray-reconstruction)

Run from the repository root after building with `VKE_BUILD_TESTS=ON`. The audit writes `build/cornell/metrics.csv` and PNGs with mode output, reference, and four-times absolute RGB error from left to right.

```powershell
$env:VKE_CORNELL_BOX_AUDIT = '1'
$env:VKE_FIXED_PRESENT_PROBE = '1'
$env:VKE_PROBE_HIDDEN = '1'
$env:VKE_FORCE_VALIDATION = '1'
$env:VK_LAYER_VALIDATE_SYNC = '1'
.\build\VK_game_engine.exe
```
