# Phase 5 Chunk 1: Material Ownership

Implemented on `phase-5`, 2026-10-28. Phase 5 remains in progress. This chunk adds material ownership within the existing single-visible-model renderer; it does not add instances, scene-graph transforms, alpha handling, emissive lighting, ReSTIR or denoising.

## Asset and shader contracts

- `src/scene/meshlet_payload.hpp` is the shared CPU format/validator. MLOD v2 retains the 288-byte header and geometry arrays: 16-byte meshlets, uint32 vertex references, byte triangle references and 48-byte bounds. Header word 6 holds the surface count; word 7 is reserved. The geometry is followed by 16-byte source-surface records `(mesh, primitive, material, flags)` and one uint32 surface ID per meshlet. The existing uint64 payload-size / `MLOD` footer remains. Scalars use little-endian encoding on the supported Windows target.
- Surface IDs follow source mesh/primitive order and remain unchanged across all six LODs. A source material of `UINT32_MAX` means the glTF default material, appended after the declared material slots. Static flag 1 preserves that primitive's original geometry at every LOD.
- `meshlet_builder.hpp` welds/simplifies within each primitive, retains border locking and UV attributes, and clusters each primitive independently. Original split vertices are retained at LOD 0 and for static geometry. Each model LOD error is the maximum cumulative primitive error.
- The loader validates trailer placement, lengths/counts, versions, LOD ranges/errors, bounds, vertex/triangle references and source-surface/material references before allocating geometry. Legacy MLOD v1 and MESH v1 synthesize surface 0 only when every referenced primitive resolves to one material. Their primitive identity is coarse and reported as such. Ambiguous multi-material legacy payloads are rejected with a rebuild instruction.
- Model set 0, binding 5 is an immutable scalar-layout SSBO of 8-byte `(surface ID, global material ID)` entries, one per meshlet. Asset registration remaps local material slots once and uploads the buffer; model destruction releases it. Deferred shading uses this lookup instead of the first submesh's material.
- Visibility remains `R32G32_UINT (meshlet, triangle)`. Push constants remain 256 bytes, including the reserved old material slot. The frame constants remain 704 bytes.
- Current/previous temporal identity images now use `R32G32B32A32_UINT (entity, surface, raster LOD, global material)`. All four components must match during history rejection, in addition to the existing projection/depth/normal checks. Guide generation remains demand-driven.

## Compiler usage and compatibility

Explicit glTF/GLB inputs receive no generated ground by default. Add `--demo-ground` to request the un-decimated ground primitive. Directory-mode compilation retains ground for `boulder_01_1k.gltf`; existing outputs are still skipped when up to date. Use separate input/output paths; in-place compilation is rejected to avoid stale trailers.

The existing default-material OBJ conversion remains supported; material-bearing OBJ requires conversion to glTF instead of silently discarding its materials. General glTF node transforms/instancing, alpha masking/blending and ray-hit material evaluation remain outside this chunk. The checked-in boulder and its textures were not regenerated or migrated; they retain the supported legacy payload.

## CPU/build verification

Exact configuration:

```text
cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug -DVKE_BUILD_TESTS=ON -DVKE_ENABLE_DLSS=ON
```

Toolchain: MinGW, Vulkan SDK 1.4.350.0, pinned Streamline 2.14.16. Before renderer changes, `temporal_math`, `hiz_history` and `reconstruction` were built and all three passed.

After implementation:

```text
cmake --build build --target VK_game_engine model_compiler meshlet_surface_tests temporal_math_tests hiz_history_tests reconstruction_tests -j8
ctest --test-dir build --output-on-failure
```

The engine/model compiler and affected shaders compiled successfully. All six CTest entries passed:

| Test | Coverage |
| :--- | :--- |
| `temporal_math` | Existing temporal CPU contracts. |
| `hiz_history` | Existing independent depth-history CPU contracts. |
| `reconstruction` | Existing settings, guide-selection, jitter and reconstruction CPU contracts. |
| `meshlet_surfaces` | Coincident geometry/UVs with different materials, deterministic six-LOD ownership, real simplification, static geometry/split vertices, maximum primitive error, default/global material slots, v2 round trips, legacy synthesis/rejection, malformed payloads and temporal identity components. |
| `legacy_boulder_payload` | CPU parsing/validation of the actual unchanged boulder GLB, all six LODs and synthesized ownership. |
| `compiler_surface_fixtures` | Offline compiler to GLB to CPU decoder, indexed/nonindexed and missing-material primitives across two meshes, with/without explicit static ground; default-material OBJ/interleaved input; rejection of primitive-local index overflow and in-place compilation without modifying the input. |

Generated fixtures are confined to ignored `build/surface-fixtures/`. Configure/build logs use `build/phase5-chunk1-*.log`; CTest records results under `build/Testing/Temporary/`. Dependency warnings remain non-fatal. These checks did not launch the engine or exercise Vulkan on a GPU.

## Remaining verification and next chunk

The Phase 4.1 resource-reuse and Native fast/full-guide image-parity assertions remain GPU-unverified. No current-loader timing baseline was recorded. The new ownership SSBO, deferred material selection, identity formats and Vulkan synchronization still need GPU/image validation; earlier boulder GPU results do not establish correctness of these changes. No performance or image-quality improvement is claimed.

The next implementation chunk is multiple opaque instances with per-instance surface access, transforms, motion/history, culling and TLAS identity. Emissive sampling/reference lighting and staged ReSTIR follow. DLSS RR remains Phase 5.1; custom SVGF remains deferred.

## Follow-up live verification (2026-10-28)

The user subsequently requested a live launch. The boulder rendered correctly in a fixed 1280x720 Native window on the RTX 3090. The first session then exposed an ImGui assertion: an out-of-date acquire discarded the frame without ending its UI frame. The acquire-out-of-date and close-before-draw paths now end the ImGui frame before returning.

After rebuilding, the existing fixed-window Reflex smoke passed all 15 Native/DLAA/Quality/Balanced/Performance and Reflex Off/On/On + Boost combinations under synchronization validation, including scripted camera motion and Hi-Z toggles. Two additional fixed-window Native/DLAA runs each forced one acquire-out-of-date event, enabled all temporal guides, recovered successfully and completed 240 submitted frames; DLAA performed 240 real evaluations. No Vulkan validation/synchronization errors or assertions were found in those three test logs. All six CPU tests passed again, and tracked assets remained unchanged.

Test-build-only flags `VKE_TEST_ACQUIRE_OUT_OF_DATE=1` and `VKE_TEST_FULL_GUIDES=1` exercise the discarded-frame path and complete guide writes. They are absent from ordinary startup. Logs: `build/phase5-chunk1-fixed-smoke.log` and `build/phase5-full-guides-mode{0,1}.log`; the initial failing session is retained in `build/phase5-chunk1-live.log`. These replace the earlier no-GPU-testing status only for the stated boulder coverage. Tests were unfocused, so their timing output is not a controlled performance/latency baseline. Exact Native fast/full-guide image parity, image readback/resource-reuse assertions, GPU rendering of multi-material v2 assets, broader hardware and extended visual review remain pending.
