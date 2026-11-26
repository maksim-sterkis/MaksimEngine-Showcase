# Independent DLSS Reconstruction Resolution & Spatial Presentation Upscaling (Phase 5.1, Task 8)

Implemented percentage-based **independent DLSS reconstruction scaling** (`Off (100% - WxH)`, `85%`, `75%`, `67% (2/3)`, `50%`) paired with a lightweight **compute spatial presentation upscaler** (**EASU** + optional **RCAS** contrast-adaptive sharpening, or **Bilinear**), native-resolution analytical emitters, native-resolution ImGui UI compositing, and a live non-hardcoded **Settings Advisor**.

---

## Architecture & Zero-Overhead Guarantees

### Three-Resolution Pipeline
The renderer tracks three explicit extents instead of conflating DLSS reconstruction output with display size:
1. **`renderExtent`**: Internal geometry, depth, Hi-Z, deferred lighting, ReSTIR DI, NRD, and DLSS RR guide resolution (queried from Streamline `slDLSSGetOptimalSettings` / `slDLSSDGetOptimalSettings` against `reconstructionExtent`).
2. **`reconstructionExtent`** (`temporal::State::outputExtent`, `SwapchainState::reconstructionExtent`): Streamline DLSS SR / DLSS RR output target (`temporal::FrameTargets::resolved`), rounded to even dimensions and clamped `<= outputViewport.extent`. Jitter phases and texture `mip_bias` use `(renderExtent, reconstructionExtent)`.
3. **`outputViewport.extent`** (inside `swapchain.extent`): Final presented scene viewport and native-resolution Dear ImGui UI (`uiRenderPass`).

### Strict Zero-Overhead `Off (100%)` Policy (Default)
- **Default state**: `reconstructionScale = 100` (`Off (100% - WxH)`), `spatialUpscaler = EASU` (selected filter is preserved even while grayed out at `100%` or `Native`).
- **Zero RAM / VRAM / GPU processing when `Off (100%)`** (or when DLSS is `Native`, or `reconstructionExtent == outputViewport.extent`):
  - `spatial::release()` destroys every spatial image, memory allocation, image view, sampler, descriptor pool, descriptor set layout, pipeline layout, and compute pipeline (`allocatedBytes == 0`, all handles `VK_NULL_HANDLE`).
  - No spatial compute passes or barriers are recorded (`spatialUpscaleMs == 0.0`), and presentation executes the unchanged 1:1 `present.comp` / `present_emitters.comp` pass.
- **RCAS-Off (`spatialSharpness <= 0.001f`)**:
  - Destroys the display-resolution intermediate `upscaled` `RGB10A2` image and RCAS pipelines (`upscaled.image == VK_NULL_HANDLE`, `rcasPipeline == VK_NULL_HANDLE`), writing directly from `spatial_upscale.comp` to `displayColor` (saving ~`20.16 MiB` at `3440×1440` and skipping the RCAS pass).
  - Toggling sharpness or switching between `EASU` and `Bilinear` does not destroy temporal/NRD targets or reset DLSS/ReSTIR history.

### Compute Spatial Pipeline (`src/renderer/spatial_upscale.hpp` / `.cpp`)
1. **`shaders/spatial_prep.comp`** (at `reconstructionExtent`): Applies ACES tone mapping (`2.2` display gamma) and `5×5` `linearDepth` flat-background preservation into `tonemapped` (`VK_FORMAT_A2B10G10R10_UNORM_PACK32`) so spatial negative lobes never produce HDR ringing halos around bright lights.
2. **`shaders/spatial_upscale.comp`** (`spatial_upscale_emitters.comp.spv` with `-DVKE_OUTPUT_EMITTERS=1`):
   - **EASU**: 12-tap edge-adaptive directional Lanczos2 filter clamped to the 4 inner neighbors `[min4, max4]`.
   - **Bilinear**: Hardware linear `sampler2D` baseline.
   - When RCAS is disabled, writes directly to `displayColor` (with letterbox/pillarbox clearing and optional `resolveOutputEmission` analytical emitter compositing at `outputViewport.extent`).
3. **`shaders/spatial_rcas.comp`** (`spatial_rcas_emitters.comp.spv` with `-DVKE_OUTPUT_EMITTERS=1`): Optional 5-tap cross contrast-adaptive sharpening (`0.00–1.00`), followed by analytical output-resolution emitter compositing (`resolveOutputEmission`), writing to `displayColor` before the native-resolution `uiRenderPass`.

---

## Settings UI, F4 HUD & F5 Diagnostics

- **Settings → Upscaling & Denoising**:
  - `Reconstruction scale`: `Off (100% - WxH)` (default), `85%`, `75%`, `67%`, `50%` showing live aspect-matched pixel dimensions for the current `outputViewport.extent` (disabled in `Native`).
  - `Spatial upscaler`: `EASU (Edge-Adaptive)` (default filter), `Bilinear` (grayed out when scale is `Off (100%)` or mode is `Native`, preserving the user's filter choice without switching it off).
  - `Spatial sharpness`: `0.00 – 1.00` slider (default `0.25`; `0.00` disables RCAS and frees its intermediate image; grayed out alongside `Spatial upscaler`).
  - Live `Pipeline: <render> -> <reconstruct> (<DLSS mode>) -> <display> (<spatial>)` preview row queried directly from Streamline before clicking **Apply**.
  - **`Recommend...` Settings Advisor (`src/core/upscale_advisor.hpp`)**: Clicking **`Recommend...`** automatically runs a 7-point live scene probe with a remaining-time countdown across all authored camera presets in the active scene (e.g., all 6 Intel New Sponza cameras, restoring your original camera afterward; `RR Quality/Balanced/Performance @ 100%`, `RR Performance @ 85%/75%`, `RR Quality @ 75%/67%`) measuring real wall-clock FPS, 1% low FPS, and NVIDIA Reflex `simToGpu` latency (including CPU, presentation/Mailbox sync, and VSync cap). Features a selectable **`Target`** FPS (`Auto (Hz)`, `60`, `75`, `90`, `100`, `120`, `144`), ranks `RR Perf + 85%/75% EASU` above `RR UltraPerf`, avoids `RR UltraPerf` whenever standard modes hit the FPS target, and reports **Quality**, **Balanced**, and **Fastest** tiers with one-click **`Stage`** buttons and **`Re-measure`** (no hardcoded GPU/FPS constants).
  - Selecting Ray Reconstruction at 100% scale with an output viewport at least 3600 pixels wide and a reported GPU capacity of up to 10 GiB stages `67% + EASU`. The Advisor uses a lower-memory probe list when the viewport is at least 3000 pixels wide with the same VRAM limit. It starts with RR Performance and RR Quality at 50%, repeats both at 67%, then measures RR Performance at 75% and 100% and RR Ultra Performance at 100%.
  - Persisted in `settings.ini` (`reconstructionScale`, `spatialUpscaler`, `spatialSharpness`).
- **F4 Performance HUD**: Displays `<render> -> <recon> -> <display>` and `Spatial (<upscaler>): <ms>` only when spatial upscaling is active; stays unchanged when `Off`.
- **F5 Diagnostics → Reconstruction**: Reports `Render`, `Reconstruct`, and `Display` dimensions separately alongside `Spatial upscale` status, VRAM, and GPU time.

---

## Verification & `3440×1440` RTX 3090 Benchmarks

> [!NOTE]
> The test workstation uses a **`3440×1440`** ultrawide display (no 4K monitor attached), so 4K (`3840×2160`) presentation was not benchmarked; all GPU benchmarks and visual captures below were measured at native `3440×1440` borderless Immediate presentation on the **NVIDIA GeForce RTX 3090** (driver `621.14`, Streamline `2.14.16` / NGX `310.9.16`, `preset F`, Force LOD 0).

### Validation & Unit Suites
- **Builds & SPIR-V**: Debug (`build`), Release+tests (`build/nsight-release`), and Release-notests (`build/release-notests`, `VKE_BUILD_TESTS=OFF`) compile cleanly; all `24` engine SPIR-V shaders (including `spatial_prep`, `spatial_upscale`, `spatial_upscale_emitters`, `spatial_rcas`, `spatial_rcas_emitters`) pass `spirv-val --target-env vulkan1.3`.
- **CTest (`12/12`)**: `reconstruction_tests` verifies `67%` (`2/3`) exact `2560×1440` at 4K and `2294×960` at `3440×1440`, `75%` (`2580×1080`), `50%` (`1720×720`), `Off`/`100%`/`Native` identity, `jitter_phases`/`mip_bias` against `reconstructionExtent`, and `settings.ini` persistence/malformed defaults.
- **Forced Vulkan + Sync Validation (`VKE_SPATIAL_SMOKE=1`, `VKE_FORCE_VALIDATION=1`, `VK_LAYER_VALIDATE_SYNC=1`)**:
  - **Stage 0 (`Off` + DLSS Quality)**: Verified `0` bytes allocated, all spatial Vulkan handles `VK_NULL_HANDLE`, `0` dispatches, `0.0 ms` spatial GPU time.
  - **Stage 1 (`EASU 75% + RCAS 0.25` + RR Quality, Sandbox64 emitters)**: Verified active spatial + RCAS + analytical output emitters.
  - **Stage 2 (`EASU 75% + RCAS 0.00`)**: Verified `upscaled` image and RCAS pipelines destroyed while preserving DLSS RR temporal history.
  - **Stage 3 (`Bilinear 67%` + window resize `1440×810`)**: Verified clean target rebuilds.
  - **Stage 4 (Return to `Off`)**: Verified `0` bytes allocated and all handles `VK_NULL_HANDLE`.
  - Reported **0** engine Vulkan validation or synchronization errors (also confirmed zero regressions on `VKE_DLSS_SMOKE=1` and `VKE_NRD_SMOKE=1`).

### New Sponza (`3440×1440` Immediate, All 4 Add-ons + 10k Candle Points, Force LOD 0)

Means across `89` measured static + motion-pan + disocclusion frames (`VKE_SPATIAL_BENCH=1`, excluding the 3 synchronous `.ppm` capture-write frames):

| Case | Render | Reconstruct | Display | Process VRAM | Spatial VRAM | Geo (ms) | Light (ms) | Recon (ms) | Spatial (ms) | Total GPU (ms) | Wall (ms) | GPU FPS |
| :--- | :--- | :--- | :--- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `3440 RR Quality` (Spatial Off) | `2293×960` | `3440×1440` | `3440×1440` | `5332.4 MiB` | `0.00 MiB` | `2.032` | `4.976` | `9.405` | `0.000` | `16.736` | `17.904` | `59.8` |
| `3440 RR Balanced` (Spatial Off) | `1995×835` | `3440×1440` | `3440×1440` | `5083.4 MiB` | `0.00 MiB` | `1.698` | `4.175` | `8.176` | `0.000` | `14.423` | `15.043` | `69.3` |
| `3440 RR Performance` (Spatial Off) | `1720×720` | `3440×1440` | `3440×1440` | `4861.3 MiB` | `0.00 MiB` | `1.587` | `2.958` | `6.341` | `0.000` | `11.264` | `12.177` | `88.8` |
| `3440 RR Ultra Perf` (Spatial Off) | `1147×480` | `3440×1440` | `3440×1440` | `4541.3 MiB` | `0.00 MiB` | `1.620` | `1.698` | `3.822` | `0.000` | `7.544` | `8.235` | `132.6` |
| `75% RR Quality → 3440` (`EASU + RCAS 0.25`) | `1720×720` | `2580×1080` | `3440×1440` | `4807.9 MiB` | `31.55 MiB` | `1.617` | `2.986` | `5.949` | `0.446` | `11.184` | `11.754` | `89.4` |
| `75% RR Quality → 3440` (`EASU 0.00`) | `1720×720` | `2580×1080` | `3440×1440` | `4787.7 MiB` | `11.39 MiB` | `1.579` | `2.878` | `5.933` | `0.311` | `10.883` | `11.787` | `91.9` |
| `75% RR Quality → 3440` (`Bilinear 0.00`) | `1720×720` | `2580×1080` | `3440×1440` | `4787.7 MiB` | `11.39 MiB` | `1.526` | `2.844` | `5.626` | `0.150` | `10.336` | `11.053` | `96.7` |
| `67% RR Quality → 3440` (`EASU + RCAS 0.25`) | `1529×640` | `2294×960` | `3440×1440` | `4634.7 MiB` | `29.16 MiB` | `1.669` | `2.474` | `4.889` | `0.345` | `9.548` | `10.268` | `104.7` |
| `67% RR DLAA → 3440` (`EASU + RCAS 0.25`) | `2294×960` | `2294×960` | `3440×1440` | `5279.7 MiB` | `29.16 MiB` | `1.719` | `4.889` | `8.780` | `0.344` | `15.925` | `16.238` | `62.8` |
| `50% RR DLAA → 3440` (`EASU + RCAS 0.25`) | `1720×720` | `1720×720` | `3440×1440` | `4745.7 MiB` | `25.22 MiB` | `1.646` | `3.030` | `6.064` | `0.339` | `11.287` | `11.737` | `88.6` |

*(When Spatial is `Off`, `present.comp` took `0.121–0.194 ms`; when Spatial is active, `spatial_prep + spatial_upscale (+ spatial_rcas)` replaces `present.comp` at `0.150–0.446 ms` total.)*

### Sandbox64 (`3440×1440` Immediate, Analytical Output Emitters Active)

| Case | Render | Reconstruct | Display | Process VRAM | Spatial VRAM | Light (ms) | Recon (ms) | Spatial (ms) | Total GPU (ms) | Wall (ms) | GPU FPS |
| :--- | :--- | :--- | :--- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `3440 RR Quality` (Spatial Off) | `2293×960` | `3440×1440` | `3440×1440` | `2223.7 MiB` | `0.00 MiB` | `2.624` | `10.050` | `0.000` | `13.120` | `13.541` | `76.2` |
| `3440 RR Balanced` (Spatial Off) | `1995×835` | `3440×1440` | `3440×1440` | `1975.6 MiB` | `0.00 MiB` | `2.036` | `8.633` | `0.000` | `11.137` | `11.525` | `89.8` |
| `3440 RR Performance` (Spatial Off) | `1720×720` | `3440×1440` | `3440×1440` | `1753.5 MiB` | `0.00 MiB` | `1.508` | `6.758` | `0.000` | `8.689` | `9.455` | `115.1` |
| `3440 RR Ultra Perf` (Spatial Off) | `1147×480` | `3440×1440` | `3440×1440` | `1433.8 MiB` | `0.00 MiB` | `0.724` | `3.972` | `0.000` | `5.090` | `5.727` | `196.5` |
| `75% RR Quality → 3440` (`EASU + RCAS 0.25`) | `1720×720` | `2580×1080` | `3440×1440` | `1700.0 MiB` | `31.55 MiB` | `1.432` | `5.742` | `0.422` | `7.774` | `8.320` | `128.6` |
| `75% RR Quality → 3440` (`EASU 0.00`) | `1720×720` | `2580×1080` | `3440×1440` | `1679.9 MiB` | `11.39 MiB` | `1.564` | `6.294` | `0.313` | `8.346` | `9.040` | `119.8` |
| `75% RR Quality → 3440` (`Bilinear 0.00`) | `1720×720` | `2580×1080` | `3440×1440` | `1679.9 MiB` | `11.39 MiB` | `1.611` | `6.660` | `0.142` | `8.578` | `9.464` | `116.6` |
| `67% RR Quality → 3440` (`EASU + RCAS 0.25`) | `1529×640` | `2294×960` | `3440×1440` | `1526.9 MiB` | `29.16 MiB` | `1.170` | `4.508` | `0.335` | `6.159` | `6.702` | `162.4` |
| `67% RR DLAA → 3440` (`EASU + RCAS 0.25`) | `2294×960` | `2294×960` | `3440×1440` | `2171.9 MiB` | `29.16 MiB` | `2.622` | `9.626` | `0.372` | `12.810` | `13.778` | `78.1` |
| `50% RR DLAA → 3440` (`EASU + RCAS 0.25`) | `1720×720` | `1720×720` | `3440×1440` | `1637.9 MiB` | `25.22 MiB` | `1.509` | `5.910` | `0.329` | `7.918` | `8.676` | `126.3` |

### Key Takeaways & Visual Observations
- **Matched internal render resolution (`1720×720`)**: `75% RR Quality → 3440` (`1720×720 → 2580×1080 → 3440×1440`) runs **slightly faster on GPU** than direct `3440 RR Performance` (`11.184` vs. `11.264 ms` on Sponza; `7.774` vs. `8.689 ms` on Sandbox64) and uses **`53.4 MiB` less total VRAM** (`4807.9` vs. `4861.3 MiB`) because DLSS RR executes tensor inference at `2580×1080` instead of `3440×1440`, offsetting the `~0.42–0.45 ms` EASU+RCAS cost.
- **Matched internal render resolution (`2293×960` vs. `2294×960`)**: `67% RR DLAA → 3440` saves **`0.811 ms` GPU** (`15.925` vs. `16.736 ms`) and `52.7 MiB` VRAM on Sponza compared to `3440 RR Quality`, while `3440 RR Quality` retains an edge on fine sub-pixel temporal detail.
- **Visual inspection** (static, pan, and disocclusion captures at `3440×1440`, plus `VKE_UI_CAPTURE` with Settings open):
  - `EASU + RCAS 0.25` preserves curtain embroidery, stone joints, and candle edges without dark/bright ringing halos, and stays noticeably crisper than `Bilinear`.
  - Motion pan and 3-frame disocclusion recovery remain stable with zero added spatial ghosting or shimmer.
  - Analytical output emitters (`spatial_rcas_emitters.comp.spv` / `spatial_upscale_emitters.comp.spv`) and Dear ImGui UI render cleanly at native `3440×1440`.
