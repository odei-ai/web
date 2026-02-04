# Health Module — Body (Recovery) Sub-View UI & Data Spec

> **Note:** This document describes the "Body" (Recovery) sub-view within the Health agent module.
> The Health agent has 4 sub-pages: Body (Recovery), Mind, Activity, Nutrition.

## Purpose

- The Body (Recovery) screen is a live organism status panel that the AI layer reads and that controls load and risk allowances.
- Strictly Stage 1: Apple Watch/HealthKit as primary source, no simulations. Secondary sensors will be connected in future.
- All UI follows the premium dark theme (`docs/premium-dark-theme.md`, tokens in `src/input.css`/`src/health-dashboard.css`).

## Key Screen Tasks

- **Morning**: Capture sleep/HRV/RHR → calculate Readiness/Capacity → provide recommendations and limits.
- **Day**: Monitor heart rate/stress/activity → catch spikes → suggest protocols.
- **Evening**: Recap load/stress/focus → correlate with results (Work/Finance) → update baselines.
- **Anytime**: Display status (OK / Soft Work / No-Risk) and active restrictions.

## Main UI Surfaces

- **Hero**: Readiness/Capacity Score (0–100), mode status (Normal/Soft Work/No-Risk), date, brief recommendations (what's allowed/forbidden).
- **Sleep**: Duration, efficiency, deep/REM, sleep onset/wake time, sleep variability 7/30d, Debt vs Baseline.
- **HR/HRV**: Current HR, RHR, HRV (rMSSD/ln), baseline vs today, load zones, stress spikes.
- **Activity**: Steps, active calories, workouts (type/zone/duration/energy), standing/sitting time, ring progress.
- **Stress/Events**: Stress spikes, tilt (high HR + prolonged sitting), loud noise, low movement.
- **Environment (optional)**: CO₂, temperature, noise, recommendations (ventilate/ears/break).
- **Metabolic (optional)**: Glucose (stability, spikes), nutrition (manual tags: coffee, alcohol, late dinner).
- **Subjective**: Quick 1–5 scales (energy/focus/mood), short note.
- **Day Timeline**: Sleep → morning check → work/workouts → stress spikes → evening recap; correlation markers.
- **Alert stream**: Event cards (HRV < baseline, HR spike, sleep <7h, sitting >60m, glucose spike) with protocol CTA.
- **Protocols/Actions**: Buttons for "Reduce risk -30%", "4-7-8 Breathing (10m)", "15m walk", "Light stretch", "Postpone heavy tasks".

## States and Allowances

- **Normal**: All limits standard.
- **Soft Work**: Readiness below target; restrictions: ↓ risk in Finance, ↓ cognitive load in Work (soft tasks), break reminders.
- **No-Risk / Block**: HRV significantly below baseline or sleep failed 2+ days; financial risk = 0 or minimal; Work tasks only supportive.
- **Offline/Degraded**: Source unavailable → show last valid snapshot + clearly mark OFFLINE (no simulations).

## Interactions (Flows)

- **Morning**: Auto-appearing "Morning sync" card after sleep ends → Readiness + daily plan/risk limit/protocol.
- **Day**: Event/alert stream; CTAs launch protocols and log completion.
- **Evening**: "Evening recap" → subjective input (energy/focus/mood 1–5) + note; linked with day's data.
- **Manual tags**: Coffee/alcohol/late dinner/illness/medication → affect interpretation of sleep and load.

## Cards and Layout (Recommended)

- **2-column stack (>=1280px)**: Left side Hero + sleep + HR/HRV; right side stress/activity/alerts/protocols; timeline full-width below.
- **Mobile/narrow**: Card stack, timeline as tab.
- **Glass cards** (`--glass-bg`, `--glass-border`), hover lift + `--shadow-sm`, CTAs with `.primary-action-btn`/`.conv-action-btn`.
- **Typescale**: `h1/h2/h3` from tokens; mono for numbers/units, label 11px for axis/legend labels.

## Dataset/Fields (Minimum for Stage 1)

- **Sleep**: total, efficiency, deep, REM, sleep latency, wake after sleep onset, bedtime, wake time, disturbances, 7/30d averages.
- **HR**: current, min/max/avg day, resting HR (RHR), zones (Z1–Z5) with time, heart rate recovery (post-workout).
- **HRV**: rMSSD (or ln), baseline (7/30d), delta vs baseline %, daily variability, stress indicator (HR high + HRV low).
- **Activity**: steps, stand hours, move calories, exercise minutes, sedentary streaks, workouts list (type/intensity/duration/energy/HR zones).
- **Stress events**: time, trigger (HR spike, HRV drop, noise), severity, suggested action.
- **Environment (optional)**: CO₂ ppm, noise dB, temp; status: OK/Warning/Bad.
- **Glucose (optional)**: current, variability, time-in-range, peaks count.
- **Subjective**: energy/focus/mood 1–5, note.
- **Meta**: data freshness timestamps, source (HealthKit/iOS Bridge), last sync event.

## Readiness/Capacity (Display, No Hard Formula)

- **Inputs**: Sleep (duration/efficiency), HRV vs baseline, RHR vs baseline, recent stress, recovery streak.
- **Output**: 0–100 + verbal status (High/Medium/Low) and mode (Normal/Soft/No-Risk).
- **Show key drivers**: "HRV -12% vs baseline", "RHR +6 bpm vs baseline", "Sleep efficiency 78%".
- **Auto-rules**: Low → Soft Work; Very Low → No-Risk; display active restrictions.

## Alerts/Triggers (UI Rules)

- HRV < baseline -10% → warn; < -20% → Soft; < -30% or 2 consecutive days → No-Risk.
- RHR > baseline +5 → warn; +10 → Soft.
- Sleep < 7h or efficiency < 80% → warn; 2 consecutive days → Soft.
- HR spike (sustained high HR outside workout) → instant alert + CTA breathing/walk.
- Sedentary >60m → alert + CTA stretch.
- Noise > threshold → alert + CTA pause/headphones.
- Glucose peak (if available) → alert + CTA walk/water.

## Integration with Finance/Work

- **Finance**: Normal/Soft/No-Risk modes set dynamic risk limit; blocks or position size reduction on alerts.
- **Work**: On Soft — tasks go to soft list; on No-Risk — remove heavy/stressful tasks; on stress spikes → suggest pause/breathing.
- **Agents receive context**: readiness, stress state, sleep debt, active restrictions.

## Charts and Visualization

- Line charts with points for HR/HRV, stacked bars for sleep (phases), area for activity/energy.
- Axes/grids thin `--glass-border`; accent highlights only on events/points.
- Entry animation 180–220ms, no harsh gradient flashes.
- Dark background `bg-02/03`, text `--ink-high`, secondary text minimal; accents sparingly.

## Error/Offline States

- **OFFLINE banner**: Source unavailable, show last valid snapshot timestamp.
- **PARTIAL**: No HRV or sleep — mark fields as N/A, but show the rest.
- **STALE DATA**: If no data >4h (daytime) or no sleep by morning, display warning.

## Extension Points (Future)

- CGM in more detail (SD, time-in-range), musculoskeletal markers (DOMS/manual), breathing/SpO2, posture from cameras/keyboard/mouse.
- WebSocket for push events; "modes" protocol so AI/agents can query state.

## Technical Notes for Frontend

- All values labeled with units (ms, bpm, ppm, %), baseline alongside.
- Colors/styles only via tokens; CTAs use `.primary-action-btn`/`.conv-action-btn`; glass panels with `--glass-border`.
- Store last successful snapshot, do not display "simulated" data.
- Add data freshness and source on each card (tooltip/label).
