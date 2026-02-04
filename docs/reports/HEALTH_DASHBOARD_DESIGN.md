# Health Dashboard Visual Design Specification

## Overview

Заменяет неиспользуемую панель "Companion AI Consultation" в правой части UI.

**Размеры:**

- Ширина: 380px (фиксированная)
- Высота: 100vh (весь экран)
- Отступы: 20px внутренние

---

## Component Hierarchy

```
┌─ health-dashboard-panel ─────────────────────────┐
│                                                   │
│  ┌─ health-score-widget ────────────────────┐   │
│  │  ┌──────────────────┐                     │   │
│  │  │   [Score Circle] │  Health Score       │   │
│  │  │       92         │  Grade: A           │   │
│  │  │    ████████      │  Status: Excellent  │   │
│  │  └──────────────────┘                     │   │
│  │                                            │   │
│  │  Current: Sun, Nov 10, 2025 14:32 UTC     │   │
│  │  Period: Q4 2025, Week 45                 │   │
│  └────────────────────────────────────────────┘   │
│                                                   │
│  ┌─ active-goals-widget ─────────────────────┐   │
│  │  Active Goals                              │   │
│  │  ┌──────────────────────────────────────┐ │   │
│  │  │ Q4: Launch Memory Atlas v2.0         │ │   │
│  │  │ 📅 Due: Dec 31, 2025 (52 days)       │ │   │
│  │  └──────────────────────────────────────┘ │   │
│  │  ┌──────────────────────────────────────┐ │   │
│  │  │ Nov: Complete temporal service       │ │   │
│  │  │ 📅 Due: Nov 30, 2025 (20 days)       │ │   │
│  │  └──────────────────────────────────────┘ │   │
│  │  ┌──────────────────────────────────────┐ │   │
│  │  │ Week 45: Implement health daemon     │ │   │
│  │  │ 📅 Due: Nov 17, 2025 (7 days)        │ │   │
│  │  └──────────────────────────────────────┘ │   │
│  └────────────────────────────────────────────┘   │
│                                                   │
│  ┌─ memory-coverage-widget ──────────────────┐   │
│  │  Memory Coverage                           │   │
│  │                                             │   │
│  │  Foundation    ████████████████  95%       │   │
│  │  Vision        ████████████      78%       │   │
│  │  Strategy      ██████████        65%       │   │
│  │  Tactics       ████████████████  88%       │   │
│  │  Execution     ██████████████    82%       │   │
│  │  Track         ████████          58%       │   │
│  │  Mind          ██████████        67%       │   │
│  │                                             │   │
│  │  Overall: 76% coverage                     │   │
│  └────────────────────────────────────────────┘   │
│                                                   │
│  ┌─ issues-widget (collapsible) ─────────────┐   │
│  │  ▼ Issues & Alerts (2)                     │   │
│  │  ┌──────────────────────────────────────┐ │   │
│  │  │ ⚠️  STALE_GOAL                       │ │   │
│  │  │ "Q1 2025 Goal" is marked active      │ │   │
│  │  │ but period has ended                 │ │   │
│  │  │ [Auto-Fix] [Dismiss]                 │ │   │
│  │  └──────────────────────────────────────┘ │   │
│  │  ┌──────────────────────────────────────┐ │   │
│  │  │ ℹ️  LOW_COVERAGE                     │ │   │
│  │  │ Track layer at 58% - consider        │ │   │
│  │  │ adding metrics                       │ │   │
│  │  │ [View Gaps] [Dismiss]                │ │   │
│  │  └──────────────────────────────────────┘ │   │
│  └────────────────────────────────────────────┘   │
│                                                   │
│  ┌─ recommendations-widget (collapsible) ───┐   │
│  │  ▶ Recommendations (3)                     │   │
│  │  (Collapsed by default)                    │   │
│  └────────────────────────────────────────────┘   │
│                                                   │
│  ┌─ actions-widget ───────────────────────────┐   │
│  │  [🔄 Refresh Now]  [⚙️ Settings]          │   │
│  │                                             │   │
│  │  Last updated: 2 minutes ago               │   │
│  │  Next check: in 3 minutes                  │   │
│  └────────────────────────────────────────────┘   │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

## 1. Health Score Widget

### Score Circle

```css
.health-score-circle {
  width: 120px;
  height: 120px;
  position: relative;
  margin: 0 auto 20px;
}

/* SVG circular progress */
.score-ring {
  stroke-width: 12;
  fill: none;
  stroke-linecap: round;
}

/* Color mapping */
.score-excellent {
  stroke: #10b981;
} /* Green - 90-100 */
.score-good {
  stroke: #3b82f6;
} /* Blue - 80-89 */
.score-fair {
  stroke: #f59e0b;
} /* Yellow - 70-79 */
.score-poor {
  stroke: #ef4444;
} /* Red - <70 */

.score-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 36px;
  font-weight: 700;
  color: var(--text-primary);
}

.score-background {
  stroke: var(--bg-tertiary);
  opacity: 0.2;
}
```

### Health Info

```html
<div class="health-info">
  <div class="health-grade">Grade: A</div>
  <div class="health-status">Status: Excellent</div>
</div>
```

### Current Time Display

```css
.current-time {
  font-size: 13px;
  color: var(--text-secondary);
  text-align: center;
  margin-top: 12px;
  line-height: 1.5;
}

.current-period {
  font-size: 12px;
  color: var(--text-tertiary);
  text-align: center;
}
```

---

## 2. Active Goals Widget

### Goal Cards

```css
.goal-card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 8px;
  transition: all 0.2s ease;
}

.goal-card:hover {
  background: var(--bg-tertiary);
  border-color: var(--accent-primary);
  transform: translateX(2px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.goal-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 6px;
}

.goal-deadline {
  font-size: 12px;
  color: var(--text-secondary);
  display: flex;
  align-items: center;
  gap: 6px;
}

/* Deadline urgency colors */
.deadline-critical {
  color: #ef4444;
} /* < 7 days */
.deadline-warning {
  color: #f59e0b;
} /* 7-30 days */
.deadline-normal {
  color: #6b7280;
} /* > 30 days */
```

---

## 3. Memory Coverage Widget

### Progress Bars

```css
.coverage-row {
  display: flex;
  align-items: center;
  margin-bottom: 12px;
  gap: 12px;
}

.coverage-label {
  width: 80px;
  font-size: 12px;
  color: var(--text-secondary);
  text-align: right;
}

.coverage-bar-container {
  flex: 1;
  height: 20px;
  background: var(--bg-tertiary);
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}

.coverage-bar-fill {
  height: 100%;
  border-radius: 10px;
  transition: width 0.6s ease;
  background: linear-gradient(90deg, var(--gradient-start), var(--gradient-end));
}

/* Coverage level colors */
.coverage-excellent {
  /* 90-100% */
  --gradient-start: #10b981;
  --gradient-end: #059669;
}
.coverage-good {
  /* 70-89% */
  --gradient-start: #3b82f6;
  --gradient-end: #2563eb;
}
.coverage-fair {
  /* 50-69% */
  --gradient-start: #f59e0b;
  --gradient-end: #d97706;
}
.coverage-low {
  /* <50% */
  --gradient-start: #ef4444;
  --gradient-end: #dc2626;
}

.coverage-percent {
  width: 40px;
  font-size: 12px;
  font-weight: 600;
  color: var(--text-primary);
  text-align: right;
}

/* Animated shimmer effect */
.coverage-bar-fill::after {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.3), transparent);
  animation: shimmer 2s infinite;
}

@keyframes shimmer {
  to {
    left: 100%;
  }
}
```

### Overall Coverage

```css
.overall-coverage {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid var(--border-secondary);
  text-align: center;
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
}
```

---

## 4. Issues Widget (Collapsible)

### Issue Cards

```css
.issue-card {
  background: var(--bg-alert);
  border-left: 4px solid var(--alert-color);
  border-radius: 6px;
  padding: 12px;
  margin-bottom: 8px;
}

.issue-card.warning {
  --bg-alert: #fef3c7;
  --alert-color: #f59e0b;
  --text-alert: #92400e;
}

.issue-card.info {
  --bg-alert: #dbeafe;
  --alert-color: #3b82f6;
  --text-alert: #1e3a8a;
}

.issue-card.error {
  --bg-alert: #fee2e2;
  --alert-color: #ef4444;
  --text-alert: #7f1d1d;
}

.issue-type {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  color: var(--text-alert);
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 6px;
}

.issue-message {
  font-size: 13px;
  color: var(--text-alert);
  line-height: 1.4;
  margin-bottom: 10px;
}

.issue-actions {
  display: flex;
  gap: 8px;
}

.issue-button {
  padding: 4px 12px;
  font-size: 11px;
  font-weight: 600;
  border-radius: 4px;
  border: 1px solid var(--alert-color);
  background: white;
  color: var(--alert-color);
  cursor: pointer;
  transition: all 0.2s ease;
}

.issue-button:hover {
  background: var(--alert-color);
  color: white;
}

.issue-button.primary {
  background: var(--alert-color);
  color: white;
}

.issue-button.primary:hover {
  filter: brightness(1.1);
}
```

### Collapsible Header

```css
.collapsible-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  cursor: pointer;
  user-select: none;
  transition: background 0.2s ease;
}

.collapsible-header:hover {
  background: var(--bg-hover);
}

.collapsible-icon {
  transition: transform 0.2s ease;
}

.collapsible-header.expanded .collapsible-icon {
  transform: rotate(90deg);
}

.issue-count {
  display: inline-block;
  background: var(--bg-badge);
  color: var(--text-badge);
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 700;
  margin-left: 8px;
}
```

---

## 5. Actions Widget

### Action Buttons

```css
.actions-container {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.action-button {
  flex: 1;
  padding: 10px 16px;
  font-size: 13px;
  font-weight: 600;
  border-radius: 6px;
  border: 1px solid var(--border-primary);
  background: var(--bg-secondary);
  color: var(--text-primary);
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.action-button:hover {
  background: var(--accent-primary);
  color: white;
  border-color: var(--accent-primary);
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.action-button.loading {
  pointer-events: none;
  opacity: 0.6;
}

.action-button.loading .icon {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
```

### Status Footer

```css
.status-footer {
  font-size: 11px;
  color: var(--text-tertiary);
  text-align: center;
  line-height: 1.6;
}

.status-time {
  font-weight: 600;
  color: var(--text-secondary);
}
```

---

## Color Scheme (Dark Theme)

```css
:root {
  /* Backgrounds */
  --bg-primary: #111827;
  --bg-secondary: #1f2937;
  --bg-tertiary: #374151;
  --bg-hover: #4b5563;

  /* Text */
  --text-primary: #f9fafb;
  --text-secondary: #d1d5db;
  --text-tertiary: #9ca3af;

  /* Borders */
  --border-primary: #374151;
  --border-secondary: #4b5563;

  /* Accents */
  --accent-primary: #3b82f6;
  --accent-secondary: #8b5cf6;

  /* Status colors */
  --success: #10b981;
  --warning: #f59e0b;
  --error: #ef4444;
  --info: #3b82f6;
}
```

---

## Animation & Interactions

### 1. Page Load Animation

```css
@keyframes slideInRight {
  from {
    opacity: 0;
    transform: translateX(20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.widget {
  animation: slideInRight 0.4s ease;
}

.widget:nth-child(1) {
  animation-delay: 0.1s;
}
.widget:nth-child(2) {
  animation-delay: 0.2s;
}
.widget:nth-child(3) {
  animation-delay: 0.3s;
}
```

### 2. Score Update Animation

```css
@keyframes scorePopIn {
  0% {
    transform: scale(0.8);
    opacity: 0;
  }
  50% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.score-text.updated {
  animation: scorePopIn 0.6s ease;
}
```

### 3. Coverage Bar Fill Animation

```css
.coverage-bar-fill {
  transform-origin: left;
  animation: fillBar 1s ease forwards;
}

@keyframes fillBar {
  from {
    transform: scaleX(0);
  }
  to {
    transform: scaleX(1);
  }
}
```

---

## Data Refresh Logic

### Auto-Refresh Cycle

```javascript
// Every 5 minutes
setInterval(
  async () => {
    await refreshHealthDashboard();
  },
  5 * 60 * 1000
);

// Smooth transition during refresh
async function refreshHealthDashboard() {
  // Add loading state
  dashboard.classList.add('refreshing');

  // Fetch new data
  const [temporal, meta] = await Promise.all([
    callMCP('odei.neo4j.temporal.context.v1', {}),
    callMCP('odei.neo4j.meta.snapshot.v1', {}),
  ]);

  // Update UI with animation
  updateScoreWithAnimation(temporal.healthScore);
  updateGoals(temporal.activeGoals);
  updateCoverage(meta.coverage);
  updateIssues(temporal.issues);

  // Remove loading state
  dashboard.classList.remove('refreshing');
}
```

### Manual Refresh

```javascript
refreshButton.addEventListener('click', async () => {
  refreshButton.classList.add('loading');
  await refreshHealthDashboard();
  refreshButton.classList.remove('loading');

  // Show toast notification
  showToast('Dashboard updated', 'success');
});
```

---

## Responsive Behavior

### Scroll Handling

```css
.health-dashboard-panel {
  overflow-y: auto;
  overflow-x: hidden;
  scrollbar-width: thin;
  scrollbar-color: var(--border-primary) transparent;
}

.health-dashboard-panel::-webkit-scrollbar {
  width: 6px;
}

.health-dashboard-panel::-webkit-scrollbar-track {
  background: transparent;
}

.health-dashboard-panel::-webkit-scrollbar-thumb {
  background: var(--border-primary);
  border-radius: 3px;
}

.health-dashboard-panel::-webkit-scrollbar-thumb:hover {
  background: var(--border-secondary);
}
```

### Widget Spacing

```css
.widget {
  margin-bottom: 16px;
  background: var(--bg-secondary);
  border: 1px solid var(--border-primary);
  border-radius: 12px;
  padding: 16px;
}

.widget:last-child {
  margin-bottom: 0;
}

.widget-header {
  font-size: 14px;
  font-weight: 700;
  color: var(--text-primary);
  margin-bottom: 16px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
```

---

## Implementation Files

### 1. `/src/components/HealthDashboard.js`

Main component logic (300 lines)

### 2. `/src/styles/health-dashboard.css`

All widget styles (450 lines)

### 3. `/src/utils/healthApi.js`

MCP call wrappers (80 lines)

### 4. Modify `/src/renderer.js`

- Remove `initCompanion()` function
- Add `initHealthDashboard()` function
- Replace companion panel mount point

### 5. Modify `/src/index.html`

- Replace `#companion-panel` with `#health-dashboard-panel`
- Update panel structure

---

## Key Features Summary

✅ **Real-time health monitoring** (5-min auto-refresh)
✅ **Visual health score** (0-100 with A-F grade)
✅ **Active goals tracking** (with deadline urgency)
✅ **Memory coverage visualization** (7 layers)
✅ **Issue detection & auto-fix** (collapsible alerts)
✅ **Smooth animations** (slide-in, shimmer, pop-in)
✅ **Dark theme optimized** (matches ODEI aesthetic)
✅ **Interactive elements** (hover states, click actions)
✅ **Responsive scrolling** (custom scrollbar)
✅ **Status indicators** (last updated, next check)

---

## Final Result

**Before:** Unused "Companion AI Consultation" panel (~40% wasted space)
**After:** Live Health Dashboard with 5 interactive widgets showing temporal health, active goals, memory coverage, issues, and quick actions.

**User benefit:** At-a-glance visibility into system health without needing to query agents manually.
