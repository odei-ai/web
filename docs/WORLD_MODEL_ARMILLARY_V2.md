# World Model Armillary V2

## Domain Bands (Latitude)
Normalized Y positions and thickness (see `WorldModelTuning.js`):
- destination: y=+0.75, thickness=0.05
- path:        y=+0.25, thickness=0.28
- state:       y=-0.05, thickness=0.12
- reality:     y=-0.45, thickness=0.22
- evidence:    y=-0.70, thickness=0.14
- policy:      y=+0.05, thickness=0.08

## District Allocation
- Nodes are grouped into districts (project/area/container/etc).
- District weights are computed from node sizes.
- Angular spans are allocated proportionally to sqrt(weight), with clamps.
- `__goal__` and `__self__` reserved sectors can be enabled.

## Armillary Layout Rules
- Graph radius: `Rg = clamp(360 + 0.75 * sqrt(N) * 75, 360, 1000)`
- Planet radius: `planet = Rg * 1.35`
- Node Y = bandY * Rg + deterministic jitter
- Radial distance: derived from latitude radius, layer factor, salience, and jitter
- Theta: deterministic per node within its district sector
- Anchors:
  - destination: x=z=0, high Y
  - state/self: x=z=0, state band center
  - policy: r=0.95*Rg, y=policy band
  - evidence: r in [0.35..0.65]*Rg

## Policy Ring
The policy ring is a dashed arc at y=policy band and r=0.95*Rg.
Districts with blocked nodes can glow their arc segment (optional).

## Transit Routing (Journey / Focus)
When journey corridors are active, edges route through:
node → hub(district,domain) → rail(domain) → arc(thetaA→thetaB) → rail → hub → node.
Routing is cached per layout version.

## Tuning
All constants live in:
`src/Renderer/MonumentalGraph/worldmodel/config/WorldModelTuning.js`

Tune without breaking stability:
- domain band Y/Thickness
- radius formula and clamp
- district min/max spans
- smoothing tau (pose damp)
- guide opacities

## Manual Test Checklist
- 50 nodes: bands and districts are visible, planet encloses graph, no jitter.
- 500 nodes: budgets respected, FPS stable, guides stay subtle.
- Filters: visibility changes without layout shifts or teleports.
- Journey: corridor edges route along rails; base edges remain faint.
- Policy: blocked nodes illuminate their district arc only.
- Zoom/fit: `zoomToFit` centers the armillary world. 
