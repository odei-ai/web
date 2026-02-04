# Premium Dark UI Spec (ODEI)

Purpose

- Shared reference for the midnight/jade/champagne system now used in `src/input.css` and `src/health-dashboard.css`.
- Keep contrast high, motion restrained, and accents scarce so the UI reads premium instead of neon.

Palette

- Base: `bg-01 #05060A`, `bg-02 #0B0E14`, `bg-03 #121622`, `bg-04 #1D2331`.
- Ink: `ink-high #E8ECF4`, `ink-med #B6C0D2`, `muted #7C889A`.
- Accents: `accent #4FD1C5` (primary jade), `accent-2 #F4C95D` (champagne highlight, low frequency), `danger #FF7B7B`.
- Border/Glass: `glass-border rgba(79,209,197,0.20)`, `glass-border-bright rgba(79,209,197,0.35)`, glass BGs `rgba(11,14,20,0.86)` → `rgba(26,32,44,0.94)`.
- Shadows: `shadow-sm 0 12px 26px rgba(0,0,0,0.28)`, `shadow-md 0 18px 34px rgba(0,0,0,0.40)`, `shadow-lg 0 28px 56px rgba(0,0,0,0.52)`, `glow rgba(79,209,197,0.2)`, `glow-2 rgba(244,201,93,0.35)`.

Typography

- Single family: Space Grotesk for display and body (`--font-display` = `--font-body`), fallbacks `Inter`, `Helvetica Neue`, system.
- Sizes: `h1 clamp(22px,2vw,28px)`, `h2 clamp(18px,1.6vw,22px)`, `h3 clamp(15px,1.2vw,18px)`, `label 11px`, `mono 13px`.
- Headings letter-spacing −0.01em; body letter-spacing 0.01em; keep uppercase labels brief.

Backgrounds & Gradients

- Page: `linear-gradient(135deg,#0B0E14 0%,#121622 60%,#1D2331 100%)` with soft radial jades/champagnes; keep attachment fixed.
- Hero/card gradient: `linear-gradient(135deg,#0B0E14,#121622,#1D2331)`.
- Sub-panels: glass backgrounds with subtle linear top sheen (`rgba(255,255,255,0.02)` overlay).

Tokens (CSS)

```css
:root {
  --bg-01: #05060a;
  --bg-02: #0b0e14;
  --bg-03: #121622;
  --bg-04: #1d2331;
  --ink-high: #e8ecf4;
  --ink-med: #b6c0d2;
  --muted: #7c889a;
  --accent-primary: #4fd1c5;
  --accent-secondary: #f4c95d;
  --accent-danger: #ff7b7b;
  --glass-bg: rgba(11, 14, 20, 0.86);
  --glass-bg-hover: rgba(18, 22, 32, 0.9);
  --glass-bg-active: rgba(26, 32, 44, 0.94);
  --glass-border: rgba(79, 209, 197, 0.2);
  --glass-border-bright: rgba(79, 209, 197, 0.35);
  --shadow-sm: 0 12px 26px rgba(0, 0, 0, 0.28);
  --shadow-md: 0 18px 34px rgba(0, 0, 0, 0.4);
  --shadow-lg: 0 28px 56px rgba(0, 0, 0, 0.52);
  --shadow-glow: 0 0 24px rgba(79, 209, 197, 0.2);
  --shadow-glow-bright: 0 0 36px rgba(244, 201, 93, 0.35);
  --text-primary: rgba(232, 236, 244, 0.98);
  --text-secondary: rgba(182, 192, 210, 0.85);
  --text-muted: rgba(124, 136, 154, 0.75);
  --font-display: 'Space Grotesk', 'Inter', 'Helvetica Neue', -apple-system, system-ui, sans-serif;
  --font-body: var(--font-display);
}
```

Usage Rules

- Backgrounds: pair `bg-01/02` for body/hero; use glass layers for cards and modals.
- Text: primary on dark bases; use `--text-secondary` sparingly; `--text-muted` for meta only.
- Accents: `--accent-primary` for CTAs/focus/selection; `--accent-secondary` only for highlights, data points, or dual-tone gradients; never use both on large surfaces.
- Borders/dividers: `--glass-border`; elevate states with `--glass-border-bright` + glow shadow.
- Selection: jade tint; focus rings use `--glass-border-bright` with subtle outer shadow.
- Scrollbars: jade gradient thumb, dark track; hover adds champagne tint.

Button Hierarchy

- **Primary Action Buttons** (Save, Refresh, main CTAs): Use `.primary-action-btn` class
  - Gradient: jade → champagne `linear-gradient(135deg, rgba(79,209,197,0.42), rgba(244,201,93,0.32))`
  - Border: `rgba(79,209,197,0.45)`
  - Shadow: `0 16px 40px rgba(0,0,0,0.35), 0 0 18px rgba(79,209,197,0.24)`
  - Hover: intensified gradient + lift + glow
- **Secondary Action Buttons** (Go to X, Refresh list): Use `.conv-action-btn` class
  - Gradient: subtle jade `linear-gradient(145deg, rgba(79,209,197,0.15), rgba(79,209,197,0.08))`
  - Border: `rgba(79,209,197,0.4)`
  - Text: jade `rgba(79,209,197,1)`
- **Glass Buttons** (toggles, utility): Use glass base with jade tint
  - Background: `rgba(79,209,197,0.18)`
  - Border: `rgba(79,209,197,0.35)`

Button CSS Classes (defined in `input.css`):

```css
/* Primary CTA - jade/champagne gradient */
.primary-action-btn {
  background: linear-gradient(135deg, rgba(79, 209, 197, 0.42), rgba(244, 201, 93, 0.32)) !important;
  border: 1px solid rgba(79, 209, 197, 0.45) !important;
  box-shadow:
    0 16px 40px rgba(0, 0, 0, 0.35),
    0 0 18px rgba(79, 209, 197, 0.24) !important;
  color: #fff !important;
}

/* Secondary action - subtle jade */
.conv-action-btn {
  background: linear-gradient(145deg, rgba(79, 209, 197, 0.15), rgba(79, 209, 197, 0.08)) !important;
  border: 1px solid rgba(79, 209, 197, 0.4) !important;
  color: rgba(79, 209, 197, 1) !important;
}
```

## Layer-Specific Colors (Semantic Extensions)

The UI implements a semantic color system for the 4-layer hierarchy. These extend the core jade/champagne palette for distinct layer identification:

| Layer         | Color     | Hex       | RGB                   | Usage                                     |
| ------------- | --------- | --------- | --------------------- | ----------------------------------------- |
| **Strategy**  | Purple    | `#A78BFA` | `rgba(167,139,250,X)` | Layer tabs, feed icons, badges            |
| **Tactics**   | Blue      | `#60A5FA` | `rgba(96,165,250,X)`  | Layer tabs, feed icons, status indicators |
| **Execution** | Jade      | `#4FD1C5` | `rgba(79,209,197,X)`  | Primary accent (from core palette)        |
| **Track**     | Champagne | `#F4C95D` | `rgba(244,201,93,X)`  | Secondary accent (from core palette)      |

### Layer Tab Styling

```css
.layer-tab[data-layer='strategy'].active {
  border-color: #a78bfa;
  color: #a78bfa;
}
.layer-tab[data-layer='tactics'].active {
  border-color: #60a5fa;
  color: #60a5fa;
}
.layer-tab[data-layer='execution'].active {
  border-color: #4fd1c5;
  color: #4fd1c5;
}
.layer-tab[data-layer='track'].active {
  border-color: #f4c95d;
  color: #f4c95d;
}
```

### When to Use Layer Colors

- **Layer navigation tabs** — active state border and text
- **Activity feed icons** — type-specific coloring
- **Status badges** — layer-context indicators
- **NEVER** for general UI elements — use jade/champagne from core palette

### CSS Variables (recommended addition)

```css
:root {
  --layer-strategy: #a78bfa;
  --layer-tactics: #60a5fa;
  --layer-execution: #4fd1c5; /* same as --accent-primary */
  --layer-track: #f4c95d; /* same as --accent-secondary */
}
```

Motion

- Use 180–220ms ease transitions; avoid springy curves.
- Hover: small translateY(-1px) + soft glow.
- Page/section load: fade/slide 120–180px max, opacity 0.0→1 over ~200ms; stagger if multiple cards.

Accessibility & Contrast

- Target WCAG AA for text on `bg-02/03`; avoid using `--accent-secondary` for text on light tones.
- Keep accent usage under 10% of visible area; rely on ink neutrals for hierarchy.
- Preserve focus outlines even on glass backgrounds.

Integration Notes

- Core tokens live in `src/input.css`; health dashboard styles mirror them in `src/health-dashboard.css`.
- Build pipeline: run `npm run build:css` to regenerate `src/styles.css` after token changes.
- Fonts shipped in `assets/fonts` (Space Grotesk weights 400/500/700); no second family in stack.
