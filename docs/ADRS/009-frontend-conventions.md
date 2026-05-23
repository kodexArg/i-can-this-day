# ADR-009 — Frontend Conventions: Pico.css + Lucide SVG, no build step

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md §4 (surfaces), §12 (design principles), NFR-1 (mobile-first) · docs/ADRS/001-stack.md · docs/ADRS/002-hosting-topology.md

## Context

ADR-001 forbids a frontend build pipeline (no npm, no PostCSS). ADR-002 puts
static assets in the container under Whitenoise. PRD §12.5 says "the calendar
IS the product" — most of the visual surface is bespoke, so a heavy component
library would not earn its weight. We pin the smallest possible front-of-house
stack that delivers good defaults for forms/typography and gets out of the way
for the bespoke calendar.

## Decision

**CSS framework**
- **Pico.css v2**, vendored at `static/vendor/pico.min.css`. Served by
  Whitenoise (ADR-002). NOT from a CDN — self-contained, no third-party
  network dependency, no privacy surface.
- Use Pico **classless**: write semantic HTML and let the framework style it.
  Class-prefixed Pico features only when needed.
- Pico defaults cover: typography, forms, tables, dialogs, color theme. The
  day modal uses `<dialog>` and picks up Pico's modal styling for free.

**Custom CSS**
- One file: `static/app.css`. Target ~50–150 lines, hard ceiling 300 lines.
- Three sections: (1) CSS custom properties (color tokens, spacing scale
  extending Pico's), (2) calendar grid (the bespoke piece), (3) day-modal
  layout overrides.
- NEVER inline styles in templates. NEVER `<style>` blocks per page.

**Icons**
- **Lucide** (https://lucide.dev) icon set.
- Icons are **inlined as SVG** directly into templates — never an icon font,
  never `<img src=...>`, never a sprite sheet.
- A reusable Django template tag (e.g., `{% lucide "calendar" %}`) renders the
  SVG inline with sensible defaults. Implementation detail, not part of this ADR.
- Vendor only the Lucide SVGs actually used into `static/vendor/lucide/`.

**JS**
- **htmx.min.js** only (ADR-001). No Alpine, no Stimulus, no jQuery, no
  Pico's optional JS bundle.
- Vendored at `static/vendor/htmx.min.js`. NOT from a CDN.

**Build step**
- **NONE.** `collectstatic` copies vendored files; Whitenoise compresses and
  serves them with far-future cache headers. No PostCSS, no Tailwind CLI, no
  esbuild, no rollup.

**Mobile-first posture**
- Base styles target the 375 px viewport (PRD NFR-1).
- Use Pico's default breakpoints (576/768/1024/1280 px) only when a layout
  genuinely needs to change; no desktop-first overrides as a default.
- Touch targets ≥ 44×44 px (PRD NFR-2) — enforced by calendar-cell sizing
  in `app.css`.
- `<meta name="viewport" content="width=device-width, initial-scale=1">` is
  mandatory in the base template.

**Color and theme**
- Single light theme in v1. Dark mode is NOT shipped (Pico supports
  `data-theme="dark"` — one-line opt-in later).
- Color tokens live in CSS custom properties at `:root` in `app.css`. NEVER
  hard-coded hex values in templates.

**Typography**
- Pico's system-font stack — no web fonts, no `@font-face`. Honors the
  WhatsApp-in-app-browser performance target.

## Consequences

- First-load CSS payload: ~15 KB gzipped (Pico ~12 KB + custom ~3 KB).
  Negligible on 4G.
- No build step means PRs can be reviewed and run with `manage.py runserver`
  only — no `npm install`, no Node version drift, no lockfile churn.
- Vendored assets keep the app working offline / on a plane during development
  and remove a third-party uptime dependency from the request path.
- Inlining SVG icons makes them color-customizable via `currentColor` and
  removes the HTTP round-trip per icon. Slightly larger HTML, better perceived
  latency.
- Pico is classless → switching frameworks later is a CSS swap, not a template
  rewrite. Low lock-in.
- Calendar grid is bespoke regardless of framework choice; Pico's lack of grid
  utilities is not a cost.

## Alternatives Considered

- **Tailwind v4 standalone CLI** — utility-first flexibility but adds a build
  step to the Dockerfile and clutters HTML with long class strings. Rejected:
  build-step cost not earned at this scale.
- **Bootstrap 5 CDN + Bootstrap Icons** — fastest component bootstrap but
  corporate aesthetic clashes with the casual social tone, larger payload,
  tempts using Bootstrap JS we don't want. Rejected.
- **Hand-rolled CSS + CSS custom properties (no framework)** — smallest
  possible payload but slowest start; solo-dev visual polish becomes a
  bottleneck. Rejected: Pico gives the same escape hatch with better defaults.

## Open Follow-ups

- Add an "asset sources" line to README pointing at the Pico, Lucide, and
  HTMX upstream versions so updates are reproducible.
- The reusable `{% lucide %}` template tag is implementation work, not
  architecture; tracked elsewhere.
- Dark theme is one attribute away (`data-theme="dark"` on `<html>`); revisit
  if users ask.
- If the calendar grid ever needs CSS Grid features not well supported on iOS
  16 (PRD NFR-7), fall back to flexbox — handled in `app.css`, not via a
  framework change.
