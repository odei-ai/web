# Task for Codex — ALL COMPLETE

Polish `index.html` to production quality. This is a single-file roadmap page for ODEI (World Model as a Service), live at https://api.odei.ai/ on GitHub Pages.

## 1. CRITICAL BUG — IntersectionObserver Broken — DONE

Replaced IntersectionObserver with `setInterval` polling approach (150ms). Immune to background tab throttling. Added 3-second failsafe that reveals all remaining elements. Added `.no-js .fade-in` CSS fallback.

**Result:** 16/16 fade-in elements visible on every load.

## 2. Fix Broken Links — DONE

- Hero pill: changed from `<a>` to `<span>` (no href)
- Footer: removed API Docs and Health links
- Only internal links: `#main` (skip-link) and `/llms.txt` (valid)

## 3. Add OG Image — DONE

Using OdeiCard.jpeg (1424x752, 985KB) with full meta tags:
- `og:image`, `og:image:width`, `og:image:height`, `og:image:type`
- `twitter:image`, `twitter:card=summary_large_image`

## 4. Content Update — DONE

Phase 3 HTTPS task changed from in-progress to done.

## 5. Visual Polish — DONE

- Animated progress bar fill on scroll reveal
- Left-border color per phase: green=done, jade=in-progress, gray=upcoming
- Count-up animation (0→255+) on status bar
- Zebra striping on priority matrix
- rAF-throttled mousemove handler
- `will-change: transform, opacity` on `.fade-in`
- CSS-only armillary sphere (5 rotating rings)
- Hero wordmark glow
- Scroll progress bar (jade→gold gradient)
- Phase status tags (COMPLETE / IN PROGRESS badges)
- Priority matrix Status column (Done/WIP/Todo)
- 6-Layer Architecture diagram
- Copy CA button in footer

## 6. Mobile (375px) — DONE

- No horizontal overflow at 375px
- Footer links stack vertically
- Priority matrix scrollable with `-webkit-overflow-scrolling: touch`
- Hero wordmark scales to 2.5rem
- Matrix min-width: 580px for 5-column layout

## 7. Accessibility — DONE

- `<main>` element wrapping all content
- 27 `aria-label` attributes on checkmarks and interactive elements
- Focus-visible styles on all links/buttons (jade outline)
- Skip-to-content link
- `prefers-reduced-motion: reduce` media query
- Smooth scroll (`scroll-behavior: smooth`)
- Color contrast verified: #7C889A on #121622 = 5.14:1 (passes WCAG AA)

## Final Stats

- **Lines:** 1139 / 1200 max
- **Size:** ~43KB / 100KB target
- **Fade-in:** 16/16 visible
- **Lighthouse a11y:** All checks pass
- **Commits:** 12 commits in this session
- **Design system:** Jade/champagne/midnight preserved, zero indigo/purple/violet

## Design System — DO NOT CHANGE

Colors: `--accent: #4FD1C5` (jade), `--gold: #F4C95D` (champagne), `--bg-deep: #05060A`. Never use indigo/purple/violet. Single file, no external dependencies, keep under 1200 lines.
