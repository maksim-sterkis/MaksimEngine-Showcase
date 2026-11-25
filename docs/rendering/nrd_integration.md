# NVIDIA Real-Time Denoisers (NRD): RELAX, REBLUR, and SIGMA

## Overview & Mutual Exclusion Policy

Phase 5.1 Task 2 integrates **NVIDIA Real-Time Denoisers (NRD `v4.18.0`)** directly on Vulkan without NRI, providing analytical spatio-temporal denoising across non-Ray-Reconstruction paths:

- **NRD RELAX** (`RELAX_DIFFUSE_SPECULAR`): Primary denoiser for low-spp emissive direct lighting and ReSTIR DI, compatible with `Native`, `DLAA`, and `DLSS Quality`/`Balanced`/`Performance`. ReSTIR final shading averages four fresh, visibility-tested RIS estimates. These draws stay out of reservoir history. The diffuse hit-distance guide uses the nearest sampled emitter, including when a blocker stops the visibility ray.
- **NRD REBLUR** (`REBLUR_DIFFUSE_SPECULAR`): Alternative hit-distance- and luminance-aware diffuse/specular denoiser for the emissive direct-lighting pipeline.
- **NRD SIGMA** (`SIGMA_SHADOW`): Ray-traced directional soft-shadow penumbra denoiser active on the non-sandbox Boulder scene (`sigmaSunAngularRadiusDeg = 1.5°` default).

**Mutual exclusion with DLSS Ray Reconstruction**: In `src/core/denoiser_settings.hpp` (`denoiser::resolve_effective`), selecting any DLSS Ray Reconstruction mode (`RayReconstructionDLAA`..`RayReconstructionUltraPerformance`) forces effective NRD to `Off` so lighting is never double-denoised. Persistent settings store `denoiserMode` separately from `effective`, freeing all NRD I/O images and permanent/transient pools (`0 MiB`) whenever NRD becomes `Off` (or when switching back from NRD or RR). `VKE_DENOISER_MODE=0..3` overrides the initial denoiser mode (`Off`, `RELAX`, `REBLUR`, `SIGMA`).

## Build & Native Vulkan Backend

- **Pinned third-party sources (`cmake/NRD.cmake`)**: Downloads and SHA-256-verifies `NVIDIA-RTX/NRD v4.18.0`, `MathLib v11`, and `ShaderMake 3f01623a` into `.cache/nrd/` (no modified upstream sources; MinGW GCC 16 `<cmath>` ambiguity resolved via compiler `-include nrd_mingw_fix.hpp` for `using std::abs;`).
- **SPIR-V embedding**: Compiles all `159` NRD compute shaders at build time via `ShaderMake` + Vulkan SDK `dxc.exe` with `NRD_NORMAL_ENCODING=2` (`VK_FORMAT_A2B10G10R10_UNORM_PACK32`, dedicated binding `31`), `NRD_ROUGHNESS_ENCODING=1` (linear roughness), `NRD_SUPPORTS_QUAD_INTRINSICS=0`, and `NRD_STATIC_LIBRARY=1`, linking `libNRD.a` statically with zero external runtime DLL dependencies and staging MIT notices to `nvidia-notices/nrd-license.txt`.
- **Demodulation & Composition (`src/renderer/nrd_denoiser.cpp`, `shaders/nrd_common.glsl`, `shaders/nrd_compose.comp`)**:
  - `deferred.comp` and `restir_final.glsl` write demodulated diffuse and specular radiance (`R16G16B16A16_SFLOAT`) plus linear view-space depth (`viewZ`), unjittered 2.5D motion (`motion`), packed `R10G10B10A2` world normal + linear roughness, and SIGMA shadow data / penumbra inputs while leaving ReSTIR reservoirs untouched.
  - NRD dispatches execute inside the `gpu_timing::denoiserBegin..denoiserEnd` GPU timestamp bracket, tracking per-image layout/access barriers across shared internal resources.
  - `nrd_compose.comp` remodulates denoised diffuse and specular irradiance by surface diffuse/specular albedo plus emissive radiance (or composites denoised SIGMA sun visibility against hard direct lighting) into `temporal.color` before ordinary `DLAA`/`DLSS SR` or tone mapping.
- **Diagnostics**: F5 exposes `Real-Time Denoiser (NRD)` controls (active/requested/effective mode, pipeline/VRAM telemetry, anti-firefly, history frame cap, SIGMA sun angular radius) and `Temporal view` indices `10..15` (`NRD demod diffuse`, `NRD demod specular`, `NRD normal/roughness`, `NRD viewZ/hitDist`, `NRD shadow data/result`, `NRD validation overlay`).

## Verification & RTX 3090 Measured Results

- **CPU & Shader Verification**: `12/12` `ctest` suites pass (including `denoiser` unit tests); all engine and NRD SPIR-V compute/graphics shaders pass `spirv-val --target-env vulkan1.3`.
- **Vulkan & Sync Validation (`VKE_NRD_SMOKE=1`)**: Passed under `VKE_FORCE_VALIDATION=1` and `VK_LAYER_VALIDATE_SYNC=1` with **zero engine or NRD validation/synchronization errors** across `NRD RELAX` (`1280×720`, debug views `10..15` + validation overlay), `NRD RELAX + DLSS Quality` (`853×480`), `NRD REBLUR` (`1280×720`), `RR DLAA` mutual-exclusion (`0 MiB` NRD), and Boulder `NRD SIGMA` (`1280×720`) + switchback cleanup (`0 MiB` NRD).
- **NRD Validation Overlay (`OUT_VALIDATION`)**: Visually inspected in `build/phase51-nrd-audit/static_validation_relax.png`: clean `UNITS` grid, `ROUGHNESS-1`, `Z` (`>0` surface / `INF` sky), `MV` (`0` static error, no jittered-MV artifacts), `NORMALS-2`, and full violet temporal convergence.

### 1. Static Quality vs. 1,024-spp Direct Reference (`3440×1440`, `VKE_NRD_AUDIT=1`)

The benchmark tables below use the original single-estimate RELAX path.

Measured on `Sandbox` (64 emitters and 3 emitters, ReSTIR DI 4 candidates, 64 settled frames) and `Boulder` (`SIGMA_SHADOW` `1.5°` sun radius compared against hard-RT baseline):

| Scene | Configuration | Internal Res | RGB RMSE | Lum RMSE | PSNR (dB) | Mean Lum Ratio |
| :--- | :--- | ---: | ---: | ---: | ---: | ---: |
| Sandbox 64 | Raw ReSTIR (`Off` + `Native`) | 3440×1440 | 0.04209 | 0.01890 | 21.66 | 1.0002 |
| Sandbox 64 | **NRD RELAX** + `Native` | 3440×1440 | **0.00517** | **0.00439** | **34.34** | 0.9955 |
| Sandbox 64 | **NRD RELAX** + `DLAA` | 3440×1440 | 0.00528 | 0.00444 | 34.24 | 0.9932 |
| Sandbox 64 | **NRD RELAX** + `DLSS Quality` | 2293×960 | 0.00530 | 0.00445 | 34.22 | 0.9934 |
| Sandbox 64 | **NRD REBLUR** + `Native` | 3440×1440 | 0.00552 | 0.00472 | 33.72 | 0.9871 |
| Sandbox 64 | `Off` + `DLSS RR DLAA` | 3440×1440 | 0.00521 | 0.00447 | 34.18 | 1.0086 |
| Sandbox 3 | Raw ReSTIR (`Off` + `Native`) | 3440×1440 | 0.03835 | 0.01955 | 22.92 | 0.9994 |
| Sandbox 3 | **NRD RELAX** + `Native` | 3440×1440 | **0.00329** | **0.00233** | **41.40** | 0.9949 |
| Sandbox 3 | **NRD REBLUR** + `Native` | 3440×1440 | 0.00350 | 0.00268 | 40.19 | 0.9912 |
| Boulder | **NRD SIGMA** + `Native` *(vs. Hard RT)* | 3440×1440 | 0.00092 | 0.00089 | 49.08 | 0.9996 |

### 2. Dynamic-Scene Quality (`Sandbox64`, `3440×1440`, `VKE_NRD_AUDIT=1`)

Replays the 64-frame camera + moving-cube + cube-snap (`frame 32`, `474,553` disoccluded px) + moving-emitter (`frames 32–47`) trajectory against 1,024-spp references:

| Mode | History Resets | Whole `f=24` lumRMSE / PSNR | Snap Disoccluded `f=32` lumRMSE / PSNR | Emitter-Move `f=40` whole lumRMSE / PSNR | Settled `f=63` whole lumRMSE / PSNR |
| :--- | ---: | :--- | :--- | :--- | :--- |
| **NRD RELAX** (`Native`) | **0** | 0.00488 / 33.07 dB | **0.00378 / 31.82 dB** | 0.00738 / 29.69 dB | 0.00504 / 32.93 dB |
| **NRD REBLUR** (`Native`) | **0** | 0.00530 / 32.36 dB | 0.00688 / 26.63 dB | 0.00889 / 28.07 dB | 0.00561 / 32.00 dB |
| `DLSS RR DLAA` (`Off`) | **0** | 0.00482 / 33.18 dB | 0.00710 / 26.36 dB | **0.00485 / 33.33 dB** | **0.00466 / 33.60 dB** |

### 3. Visible `3440×1440` GPU Timing (`VKE_NRD_BENCH=1`, 90-frame Medians)

| Scene | Mode | Internal Res | GPU Frame (ms) | Lighting (ms) | Denoiser NRD (ms) | Recon / SDK (ms) |
| :--- | :--- | ---: | ---: | ---: | ---: | ---: |
| Sandbox 64 | `Off` + `Native` | 3440×1440 | 3.341 | 2.998 | 0.000 | 0.001 |
| Sandbox 64 | **NRD RELAX** + `Native` | 3440×1440 | **9.969** | 3.889 | 5.188 | 0.001 |
| Sandbox 64 | **NRD RELAX** + `DLAA` | 3440×1440 | 11.661 | 3.848 | 5.532 | 1.447 |
| Sandbox 64 | **NRD RELAX** + `DLSS Quality` | 2293×960 | **6.102** | 1.745 | **2.441** | 1.370 |
| Sandbox 64 | **NRD REBLUR** + `Native` | 3440×1440 | **8.564** | 4.130 | **4.227** | 0.001 |
| Sandbox 64 | `Off` + `RR DLAA` | 3440×1440 | 24.917 | 5.800 | 0.000 | 17.981 |
| Sandbox 64 | `Off` + `RR Quality` | 2293×960 | 11.778 | 2.419 | 0.000 | 8.660 |
| Boulder | `Off` + `Native` *(Hard RT)* | 3440×1440 | 0.732 | 0.486 | 0.000 | 0.000 |
| Boulder | **NRD SIGMA** + `Native` | 3440×1440 | **1.309** | 0.593 | **0.411** | 0.001 |
| Boulder | **NRD SIGMA** + `DLAA` | 3440×1440 | 3.153 | 0.635 | **0.434** | 1.371 |

### 4. Process VRAM Footprint (`3440×1440`, `VK_EXT_memory_budget`, `VKE_NRD_MEMORY_AUDIT=1`)

| Case | Effective NRD / Recon | Process VRAM (MiB) | NRD IO (MiB) | NRD Pools (MiB) | NRD Total (MiB) | RR Guides (MiB) |
| :--- | :--- | ---: | ---: | ---: | ---: | ---: |
| `off_native` | `Off` / `Native` | 2004.73 | 0.00 | 0.00 | 0.00 | 0.00 |
| `relax_native` | `NRD RELAX` / `Native` | 2750.92 | 342.66 | 403.34 | 746.00 | 0.00 |
| `relax_dlaa` | `NRD RELAX` / `DLAA` | 3077.36 | 342.66 | 403.34 | 746.00 | 0.00 |
| `relax_quality` | `NRD RELAX` / `DLSS Quality` *(2293×960)* | **1784.04** | 152.56 | 179.51 | 332.07 | 0.00 |
| `reblur_native` | `NRD REBLUR` / `Native` | 2838.04 | 342.66 | 353.38 | 696.03 | 0.00 |
| `sigma_boulder_native` | `NRD SIGMA` / `Native` *(Boulder)* | 2419.54 | 201.66 | 75.91 | 277.57 | 0.00 |
| `rr_dlaa_exclusivity` | `Off` / `RR DLAA` | 3649.61 | 0.00 | 0.00 | **0.00** | 282.19 |
| `switchback_off_native` | `Off` / `Native` | 2572.98 | 0.00 | 0.00 | **0.00** | **0.00** |

## Reproduction Commands (Release Test Build `build/nsight-release`)

- Smoke + forced Vulkan sync validation: `VKE_FORCE_VALIDATION=1 VK_LAYER_VALIDATE_SYNC=1 VKE_NRD_SMOKE=1 VKE_FIXED_PRESENT_PROBE=1 VKE_PROBE_HIDDEN=1 ./build/nsight-release/VK_game_engine.exe`
- Static/dynamic/SIGMA quality audit: `VKE_NRD_AUDIT=1 VKE_FIXED_PRESENT_PROBE=1 VKE_PROBE_HIDDEN=1 VKE_PROBE_BORDERLESS=1 ./build/nsight-release/VK_game_engine.exe`
- Memory audit: `VKE_NRD_MEMORY_AUDIT=1 VKE_FIXED_PRESENT_PROBE=1 VKE_PROBE_HIDDEN=1 VKE_PROBE_BORDERLESS=1 ./build/nsight-release/VK_game_engine.exe`
- Visible `3440×1440` GPU timing benchmark: `VKE_NRD_BENCH=1 VKE_FIXED_PRESENT_PROBE=1 VKE_PROBE_BORDERLESS=1 ./build/nsight-release/VK_game_engine.exe`
