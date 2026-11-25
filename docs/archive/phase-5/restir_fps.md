# ReSTIR FPS checkpoint — 2026-10-30

This is the historical checkpoint. See the [2026-10-31 resume](restir_fps_resume.md) for the current compact/fused paths, Release build and approximately 200 FPS starting-view result, with lower interior results.

The approximately 200 FPS DLAA target at 3440x1440 remains unmet. No GI, denoiser, RR, SVGF or outlier filter was added. Work remains uncommitted on `phase-5`.

## Changes

- Separately compiled **Basic (real time)** and **Ray traced (reference)** temporal/spatial correction kernels. Basic uses unoccluded source targets and retains invisible samples in history; both trace fresh final shadows. F5 labels the accuracy/performance policy, and changes reset incompatible history.
- Basic defaults to one bounded spatial proposal, with up to eight on temporal rejection. Candidate count remains eight; history limit remains twenty.
- A dedicated full-DI material/initial specialization excludes directional/legacy lighting and legacy reservoir access.
- Legacy initial-only storage now uses two initialized 48-byte stubs unless that diagnostic mode is active: **475,545,504 bytes** less allocation at ultrawide. Mode transitions retire users and update actual descriptor ranges.
- Visible full-DI records avoid a redundant 160-byte clear; invalid/background records still clear. Unused legacy per-pixel clears are removed.
- Spatial reads temporal scratch and overwrites the consumed initial array. Post-submission ownership rotation preserves temporal-only history without the **317 MB GPU copy** (about 634 MB read/write traffic) each frame. Recursive spatial feedback remains open work.

## Measurements

RTX 3090, existing MinGW Debug/pinned DLAA, synchronization validation enabled, **3440x1440 output and internal resolution**, locked interior camera, three emitters, fresh shadows, F4/F5 drawn. Each case: 60 warmup + 180 measured frames, 180/180 focused frames and 180 GPU samples. No pixel readbacks or diagnostic atomics.

| Mode | Loop FPS | GPU frame | Lighting | Spatial |
| --- | ---: | ---: | ---: | ---: |
| Ray traced / four neighbors | 49.31 | 19.00 ms | 17.31 ms | 12.99 ms |
| Basic / four proposals | 49.43 | 18.68 ms | 16.98 ms | 12.69 ms |
| Basic / one proposal | 89.40 | 10.19 ms | 8.47 ms | 4.16 ms |
| MC8 comparison | 165.92 | 4.95 ms | 3.22 ms | — |
| Basic / one repeat | 88.79 | 10.20 ms | 8.47 ms | 4.16 ms |

Initial/temporal/final passes are about 1.75/1.82/0.74 ms; DLAA is about 1.35 ms. Log: `build/restir-fps-3440.log`. These are matched policy/work comparisons, not equal-noise evidence or minimum-FPS guarantees. Removing correction rays alone did little; general spatial target/data work remains the main bottleneck. A scalar one-neighbor kernel, persistent-data packing and target evaluation profiling are next throughput work.

## Verification

All eleven CPU/offline suites pass, including Basic normalization cases with changing visibility/support, retained invisible samples, rejection, unequal counts and capped history. Four affected SPIR-V modules validate. The focused full-resolution timing run and hidden Basic GPU suite report no Vulkan validation errors. The latter passes the full 4096-spp/512-frame-per-method reference matrix, reuse, motion, light response, modes, Hi-Z, topology, cuts, LOD, area and resize cases. Logs: `build/restir-fps-cpu.log`, `build/restir-fps-basic-reference.log`.

The legacy initial-only GPU allocation-transition regression and many-emitter/full-resolution temporal-quality checks remain pending at this checkpoint. Tracked assets are unchanged. No commits, pushes, PowerShell scripts or documentation sync occurred. The 5 ms total frame target still requires substantial work.
