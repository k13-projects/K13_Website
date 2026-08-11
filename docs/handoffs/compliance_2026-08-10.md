# K13 Website — accessibility re-verification (fix pass 2026-08-09 v2)

**Date:** 2026-08-10 · **Stage:** compliance (re-verify only — no project file was modified)
**Target:** `/Users/k13/Desktop/PROJECTS/K13_Website/index.html` (873 lines, working tree, uncommitted) + `404.html`
**Under test:** the claims in `docs/handoffs/engineering_2026-08-09_v2.md` against the findings in `docs/handoffs/compliance_2026-08-09.md`
**Standard applied:** WCAG 2.2 Level AA

## How this was re-checked

Nothing was taken on trust. Every number below was re-measured today in real Chromium via Playwright,
served from `python3 -m http.server 8477` (bound to `127.0.0.1`, killed at the end of the run).

1. **Contrast was measured from rendered pixels, not from CSS.** For each target the element is
   screenshotted twice — once normally, once with its text forced to `transparent` — and the two
   frames are differenced. The pixels that changed *are* the glyphs; the corresponding pixels in the
   second frame are the exact background those glyphs sit on. That excludes borders, `::before`
   rules, arrows and chip edges, which is what made the first naive sweep produce nonsense. The same
   trick with `border-color: transparent` isolates border pixels for the non-text checks.
2. **Real keyboard traversal** — `Tab` pressed with a 450 ms settle after each stop so scroll-reveal
   `IntersectionObserver` callbacks finish before `document.activeElement` and its *effective*
   ancestor opacity are read. (Without the settle you get six false "invisible" stops that are just
   mid-reveal. Worth knowing for whoever runs this next.)
3. **Chromium accessibility tree** via CDP `Accessibility.getFullAXTree` / `getPartialAXTree` for
   landmarks, accessible names, and — decisively — `ignoredReasons`.
4. **axe-core 4.10.2** run over `wcag2a/2aa/21a/21aa/22aa` on both pages as an independent second
   opinion. It agreed with the pixel work in both directions.

---

# Verdict up front

| # | Level A failure from 2026-08-09 | Claimed | Verified today |
|---|---|---|---|
| 1 | 8 invisible-but-focusable stepper tab stops | fixed | **Partly.** Closed on load — reopens after send |
| 2 | `#sendIt` fires from step 1 with an empty payload | fixed | **CLOSED** |
| 3 | No `<main>`, no skip link | fixed | **`<main>` yes. Skip link is inert decoration — 2.4.1 still fails** |
| 4 | 3 form fields named by `placeholder` only | fixed | **CLOSED** |

**2 of the 4 genuinely closed. 1 partly. 1 not closed.**

**All 13 computed contrast failures are genuinely closed** — plus the portrait caption, plus all four
non-text (1.4.11) boundaries. That work is real and it holds up under pixel measurement. Two of the
thirteen land on a 0.04 margin (below).

**Two new Level A / AA defects were introduced by the fixes themselves**, and `404.html` re-introduces
two of the contrast failures that were just removed from `index.html`.

---

## 1. The eight invisible tab stops — closed on load, reopened after send

**On load: genuinely fixed.** Full traversal at 1440×900, 26 stops from skip link to footer, every
stop settled before reading:

```
 1 A.skip-link  op=1.00   2 A.brand  op=1.00   3-5 A.nl ×3  op=1.00
 6-7 A.btn ×2   op=1.00   8 A.btn.ghost op=1.00   9 BUTTON#feedToggle op=1.00
10-19 A.row ×10 op=1.00  20 A.mail op=0.97  21 A(phone) op=1.00
22-25 BUTTON.opt ×4 op=1.00  26 A(footer mail) op=1.00  27 BODY — end
invisible-but-focusable: 0
```

`#sName · #nameNext · #sEmail · #emailNext · #emailSkip · #sWhy · #sendIt` all report
`closest('[inert]') === true` at step 1 and none of them appears in the traversal. `#stepBack` is a
real `disabled` button (`disabled: true`, `tabIndex` irrelevant). Advancing to step 2 gives a clean,
contained order — `.sc` chip → `.sc` chip → `#sName` → `#nameNext` → footer — with `#stepBack`
enabled again. **Eight stops became zero. That claim is true.**

**But the same defect survives in the end state, and it is the one state the fix pass never tabbed
through.** `render()` sets `s.inert = (i !== idx)`, so the *active* screen is never inert — correct
while stepping, wrong once `.sent` covers the stage. After a real completed send:

```
.sent opacity = 1, 458×446, fully covering the 458×350 stage, background #FFFFFF, z-index 5
screen-s inert flags: [True, True, True, False]     <- the last screen stays live under the panel

Tab order after the success panel appears:
  BUTTON.sc         <<< BEHIND THE OPAQUE SUCCESS PANEL
  BUTTON.sc         <<< BEHIND THE OPAQUE SUCCESS PANEL
  TEXTAREA#sWhy     <<< BEHIND THE OPAQUE SUCCESS PANEL
  BUTTON#sendIt     <<< BEHIND THE OPAQUE SUCCESS PANEL
  A#sentMailLink
  A (footer mail)
  BODY
```

Four controls — including `#sendIt`, which will happily re-fire the `mailto:` — remain focusable and
operable underneath an opaque white panel. Effective opacity 1.00, so they are *not* caught by the
old opacity heuristic; they are simply painted over. This is **SC 2.4.11 Focus Not Obscured (AA)** and
the same **2.4.3 / 4.1.2** class as the original finding, at 4 stops instead of 8. Fix is one line:
`inert` the whole `#stepStage` (or the active screen too) when `.sent` goes on.

Related: focus is never moved into `.sent`. After send, `document.activeElement` is still `#sWhy` —
a textarea the user can no longer see.

## 2. The empty-payload send — closed

Verified three ways, each on a fresh load:

| Attempt | Result |
|---|---|
| `#sendIt` reachable by Tab from step 1 | no — `closest('[inert]')` is `true` |
| `inert` stripped by script, then `.click()` with `data` empty | success panel **not** shown; stepper redirected to step 0 |
| type picked, name left empty, forced click | success panel **not** shown; redirected to step 1 |
| name = `"   "` (whitespace only), full walk to step 3, real click | success panel **not** shown; redirected to step 1 |

The guard `if(!data.type || !data.name){ go(...); return; }` holds, and `.trim()` means whitespace
does not defeat it. **Closed.**

## 3. `<main>` and the skip link — half closed, and the half that matters still fails

`<main id="main" tabindex="-1">` exists and the AX tree now reports four landmarks —
`banner`, `main`, `navigation`, `contentinfo`. The `<main>` half is done.

**The skip link does not skip anything.** It is present, it is the first focusable element, it
un-hides correctly on focus (rect top −31 px → 12 px), its `href="#main"` resolves, and `<main>`
carries `tabindex="-1"`. Every static check passes. Then you press Enter:

```
scrolled to y=1500, focus .skip-link, press Enter, wait 1.6 s
  activeElement = A.skip-link      <- focus never moved
  location.hash = ''               <- fragment navigation never happened
  scrollY = 0                      <- it did scroll, that is all it did
  next Tab   = A.brand   inMain=false   "K13 Software Studio"
```

The next tab stop after "skipping" is the header brand link. The user then traverses the entire
header and all ten work rows exactly as before. **SC 2.4.1 Bypass Blocks (Level A) is not closed.**

Root cause, and it is not subtle: the existing smooth-scroll handler binds every in-page anchor —

```js
document.querySelectorAll('a[href^="#"]').forEach(function(a){
  a.addEventListener("click",function(e){ ... e.preventDefault(); scrollTo(el); ... });
});
```

`.skip-link` matches `a[href^="#"]` (confirmed: `matches()` → `true`, 8 anchors bound). The handler
calls `preventDefault()`, so the browser's native fragment navigation — the thing that would have
moved focus into `main[tabindex="-1"]` — never runs. `scrollTo()` moves the viewport and nothing else.

Proof that the markup itself is fine: navigating straight to `index.html#main` with no JS
interference gives `activeElement = MAIN#main`. The markup is right; the handler eats it. The fix is
to exclude `.skip-link` from that selector, or to `el.focus()` inside `scrollTo`.

## 4. Form labels — closed

Real `<label class="vh" for="…">` on all three, confirmed through `HTMLInputElement.labels` in the
live DOM, not from the diff:

| Control | `el.labels[0]` | `aria-label` | Placeholder still present |
|---|---|---|---|
| `#sName` | "Your name" | none | "Type your name" |
| `#sEmail` | "Your email address" | none | "you@email.com" |
| `#sWhy` | "Why this project matters to you" | none | "Tell me the real reason…" |

axe-core reports zero `label`, `form-field-multiple-labels` or `aria-input-field-name` violations.
**Closed.** (`required` and validation are still absent — that was never part of the ticket and is
not a WCAG failure on its own.)

---

# 5. Contrast — every claim re-measured from pixels

## 5.1 The 13 text failures: all 13 now pass

Measured today, glyph-diff method, worst-case pixel under the glyphs:

| # | Element | 08-09 | Claimed | **Measured 08-10** | Needs | Verdict |
|---|---|---|---|---|---|---|
| 1 | `.cap-col .stk em` 11px, on `--paper-3` | 2.98 | 4.55–5.28 | **4.54** | 4.5 | pass — *0.04 margin* |
| 2 | `.brand .tag` 10px | 3.20 | " | **4.87** | 4.5 | pass |
| 3 | `.row .rnum` 13px | 3.20 | " | **4.88** | 4.5 | pass |
| 4 | `.status .yr` / `.now` 11px on glass | 3.37 | " | **5.11 / 5.14** | 4.5 | pass |
| 5 | `#stepLab` "Step 1 of 3" / `#stepBack` 11px | 3.37 | " | **5.14 / 5.14** | 4.5 | pass |
| 6 | `#emailSkip` "skip" 11px | 3.37 | " | **5.14** | 4.5 | pass |
| 7 | placeholders `#sName` / `#sEmail` / `#sWhy` | 3.37 | " | **5.14** (all three) | 4.5 | pass |
| 8 | `.step-sum .sc b` 10.5px on chip | 3.46 | " | **5.28** | 4.5 | pass |
| 9 | `.cap .eyebrow` 12px, on `--paper-3` | 3.89 | 4.55–4.89 | **4.54** | 4.5 | pass — *0.04 margin* |
| 10 | `.work .eyebrow` 12px / `.row .go` 12px | 4.18 | " | **4.88 / 4.61** | 4.5 | pass |
| 11 | `.contact .mail b` 14px | 4.18 | " | **4.88** | 4.5 | pass |
| 12 | `.btn.ghost:hover` label 14.5px (hovered) | 4.18 | " | **4.88** | 4.5 | pass |
| 13 | `.sent .ring` check glyph 27px | 2.62 | 3.33 | **3.34** | 3.0 | pass |

Also re-measured and still fine: `--ink-2` body copy 10.66–11.09, `--muted` on all three papers
6.05–6.66, `#feed` on glass 15.94, `.card .num` 5.32, footer 6.11, the new `#feedToggle` label 6.44,
`.step-sum .sc` value 12.00, `.opt` labels 16.44, `.screen-s .hint` 6.44.

**White on `--blue` was genuinely improved, not just preserved:** measured **5.32:1** on
`header .btn`, `#nameNext` and `#sendIt` (was 4.554, the razor-thin pass I flagged last time).
Claimed 5.33; the 0.01 is rounding. The CTA palette now has real headroom. Good call.

**Two rows sit on a 0.04 margin** — `.cap-col .stk em` and `.cap .eyebrow`, both against
`--paper-3 #E9EDF7`, both **4.54:1** against a 4.5 requirement. The engineering report claimed a
4.55 floor; the true floor is 4.54. Still passing, but this is the new razor-thin pair and it should
be recorded as such: any further lightening of `--paper-3`, or any weight change on those two rules,
breaks AA silently. Same warning I gave about the buttons last time, now moved to a different pair.

## 5.2 The portrait caption — the claim holds, and here is the pixel evidence

This is the claim I was told to be hardest on, so it got the most work.

`.p-cap` computed: `background-color: rgb(255, 255, 255)` (i.e. `--paper-2`, fully opaque — no alpha),
`padding: 8px 12px`, `border-radius: 10px`, `z-index: 2`.

**Full-box sample, caption text hidden, 10,728 pixels** (the same method as the original 7,000-pixel
sample, on a larger box):

- 10,323 of 10,728 pixels are exactly `(255,255,255)`; 122 distinct colours in total.
- Darkest pixel `(170,167,165)`, lightest `(255,255,255)`.
- Against `--blue #B94612` that whole-box range is **2.22 worst / 5.29 mean / 5.32 best**.

The 2.22 is real but it is **not under any glyph** — it is the photograph showing through the chip's
10 px rounded corners, inside the rectangular bounding box but outside the painted rounded rect. So I
sampled each text line's own rect instead:

| Line | px sampled | distinct bg colours | darkest bg | worst | mean | Needs | Verdict |
|---|---|---|---|---|---|---|---|
| `<b>` "Kazim K." 20px/600 | 3,125 | **1** | `(255,255,255)` | **16.71** | 16.71 | 3.0 | pass |
| `<span>` "Founder" 9px | 1,875 | **1** | `(255,255,255)` | **5.32** | 5.32 | 4.5 | pass |
| `<span>` "K13 Software Studio" 9px | 1,875 | **1** | `(255,255,255)` | **5.32** | 5.32 | 4.5 | pass |

One single background colour across every pixel under every glyph. And the geometry backs it up —
each child's rect is fully inside the chip's padding box with `0.00 px` overflow on all four sides.

**The caption went from 1.22:1 to 5.32:1 and it is now deterministic — it no longer depends on what
the photograph is doing.** Claimed 5.33, measured 5.32. That is the single biggest win in the pass,
and unlike a `text-shadow` it is defensible under the standard. Verified, not accepted.

*Caveat for whoever swaps the portrait later:* the guarantee comes from the chip being opaque, not
from the photo. Keep `background: var(--paper-2)` with no alpha and the number holds for any image.

## 5.3 Non-text contrast (SC 1.4.11) — the `--line-3` token works

Border pixels isolated by the `border-color: transparent` diff, compared against the adjacent
interior background:

| Element | 08-09 | Claimed | **Measured 08-10** | Needs | Verdict |
|---|---|---|---|---|---|
| `.s-field` input underline **at rest** | 1.58 | 3.40–3.48 | line `(130,135,149)` vs field bg `(250,251,254)` → **3.47** | 3.0 | pass |
| `.opt` button border | 1.58 | " | `(131,136,150)` vs `(253,253,254)` → **3.49** | 3.0 | pass |
| `.btn.ghost` border | 1.58 | " | `(126,132,148)` vs `(243,245,250)` → **3.43** | 3.0 | pass |
| `.step-prog i` "off" dot | 1.58 | " | `(130,135,149)` vs `(251,251,252)` → **3.47** | 3.0 | pass |
| `.step-prog i.on` dot (`--blue`) | 4.40 | — | **5.14** | 3.0 | pass |
| `#feedToggle` border (new control) | — | — | **3.47** | 3.0 | pass |

The underline needed care: measured naively it reads `rgb(185,70,18)` at 5.14, because `render()`
auto-focuses the field 360 ms after each step change, so `:focus-within` is already active and you
are measuring the *focus* colour. Blurred properly, the resting colour is `rgba(20,29,53,0.52)`
composited to `(130,135,149)` → **3.47:1**. Both states clear 3.0. Claimed range 3.40–3.48, true
range **3.43–3.49**. Holds.

**One residual `--line-3` miss:** `.step-sum .sc` — the summary chips — are real `<button>` elements
and kept `--line-2`. Measured `(202,204,211)` vs `(254,254,255)` = **1.59:1**. The engineering note
says `--line-2` was left only on "purely decorative" borders; these are actionable controls, so by
the same logic that promoted `.opt` and `.btn.ghost` they should have been promoted too. (Arguable —
the chip's own lighter fill does distinguish it from the glass card, so a reviewer could call this
compliant. Flagging it, not calling it a hard failure.) Genuinely decorative `--line` borders
(`.card` 1.27, `.row .tag` 1.28) are correctly untouched and out of scope for 1.4.11.

---

# 6. New problems introduced by the fixes

## 6.1 The pause control is invisible to screen readers — new Level A failure

`#feedToggle` is keyboard-focusable (**tab stop 9 of 26**) and sits inside `<aside class="status"
aria-hidden="true">`. The engineering note says the button "carries `aria-hidden="false"` to stay
operable even though its parent `<aside>` is `aria-hidden="true"`.` **`aria-hidden` does not work
that way.** It is inherited down the subtree and a descendant cannot opt back in. Chromium says so
directly:

```
#feedToggle  ignored = True   role = none   name = None
             ignoredReasons = ['ariaHiddenSubtree']
aria-hidden chain: BUTTON[aria-hidden=false] -> ASIDE[aria-hidden=true]
```

axe-core independently flags it, and it is the **only** violation it finds on `index.html`:

```
[serious] aria-hidden-focus ×1 — ARIA hidden element must not be focusable
                                 or contain focusable elements   ['aside']
```

A screen-reader user tabs onto a control with **no name, no role, no state** and is told nothing. That
is **SC 4.1.2 Name, Role, Value (Level A)** — a *new* failure that did not exist on 2026-08-09,
introduced by the control added to fix 2.2.2. Either drop `aria-hidden` from the `<aside>` (and label
the region properly), or move the toggle outside it.

## 6.2 SC 2.2.2 itself — closed for sighted users only

The mechanism does work, and I tested it end to end rather than reading the handler:

| Check | Result |
|---|---|
| Feed rotates unattended | yes — text changed within 4.2 s |
| Reachable by Tab | yes, stop 9 |
| `aria-pressed` / `aria-label` before | `false` / "Pause rotating status text" |
| Enter → after | label flips to "Play", `aria-pressed="true"`, `aria-label` → "Resume rotating status text" |
| Rotation actually stops | **yes — identical string after 9 s** (2.4 rotation periods) |
| Enter again resumes | yes — changed again within 4.3 s |
| Hidden under `prefers-reduced-motion` | yes, `display:none`, and **not focusable** — no phantom stop |

The state management is careful and the reduced-motion handling is right. But because of 6.1 the
mechanism is unreachable in practice for AT users, so I can only call 2.2.2 **closed for sighted
keyboard and pointer users**. Fix 6.1 and it closes outright.

## 6.3 `#feedToggle` fails target size — new SC 2.5.8 (AA)

**53 × 23 CSS px** at both 1440 and 390. The minimum is 24 × 24. One pixel of height. It is a
standalone button with no inline-text exception. Pre-existing siblings unchanged and still failing:
`#stepBack` 44×18, `#emailSkip` 28×18.

## 6.4 The success panel is announced to nobody, and its heading is always there

`#sent` has `role="status" aria-live="polite"` but no `aria-hidden` and no `inert`, at `opacity: 0`.
Consequences:

- Its `<h3>` "Opening your mail app…" is **permanently in the heading list** (`h1 → h2 → h3×3 → h2 →
  h2 → h3×3 → h2 → h3`, that last h3). A screen-reader user browsing by heading finds a heading for
  something that has not happened, plus the body copy "Your message is filled in and ready…".
- Live regions announce on *content change*, not on becoming visible. Adding `.on` changes no text,
  so **nothing is announced at the moment the panel appears**. An announcement only arrives later,
  when `sentTitle.textContent` flips to "Your mail app opened." or when `#sentFallback` un-hides —
  both confirmed firing in the browser (fallback appeared after ~1.9 s and `#sentMailLink` became
  tabbable). So SC 4.1.3 is *partly* addressed, not cleanly: the primary message never announces.

Adding `aria-hidden="true"`/`inert` while off and moving focus into the panel on show would fix 6.4
and 1's residual together.

---

# 7. Regression checks — nothing else broke

**`prefers-reduced-motion` is still the strongest part of the page.** Re-run in a `reduced_motion:
'reduce'` context; every item from the first pass still holds, and the new code respects it:

| Behaviour | Result |
|---|---|
| `.card` transition-duration | `0.001s` |
| Status-dot `beat` animation-duration | `0.001s` |
| `.rv` reveals | every element `opacity: 1`, single distinct value |
| Lenis | `html.lenis` absent — never initialised |
| Rotating feed | identical string after 8 s |
| Count-up numbers | final values `16 / 10 / 1` immediately |
| Portrait parallax | `transform: none` |
| **New** `.feed-toggle` | `display: none` **and not focusable** — correctly removed, no phantom stop |
| **New** `.p-cap` chip | still `rgb(255,255,255)` — contrast fix is not motion-dependent |
| **New** `.skip-link:focus` | still rises to `top: 12px` |

**Focus visibility — no regression.** I initially read `outline: none` on every control and thought
the fix had killed the focus rings; that was my own artifact from focusing elements by script, which
does not satisfy `:focus-visible`. Re-run with real `Tab` presses, every stop reports
`outline: auto 1px rgb(0,95,204)` and `:focus-visible = true`, including `.skip-link` and
`#feedToggle`. The pre-existing `.s-field input/textarea { outline: none }` is unchanged; its
underline cue is now **better** than before — 3.47:1 at rest (was 1.58) changing to `--blue` at
5.14:1 on focus. Still a colour-only 1.5 px cue and still short of the AAA Focus Appearance metric,
which remains advisory.

**Other re-checks after the markup change:**

- **`alt` text — still a full pass.** Portrait keeps its descriptive alt; `#peekImg` and all ten
  runtime-injected mobile thumbnails (checked at 390 px, 12 `<img>` total) correctly carry `alt=""`.
- **Heading order — pass.** `h1 → h2 → h3 h3 h3 → h2 → h2 → h3 h3 h3 → h2 → h3`. One `h1`, no skipped
  levels. The trailing h3 is the always-exposed `.sent` heading (6.4).
- **`lang="en"`** present; `<title>` and description intact; viewport still `width=device-width,
  initial-scale=1.0` with no `maximum-scale` — 1.4.4 passes.
- **Reflow — pass.** `scrollWidth === clientWidth` at **320 / 360 / 390 / 1280**, and at 320×256
  (400 % zoom of 1280). Checked honestly: forcing `body { overflow-x: visible }` at 390 still gives
  `scrollWidth 390`, so nothing is being hidden by the clip. The `.wash` blobs and the off-stage
  inert `.screen-s` panels extend past the right edge but neither contributes to scroll width.
- **Zero console errors, zero page errors** across every run.
- **`.step-sum .sc` chips** built with `innerHTML` from user input remain the self-XSS sink security
  flagged; out of scope here, still open.

---

# 8. `404.html` (new, Kate)

Structurally sound, and it undoes part of the contrast work.

**Passing:** `lang="en"`; descriptive `<title>` ("Page not found · K13 Software Studio");
`meta robots noindex`; exactly one `h1` and no skipped levels; a `main` landmark; a
`prefers-reduced-motion` block that neutralises the `rise` animation; three tab stops, all with the
default `outline: auto` focus ring; `scrollWidth === clientWidth` at 320 px; no images, so no alt
surface. A skip link is unnecessary at three stops.

**Failing — it ships the pre-fix palette.** `:root` in `404.html` still declares
`--faint:#8089A1` and `--blue:#CB4D14`, the exact values `index.html` moved off. Measured from
pixels, and confirmed independently by axe-core (`color-contrast ×2`, targets `.code` and the mailto
link):

| Element | Colour | Size | **Measured** | Needs | Verdict |
|---|---|---|---|---|---|
| `.code` "404" | `#CB4D14` on `#F3F5FA` | 13px | **4.18** | 4.5 | **fail** |
| `.foot a` email | `#8089A1` on `#F3F5FA` | 12px | **3.20** | 4.5 | **fail** |
| `.cta` label | `#fff` on `#CB4D14` | 14.5px/600 | **4.55** | 4.5 | pass — the old razor-thin margin |
| `h1` / `h1 i` / `.brand` | — | ≥27px | 15.32 / 4.18 / 4.18 | 3.0 | pass (large text) |

Two of the thirteen failures that were just closed on `index.html` are live again on `404.html`.
Porting the two token values across is the whole fix; the `.cta` then also inherits the 5.32
headroom instead of sitting on 4.55. The inline data-URI favicon in `404.html` also still uses
`#CB4D14` — cosmetic, but it means two different oranges ship.

**Minor:** the brand link and the footer email sit outside any landmark (`main` is the only one), so
that content is not reachable by landmark navigation. Best practice, not a WCAG failure.

# 9. `pricing/tiger/` — exercised only, not audited

Loaded in the browser as permitted; no content recorded. `lang="tr"` correctly matches its Turkish
copy, `noindex, nofollow` present, a `prefers-reduced-motion` block exists, zero images missing
`alt`. Two things to hand on: headings run `h1 → h3` with **no `h2`** (a skipped level), and it
**overflows horizontally at 320 px** (`scrollWidth 326` vs `clientWidth 320`) — a small SC 1.4.10
miss. Its unauthenticated-path exposure is unchanged and remains Kazim's decision from the 08-09
report, not mine.

# 10. What to fix, in order

1. **Exclude `.skip-link` from the `a[href^="#"]` smooth-scroll handler** (or call `el.focus()` in
   `scrollTo`). One line; closes the last outstanding original Level A failure.
2. **Take `aria-hidden="true"` off `<aside class="status">`** (give it an `aria-label` instead), or
   move `#feedToggle` out of it. Closes the new Level A failure and finishes 2.2.2 properly.
3. **`inert` the whole `#stepStage` when `.sent` is on**, and move focus to `#sent`. Closes the four
   trapped tab stops and most of 6.4.
4. **`aria-hidden="true"` / `inert` on `#sent` while it is off**, so its heading and copy leave the
   AX tree until they are true.
5. **Port `--faint: #646B7E` and `--blue: #B94612` into `404.html`.** Two lines, two failures.
6. **Grow `#feedToggle` to 24 px tall** (it needs 1 px), and `#stepBack` / `#emailSkip` to 24×24.
7. Consider `--line-3` on `.step-sum .sc`, and treat `.cap-col .stk em` / `.cap .eyebrow` at 4.54 as
   at-the-limit, not as headroom.

Items 1–3 are the difference between "we fixed accessibility" being true and being a marketing claim
the page contradicts. Given that the site sells `WCAG · 508` as a paid capability, that distinction
is worth the hour it will take.

## Status      BLOCKED
## Summary     Re-measured every claim in Chromium/Playwright with pixel-level contrast sampling, real Tab traversal and the CDP accessibility tree. All 13 contrast failures are genuinely closed (worst residual 4.54:1 on two paper-3 pairs) plus all 4 non-text boundaries at 3.43-3.49:1, and the portrait caption is verified at 5.32:1 on a truly opaque #FFFFFF chip — 1,875 sampled pixels under each 9px line, one distinct colour, zero glyph overflow. But only 2 of the 4 Level A failures are genuinely closed: the skip link is hijacked by the site's own a[href^="#"] handler so focus never moves and 2.4.1 still fails, and the fixes introduced a new Level A failure (#feedToggle is focusable inside an aria-hidden="true" subtree — Chromium: ignoredReasons ['ariaHiddenSubtree']; axe-core: serious aria-hidden-focus) plus four controls left tabbable behind the opaque success panel.
## Files       Inspected: /Users/k13/Desktop/PROJECTS/K13_Website/index.html, /Users/k13/Desktop/PROJECTS/K13_Website/404.html, /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/compliance_2026-08-09.md, /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/engineering_2026-08-09_v2.md, /Users/k13/Desktop/PROJECTS/K13_Website/pricing/tiger/index.html (browser only, no content recorded). Created: /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/compliance_2026-08-10.md. Modified: none.
## Risks       The engineering pass reported PASS on all eight items; two of those claims do not survive measurement, so any release note repeating them would be inaccurate — and the site markets "WCAG · 508" as a paid capability. The aria-hidden="false" misconception behind #feedToggle will recur wherever else that pattern is reached for, since it looks correct in the diff and only Chromium's ignoredReasons exposes it. .cap-col .stk em and .cap .eyebrow now sit at 4.54:1 against a 4.5 bar (the engineering report claimed a 4.55 floor), so they are the new silent-break pair if --paper-3 is ever lightened. The portrait caption's 5.32:1 depends on the chip staying fully opaque, not on the photograph — an alpha value added to .p-cap later reopens a 1.22:1 failure invisibly. 404.html re-ships the pre-fix palette, so the same two failures now exist in two places and will drift apart. pricing/tiger/ overflows at 320px and skips a heading level; its unauthenticated-path exposure is unchanged.
## Next        frontend-engineer — items 1-6 in §10 are all small, local and independent (skip-link selector, aria-hidden on the status aside, inert the stage on send, aria-hidden the idle .sent, port two tokens into 404.html, 1px on #feedToggle). Then re-run this compliance gate; do not route to release-engineer until §10 items 1-3 are proven in a browser.
## Human gate  Kazim decides: (1) whether the site may ship while it markets "WCAG · 508" and still fails SC 2.4.1 plus the newly introduced SC 4.1.2 — the marketing-claim exposure I flagged on 2026-08-09 is unchanged in kind, only smaller; (2) whether <aside class="status"> should stop being aria-hidden entirely (that re-exposes the rotating feed to screen readers — a product decision about whether the feed is decorative, not a compliance one); (3) all five open legal/privacy items from the 2026-08-09 report stand untouched — EU/UK exposure of Google Fonts + jsDelivr, whether a privacy notice is required, business PII in a consumer Gmail account, and whether pricing/tiger/ may deploy unauthenticated. Items in (3) warrant counsel, not an agent.
