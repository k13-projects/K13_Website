# K13 Website — accessibility re-verification against the merged, shipped state

**Date:** 2026-08-11 · **Stage:** compliance (re-verify only — no project file was modified)
**Target:** `main` @ `d152fe4` — `index.html` (904 lines), `404.html` (70 lines)
**Under test:** the five items my 2026-08-10 pass returned BLOCKED on, claimed fixed in `0c7c8a5`
**Standard applied:** WCAG 2.2 Level AA

## Verdict up front

**My 2026-08-10 BLOCKED verdict is stale. All five items hold under measurement.**

| # | 2026-08-10 finding | SC | Verified today |
|---|---|---|---|
| 1 | 4 controls tabbable behind the opaque `.sent` panel | 2.4.11 / 4.1.2 (A) | **CLOSED** — 0 focusable |
| 2 | `#feedToggle` focusable inside an `aria-hidden="true"` subtree | 4.1.2 (A) | **CLOSED** — not in a hidden subtree, named, 63×24 |
| 3 | `.sent` never inert, never announced | 4.1.3 (AA) | **CLOSED** — inert at load, real mutation at reveal |
| 4 | Skip link scrolled but never moved focus | 2.4.1 (A) | **CLOSED** — focus lands on `<main>` |
| 5 | `404.html` shipping the pre-fix palette | 1.4.3 (AA) | **CLOSED** — 4.877 / 4.879 |

**WCAG Level A failures remaining on `index.html` and `404.html`: zero.**
**axe-core 4.11.3: 0 violations, in all six states tested.**

One genuine regression was introduced by the same commit, and it is **not** a WCAG failure — the
reduced-motion pause control (§7). One pre-existing AA miss survives on `pricing/tiger/` (§8), which
was never in this fix ticket's scope.

> **Tree state during this run.** `index.html` was clean at `d152fe4` when I started and shows as
> modified at the end — not by me (I edited nothing but this file). A concurrent QA agent changed
> one line: `twitter:description`, em dash → colon. It renders nothing, styles nothing and scripts
> nothing, so no measurement below is affected either way. Flagging it so the record is exact.

## How this was re-checked

Nothing was taken on trust; the numbers below are today's, on the merged code. Real Chromium
(Chrome 151.0.7922.109) via Playwright, served from `python3 -m http.server 8479` bound to
`127.0.0.1`, killed at the end of the run. Same methodology as 2026-08-10:

1. **Contrast from rendered pixels, not from CSS.** Each target is screenshotted twice — once
   normally, once with its own text forced `transparent` — and the frames differenced. The pixels
   that change *are* the glyphs; the same pixels in the second frame are the exact background under
   them. Borders, `::before` rules and chip edges are excluded by construction.
2. **Real `Tab` presses** with a 450 ms settle per stop, so `IntersectionObserver` reveal callbacks
   finish before `document.activeElement` and effective ancestor opacity are read.
3. **Chromium accessibility tree** via CDP `Accessibility.getPartialAXTree` — decisively,
   `ignoredReasons`.
4. **axe-core 4.11.3** over `wcag2a/2aa/21a/21aa/22aa`, on six page states.

> **Method note for whoever runs this next — read this before you walk the stepper.**
> `#sendIt` ends in `window.location.href = "mailto:…"`. Chromium hands that to the OS **even
> headless**, so a scripted walk-through opens a real Mail compose window per run. My first
> traversals did exactly that and filled Kazim's desktop with identical drafts. Every later run
> loaded the page inside a **sandboxed iframe** (`sandbox="allow-scripts allow-same-origin"`, i.e.
> no `allow-top-navigation*`, no `allow-popups`) and activated every control **programmatically**
> (`.click()` from `evaluate`, so there is no transient user activation — Chrome requires one for
> external protocols). Both blocks were proven with a harmless unregistered scheme
> (`k13probe://safety-check`) *before* any `mailto:` was composed:
> `Navigation to external protocol blocked by sandbox…`. Page code is untouched by this; the
> composed URL is then recovered from CDP `Page.frameRequestedNavigation` instead of the OS.

---

## 1. Inert after send — closed

`render()` still only tracks the *step* (`s.inert = i !== idx`), so the active screen is still
`false` in the end state — but it no longer matters, because the whole `#stepBody` wrapper is
inerted the moment `.sent` is shown (`index.html` line 888). Measured after a real completed send:

```
stepBodyInert: true      sentInert: false     sentOpacity: 1
screen-s inert flags: [True, True, True, False]   <- last screen live, but inside the inert wrapper
focusable elements inside #stepBody not covered by an inert ancestor:  []   (was 4)
```

The four controls I found trapped on 2026-08-10 — 2 `.sc` chips, `#sWhy`, `#sendIt` — are all gone
from the tab order. Real traversal after send:

```
1 A#sentMailLink   "projects.k13@gmail.com"     <- inside the visible panel
2 A (footer mail)  "projects.k13@gmail.com"
3 A.skip-link  -> 4 A.brand -> …                 (normal wrap to the top of the document)
```

**The `#stepBody` wrapper did not break focus order.** Mid-flow traversal is still clean and
contained, chips first, then the field, then the button:

```
step 2:  .sc "Building A product" -> #sName -> #nameNext -> footer mail
step 3:  .sc "Building A product" -> .sc "Name Ada" -> #sEmail -> #emailNext -> #emailSkip -> footer
```

**Two residuals, neither a failure.** Focus is still never moved *into* `.sent` — when `#stepBody`
goes inert Chromium blurs the focused field and `document.activeElement` becomes `BODY`. I checked
the consequence rather than assuming it: Chrome preserves the sequential-navigation starting point,
so the next `Tab` goes *forward* to `#sentMailLink`, not back to the top. Focus order is therefore
not broken (2.4.3 holds) — but an explicit `sent.focus()` would still be better than relying on that
browser behaviour. Second: the form is sealed permanently once sent. There is no control inside
`#sent` to reopen or correct it, so a user who mistyped their email must reload the page. UX, not
WCAG.

## 2. `#feedToggle` — closed, and the AX tree agrees

It is now a DOM sibling of the `<aside>`, inside `.status-wrap`:

```
parent: DIV.status-wrap        aria-hidden ancestor chain: []        closest('[aria-hidden=true]'): null
AX:  role=button   name="Pause rotating status text"   ignored=False   ignoredReasons=[]
```

Compare 2026-08-10, which read `ignored=True, ignoredReasons=['ariaHiddenSubtree'], role=none,
name=None`. The `aria-hidden="false"` misconception is gone from the markup along with it. **SC
4.1.2 closed.**

**Target size: 63 × 24 CSS px** at 1440, 390 and 320 — measured, matching the claim exactly. The
minimum is 24 × 24, so it now passes **SC 2.5.8** on its own dimensions (it was 53 × 23, one pixel
short). It is absolutely positioned on the card's top edge; I checked it is never clipped or
off-viewport and that nothing else paints over its centre at 390 and 320.

**Its label is honest about what it controls.** The rotating feed itself remains `aria-hidden`
(decorative ticker), so an AT user meets a button for content they cannot perceive — but SC 2.2.2
exists for the sighted user watching the text move, and for them the mechanism works. That reading
is unchanged from my last two passes.

## 3. `.sent` — starts inert, and the live region gets a real mutation

At load, `#sent` carries `inert` and both text nodes are genuinely empty:

```
sentInert: true    sentTitle.textContent: ""    sentMsg.textContent: ""
AX: role=none  ignored=True  ignoredReasons=['inertElement']
heading list at load: h1 -> h2 -> h3 h3 h3 -> h2 -> h2 -> h3 h3 h3 -> h2 -> [h3 INERT, empty]
```

The phantom "Opening your mail app…" heading is out of the exposed heading list at load — it only
joins it after a send, when it is true. That closes the second half of my 6.4 finding.

At reveal, a `MutationObserver` on `#sent` recorded, in order:

```
1 attributes/inert on #sent      <- region enters the AX tree
2 childList on #sentTitle        added: "Opening your mail app…"
3 childList on #sentMsg          added: "Your message is filled in and ready…"
4 attributes/class on #sent      <- .on, the visual reveal
```

and the AX tree afterwards reports `role=status, ignored=False`. The text arrives **as an insertion
into a live region that is already exposed**, which is the announcing case — `aria-live="polite"`
defaults to `aria-relevant="additions text"`. This is the right shape and materially better than
2026-08-10, where adding `.on` changed no text and nothing was announced at all.

**Honest limit on this one:** I can prove the mutation and the AX exposure; I cannot prove what
NVDA/VoiceOver actually speak without a screen reader in the loop. The un-inert and the text write
land in the same task, which is the one arrangement where a screen reader could plausibly treat the
subtree as newly-inserted-and-suppressed. If this is ever bench-tested with a real AT, that is the
case to try.

## 4. Skip link — closed

Tested in both motion modes, with no prior scroll (scrolling moves Chrome's sequential-focus
starting point and silently invalidates this test — my first attempt today fell into exactly that
trap and had to be redone):

```
prefers-reduced-motion: reduce          | no-preference
1st Tab      A.skip-link                | A.skip-link
after Enter  MAIN#main   inMain=True    | MAIN#main   inMain=True
next Tab     A.btn "Start a project"    | A.btn "Start a project"   inMain=True
```

Focus moves into `<main>` and the next stop is the first control inside it, not the header brand
link. **SC 2.4.1 closed.** Ordinary in-page anchors are unaffected — the exemption is
`a[href^="#"]:not(.skip-link)` plus a dedicated listener, and the shared `scrollTo()` is reused.

## 5. `404.html` contrast — closed, claim exact

Tokens now read `--faint:#646B7E`, `--blue:#B94612`, `--paper:#F3F5FA` — the `index.html` values.
Measured from pixels, one distinct background under every glyph:

| Element | 2026-08-10 | Claimed | **Measured 2026-08-11** | Needs | Verdict |
|---|---|---|---|---|---|
| `.code` "404" 13px | **4.18 fail** | 4.877 | **4.877** | 4.5 | pass |
| `.foot a` email 12px | **3.20 fail** | 4.879 | **4.879** | 4.5 | pass |
| `.cta` white-on-blue 14.5px/600 | 4.55 (razor) | — | **5.320** | 4.5 | pass — real headroom now |
| `h1` 54px | 15.32 | — | **15.06** | 3.0 | pass |
| `h1 i` 54px | 4.18 | — | **4.877** | 3.0 | pass |

Both claimed numbers reproduce to three decimals. axe-core finds **0 violations** on `404.html`
(it found `color-contrast ×2` there last time).

## 6. The 13 pairs, 4 non-text boundaries and the portrait caption — still closed

Re-measured on merged code, glyph-diff, worst-case pixel under the glyphs:

| Element | Needs | **Measured** | | Element | Needs | **Measured** |
|---|---|---|---|---|---|---|
| `.cap-col .stk em` 11px | 4.5 | **4.541** ⚠ | | `.contact .mail b` 14px | 4.5 | 4.877 |
| `.cap .eyebrow` 12px | 4.5 | **4.540** ⚠ | | `#feedToggle` label 10px | 4.5 | 6.554 |
| `.brand .tag` 10px | 4.5 | 4.797 | | white on `--blue` (`header .btn`) | 4.5 | 5.320 |
| `.row .rnum` 13px | 4.5 | 4.701 | | white on `--blue` (`#nameNext`) | 4.5 | 5.320 |
| `.status .yr` / `.now` 11px | 4.5 | 4.971 / 5.143 | | `.step-sum .sc b` 10.5px | 4.5 | 5.280 |
| `#stepLab` / `#stepBack` 11px | 4.5 | 5.140 / 5.099 | | `.opt` label 16px | 4.5 | 16.425 |
| `#sName` placeholder | 4.5 | 5.143 | | footer email 12px | 4.5 | 6.108 |
| `.work .eyebrow` 12px | 4.5 | 4.738 | | `.sent .ring` glyph 27px | 3.0 | 3.344 |
| `.row .go` 12px | 4.5 | **4.549** ⚠ | | `.sent` h3 / p | 4.5 | 16.708 / 6.663 |

**Non-text (SC 1.4.11)**, composited from the real `rgba` tokens against the resolved backing:
`.s-field` underline, `.opt` border, `.btn.ghost` border and `#feedToggle` border all composite to
`(127,133,148)` on `(243,245,250)` = **3.386**; `.step-prog i` **11.092**; `.step-prog i.on`
**4.877**. All clear 3.0.

**The portrait caption holds, with the same pixel evidence.** `.p-cap` computes
`background-color: rgb(255,255,255)`, no alpha, `z-index: 2`, 148.8 × 71.6:

| Line | glyph px | distinct bg | worst | needs | verdict |
|---|---|---|---|---|---|
| `.p-cap b` "Kazim K." 20px/600 | 727 | **1** — `(255,255,255)` | **16.708** | 3.0 | pass |
| `.p-cap span` "Founder" 9px | 238 | **1** | **5.320** | 4.5 | pass |
| `.p-cap span` "K13 Software Studio" 9px | 521 | **1** | **5.320** | 4.5 | pass |

One background colour under every glyph of every line. **1.22:1 → 5.320:1, and deterministic** — it
no longer depends on what the photograph is doing. Unchanged from my last measurement to three
decimals.

⚠ **The razor-thin set has grown by one.** `.cap .eyebrow` **4.540**, `.cap-col .stk em` **4.541**
and now `.row .go` **4.549** sit within 0.05 of the 4.5 bar. Any further lightening of `--paper-3`,
or a weight change on those rules, breaks AA silently and axe will not catch it (axe returns
`color-contrast` as *incomplete*, not *pass*, on this page — 39–67 nodes per state — because the
backgrounds are gradients and images; the pixel work above is what actually covers them).

## 7. What the fixes broke — the reduced-motion pause control

This is the one thing I found that is genuinely worse than on 2026-08-10, and it is a direct
consequence of the `.status-wrap` restructuring in fix #2.

```css
line 142:  .status-wrap .feed-toggle{ … display:inline-flex; … }          /* specificity 0,2,0 */
line 334:  @media(prefers-reduced-motion:reduce){ .feed-toggle{display:none} }   /* 0,1,0 */
```

The new rule out-specifies the reduced-motion rule, so the hide no longer applies. Measured under
`prefers-reduced-motion: reduce`:

| | 2026-08-10 | **2026-08-11** |
|---|---|---|
| `display` | `none` | **`flex`** |
| focusable | no — no phantom stop | **yes, 63×24, in the tab order** |
| clicking it | n/a | **does nothing at all** |

It does nothing because the feed IIFE bails at line 716 (`if(!feed||reduced) return;`) before the
toggle's click listener is ever attached. Measured: after a real click the label stays `"Pause"`,
`aria-pressed` stays `"false"`, `aria-label` stays `"Pause rotating status text"` — and the feed was
already frozen, so there was nothing to pause.

**This is not a WCAG Level A or AA failure** and I am not counting it as one: the control has a
correct name, role and state, and its state never lies because it never changes. The closest hook is
SC 2.4.6 Labels (AA) — a label promising an action the control cannot perform — and that is a
stretch. What it plainly is: a **dead tab stop shipped to exactly the users who asked for less
motion**, on a page that sells accessibility. One line fixes it — scope the reduced-motion rule to
`.status-wrap .feed-toggle` (or `!important`).

Everything else under reduced motion still holds: `.card` transition `0.001s`, `html.lenis` absent,
every `.rv` at opacity 1 (single distinct value), feed text identical after 8 s, portrait
`transform: none`, count-ups final immediately, skip link still lands on `<main>`.

## 8. Regression sweep — everything else

**axe-core 4.11.3, `wcag2a/2aa/21a/21aa/22aa`:**

| State | Violations |
|---|---|
| `index.html` on load | **0** |
| `index.html` after full scroll (all reveals fired) | **0** |
| `index.html` mid-stepper (step 3) | **0** |
| `index.html` after send (success panel up) | **0** |
| `index.html` @390 mobile | **0** |
| `404.html` | **0** |

Last pass this was 1 serious `aria-hidden-focus` on `index.html` and 2 `color-contrast` on
`404.html`. `color-contrast` appears as *incomplete* only, for the gradient/image backgrounds — §6
covers those by pixel.

**Reflow (1.4.10) and zoom (1.4.4) — pass everywhere**, checked honestly by also re-reading
`scrollWidth` with `body{overflow-x:visible}` forced, so nothing is hidden by a clip:

| Viewport | `index.html` sw/cw (unclipped) | `404.html` |
|---|---|---|
| 320 | 320/320 (320) | 320/320 |
| 360 | 360/360 (360) | 360/360 |
| 390 | 390/390 (390) | 390/390 |
| 1280 | 1280/1280 (1280) | 1280/1280 |
| 320×256 (= 1280 @ 400 %) | 320/320 (320) | 320/320 |

**Target size (SC 2.5.8)** — every undersized target clears the 24 px **spacing exception** (a 24 px
circle on each centre reaches no other target), measured at 1440, 390 and 320:

| Target | Size | Nearest other target | Verdict |
|---|---|---|---|
| `#stepBack` | 44×18 | 40.1 px | pass (spacing) |
| `#emailSkip` | 28×18 | 28.1 px | pass (spacing) |
| `.mail` / phone / footer links | 331×23 / 164×21 / 158×16 | 47.2 / 18.7 / 204.5 px | pass (spacing + inline text) |
| `#feedToggle` | **63×24** | — | pass outright |

**`mailto:` payload — verified for the first time** (nobody had checked this). Composed URL captured
from the blocked navigation, typed values `A brand site` / `Ada Lovelace` / `ada@example.invalid` /
`Because the numbers deserve a machine.`:

```
to:      projects.k13@gmail.com
subject: Project inquiry: A brand site (Ada Lovelace)
body:    Hi Kazim,\r\n\r\nWe're building: A brand site\r\n\r\nWhy it matters: Because the numbers
         deserve a machine.\r\n\r\n--\r\nAda Lovelace\r\nada@example.invalid
```

Every field the user typed arrives intact and in the right slot; everything is percent-encoded,
including the CRLFs (`%0D%0A`), so there is no header-injection surface through the name or reason
fields. The fallback line un-hides on schedule when nothing takes focus, and `#sentMailLink` becomes
tabbable at that point — measured, not read.

**Also re-checked and unchanged:** `lang="en"`; viewport `width=device-width, initial-scale=1.0` with
no `maximum-scale`; exactly one `h1`, no skipped levels, and the `.sent` h3 correctly absent while
inert; 12 `<img>` at 390 with **0 missing `alt`** — the portrait keeps its descriptive alt, `#peekImg`
and all ten injected mobile thumbnails correctly carry `alt=""`; landmarks `header/nav/main/aside/footer`;
zero links without an accessible name; **zero console errors and zero page errors** across every run.

**Two residuals I am carrying forward unchanged, neither newly broken:**

- **`.step-sum .sc` chip border 1.58:1** against a 3.0 bar (`rgba(20,29,53,.22)` → `(194,197,207)` on
  `(243,245,250)`). These are real `<button>`s that kept `--line-2` while `.opt` and `.btn.ghost`
  were promoted to `--line-3`. Same call as 2026-08-10: arguable, because the chip's own lighter
  fill distinguishes it from the card. Flagged, not scored as a failure.
- **`pricing/tiger/`** — exercised in the browser only, no content recorded, per instruction. Still
  `lang="tr"`, `noindex, nofollow`, a reduced-motion block, 0 images missing `alt`. Still
  **overflows at 320 px (`scrollWidth 326` vs `clientWidth 320`) — an open SC 1.4.10 (AA) miss** —
  and still runs `h1 → h3` with no `h2`. Unchanged since 2026-08-09 and never in this ticket's
  scope; it is the only WCAG failure of any level I can find anywhere in the repo today.

## 9. What I would do next, in order

1. **Scope the reduced-motion hide to `.status-wrap .feed-toggle`** (§7). One line. It is the only
   thing this pass found broken.
2. `sent.focus()` when the success panel appears, and a way back into the form after a send (§1).
   Both are polish, not compliance.
3. Treat `.cap .eyebrow` 4.540, `.cap-col .stk em` 4.541 and `.row .go` 4.549 as **at the limit, not
   as headroom** — put a comment next to `--paper-3` saying so (§6).
4. Consider `--line-3` on `.step-sum .sc` (§8).
5. `pricing/tiger/` 320 px overflow and heading skip — only if Kazim decides that page ships.

The site markets "WCAG · 508" as a paid capability. On `index.html` and `404.html`, as of `d152fe4`,
that claim now survives an adversarial re-measurement. That was not true yesterday, and the
difference is real work, not a re-graded opinion.

## Status      PASS
## Summary     Re-measured all five 2026-08-10 BLOCKED items on merged main @ d152fe4 in Chrome 151 via Playwright, with pixel glyph-diff contrast, real Tab traversal and CDP ignoredReasons — all five hold, so my prior BLOCKED verdict is stale. Zero WCAG Level A failures remain on index.html and 404.html, and axe-core 4.11.3 reports 0 violations across six page states (was 1 serious + 2 color-contrast). 404.html measures .code 4.877 and footer email 4.879 (claims reproduce exactly); the portrait caption measures 5.320:1 on a single #FFFFFF background under every glyph. One real regression found, not a WCAG failure: the reduced-motion pause control (§7).
## Files       Inspected: /Users/k13/Desktop/PROJECTS/K13_Website/index.html, /Users/k13/Desktop/PROJECTS/K13_Website/404.html, /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/compliance_2026-08-10.md, /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/engineering_2026-08-09_v2.md, /Users/k13/Desktop/PROJECTS/K13_Website/pricing/tiger/index.html (browser only, no content recorded). Created: /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/compliance_2026-08-11.md. Modified: none.
## Risks       .status-wrap .feed-toggle (0,2,0) out-specifies the reduced-motion .feed-toggle{display:none} (0,1,0), so reduced-motion users get a visible, focusable Pause button whose click listener was never attached — it is inert in effect and its label promises an action it cannot perform; not a WCAG failure, but it ships to the exact audience the page claims to serve. Three text pairs now sit within 0.05 of the 4.5 bar (.cap .eyebrow 4.540, .cap-col .stk em 4.541, .row .go 4.549) and axe cannot see them — it returns color-contrast as incomplete on this page because the backgrounds are gradients — so any future lightening of --paper-3 breaks AA silently. The portrait caption's 5.320 depends on .p-cap staying fully opaque, not on the photograph; adding any alpha reopens a 1.22:1 failure invisibly. .step-sum .sc chips remain at 1.58:1 on --line-2 while sibling controls were promoted to --line-3. pricing/tiger/ still overflows at 320px (326 vs 320, SC 1.4.10 AA) and skips h2 — the only WCAG failure left anywhere in the repo. Testing hazard for the next agent: #sendIt hands a real mailto: to macOS even headless — walk the stepper only inside a sandboxed iframe with programmatic clicks (method note above), or it will open a Mail window per run.
## Next        frontend-engineer — a single one-line CSS fix (scope the reduced-motion hide to .status-wrap .feed-toggle), then release-engineer. Nothing here blocks a release on WCAG grounds.
## Human gate  Kazim decides: (1) whether pricing/tiger/ ships at all, and if so whether its 320px reflow miss and heading skip get fixed first — it is the only remaining WCAG failure in the repo and it sits on a client-confidential page I am not permitted to edit; (2) whether the rotating status feed should stay aria-hidden — keeping it decorative means AT users meet a pause control for content they cannot perceive, which is defensible but is a product call, not a compliance one; (3) all five open legal/privacy items from the 2026-08-09 report still stand untouched — EU/UK exposure of Google Fonts + jsDelivr, whether a privacy notice is required, business PII in a consumer Gmail account, and whether pricing/tiger/ may deploy unauthenticated. Those warrant counsel, not an agent.
