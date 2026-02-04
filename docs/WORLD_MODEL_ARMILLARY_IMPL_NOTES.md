# World Model Armillary Implementation Notes

Files to modify / add for Armillary layout integration:
- `src/Renderer/MonumentalGraph/index.js` (layout mode switch, armillary integration, focus/zoom behavior)
- `src/Renderer/MonumentalGraph/core/LayoutEngine.js` (kept for non‑armillary layouts)
- `src/Renderer/MonumentalGraph/renderers/SceneManager.js` (planet center/radius sync, atmosphere tuning hooks)
- `src/Renderer/MonumentalGraph/renderers/NodeRenderer.js` (AI emphasis + confidence halo hooks)
- `src/Renderer/MonumentalGraph/renderers/EdgeRenderer.js` (journey routing points)
- `src/Renderer/MonumentalGraph/worldmodel/config/WorldModelTuning.js` (new)
- `src/Renderer/MonumentalGraph/worldmodel/layout/armillaryLayout.js` (new)
- `src/Renderer/MonumentalGraph/worldmodel/layout/poseSmoother.js` (new)
- `src/Renderer/MonumentalGraph/worldmodel/render/armillaryGuides.js` (new)
- `src/Renderer/MonumentalGraph/worldmodel/render/transitRouting.js` (new)

Integration points to validate:
- World Model view uses MonumentalGraph (not ForceGraph3D).
- Deterministic armillary layout is used only for World Model.
- Filtering does not re‑layout; only visibility changes.
- Journey mode edges use routed paths (optional).
