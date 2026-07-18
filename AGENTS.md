# AGENTS.md — NyayOps marketing landing page

Public marketing site for NyayOps, meant to live at the bare `nyayops.in` domain. Separate
project from `frontend/` (the dashboard SPA at `workspace.nyayops.in`) on purpose — this is a
brand/marketing surface (SSG, SEO, fast first paint), not an authenticated app; see root
`CLAUDE.md`'s workspace table for how the two relate.

**Not yet wired as a git submodule** (unlike `backend v1/`, `frontend/`,
`Court Data Extractor/court-data-service/`) — it's a local-only directory for now. Turning it into
one (new GitHub repo under the org, `.gitmodules` entry, Vercel project, DNS for `nyayops.in`) is a
deliberate next step that needs the user's go-ahead before creating remote resources, not something
to do silently.

## Commands

```bash
npm install
npm run dev        # Astro dev server, localhost:4321 (add --background to run detached;
                     # manage with `astro dev stop` / `astro dev status` / `astro dev logs`)
npm run build       # astro build -> dist/ (static)
npm run preview     # serve the production build locally
```

No lint/test tooling configured — `astro build` (which runs Astro's own type checking) is the only
automated gate. If a test suite or linter is added later, record it here.

## Architecture

- `src/pages/index.astro` — the entire site is one long-scroll page. Add new routes here if the
  site grows beyond one page (e.g. a `/security` or `/pricing` page).
- `src/components/` — one component per section (`Hero.astro`, `Features.astro`,
  `HowItWorks.astro`, etc.), assembled in `index.astro`. `Wordmark.astro` and `Nav.astro` are
  shared chrome.
- `src/layouts/Layout.astro` — HTML shell: SEO meta/OG tags, self-hosted font imports, and the
  page-wide `<script>` (`.reveal` scroll animation via `motion`'s `inView`/`animate` + mobile nav
  toggle + nav scroll-state + anchor-link smooth scroll + scroll-restoration override).
- `src/styles/tokens.css` — **ported verbatim** from `frontend/src/tokens.css` (the "Ledger Blue"
  OKLCH palette). If the brand palette changes, update `frontend/src/tokens.css` first and mirror
  the change here — this file has no other source of truth.
- `src/styles/global.css` — Tailwind v4 `@theme` (fonts/radius/shadow/z-index/motion) + base
  styles. Same font stack as the dashboard (Outfit/Public Sans/JetBrains Mono) via
  `@fontsource/*` packages (self-hosted, not the Google Fonts CDN link `frontend/index.html`
  uses — that was a real Lighthouse performance regression here: render-blocking cross-origin
  font CSS cost ~15 perf points before the switch).
- `src/assets/screenshots/` — real product screenshots (dashboard, case list), captured via
  Playwright against a locally-seeded staging stack, not stock imagery or mockups. If the product
  UI changes meaningfully, these will drift and should be recaptured.

## Conventions / invariants

- **Brand register, not product register.** This project follows `impeccable`'s brand rules
  (`~/.claude/skills/impeccable/reference/brand.md`), not the product rules `frontend/` follows —
  bolder color commitment is expected and intentional (the navy-drenched hero/CtaBand/HowItWorks
  sections vs. the Restrained white sections between them). Don't flatten this to match the
  dashboard's uniformly-Restrained chrome.
- **Reused tokens are contrast-verified only against the surfaces `frontend/DESIGN.md` verified
  them against** (`--color-ink-muted` ≥4.5:1 on `--color-surface`/white specifically). Using
  `ink-muted` body text on the tinted `--color-bg`/`--color-surface-muted` backgrounds drops
  below 4.5:1 and fails WCAG AA — use `text-ink` (full ink) for body copy on those tints instead.
  Learned the hard way via a Lighthouse a11y regression during the initial build; re-check with
  Lighthouse/axe before reusing a token pairing not already used elsewhere in this project.
- **`Wordmark.astro`'s default colors (`shell-ink`/`accent`) are calibrated for the navy shell
  only** (mirrors `frontend/src/components/layout/Wordmark.tsx`'s own constraint). Pass
  `onLight` when placing it on a white/near-white background (e.g. `Footer.astro`) — the
  unguarded default is a near-invisible white-on-white bug, not a style choice.
- **`.reveal` must never gate content visibility.** Content is visible by default (`opacity`
  unset in CSS); the fade-up-on-scroll-into-view effect is layered on top via `motion`'s
  `inView`/`animate` in `Layout.astro` (WAAPI, not a CSS class toggle — see below for why). Don't
  set `.reveal { opacity: 0 }` anywhere — full-page/headless captures and paused-tab renders don't
  reliably fire the JS that would bring it back to 1, so gated sections would ship visually blank
  (happened once during the initial build; keep the fix).
- **`.reveal` uses `motion` (the vanilla `inView`/`animate` package, not framer-motion/React — no
  framework island), not a hand-rolled `IntersectionObserver` + CSS-class-toggle +
  `@keyframes`.** The original hand-rolled version was a confirmed cause of a real scroll-jump bug
  (fast scroll -> page briefly jumps ~1 viewport, then corrects) — root-caused by removing the
  whole reveal system and having the user confirm the jump was gone, then reintroducing it via
  `motion` instead. Chrome DevTools' Layout Shift Regions showed zero shift flashes throughout
  (ruling out CLS/reflow); the working theory is a compositor-level race — many `.reveal` elements
  crossing their intersection threshold in the same scroll gesture each spinning up a fresh CSS
  `animation` layer at once, desyncing the compositor's independent scroll-position thread from
  main-thread paint. `motion`'s `animate()` schedules a WAAPI animation per element directly
  instead, which the browser handles more predictably under concurrent-start conditions. If this
  regresses: don't go back to IntersectionObserver+classList; try native
  `animation-timeline: view()` (zero JS, runs purely on the compositor, structurally can't have
  this race) before re-introducing any JS-driven trigger. Also: **verify any fix here with a real
  browser scroll, not synthetic/automated wheel events** — Playwright's `mouse.wheel()` (discrete
  jumps, small-delta trackpad-style, and momentum coast-down were all tried) never reproduced this
  bug once, only an actual user session did.
- **Never set `html { scroll-behavior: smooth }` globally.** It intercepts *all* scrolling, not
  just anchor-link navigation — on fast wheel/trackpad input, Chrome queues/overlaps a
  smooth-scroll animation per wheel tick, which can read as the page overshooting and snapping
  back. (This turned out not to be the actual cause of the scroll-jump bug above, but it's a
  correctness issue in its own right — leaving it removed.) Smooth-scroll for the nav/hero anchor
  links is implemented in `Layout.astro` instead: a scoped `click` handler on `a[href^="#"]` calls
  `target.scrollIntoView({ behavior: 'smooth' })` per-click, which doesn't fight with ordinary
  scroll input because it's a one-shot JS call, not a persistent CSS mode.
- `history.scrollRestoration = 'manual'` is set synchronously at the top of `<head>` in
  `Layout.astro`, so refreshing never restores a stale pre-refresh scroll position — this is a
  single-page site with no meaningful "resume where I left off" case. (Also not the cause of the
  scroll-jump bug above, but correct to keep regardless.)
- Hero's decorative particles/glow blobs pause via `animation-play-state` (an
  `IntersectionObserver` on the `<section data-hero>` toggles `data-in-view`) the instant Hero
  scrolls out of view — they were previously animating unconditionally forever, which is wasted
  compositor work for content nobody can see. `Nav.astro`'s scrolled state (`data-scrolled`) uses
  a solid `bg-shell`, not `bg-shell/95 backdrop-blur` — toggling `backdrop-filter` on a
  `position: fixed` element mid-scroll is a separately well-documented Chromium jank source, worth
  avoiding even though it wasn't confirmed as this bug's cause either.
- Every `@fontsource` weight actually imported in `Layout.astro` is preloaded (see the
  `preloadFonts` array) — a partial preload (just the hero's weights) still let other weights
  swap in after first paint, which contributed to visible layout jitter (Public Sans is the body
  font almost everywhere, so its swap reflows most of the page at once). Keep this list in sync
  if a new weight is imported.
- CTAs point to `https://workspace.nyayops.in` (dashboard login/app) — hardcoded, no env var,
  since this is a fully static site with no build-time config surface today.
- `astro.config.mjs`'s `site: 'https://nyayops.in'` feeds the canonical URL, OG tags, and the
  sitemap (`@astrojs/sitemap`) — keep it in sync if the domain ever changes.

## Known debt / open items

- Stats/testimonials/social-proof are intentionally absent — no real numbers or customer quotes
  exist yet. Don't invent placeholder numbers; add them once real data exists.
- `public/og-image.png` is a real Playwright screenshot of the hero (1200×630), not a designed OG
  card — fine for now, worth a proper design pass later.
- Not yet deployed anywhere; no Vercel project, no DNS for `nyayops.in` pointed at it yet.
