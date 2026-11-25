# Future Engine Work

## Force LOD 0 Default & Dynamic LOD Temporal Stability (Deferred to Phase 6.1)

**Status: Active runtime/testing policy; dynamic LOD fix deferred to Phase 6.1.**

- **Testing rule**: **Always use Force LOD 0 (`manual_lod_override = 0`, hotkey `1` or `lod = 0` in probes) for all testing and benchmarking.**
- **Engine default**: The default engine configuration sets `manual_lod_override = 0` (Force LOD 0) at startup instead of automatic screen-space LOD (`-1`). Automatic LOD (`0` key) and explicit LOD overrides (`1`–`6`) remain available for manual inspection and Hi-Z/LOD tests.
- **Why**: Moving through complex multi-instance scenes such as Intel New Sponza triggers frequent per-instance LOD transitions. Because LOD switches alter meshlet topology (`identity.z`), swap RT BLAS levels (`work.x`), and mutate `geometryKey`, they invalidate temporal surface identities and reset DLSS Ray Reconstruction, NRD, and ReSTIR DI history—causing severe flashing, popping, and re-convergence artifacts during camera movement.
- **Planned resolution (Phase 6.1)**: Stabilize dynamic LOD transitions across raster meshlets, RT BLAS representation, and temporal reprojection/history retention (RR, NRD, and ReSTIR) in **Phase 6.1**.

## Win32 Mailbox 200 FPS cap

**Open.** Win32 Mailbox presentation (`VK_FULL_SCREEN_EXCLUSIVE_DISALLOWED_EXT` + pre-present fence sync) passes `VK_EXT_present_timing` audits with zero VBlank stalls, while a 200 FPS cap observed under some Win32 Mailbox configurations remains deferred for future investigation outside Phase 5.1.

Reproduce the ceiling with a light, focused workload. Record the build, driver, window mode, display refresh, reconstruction and Reflex settings, requested and effective present modes, application FPS, and displayed cadence. Compare Mailbox, Immediate, and FIFO under matched conditions and use frame timings to locate the wait. Verify any fix preserves the signed-off Mailbox behavior and live present-mode switching.

[Phase 5.1 pacing verification](../roadmap.md#win32-mailbox-presentation-pacing) and [historical 200 FPS investigation](../rendering/dlss_integration.md#historical-canonical-name-200-fps-investigation).

## Clean-Machine & Multi-GPU/Driver Verification

**Open.** The `v0.3.1` portable distribution (`dist/MaksimEngine-v0.3.1-Portable.zip`) and all Phase 5.1 tests were built and verified locally on a single **NVIDIA GeForce RTX 3090** (`driver 621.14`, `3440×1440` display) Windows 11 development machine.

- Verify `v0.3.1` (and subsequent releases) on a clean non-development Windows machine (confirming zero missing runtime DLLs, explicit RR-unavailable error reporting, and Native + NRD `RELAX`/`REBLUR` behavior).
- Test across additional GPU architectures and drivers (NVIDIA Turing/Ada/Blackwell, AMD RDNA, Intel Arc) and higher display resolutions such as 4K (`3840×2160`).
