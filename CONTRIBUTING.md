# Contributing to ODEI Web Platform

Thank you for your interest in contributing to the ODEI web platform. This guide covers the process for submitting changes.

---

## Getting Started

1. **Fork the repository** and clone it locally.
2. **Read the codebase** before proposing changes. Understand the existing architecture and patterns.
3. **Check the design system.** All UI work must follow the premium dark theme specification (jade `#4FD1C5` / champagne `#F4C95D`). Never use indigo, purple, or violet accents.

---

## Development Setup

### Prerequisites

- Node.js 20+
- npm 10+

### Local Development

The API server runs standalone with no external dependencies:

```bash
# Start the v1 API server on port 8800
node api-server.js

# Start the v2 routes in standalone mode on port 8801
node api-v2-routes.js
```

### Building CSS

After modifying styles:

```bash
npm run build:css
```

This compiles `input.css` through Tailwind CSS. The compiled output is `styles.css` -- do not edit `styles.css` directly.

---

## Design System

All UI contributions must follow the premium dark theme:

### Colors

| Token | Value | Usage |
|-------|-------|-------|
| Accent Primary | `#4FD1C5` (jade/teal) | CTAs, focus states, selection, borders |
| Accent Secondary | `#F4C95D` (champagne) | Highlights only, less than 10% of UI |
| Accent Danger | `#FF7B7B` | Errors and warnings |
| Background 01 | `#05060A` | Deepest background |
| Background 02 | `#0B0E14` | Primary background |
| Background 03 | `#121622` | Card/panel background |
| Background 04 | `#1D2331` | Elevated surfaces |
| Ink High | `#E8ECF4` | Primary text |
| Ink Medium | `#B6C0D2` | Secondary text |
| Muted | `#7C889A` | Meta text only |

### Tailwind Classes

- Primary accent: `teal-500`, `teal-400`, `teal-600`
- Never use: `indigo-*`, `purple-*`, `violet-*`
- Glass borders: `border-teal-500/30`, `border-teal-400/50` (hover)
- Glows: `shadow-teal-500/20`

### Button Hierarchy

| Type | Class | Usage |
|------|-------|-------|
| Primary CTA | `.primary-action-btn` | Save, main actions |
| Secondary | `.conv-action-btn` | Navigation, utility |
| Glass | custom | Toggles, utility buttons |

---

## Code Standards

### General

- No framework for the API server (vanilla Node.js `http` module)
- No unnecessary dependencies -- the server has zero npm dependencies in production
- Consistent JSON response envelope: `{ ok: true, data: {...} }` or `{ ok: false, error: "...", code: "..." }`

### HTML/CSS

- Semantic HTML5
- Tailwind CSS utility classes preferred over custom CSS
- Custom CSS goes in component-specific files (e.g., `health-dashboard.css`), not inline
- Mobile-responsive by default

### JavaScript

- ES2022+ features are fine (Node.js 20+ on server, modern browsers on client)
- No TypeScript in the web repo (the MCP servers use TypeScript, this repo is plain JS)
- `'use strict'` at file top for modules
- Prefer `const` over `let`, never use `var`

---

## Submitting Changes

### Before Opening a PR

1. **Test locally.** Start the API server and verify your changes work.
2. **Build CSS** if you changed any styles: `npm run build:css`
3. **Check for theme violations.** Search for `indigo`, `purple`, or `violet` in your changes -- these are not allowed.
4. **Verify social metadata.** If you changed `index.html` or `worldmodel.html`, ensure `og:title`, `og:description`, `twitter:title`, and `twitter:description` meta tags are present.

### PR Guidelines

- **One concern per PR.** Don't mix unrelated changes.
- **Clear title.** Use the format: `feat(component): description`, `fix(api): description`, or `refactor(css): description`.
- **Describe the "why."** The PR description should explain why the change is needed, not just what it does.
- **Include screenshots** for any visual changes.
- **Test the deploy script** if you changed server-side files. The deploy validates required files and will reject incomplete changes.

### Commit Messages

Follow the conventional commits format:

```
feat(api): add rate limit headers to v2 responses
fix(ui): correct holder map gradient on mobile
refactor(server): extract guardrail rules to separate module
docs(api): add signal endpoint examples
```

---

## Architecture Notes

### File Organization

- `api-server.js` -- v1 API endpoints (health, worldmodel, intake, token price)
- `api-v2-routes.js` -- v2 API endpoints (auth, quota, guardrails, graph queries)
- `blinks-handler.js` -- Solana Actions (Blinks) handler
- `config.js` -- Site configuration
- `*.html` -- Page templates (static, served by nginx)
- `*.css` -- Stylesheets (compile from `input.css` via Tailwind)
- `*.js` (client-side) -- UI components (nav-rail, site-shell-router, graph renderer)
- `world-model-demo.json` -- Graph snapshot (auto-exported during deploy, do not edit manually)
- `sentinel-feed.json` -- Sentinel data (auto-exported during deploy, do not edit manually)

### Key Constraints

- The API server binds to `127.0.0.1` only. nginx is the public boundary.
- Rate limiting is 20 requests per minute per IP.
- The server has zero npm runtime dependencies in production. Do not add Express, Fastify, or similar.
- `world-model-demo.json` and `sentinel-feed.json` are generated by export scripts. Do not commit manual edits to these files.

---

## Reporting Issues

Open an issue with:

- **Steps to reproduce** the problem
- **Expected behavior** vs. actual behavior
- **Environment** (browser, OS, API client)
- **Screenshots or curl output** if applicable

---

## Security

If you discover a security vulnerability, **do not** open a public issue. Instead, contact the team directly via [Telegram](https://t.me/odei_ai) or email. See [SECURITY.md](SECURITY.md) for the full policy.

---

## License

By contributing, you agree that your contributions will be licensed under the project's proprietary license. All contributions become the property of ODEI Symbiosis.
