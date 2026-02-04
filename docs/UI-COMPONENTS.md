# ODEI UI Component Inventory

**Last Updated:** 2025-12-25
**Theme Reference:** `/docs/premium-dark-theme.md`

---

## Button Hierarchy

All buttons must use the correct hierarchy classes to maintain visual consistency.

### Primary Action Buttons
**Class:** `.primary-action-btn`
**Usage:** Main CTAs, save operations, critical actions
**Style:**
- Jade → Champagne gradient: `linear-gradient(135deg, rgba(79,209,197,0.42), rgba(244,201,93,0.32))`
- Border: `rgba(79,209,197,0.45)`
- Shadow: Glow effect with jade tint
- Hover: Intensified gradient + translateY(-1px) + enhanced glow

**Examples:**
```html
<button class="primary-action-btn" id="save-config">Save</button>
<button class="primary-action-btn">Create Backup</button>
```

**Current Usage:**
- Memory backup create button
- Modal confirm buttons (error dialogs)
- Memory empty state CTA

### Secondary Action Buttons
**Class:** `.conv-action-btn`
**Usage:** Navigation, refresh, utility actions
**Style:**
- Subtle jade gradient: `linear-gradient(145deg, rgba(79,209,197,0.15), rgba(79,209,197,0.08))`
- Border: `rgba(79,209,197,0.4)`
- Text: Jade `rgba(79,209,197,1)`
- Hover: Lighter gradient + white text

**Examples:**
```html
<button class="conv-action-btn" onclick="refresh()">Refresh</button>
<button class="conv-action-btn">Go to Discuss</button>
```

**Current Usage:**
- Empty state actions (UIManager)
- Context dashboard actions
- Agent navigation
- Refresh list buttons

### Glass Buttons
**Classes:** `.ghost-btn`, `.memory-icon-btn`, `.memory-circle-btn`
**Usage:** Icon buttons, toggles, utility controls
**Style:**
- Glass background: `rgba(79,209,197,0.18)` or subtle
- Border: `rgba(79,209,197,0.35)`
- Minimal padding, icon-focused
- Hover: Brightened border + subtle glow

**Examples:**
```html
<button class="ghost-btn memory-icon-btn" title="Collapse panel">
  <svg>...</svg>
</button>
<button id="zoom-in-btn" class="memory-circle-btn">+</button>
```

**Current Usage:**
- Memory panel collapse/expand
- Zoom controls
- Header icon buttons
- Agent reload buttons

### Specialized Button Types

#### Agent Tabs
**Class:** `.agent-tab`
**Usage:** Agent/view selection in navbar
**Style:** Glass background, active state with jade accent

#### Layer Tabs
**Class:** `.memory-filter-tab`, `.layer-tab`
**Usage:** Layer filtering, memory view mode selection
**Style:** Glass background, active state border accent

#### Action Buttons (Node Details)
**Classes:** `.action-btn-primary`, `.action-btn-secondary`
**Usage:** Node detail panel actions
**Follows primary/secondary hierarchy**

#### Context Action Chips
**Class:** `.context-action-chip`
**Usage:** Health dashboard quick actions
**Style:** Pill-shaped, jade border, glass fill

#### Task Buttons
**Classes:** `.task-start-btn`, `.capture-btn-physical`, `.capture-btn-digital`
**Usage:** Today view task management
**Style:** Specialized colors (physical vs digital distinction)

---

## Panel Components

### Memory Panel Left (Filter Panel)
**ID:** `#memory-filter-panel`
**Class:** `.memory-panel.memory-panel-left`
**Structure:**
```html
<aside class="memory-panel memory-panel-left">
  <div class="memory-panel-header">
    <div class="memory-panel-heading">
      <h2 class="memory-panel-title">Title</h2>
      <p class="memory-panel-subtitle">Subtitle</p>
    </div>
  </div>
  <!-- Filter content -->
</aside>
```

**Style:**
- Glass background with jade border
- Fixed width, full height
- Scrollable content area
- Collapsible sections

### Memory Panel Right (Detail Panel)
**ID:** `#memory-detail-panel`
**Class:** `.memory-panel.memory-panel-right`
**Purpose:** Node details, selection info

### Node List Panel
**Class:** `.node-list-panel`
**Usage:** Dashboard node browsing
**Structure:** Header + search + scrollable list

### Node Details Panel
**Class:** `.node-details-panel`
**Usage:** Individual node inspection
**Structure:** Header + metadata + actions + relationships

### Empty Panel
**Class:** `.empty-panel`
**Usage:** Empty state messaging
**Structure:**
```html
<div class="empty-panel">
  <h4>No items</h4>
  <p>Description</p>
  <div class="empty-actions">
    <button class="conv-action-btn">Action</button>
  </div>
</div>
```

---

## Card Components

### Memory Stat Card
**Class:** `.memory-stat-card`
**Usage:** Memory metrics display (filter panel)
**Attributes:** `data-layer-card="layer-name"`
**Style:** Glass background, jade border on active
**Contains:** Title, count, sparkline/chart

### Widget Card
**Class:** `.widget-card`
**Usage:** Today view dashboard widgets
**Variants:** `.agent-panel`, `.task-panel`, `.health-panel`
**Style:** Glass background, rounded corners, subtle shadow

### Agent Panel
**Class:** `.agent-panel`
**Usage:** Agent status cards in Today view
**Style:** Widget card variant with agent-specific colors

### Context Hero Card
**Class:** `.context-hero`
**Usage:** Health dashboard empty/hero states
**Style:** Large glass card, centered content

---

## Input Components

### Text Inputs
**Base Style:**
- Background: `rgba(11,14,20,0.86)`
- Border: `rgba(79,209,197,0.2)`
- Text: `var(--text-primary)`
- Focus: Brightened border + jade glow

**Classes:**
- `.terminal-input` - Command terminal
- `.nodes-panel-search input` - Node search
- Standard text inputs follow theme tokens

### Toggles
**Class:** `.memory-legend-toggle`
**Usage:** Layer visibility toggles in legend
**Style:** Custom checkbox, jade accent when checked

### Checkboxes
**Style:** Custom styled with jade accent
**Focus:** Jade outline ring

---

## Typography Scale

### Headings
- **h1:** `clamp(22px, 2vw, 28px)` - Page titles
- **h2:** `clamp(18px, 1.6vw, 22px)` - Section headers
- **h3:** `clamp(15px, 1.2vw, 18px)` - Subsections
- **Labels:** `11px` - Uppercase meta labels

### Body Text
- **Primary:** `var(--text-primary)` / `rgba(232,236,244,0.98)`
- **Secondary:** `var(--text-secondary)` / `rgba(182,192,210,0.85)`
- **Muted:** `var(--text-muted)` / `rgba(124,136,154,0.75)`

### Special Text Classes
- `.memory-panel-title` - Panel headers
- `.memory-panel-subtitle` - Panel subheaders
- `.rc-group-title` - Recent conversations group
- `.rc-group-meta` - Meta text (counts, dates)

---

## Icon Components

### SVG Icons
**Color:** Inherit from parent or `currentColor`
**Size:** 12-20px typical
**Stroke:** 1.5-2px for clarity

**Common Usage:**
- Agent icons (navbar)
- Collapse/expand chevrons
- Action buttons (edit, delete, archive)
- View mode switchers

### Memory Legend Dots
**Class:** `.memory-legend-dot`, `.memory-legend-dot-small`
**Usage:** Layer and type color indicators
**Style:** Circular, inline-flex, background color indicates type

---

## Layout Components

### Navbar
**Class:** `.app-nav`
**Structure:**
```html
<nav class="app-nav">
  <div class="nav-shell">
    <div class="nav-inline">
      <!-- Agent tabs -->
    </div>
    <div class="nav-meta-row">
      <!-- Session info, metrics -->
    </div>
  </div>
</nav>
```

**Style:**
- Glass background with gradient overlay
- Jade bottom border
- Radial accent gradients
- Fixed at top, z-index 100

### Sidebar (Conversations)
**ID:** `#sidebar`
**Purpose:** Recent conversations, agent-specific content
**Style:** Glass background, scrollable, collapsible sections

### Main Content Area
**ID:** `#main-content`
**Purpose:** Primary view (chat, memory, dashboard, today)
**Style:** Flexible, full height, padded

### Memory Canvas
**ID:** `#memory-canvas`
**Purpose:** 3D graph visualization (Three.js)
**Style:** Absolute positioned, full container

---

## Modal/Overlay Components

### Error Modal
**Class:** `.error-modal-overlay`
**Structure:**
```html
<div class="error-modal-overlay">
  <div class="error-modal">
    <div class="error-modal-header">
      <h3>Error Title</h3>
      <button class="error-modal-close">×</button>
    </div>
    <div class="error-modal-body">
      <p>Error message</p>
    </div>
    <div class="error-modal-actions">
      <button class="error-modal-btn error-modal-btn-primary">Retry</button>
      <button class="error-modal-btn error-modal-btn-secondary">Dismiss</button>
    </div>
  </div>
</div>
```

**Style:**
- Full viewport overlay: `rgba(5,6,10,0.95)`
- Centered modal: glass background, jade border
- Backdrop blur effect

### Toast Notifications
**Class:** `.error-toast`
**Usage:** Temporary status messages
**Style:** Small glass card, bottom-right position, auto-dismiss

---

## Color Usage Guidelines

### Accent Colors
- **Primary (Jade):** `#4fd1c5` - CTAs, focus, selection, primary borders
- **Secondary (Champagne):** `#f4c95d` - Highlights only, <10% usage, data points
- **Danger:** `#ff7b7b` - Errors, destructive actions

### Layer-Specific Colors (ONLY for layer identification)
- **Strategy:** `#a78bfa` (Purple) - Layer tabs, feed icons
- **Tactics:** `#60a5fa` (Blue) - Layer tabs, status indicators
- **Execution:** `#4fd1c5` (Jade) - Primary accent
- **Track:** `#f4c95d` (Champagne) - Secondary accent

**IMPORTANT:** Layer colors (purple, blue) are ONLY for:
- Layer navigation tabs active state
- Activity feed type-specific icons
- Status badges with layer context
- NEVER for general UI elements (use jade/champagne)

### Node Type Colors
Defined in `/src/modules/ThreeGraphRenderer.js` and `/src/modules/memory/memoryUtils.js`

**Foundation Types:**
- Value: `#ec4899` (Pink)
- Principle: `#4fd1c5` (Jade) ✅ Fixed from purple
- Guardrail: `#f97316` (Orange)
- Human: `#ef4444` (Red)
- Skill: `#22d3ee` (Cyan)

**Mind Layer Types:**
- Insight: `#5fddcd` (Light jade) ✅ Fixed from purple
- Pattern: `#4fd1c5` (Jade) ✅ Fixed from purple
- Evidence: `#f472b6` (Pink)

**Execution Types:**
- Decision: `#60a5fa` (Blue) ✅ Fixed from indigo
- Task/Action: `#a5b4fc` (Light blue)
- TimeBlock: `#93c5fd` (Sky blue)

---

## Glass Effects

### Backgrounds
- **Base:** `rgba(11,14,20,0.86)` - Panels, cards
- **Hover:** `rgba(18,22,32,0.9)` - Interactive elements
- **Active:** `rgba(26,32,44,0.94)` - Pressed states

### Borders
- **Default:** `rgba(79,209,197,0.2)` - Subtle jade tint
- **Bright:** `rgba(79,209,197,0.35)` - Hover, active states
- **Focus:** `rgba(79,209,197,0.45)` - Input focus

### Shadows
- **Small:** `0 12px 26px rgba(0,0,0,0.28)`
- **Medium:** `0 18px 34px rgba(0,0,0,0.4)`
- **Large:** `0 28px 56px rgba(0,0,0,0.52)`
- **Glow (Jade):** `0 0 24px rgba(79,209,197,0.2)`
- **Glow (Champagne):** `0 0 36px rgba(244,201,93,0.35)`

---

## Accessibility

### Focus States
All interactive elements MUST have visible focus indicators:
- Outline: `2px solid var(--glass-border-bright)`
- Offset: `2px`
- Shadow: Jade glow

### Contrast Requirements
- Text on dark backgrounds: WCAG AA minimum
- Primary text: `--text-primary` for highest contrast
- Secondary text: `--text-secondary` for hierarchy
- Muted text: `--text-muted` for meta only

### ARIA Attributes
Required on:
- Toggles: `aria-expanded`, `aria-controls`
- Tabs: `aria-selected`, `role="tab"`
- Panels: `aria-labelledby`, `role="tabpanel"`
- Buttons: `aria-label` for icon-only buttons

---

## Animation & Motion

### Transitions
**Default:** `0.2s ease`
**Properties:**
- `background-color`
- `border-color`
- `color`
- `transform`

### Hover Effects
- **translateY:** `-1px` for buttons
- **Opacity:** Subtle increases (0.85 → 1)
- **Glow:** Enhanced shadow on hover

### Duration Guidelines
- UI feedback: 180-220ms
- Page transitions: 200ms
- Fade in/out: 200ms
- Avoid springy/bounce curves

---

## CSS Custom Properties Reference

See `/src/input.css` for full variable definitions.

**Key Tokens:**
```css
:root {
  /* Backgrounds */
  --bg-01: #05060a;
  --bg-02: #0b0e14;
  --bg-03: #121622;
  --bg-04: #1d2331;

  /* Accents */
  --accent-primary: #4fd1c5;
  --accent-secondary: #f4c95d;
  --accent-danger: #ff7b7b;

  /* Glass */
  --glass-bg: rgba(11,14,20,0.86);
  --glass-border: rgba(79,209,197,0.2);
  --glass-border-bright: rgba(79,209,197,0.35);

  /* Typography */
  --text-primary: rgba(232,236,244,0.98);
  --text-secondary: rgba(182,192,210,0.85);
  --text-muted: rgba(124,136,154,0.75);

  /* Shadows */
  --shadow-sm: 0 12px 26px rgba(0,0,0,0.28);
  --shadow-glow: 0 0 24px rgba(79,209,197,0.2);
}
```

---

## Build & Maintenance

### CSS Build Process
After modifying `/src/input.css`:
```bash
npm run build:css
```
This generates `/src/styles.css` for production.

### Theme Violations to Avoid
❌ **Never Use:**
- Purple/indigo/violet for general UI (`#8b5cf6`, `#a855f7`, `#6366f1`)
- Hardcoded colors instead of CSS variables
- Inconsistent button classes (always use hierarchy)
- Text colors outside theme tokens

✅ **Always Use:**
- CSS custom properties (`var(--accent-primary)`)
- Button hierarchy classes (`.primary-action-btn`, `.conv-action-btn`)
- Theme tokens for text (`--text-primary`, `--text-secondary`, `--text-muted`)
- Jade (`#4fd1c5`) and Champagne (`#f4c95d`) as primary accents

---

## Component Checklist

When creating new UI components:
- [ ] Uses theme CSS variables
- [ ] Follows button hierarchy
- [ ] Has proper focus states
- [ ] Meets WCAG AA contrast
- [ ] Includes ARIA labels where needed
- [ ] Uses glass effects consistently
- [ ] Animates with 0.2s ease transitions
- [ ] No purple/indigo/violet unless layer-specific
- [ ] Text uses theme tokens (ink-high, ink-med, muted)

---

**End of UI Component Inventory**
