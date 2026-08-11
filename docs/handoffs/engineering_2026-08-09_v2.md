# Engineering fix pass — K13 Website (`index.html`)

**Date:** 2026-08-09 (v2) · **Engineer:** Natalia (Frontend) · **Scope:** `index.html` + `favicon.svg` only

Follow-up to `engineering_2026-08-09.md` / `compliance_2026-08-09.md` / `security_2026-08-09.md` /
`report_2026-08-09.md`. All eight items from the fix ticket are addressed below, verified live in
Chromium via Playwright (not inferred from the diff).

## 1. Contact stepper — the lead-capture bug

- **Invisible-but-focusable tab stops, closed.** Each `.screen-s` not currently active now carries
  the `inert` attribute (toggled in `render()`: `s.inert = i !== idx`), and `#stepBack` is a real
  `disabled` button at step 1 instead of an opacity-hidden one. Both remove the element from the tab
  order, the accessibility tree, and pointer/keyboard activation — not just from view — while leaving
  the existing opacity/transform slide transition untouched (CSS wasn't touched for this; `inert` is
  transition-agnostic).
- **`#sendIt` can no longer fire with empty data.** Verified two ways in Chromium:
  - Natural tab order: with the stepper on step 1, `#sendIt` is unreachable by Tab at all (it lives
    inside an `inert` ancestor). Confirmed via `document.activeElement`.
  - Defense-in-depth: even force-invoking the click handler directly (bypassing `inert`/`disabled` via
    script, simulating a regression) does **not** show the success panel — the handler now guards
    `if(!data.type || !data.name){ go(...); return; }` and redirects to the incomplete step instead.
- **The false "email sent" claim is gone.** `#sendIt` no longer asserts success before `mailto:` even
  opens. New copy: *"Opening your mail app…"* → if `blur`/`visibilitychange` fires (heuristic for the
  OS handing off to a mail client) it updates to *"Your mail app opened."* If nothing takes focus
  within ~1.9s of the click (no registered mail handler), a fallback line appears with a direct,
  clickable `mailto:` link and the plain email address — an actual next step instead of a false
  positive. `#sent` also got `role="status" aria-live="polite"` so screen readers hear the outcome.

**Proof (Playwright, Chromium, 1440×900):**
```
Zero invisible-but-focusable stepper controls: PASS (0 found; only the 4 visible step-1 .opt buttons are reachable)
#sendIt unreachable by Tab from step 1: PASS (blocked by inert ancestor)
Forced #sendIt click with empty data → success panel shown: PASS (did NOT show; redirected to step 1)
```

## 2. Form labels
Added real `<label class="vh" for="...">` (visually hidden, standard clip-rect pattern) for `#sName`,
`#sEmail`, `#sWhy`. Verified via `el.labels[0].textContent` in the live DOM: `"Your name"`,
`"Your email address"`, `"Why this project matters to you"` — all present.

## 3. Landmarks
Added `<main id="main" tabindex="-1">` wrapping hero→contact, and a skip link
(`<a class="skip-link" href="#main">Skip to main content</a>`) as the first focusable element in
`<body>`, visually hidden until focused. `tabindex="-1"` on `<main>` lets native fragment navigation
actually move focus there (not just scroll), per the standard skip-link pattern. Verified:
`document.querySelector('main#main')` exists; skip link `href="#main"`.

## 4. Contrast — computed before/after
All fixes are **token-level, hue-preserving darkenings** (uniform RGB scale-down — mathematically
invariant to HSV hue/saturation, only moves the "value" axis) or a solid backing chip. No palette
swap. Computed with the WCAG relative-luminance formula, cross-checked in a script, worst-case
background for each token used.

| Element / token | Before | After | New value | Needs |
|---|---|---|---|---|
| `--faint` (placeholders, `.rnum`, `.brand .tag`, `.step-lab`, `.s-skip`, `.status .yr/.now`, `.step-sum .sc b`, `.cap-col .stk em`) — worst bg `--paper-3` | **2.98–3.46** | **4.55–5.28** | `#8089A1` → `#646B7E` | 4.5 |
| `--blue` below 24px (eyebrows, `.row .go`, `.contact .mail b`, `.btn.ghost:hover`) — worst bg `--paper-3` | **3.89–4.18** | **4.55–4.89** | `#CB4D14` → `#B94612` | 4.5 |
| white on `--blue` (buttons, `.s-next`) | 4.554 (razor-thin) | **5.33** (improved, not worsened) | same token | 4.5 |
| `.p-cap span` (9px caption over portrait) | **1.22 worst / 2.84 mean** | **5.33** (deterministic — opaque backing, no longer photo-dependent) | solid `var(--paper-2)` chip added behind caption | 4.5 |
| `.p-cap b` (20px caption over portrait) | 3.01 worst / 10.43 mean | **16.71** (same chip) | — | 3.0 (large text) |
| `.sent .ring` check glyph, 27px | **2.62** | **3.33** | `#27B584` → `#229F74` | 3.0 (large text) |
| Non-text control boundaries — `.s-field` underline, `.opt` border, `.btn.ghost` border, `.step-prog i` — new `--line-3` token, worst bg `glass` | **1.58** | **3.40–3.48** | `rgba(20,29,53,.52)` (new token, added alongside existing `--line`/`--line-2` which stay untouched on purely decorative borders) | 3.0 |

Decorative-only uses of `--line-2` (peek preview frame, portrait frame border, work-index divider,
mobile thumbnail border) were deliberately left alone — SC 1.4.11 doesn't apply to pure decoration,
and touching them would have widened the diff for no compliance benefit.

## 5. Status feed — SC 2.2.2 pause control
Added a `Pause`/`Play` toggle button next to "Currently" in the status card. It stops/restarts the
`setInterval` that rotates the feed text and updates its own `aria-pressed`/`aria-label`. It carries
`aria-hidden="false"` to stay operable even though its parent `<aside>` is `aria-hidden="true"`
(unchanged — compliance's existing mitigating-factor read on that card was left as-is; this only adds
the missing control for sighted users). Hidden entirely under `prefers-reduced-motion` (rotation is
already stopped there, so a toggle for nothing would be confusing UI).

## 6. Dead code removed
`.scrawl`, `.scrawl path,.scrawl ellipse`, `.draw`, `.hero h1 .ci .scrawl`, the reduced-motion block's
stray `.draw{stroke-dashoffset:0}`, and the orphaned `<svg><filter id="pencil">` defs block — all
gone. Grepped the final file for `scrawl|pencil|\.draw\b`: zero matches.

## 7. External script hardening
- Lenis: added `integrity="sha384-tKsJDT6PlUI0pSBt9/sBKJluKgA19/a6mBrDsZaXotLB4ZYfMGM6xt6/WgGpYhTm"` +
  `crossorigin="anonymous"`. The hash was computed locally from the actual fetched file
  (`openssl dgst -sha384`), then cross-verified against jsDelivr's own published file hash via their
  data API (`data.jsdelivr.com/v1/package/npm/lenis@1.1.18`) — sha256 of the downloaded bytes matched
  jsDelivr's listed hash exactly, confirming the file wasn't tampered with in transit before hashing.
- All 10 `target="_blank"` work-index links: `rel="noopener"` → `rel="noopener noreferrer"`.

## 8. P5 head items
Added `<link rel="canonical" href="https://k13projects.com/">`, full OG (`og:type`, `og:site_name`,
`og:url`, `og:title`, `og:description`, `og:image`) and Twitter Card meta (reusing the existing
description copy and `assets/kazim.jpg` as the share image — no new image asset needed). Replaced the
inline data-URI favicon with a real file, `favicon.svg`, containing a `prefers-color-scheme: dark`
block that swaps the background/text/accent colors for dark mode — one favicon, both themes, no extra
`<link>` needed.

## Verification run
Playwright + Chromium, served locally via `python3 -m http.server 8931` (port outside 9130–9199 and
away from 5000/7000), server killed after the run.

```
Zero console errors (1440):                        PASS
Zero 4xx/5xx responses:                             PASS
No horizontal overflow @1440:                       PASS (scrollWidth - clientWidth = 0)
No horizontal overflow @390:                        PASS (scrollWidth - clientWidth = 0)
Zero invisible-but-focusable stepper controls:      PASS
#sendIt unreachable from step 1 / guarded if forced: PASS
Lenis never initialised under reduced-motion:        PASS (html.lenis absent; window.Lenis loaded but unused)
Transitions collapse under reduced-motion:           PASS (.card transition-duration = 0.001s)
#sName / #sEmail / #sWhy have associated <label>s:   PASS
<main id="main"> + skip link present:                PASS
```
Visual sanity screenshots taken at 1440 (hero, about/portrait, contact stepper) and 390 (hero) —
caption chip, pause pill, and stepper option borders all read cleanly; no layout regressions spotted.

## What was NOT touched (explicitly out of scope)
- The `innerHTML` self-XSS sink in the stepper's summary chips (`index.html` step-sum rendering) —
  flagged Low/self-XSS-only in `security_2026-08-09.md`, not listed in this ticket's 8 items. Left
  alone to keep the diff scoped; worth a follow-up pass.
- `SKIP`/`‹ BACK` touch-target size (24×24) — noted by compliance as a smaller finding, not in this
  ticket's 8 items.
- Brand-canon decision (README/brand vs shipped `beta_v7`), `pricing/tiger/` fate, Google
  Fonts/jsDelivr EU exposure — all explicitly Kazim's calls per the compliance/report handoffs.
- `vercel.json`, `robots.txt`, `sitemap.xml`, `404.html` — Kate's concurrent work; confirmed via
  `git status` that none of the four were created or modified by this pass.

## Files
- `/Users/k13/Desktop/PROJECTS/K13_Website/index.html` (modified — CSS tokens, stepper markup/CSS/JS,
  form labels, `<main>`+skip link, status feed pause control, dead-code removal, SRI + `noreferrer`,
  head meta/favicon reference)
- `/Users/k13/Desktop/PROJECTS/K13_Website/favicon.svg` (new — dark-mode-aware SVG favicon)

---

## Addendum 2026-08-10 — skip-link focus regression (Olga's adversarial re-run)

Olga's re-test found the skip link was DOM-correct but functionally inert: the pre-existing global
`a[href^="#"]` click handler (`index.html` ~line 667) intercepted its click, called
`preventDefault()`, and only scrolled — it never moved focus, so native fragment-navigation focus
(the thing that would normally land focus on `<main>`) never happened either. Proven: Tab → Enter →
Tab landed back on the nav `.brand` link, i.e. the pre-fix behaviour.

**Fix:** exempted `.skip-link` from the shared handler
(`document.querySelectorAll('a[href^="#"]:not(.skip-link)')`) and gave it its own click listener that
calls the same `scrollTo()` used everywhere else, then `el.focus({preventScroll:true})` on `<main>`.
Ordinary in-page anchors are untouched — same selector minus one element, same `scrollTo()` function,
zero behavior change for them.

**Verified live in Chromium (Playwright, port 8932, killed after):**
```
First Tab stop is the skip link:                          PASS
After Enter, focus is on <main>:                           PASS ({'tag':'MAIN','id':'main'})
Next Tab after Enter lands INSIDE <main>, not nav/header:   PASS (lands on the "Start a project" hero button)
No console errors during skip-link flow:                   PASS
Ordinary nav anchor (#work) still scrolls via Lenis:        PASS (scrollY 0 -> 855 mid-animation -> 1590 settled)
Ordinary anchor click does not steal focus onto the section: PASS
Lenis still never initialised under reduced-motion:         PASS
Skip link still moves focus into <main> under reduced-motion: PASS
Transitions still collapse under reduced-motion:            PASS (0.001s)
```
`git status` confirms only `index.html` changed this pass — no other files touched.

## Status      PASS
## Summary     Fixed the skip-link focus regression Olga found: it's now exempted from the shared smooth-scroll anchor handler and explicitly calls el.focus({preventScroll:true}) on <main> after scrolling. Verified live: Tab->Enter->Tab now lands inside <main> (not back in nav), ordinary anchors still smooth-scroll via Lenis unaffected, reduced-motion behavior unchanged. All 8 original ticket items from the prior pass still hold (stepper bug proven dead three ways per Olga).
## Files       /Users/k13/Desktop/PROJECTS/K13_Website/index.html (modified — skip-link click handler only)
## Risks       none new — this was a scoped regression fix with no behavior change to ordinary anchors.
## Next        qa-test-engineer
## Human gate  none
