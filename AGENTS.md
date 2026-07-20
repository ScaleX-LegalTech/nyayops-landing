# AGENTS.md — NyayOps marketing landing page

Public marketing site for NyayOps, meant to live at bare `nyayops.in`. Separate project from
`frontend/` (dashboard SPA at `workspace.nyayops.in`) on purpose — brand/marketing surface (SSG,
SEO, fast first paint), no auth.

**Not yet wired as a git submodule** — local-only directory for now. Turning it into one
(new GitHub repo, `.gitmodules` entry, Vercel project, DNS for `nyayops.in`) needs the user's
go-ahead before creating remote resources, not something to do silently.

## Commands

```bash
npm install
npm run dev        # Astro dev server, localhost:4321 (add --background to run detached;
                     # manage with `astro dev stop` / `astro dev status` / `astro dev logs`)
npm run build       # astro build -> dist/ (static)
npm run preview     # serve the production build locally
```
No lint/test tooling configured — `astro build` (runs Astro's own type checking) is the only
automated gate. Record it here if a linter/test suite is added later.

## Architecture

- `src/pages/index.astro` — the entire site is one long-scroll page. Add new routes here if the
  site grows beyond one page (e.g. `/security`, `/pricing`).
- `src/components/` — one component per section (`Hero.astro`, `Features.astro`,
  `HowItWorks.astro`, etc.), assembled in `index.astro`. `Wordmark.astro`/`Nav.astro` are shared
  chrome.
- `src/layouts/Layout.astro` — HTML shell: SEO meta/OG tags, self-hosted font imports, page-wide
  `<script>` (`.reveal` scroll animation via `motion`'s `inView`/`animate` + mobile nav toggle +
  nav scroll-state + anchor smooth scroll + scroll-restoration override).
- `src/styles/tokens.css` — **ported verbatim** from `frontend/src/tokens.css` ("Ledger Blue" OKLCH
  palette). If the brand palette changes, update `frontend/src/tokens.css` first and mirror here —
  no other source of truth.
- `src/styles/global.css` — Tailwind v4 `@theme` (fonts/radius/shadow/z-index/motion) + base
  styles. Same font stack as the dashboard (Outfit/Public Sans/JetBrains Mono) via `@fontsource/*`
  (self-hosted, unlike `frontend/`'s Google Fonts CDN link — render-blocking cross-origin font CSS
  cost ~15 Lighthouse perf points here before the switch).
- `src/assets/screenshots/` — real product screenshots, Playwright-captured against a
  locally-seeded staging stack, not stock/mockups. Recapture if the product UI drifts.

## Conventions / invariants

- **Brand register, not product register.** Follows `impeccable`'s brand rules
  (`~/.claude/skills/impeccable/reference/brand.md`), not `frontend/`'s product rules — bolder
  color commitment is intentional (navy-drenched hero/CtaBand/HowItWorks vs. Restrained white
  sections between them). Don't flatten to match the dashboard's uniformly-Restrained chrome.
- **Reused tokens are contrast-verified only against the surfaces `frontend/DESIGN.md` verified
  them against** (`--color-ink-muted` ≥4.5:1 on `--color-surface`/white specifically). Using
  `ink-muted` body text on tinted `--color-bg`/`--color-surface-muted` backgrounds drops below
  4.5:1, fails WCAG AA — use `text-ink` (full ink) for body copy on those tints. Learned via a
  Lighthouse a11y regression during initial build; re-check with Lighthouse/axe before reusing a
  token pairing not already used elsewhere in this project.
- **`Wordmark.astro`'s default colors (`shell-ink`/`accent`) are calibrated for the navy shell
  only** (mirrors `frontend/src/components/layout/Wordmark.tsx`'s own constraint). Pass `onLight`
  on a white/near-white background (e.g. `Footer.astro`) — the unguarded default is a
  near-invisible white-on-white bug, not a style choice.
- **`.reveal` must never gate content visibility.** Content is visible by default (`opacity`
  unset in CSS); fade-up-on-scroll-into-view is layered on top via `motion`'s `inView`/`animate`
  (WAAPI, not a CSS class toggle — see below for why). Never set `.reveal { opacity: 0 }` —
  full-page/headless captures and paused-tab renders don't reliably fire the JS that restores
  opacity, so gated sections ship visually blank (happened once during initial build; keep the
  fix).
- **`.reveal` uses `motion` (vanilla `inView`/`animate`, not framer-motion/React, no framework
  island), not a hand-rolled `IntersectionObserver` + CSS-class-toggle + `@keyframes`.** The
  hand-rolled version was a confirmed cause of a real scroll-jump bug (fast scroll → page briefly
  jumps ~1 viewport, then corrects) — root-caused by removing the whole reveal system, confirming
  the jump was gone, then reintroducing it via `motion`. DevTools' Layout Shift Regions showed
  zero shift flashes (ruling out CLS/reflow); working theory is a compositor-level race — many
  `.reveal` elements crossing their intersection threshold in the same scroll gesture each
  spinning up a fresh CSS `animation` layer at once, desyncing the compositor's scroll-position
  thread from main-thread paint. `motion`'s `animate()` schedules a WAAPI animation per element
  directly, which the browser handles more predictably under concurrent-start conditions. If this
  regresses: don't go back to IntersectionObserver+classList; try native
  `animation-timeline: view()` (zero JS, compositor-only, structurally can't have this race)
  before reintroducing a JS-driven trigger. Verify any fix with a real browser scroll, not
  synthetic/automated wheel events — Playwright's `mouse.wheel()` (discrete jumps, small-delta
  trackpad-style, momentum coast-down all tried) never reproduced this bug, only an actual user
  session did.
- **Never set `html { scroll-behavior: smooth }` globally.** Intercepts *all* scrolling, not just
  anchor-link nav — on fast wheel/trackpad input, Chrome queues/overlaps a smooth-scroll animation
  per tick, reading as overshoot-and-snap-back. (Not the actual cause of the scroll-jump bug above,
  but a correctness issue in its own right — left removed.) Anchor-link smooth scroll is
  implemented in `Layout.astro` instead: a scoped `click` handler on `a[href^="#"]` calls
  `target.scrollIntoView({ behavior: 'smooth' })` per-click, a one-shot JS call that doesn't fight
  ordinary scroll input.
- `history.scrollRestoration = 'manual'` set synchronously at top of `<head>` in `Layout.astro` —
  refresh never restores a stale pre-refresh scroll position (single-page site, no "resume where I
  left off" case). Not the scroll-jump cause either, but correct to keep.
- Hero's decorative particles/glow blobs pause via `animation-play-state` (`IntersectionObserver`
  on `<section data-hero>` toggles `data-in-view`) the instant Hero scrolls out of view — previously
  animated unconditionally forever (wasted compositor work). `Nav.astro`'s scrolled state
  (`data-scrolled`) uses solid `bg-shell`, not `bg-shell/95 backdrop-blur` — toggling
  `backdrop-filter` on a `position: fixed` element mid-scroll is a well-documented Chromium jank
  source, avoided even though not confirmed as this bug's cause.
- Every `@fontsource` weight imported in `Layout.astro` is preloaded (`preloadFonts` array) — a
  partial preload (just hero weights) let other weights swap in after first paint, causing visible
  layout jitter (Public Sans is the body font almost everywhere, so its swap reflows most of the
  page). Keep this list in sync when a new weight is imported.
- CTAs point to `https://workspace.nyayops.in` — hardcoded, no env var (fully static site, no
  build-time config surface today).
- `astro.config.mjs`'s `site: 'https://nyayops.in'` feeds canonical URL, OG tags, sitemap
  (`@astrojs/sitemap`) — keep in sync if the domain changes.

## Known debt / open items

- Stats/testimonials/social-proof intentionally absent — no real numbers/quotes yet. Don't invent
  placeholders; add once real data exists.
- `public/og-image.png` is a real Playwright screenshot of the hero (1200×630), not a designed OG
  card — fine for now, worth a proper design pass later.
- Not yet deployed — no Vercel project, no DNS for `nyayops.in`.
