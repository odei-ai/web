# World Model Filters v2.5.4 - Implementation Notes

**Date:** 2026-01-29
**Scope:** Atlas (World Model) UI -> filter state -> renderer visibility -> optional backend params

## Target Files / Paths

### UI (Atlas Panel)
- `src/index.html` - Atlas panel markup; add Domains/Loop/Status/Integrity/Advanced sections
- `src/input.css` - primary styles for new filter groups (Tailwind source)
- `src/styles.css` - compiled CSS output (keep in sync with `src/input.css`)

### Filter State + Persistence
- `src/modules/MemoryViewManager.js` - main filter wiring, UI handlers, persistence hooks
- `src/modules/memory/memoryUtils.js` - filter constants/defaults (domains/layers/presets)
- **NEW:** `src/modules/worldmodel/filters/architecture.ts` - shared types + colors + status mapping
- **NEW:** `src/modules/worldmodel/filters/filterState.ts` - filter model + defaults + migration + persistence
- **NEW:** `src/modules/worldmodel/filters/predicate.ts` - core filter predicate helpers
- **NEW:** `src/modules/worldmodel/filters/useWorldModelFilters.ts` - thin helper wrapper for filter persistence

### Normalization + Graph Stats
- `src/Renderer/MonumentalGraph/utils/normalization.js` - compute domain/loop/status/typeNormalized/etc
- `src/Renderer/MonumentalGraph/index.js` - apply filters as visibility only; maintain layout
- `src/modules/MonumentalGraphRenderer.js` - expose filter stats + filtered ids to UI

### Integrity
- **NEW:** `src/modules/worldmodel/integrity/computeIntegrity.ts` - guard computation + counts

### Optional backend alignment (not modified in this pass)
- `servers/odei-neo4j/src/tools/hybridSearch.ts`
- `servers/odei-neo4j/src/tools/hybridPlusPlus.ts`

### Tests + Docs
- `docs/WORLD_MODEL_FILTERS_V2_5_4.md` - new documentation
- `tests/unit/*` - new unit tests for mapping + filters + persistence

## Notes
- Avoid ForceGraph3D (keep MonumentalGraph / Three.js pipeline).
- Filtering must not re-run layout; visibility-only with budgets.
- Use cached stats for counts: domains, loop phases, status groups, layers, types, integrity.
