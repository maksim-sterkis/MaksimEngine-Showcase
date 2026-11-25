# Two-room Lighting Sandbox capture settings

These Phase 5.1 images are the direct-lighting baselines for the [Phase 6 ReSTIR GI comparison](../roadmap.md#phase-6-restir-gi).

## Camera views

Positions and look-at targets are world-space `(x, y, z)` in meters. Use Y up, zero roll, and the engine's 45° vertical field of view. Render each view at 2100 × 1400 before applying the specified horizontal center crop. Keep all 1400 rows.

| Baseline image | Camera position | Look-at target | Sun | PNG size |
| --- | --- | --- | --- | --- |
| [01 · Interior area lights](01-interior-area-lights.png) | `(0, 1.72, 6.85)` | `(0, 1.58, -0.5)` | Off | 1920 × 1400 |
| [02 · Overhead courtyard](02-courtyard-overhead-sun-off.png) | `(9.06, 12.8, 0.62)` | `(9.06, 0, 0.5)` | Off | 1400 × 1400 |
| [03 · Sunlit courtyard](03-courtyard-sun-on.png) | `(13.3, 3.35, 5.4)` | `(8.8, 0.83, 0.23)` | On, azimuth 75°, elevation 23° | 2100 × 1400 |
| [04 · Sun through the doorway](04-interior-sun-through-door.png) | `(2.4, 1.74, 6.5)` | `(-0.58, 1.52, -0.7)` | On, azimuth 86°, elevation 5.5° | 2100 × 1400 |

## Shared render settings

- Select **Lighting Sandbox (two rooms)** with three static area emitters at the default emission intensity of 1.0. Keep object and light animation off.
- Lock raster and ray-tracing geometry to **LOD 0**. Keep ray-traced shadows on and Hi-Z off. Use ReSTIR DI with four RIS candidates. The sun intensity is 2.5 when enabled.
- Use **DLAA Ray Reconstruction** (DLSS 4.5 model F), at native resolution with NRD off. Reset temporal history at each camera cut and capture after 95 successful RR evaluations at the new view.
- Read back the resolved HDR image without UI and use the engine's ACES tone curve and 2.2 display gamma for the PNGs.

For Phase 6, capture each view with ReSTIR GI off and on in the same build using these settings. Compare each pair side by side for indirect lighting and color bleeding, and keep the four Phase 5.1 PNGs as the historical reference.
