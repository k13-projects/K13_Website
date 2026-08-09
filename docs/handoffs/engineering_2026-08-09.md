# Engineering review — K13 Website (`index.html`)

Scope: assess-only code review of `index.html` (831 lines, single file, no build step), `assets/`, and `scripts/*.sh`. No files were modified.

## Document structure & semantics — clean

Heading order is correct and unbroken: one `<h1>` (`index.html:345`) → four `<h2>` (384, 417, 493, 566) → seven `<h3>` (391, 397, 403, 499, 509, 519, and the dynamic confirmation heading at 617). No skipped levels. `<html lang="en">`, charset first, viewport meta present. Landmarks (`header`, `nav`, `section`, `footer`) are used correctly. Links use real `<a href>` (not `<div onclick>`), buttons use `<button type="button">` — good baseline semantics throughout.

**Gap:** the three contact-stepper inputs (`#sName` line 599, `#sEmail` line 605, `#sWhy` line 611) have no `<label>` — the only accessible name comes from the `placeholder`, which most screen readers do expose but which is not a reliable accessible name per WCAG 2.4.6/3.3.2, and disappears the moment the user types. Worth a visually-hidden `<label>` per field, especially since the capabilities section markets `WCAG · 508 access` (`index.html:525`) as a service line.

No Open Graph / Twitter Card meta tags — only `<meta name="description">` (line 7). Not a defect, but a real gap if this page is ever shared on socials/Slack/iMessage (no preview image or title control). Flagging since it's part of `<head>` document structure; treat as P5 polish, not a bug.

## Images — mixed

- `assets/kazim.jpg` (`index.html:539`): correct `alt`, `loading="lazy"`, `decoding="async"`, and a CSS `aspect-ratio:3/4` that matches the source's native 900×1200 ratio — no CLS risk, sensible choice.
- `assets/kazim 2.jpg` (164 KB) and root-level `Kazim Image.png` (1.9 MB) are **not referenced anywhere** in `index.html` — orphaned assets sitting in the repo (the PNG in particular is dead weight at nearly 2 MB, sitting outside `assets/` entirely).
- `assets/shots/*.jpg` (10 files, 2.5 MB total, 93 KB–522 KB each) are all native **1500×938**. They're displayed at two sizes only: the 340px-wide desktop hover preview (`.peek img`, `index.html:59-65`, populated via JS on `mouseenter`, line 756) and the full-row mobile/touch inline thumbnail (`.rthumb`, `index.html:295-296`, inserted via JS at 734-743). Both are 3-4× oversized relative to their rendered box, there's no `srcset`/`sizes`, and everything is JPEG (no WebP/AVIF). `thg-hero.jpg` alone is 522 KB for what's ultimately a decorative hover card. Real savings available here (resize to ~800px wide, re-encode WebP — should cut ~2.5 MB down to a few hundred KB) but it's not a first-load cost today: the desktop preview only fetches on hover, and the mobile thumbnails carry `loading="lazy"` correctly, so nothing here blocks first paint.
- No `width`/`height` attributes on any `<img>`, but every image context has a matching CSS `aspect-ratio` (16/10 for thumbnails/peek, 3/4 for the portrait), so this doesn't cost CLS in practice — just non-standard, not broken.

## CSS — organized, two dead rules

The stylesheet (lines 12-307) is well-sectioned with comments per component and no obvious specificity fights. One concrete dead-code finding:

- `.scrawl` / `.scrawl path,.scrawl ellipse` / `.draw` (`index.html:53-56`) and the referencing rule at `index.html:111` (`.hero h1 .ci .scrawl`) target a hand-drawn "pencil scrawl" SVG accent that no longer exists anywhere in the HTML body — I grepped the full file and there is no element with `class="scrawl"` or `class="draw"`. The `<svg><filter id="pencil">` definition (`index.html:311-316`) that these rules depend on is likewise now orphaned. This is leftover from an earlier design pass (the header comment at line 14 references "hand-drawn accents used only as annotation") — safe to delete, ~10 lines across the CSS and the SVG defs block.
- `--wash-blue/-lilac/-mint/-peach` tokens (line 24) are all consumed (lines 319-321, 393, 399, 405) — not dead, just flagging I checked.

## JS — defensively written, one real correctness issue

The vanilla IIFE (`index.html:636-828`) is disciplined: every `getElementById`/`querySelector` result that could plausibly be null is guarded (`if(!feed||reduced)return`, `if(!wrap||reduced)return`, `if(!peek||...)return`, etc. at lines 674, 748, 764), event listeners use `{passive:true}` where appropriate, and the `IntersectionObserver` reveal (658) correctly `unobserve`s after firing so it doesn't leak. `prefers-reduced-motion` is checked once (639) and threaded through the count-up, feed rotation, and portrait-depth effects — matches the CSS `@media(prefers-reduced-motion:reduce)` fallback at line 303. I did not find anything that would throw at runtime; every referenced `id` in JS exists in the markup (`stepBack`, `stepLab`, `stepProg`, `stepSum`, `stepStage`, `sent`, `sName`, `sEmail`, `sWhy`, `nameNext`, `emailNext`, `emailSkip`, `sendIt` all confirmed present).

**Real issue — optimistic send confirmation (`index.html:816-825`):** clicking "Send it" immediately shows `.sent` ("Your email is ready... Your mail app just opened") via `sent.classList.add("on")`, then 700ms later sets `window.location.href="mailto:..."`. There is no way to detect whether a mail client actually opened — on a machine/browser with no registered mail handler (increasingly common; webmail-only users, some Chromebooks, sandboxed browsers) the user sees a false-positive success screen and nothing happens. This is the site's only conversion path (no backend form), so it's worth a fallback (e.g., also show the raw mailto/email address as a copyable string, or open it in the same tick and let the confirmation only fire after a `visibilitychange`/`blur` heuristic).

**Minor:** the email field's value is used as-is (`emailI.value.trim()`, line 813/815) despite `type="email"` — there's no `.checkValidity()`/format check before it's folded into the mailto body, so a malformed address is silently accepted. Low severity since the field is skippable anyway (line 606, 814).

**Minor / inert markup:** 24 elements carry a `data-c` attribute (e.g. `index.html:327, 332-335, 350-351...`) that is never read anywhere in the script — grepped for `data-c` usage and only false-positive matches against `data-count` turned up. Looks like a leftover hook (analytics/click-tracking?) that was never wired up or already removed. Harmless (unknown attributes are inert in HTML) but worth a decision: wire it up or strip it.

## Responsive behaviour — as-written, looks sound

Breakpoints at 920px (primary reflow: hero grid, cards, about grid, work-index columns), 880px (contact stepper), and 600px (nav link hiding) are consistent and mobile-first in intent. The work-index row grid switches from an explicit 4-column desktop layout (`index.html:144`) to a 2-column layout with `grid-column` overrides for `.rdesc`/`.rmeta` (289-291) — I traced the implicit auto-placement by hand and it resolves cleanly (number top-left, name top-right, description under the name, meta spanning full width beneath), but it's a layout that depends on DOM order matching this exact auto-placement outcome; reordering the row's children later would silently break it with no visual regression test to catch it. Not a bug today, just fragile.

The touch/hover fallback (`@media(hover:none),(max-width:920px)` at line 294) correctly swaps the desktop cursor-preview interaction for an inline thumbnail — good coverage for the "hover doesn't exist on mobile" case, which is easy to miss.

`prefers-reduced-motion` fallback exists and is correct (line 303-306): kills animation/transition duration, forces `.rv` and `.draw` to their resolved end-state instead of just shortening — this is the right pattern, not just a duration hack.

## Performance — one real render-blocking cost, otherwise lean

- The Google Fonts stylesheet (`index.html:10`) is a classic render-blocking `<link rel="stylesheet">`. `preconnect` hints (8-9) shave connection setup but the CSS itself still gates first paint; `display=swap` in the URL avoids invisible-text (FOIT) but there will be a visible font swap (FOUT) on first load. This is the only genuinely render-blocking resource in the document — Lenis is `defer`red (635) and the inline script sits at the end of `<body>`.
- No hero image — the hero is pure text/CSS, which keeps LCP cheap. Good call for a portfolio site.
- Lenis is loaded from `cdn.jsdelivr.net` pinned to an exact version (`lenis@1.1.18`) but without a `integrity` (SRI) hash — a supply-chain best-practice gap for the one third-party script in the page. Functionally the code degrades gracefully if it fails to load (`if(lenis) ... else el.scrollIntoView(...)`, line 647), so this is a hardening note, not a functional risk.
- Image weight is discussed above under Images — not a first-load cost (hover-gated + lazy-loaded), but real bytes sitting in the repo that would matter the moment any of those images get promoted to eager/above-the-fold use.

## `scripts/*.sh` — brief pass, no issues found

`hm.sh`, `new-branch.sh`, `setup-identity.sh`, `ship.sh` are small, `set -e`'d, and do what their comments say (branch/commit/push, PR+merge, git identity, push+PR). `hm.sh` does `git add -A` before committing (line 18 of `hm.sh`) — consistent with the house Hail Mary convention, just noting it stages everything indiscriminately, so anything untracked in a dirty tree rides along. No logic bugs spotted.

## Repo hygiene (not code defects, but worth a note)

- `pricing/tiger/` exists in the project root but is not linked from anywhere in `index.html` — orphaned directory of unclear current purpose.
- `README.md` documents an older brand system (Ink `#15130F` / Paper `#F1ECE2` / signal orange `#FF4F00`, Space Grotesk + Fraunces Italic) that no longer matches what's shipped in `index.html` (`--paper:#F3F5FA`, `--ink:#141D35`, `--blue:#CB4D14`, Fraunces + Inter + JetBrains Mono — labeled `beta_v7 "THE DRAFTING TABLE"` at `index.html:14`). Not a code bug, but the docs and the shipped design have drifted; a new contributor reading the README would build the wrong palette.

## Files inspected

- `/Users/k13/Desktop/PROJECTS/K13_Website/index.html` (full, 831 lines)
- `/Users/k13/Desktop/PROJECTS/K13_Website/scripts/hm.sh`
- `/Users/k13/Desktop/PROJECTS/K13_Website/scripts/new-branch.sh`
- `/Users/k13/Desktop/PROJECTS/K13_Website/scripts/setup-identity.sh`
- `/Users/k13/Desktop/PROJECTS/K13_Website/scripts/ship.sh`
- `/Users/k13/Desktop/PROJECTS/K13_Website/README.md`
- `/Users/k13/Desktop/PROJECTS/K13_Website/assets/` (listing + `file`/size audit of `assets/shots/*.jpg`, `assets/kazim.jpg`, `assets/kazim 2.jpg`)
- `/Users/k13/Desktop/PROJECTS/K13_Website/pricing/`, `/Users/k13/Desktop/PROJECTS/K13_Website/brand/` (listing only)

## Status      NEEDS-REVIEW
## Summary     index.html is well-structured (clean heading order, guarded JS, correct reduced-motion fallback) with no functional breakage found. Real findings: dead `.scrawl`/`.draw`/`#pencil` CSS+SVG, an optimistic "email sent" confirmation that can lie to users with no mail client, missing `<label>`s on the 3 contact-form inputs, oversized/unreferenced images (1.9MB orphaned PNG, 2.5MB of 3-4x oversized shot JPEGs), and a stale README describing a different brand system than what's shipped.
## Files       docs/handoffs/engineering_2026-08-09.md (created); index.html, scripts/*.sh, README.md, assets/ (inspected only, no changes)
## Risks       The optimistic mailto "send" confirmation is the site's only conversion path and can silently fail for users without a registered mail client — worth prioritizing over the cosmetic items. Missing form labels are a real WCAG gap on a site that markets WCAG/508 work as a capability.
## Next        qa-test-engineer
## Human gate  none
