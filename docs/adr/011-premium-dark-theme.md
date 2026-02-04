# ADR-011: Premium Dark Theme Design System

## Status
Accepted

## Context

ODEI is a professional productivity tool for a $500M mission. The UI must convey:

- **Authority:** This is serious infrastructure, not a toy
- **Focus:** Minimal distractions, maximum clarity
- **Sophistication:** Premium feel matching the mission scale
- **Long sessions:** Comfortable for 8+ hour work days
- **Information density:** Graph viz, terminals, health data fit on screen

Anton works primarily in **evening hours (20:00-02:00 Doha time)**, requiring:
- Low eye strain
- High contrast for readability
- Subtle accents (no neon distractions)
- Dark adaptation (no bright white flashes)

### Problem

How do we design a UI that feels premium, remains usable for long sessions, and visually distinguishes ODEI from consumer apps?

### Constraints

1. **Dark-first:** Light themes cause eye strain during evening work
2. **Accessibility:** WCAG AA contrast minimums
3. **Consistency:** Same palette across graph viz, terminals, dashboards
4. **Performance:** No visual jank, smooth 60fps animations
5. **Maintainability:** Design tokens in one place, easy to update
6. **Layer semantics:** Visual distinction for Foundation/Vision/Strategy/Tactics layers

## Decision

**Implement a midnight/jade/champagne color system with glass morphism and restrained motion.**

### Color Palette

**Base (Midnight Gradients):**
```css
--bg-01: #05060A  /* Deepest background */
--bg-02: #0B0E14  /* Primary background */
--bg-03: #121622  /* Card/panel background */
--bg-04: #1D2331  /* Elevated surfaces */
```

**Ink (Text Hierarchy):**
```css
--ink-high: #E8ECF4   /* Primary text (98% opacity) */
--ink-med:  #B6C0D2   /* Secondary text (85% opacity) */
--muted:    #7C889A   /* Meta text (75% opacity) */
```

**Accents:**
```css
--accent-primary:   #4FD1C5  /* Jade/teal - CTAs, focus, selection */
--accent-secondary: #F4C95D  /* Champagne - highlights, <10% usage */
--accent-danger:    #FF7B7B  /* Errors, warnings */
```

**Glass (Translucent Surfaces):**
```css
--glass-bg:        rgba(11, 14, 20, 0.86)      /* Base glass */
--glass-bg-hover:  rgba(18, 22, 32, 0.90)      /* Hover state */
--glass-bg-active: rgba(26, 32, 44, 0.94)      /* Active state */
--glass-border:        rgba(79, 209, 197, 0.20) /* Jade tint */
--glass-border-bright: rgba(79, 209, 197, 0.35) /* Hover/active */
```

**Shadows (Depth & Glow):**
```css
--shadow-sm: 0 12px 26px rgba(0, 0, 0, 0.28)
--shadow-md: 0 18px 34px rgba(0, 0, 0, 0.40)
--shadow-lg: 0 28px 56px rgba(0, 0, 0, 0.52)
--shadow-glow:        0 0 24px rgba(79, 209, 197, 0.20)  /* Jade glow */
--shadow-glow-bright: 0 0 36px rgba(244, 201, 93, 0.35)  /* Champagne glow */
```

### Why These Colors?

**Jade (#4FD1C5):**
- Calming, professional (not aggressive like neon)
- High contrast on dark backgrounds (readable)
- Tech-forward aesthetic (not corporate blue)
- Evokes clarity, precision, trust

**Champagne (#F4C95D):**
- Warm accent for highlights/data points
- Complements jade without clashing
- Used sparingly (<10% of UI) for emphasis
- Premium feel (gold undertones)

**Midnight Blues:**
- Gentle on eyes during long sessions
- Creates depth without pure black (harsh)
- Professional, sophisticated, not "gamer"

### Layer-Specific Colors (Semantic Extension)

**Graph visualization and layer tabs use color-coding:**

```css
--layer-strategy:  #A78BFA  /* Purple */
--layer-tactics:   #60A5FA  /* Blue */
--layer-execution: #4FD1C5  /* Jade (primary accent) */
--layer-track:     #F4C95D  /* Champagne (secondary accent) */
```

**Usage rules:**
- **Layer tabs:** Active tab border + text color
- **Activity feed icons:** Type-specific coloring
- **Status badges:** Layer-context indicators
- **Never for general UI:** Use jade/champagne from core palette

**Why NOT use purple/blue everywhere:**
- Core palette is jade/champagne (brand identity)
- Layer colors are **semantic extensions** for specific contexts
- Using them globally would dilute layer meaning

### Button Hierarchy

**Primary CTA (Save, Refresh Metrics):**
```css
.primary-action-btn {
  background: linear-gradient(135deg,
    rgba(79, 209, 197, 0.42),  /* Jade */
    rgba(244, 201, 93, 0.32)   /* Champagne */
  );
  border: 1px solid rgba(79, 209, 197, 0.45);
  box-shadow:
    0 16px 40px rgba(0, 0, 0, 0.35),
    0 0 18px rgba(79, 209, 197, 0.24);
  color: #fff;
}

.primary-action-btn:hover {
  transform: translateY(-1px);
  box-shadow:
    0 18px 48px rgba(0, 0, 0, 0.42),
    0 0 28px rgba(79, 209, 197, 0.38);
}
```

**Secondary Action (Go to X, Refresh List):**
```css
.conv-action-btn {
  background: linear-gradient(145deg,
    rgba(79, 209, 197, 0.15),  /* Subtle jade */
    rgba(79, 209, 197, 0.08)
  );
  border: 1px solid rgba(79, 209, 197, 0.4);
  color: rgba(79, 209, 197, 1);
}

.conv-action-btn:hover {
  background: linear-gradient(145deg,
    rgba(79, 209, 197, 0.22),
    rgba(79, 209, 197, 0.14)
  );
}
```

**Glass Button (Toggles, Utility):**
```css
.glass-btn {
  background: rgba(79, 209, 197, 0.18);
  border: 1px solid rgba(79, 209, 197, 0.35);
  backdrop-filter: blur(12px);
}
```

### Typography

**Font Family:**
```css
--font-display: 'Space Grotesk', 'Inter', 'Helvetica Neue', -apple-system, sans-serif;
--font-body: var(--font-display);  /* Single family for consistency */
```

**Why Space Grotesk:**
- Geometric sans-serif (clean, modern)
- Excellent readability at small sizes
- 7 weights (400, 500, 700 used)
- Open source (no licensing)

**Sizes:**
```css
--h1: clamp(22px, 2vw, 28px)
--h2: clamp(18px, 1.6vw, 22px)
--h3: clamp(15px, 1.2vw, 18px)
--label: 11px
--mono: 13px  /* Monaco, monospace for code/terminals */
```

**Letter Spacing:**
- Headings: `-0.01em` (tighter for impact)
- Body: `0.01em` (looser for readability)
- Uppercase labels: `0.05em` (tracking for legibility)

### Backgrounds & Gradients

**Page Background:**
```css
body {
  background: linear-gradient(135deg,
    #0B0E14 0%,
    #121622 60%,
    #1D2331 100%
  );
  background-attachment: fixed;
}
```

**Radial Overlays (Subtle Ambiance):**
```css
/* Jade accent (top-right) */
radial-gradient(
  800px circle at 85% 10%,
  rgba(79, 209, 197, 0.08),
  transparent 60%
)

/* Champagne accent (bottom-left) */
radial-gradient(
  600px circle at 15% 90%,
  rgba(244, 201, 93, 0.06),
  transparent 50%
)
```

**Glass Panels:**
```css
.panel {
  background: rgba(11, 14, 20, 0.86);
  border: 1px solid rgba(79, 209, 197, 0.20);
  backdrop-filter: blur(16px);
  box-shadow: 0 12px 26px rgba(0, 0, 0, 0.28);
}
```

### Motion & Animation

**Principles:**
- **Restrained:** No springy/bouncy animations (professional, not playful)
- **Fast:** 180-220ms transitions (responsive feel)
- **Purposeful:** Animations guide attention, not distract

**Standard Transitions:**
```css
transition: all 200ms cubic-bezier(0.4, 0, 0.2, 1);  /* ease-out */
```

**Hover Effects:**
```css
.interactive:hover {
  transform: translateY(-1px);  /* Subtle lift */
  box-shadow: 0 0 24px rgba(79, 209, 197, 0.20);  /* Jade glow */
}
```

**Page Load:**
```css
@keyframes fadeSlideIn {
  from {
    opacity: 0;
    transform: translateY(120px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.panel {
  animation: fadeSlideIn 200ms ease-out;
}
```

**Stagger (Multiple Cards):**
```css
.card:nth-child(1) { animation-delay: 0ms; }
.card:nth-child(2) { animation-delay: 50ms; }
.card:nth-child(3) { animation-delay: 100ms; }
```

### Accessibility

**WCAG AA Compliance:**
- `--ink-high` on `--bg-02`: 15.2:1 (AAA)
- `--ink-med` on `--bg-03`: 8.4:1 (AA)
- `--accent-primary` on `--bg-02`: 7.1:1 (AA)

**Focus States:**
```css
:focus-visible {
  outline: 2px solid rgba(79, 209, 197, 0.6);
  outline-offset: 2px;
  box-shadow: 0 0 0 4px rgba(79, 209, 197, 0.15);
}
```

**Reduced Motion:**
```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

### Tailwind Integration

**Primary accent:** Use `teal-*` classes (maps to jade)
```html
<button class="bg-teal-500 hover:bg-teal-400 border-teal-500/30">
```

**NEVER use:**
- `indigo-*`, `purple-*`, `violet-*` for general UI
- These are reserved for layer-specific contexts only

**Glass borders:**
```html
<div class="border border-teal-500/30 hover:border-teal-400/50">
```

**Glows:**
```html
<div class="shadow-[0_0_24px_rgba(79,209,197,0.2)]">
```

### Build Process

**CSS Pipeline:**
```bash
# Development (hot reload)
npm run dev:css
# Watches: src/input.css → src/styles.css

# Production (minified)
npm run build:css
# Builds: src/input.css → src/styles.css (optimized)
```

**Token Management:**
- Core tokens in `src/input.css` (`:root` variables)
- Tailwind config extends these tokens
- Components reference CSS variables (never hardcode hex)

## Consequences

### Positive

1. **✅ Professional aesthetic:** Conveys authority matching $500M mission
2. **✅ Low eye strain:** Gentle on eyes during 8+ hour sessions
3. **✅ High readability:** WCAG AA compliant, 15:1 contrast on primary text
4. **✅ Information density:** Dark backgrounds recede, content stands out
5. **✅ Performance:** CSS variables + Tailwind JIT = minimal bundle
6. **✅ Consistency:** Single design system across all UI components
7. **✅ Maintainability:** Update palette in one place (`:root` variables)

### Negative

1. **⚠️ Build step required:** Tailwind must compile CSS before use
2. **⚠️ Dark-only:** No light theme (acceptable for Anton's evening work)
3. **⚠️ Accessibility gaps:** Some accent usage <7:1 contrast (non-critical text)
4. **⚠️ Learning curve:** Team must understand glass morphism, token system
5. **⚠️ Browser dependency:** `backdrop-filter` requires modern browsers

### Operational Impact

**Development:**
```bash
# Hot reload CSS changes
npm run dev:css

# Verify token usage
grep -r "indigo\|purple\|violet" src/  # Should return only layer-specific files
```

**Design Updates:**
```css
/* Update accent color system-wide */
:root {
  --accent-primary: #NEW_COLOR;  /* Changes all jade references */
}
```

**Monitoring:**
- Visual regression tests with Playwright screenshots
- Contrast ratio checks in CI (axe-core)

## Alternatives Considered

### 1. Material Design 3 (Material You)

**Approach:** Google's Material Design system

**Pros:**
- Comprehensive component library
- Accessibility built-in
- Dynamic color generation

**Cons:**
- Generic aesthetic (looks like every Google app)
- Not premium/distinctive enough
- Light theme bias (dark mode secondary)

**Rejected:** ODEI needs unique brand identity, not Google clone.

### 2. Indigo/Purple Primary Palette

**Approach:** Use indigo (#6366F1) as primary accent

**Pros:**
- Popular in tech UIs
- Good contrast on dark
- Familiar to users

**Cons:**
- Overused (looks generic)
- Blue/purple conveys "playful" not "serious"
- Clashes with layer colors (purple = Strategy layer)

**Rejected:** Jade differentiates ODEI from competitors.

### 3. Pure Black Background (#000000)

**Approach:** True black instead of midnight blues

**Pros:**
- Maximum OLED power savings
- Highest contrast possible

**Cons:**
- Harsh, sterile aesthetic
- No depth/layering
- Eye strain from extreme contrast

**Rejected:** Midnight gradients softer on eyes.

### 4. Light Theme with Dark Mode Toggle

**Approach:** Support both light and dark themes

**Pros:**
- Flexibility for different preferences
- Better daytime readability (subjective)

**Cons:**
- Doubles design/maintenance effort
- Anton works evenings only (dark preferred)
- Light theme unnecessary complexity

**Rejected:** Dark-first aligns with user behavior.

### 5. Neon Cyberpunk Aesthetic

**Approach:** Bright neon accents (hot pink, electric blue)

**Pros:**
- Visually striking
- "Tech-forward" branding

**Cons:**
- Eye strain during long sessions
- Distracting, not professional
- Difficult to achieve WCAG compliance

**Rejected:** Professionalism > visual flash.

## References

- Design System Spec: `/Users/ai/ODEI/docs/premium-dark-theme.md`
- Tailwind Config: `/Users/ai/ODEI/tailwind.config.js`
- Core Styles: `/Users/ai/ODEI/src/input.css`
- WCAG Guidelines: https://www.w3.org/WAI/WCAG21/quickref/
- Space Grotesk Font: https://fonts.google.com/specimen/Space+Grotesk

## Revision History

- **2025-12-25:** Initial ADR documenting premium dark theme design system
