# K13 Website — ADA/508 accessibility + privacy assessment

**Date:** 2026-08-09 · **Stage:** compliance (assess-only, no files modified)
**Target:** `/Users/k13/Desktop/PROJECTS/K13_Website/index.html` (831 lines, single file, inline CSS/JS)
**Standard applied:** WCAG 2.2 Level AA (the measurable standard behind ADA Title III and Section 508)

## How this was checked

Not eyeballed. Three methods, all reproducible:

1. **Contrast** — every foreground/background pair in the stylesheet was resolved to sRGB (including
   compositing the semi-transparent `--glass`, `--line`, `--line-2` and `rgba(255,255,255,…)` layers
   against what actually sits behind them) and run through the WCAG relative-luminance formula.
   50 text pairs and 10 non-text pairs computed. Every ratio quoted below is calculated, not estimated.
2. **Live browser** — Chromium via Playwright at 1440/390/360/320px. Real tab traversal recording
   `document.activeElement` and its *effective* opacity through all ancestors; computed focus outlines;
   the Chromium accessibility tree (via CDP `Accessibility.getFullAXTree`) for landmark and
   accessible-name truth; reflow measurement; network request capture; cookie/storage inspection.
3. **Photo pixels** — for the one text-over-image case, the caption was hidden, the region screenshotted,
   and all 7,000 background pixels sampled to get the real contrast range rather than guessing.

**Verified conformance claim:** none. This page does **not** conform to WCAG 2.2 AA today. The failures
below are individually confirmed. Passing items are noted as passing *the checks listed*, which is not
the same as a full audit — no assistive-technology user testing was performed, and screen-reader
behaviour was inferred from the accessibility tree rather than observed in JAWS/NVDA/VoiceOver.

---

## 1. Critical — keyboard users fall into 12 invisible controls

**The single most serious defect on the page.** Verified in Chromium, not deduced from CSS.

The contact stepper hides its inactive steps with `opacity:0; pointer-events:none` (`.screen-s`,
line 231–233) and hides the Back button the same way (`.step-back`, line 218). Neither property
removes an element from the tab order — only `display:none`, `visibility:hidden`, `inert`, or
`tabindex="-1"` do that. So every control on every step is focusable at all times while being
completely invisible.

Recorded tab traversal, page freshly loaded, contact section scrolled into view and fully revealed:

| Tab stop | Element | Effective opacity | In viewport |
|---|---|---|---|
| 1 | `button#stepBack` "‹ BACK" | **0.00** | yes |
| 2–5 | `button.opt` ×4 (the visible step 1) | 1.00 | yes |
| 6 | `input#sName` | **0.00** | yes |
| 7 | `button#nameNext` "Continue →" | **0.00** | yes |
| 8 | `input#sEmail` | **0.00** | yes |
| 9 | `button#emailNext` "Continue →" | **0.00** | yes |
| 10 | `button#emailSkip` "SKIP" | **0.00** | yes |
| 11 | `textarea#sWhy` | **0.00** | yes |
| 12 | `button#sendIt` "Send it →" | **0.00** | yes |

A keyboard-only or screen-reader user tabbing through the contact area hits **eight controls in a row
that they are told exist but cannot see** — announced, focusable, operable, invisible. Sighted keyboard
users lose the focus ring entirely for eight consecutive stops.

**Functional bug riding along with it:** `#sendIt` is reachable and operable from step 1. Pressing it
there runs the handler at line 816 with the `data` object still empty — it shows the "Your email is
ready." success panel and opens a `mailto:` with an empty body and no name, type or reason. The
visitor is told their email is ready when nothing was collected.

- Fails **2.4.3 Focus Order** (A), **2.4.7 Focus Visible** (AA), **2.4.11 Focus Not Obscured** (AA),
  and **4.1.2 Name, Role, Value** (A) — controls are exposed in a state the user cannot perceive.

## 2. Critical — no `<main>` landmark and no skip link

The Chromium accessibility tree exposes exactly three landmarks: `banner`, `navigation`, `contentinfo`.
There is **no `main`**, and there is no skip link anywhere in the document.

There are 17 links between the top of the page and the contact form. A keyboard user who wants to
enquire must tab through all of them on every page load, with no bypass mechanism and no landmark to
jump to. The six `<section>` elements have no accessible names, so they are not exposed as regions
either — screen-reader landmark navigation offers nothing between the header and the footer.

- Fails **2.4.1 Bypass Blocks** (Level **A** — the lowest bar in the standard).

## 3. Critical — the three form fields have no labels

Confirmed against the Chromium accessibility tree. There are **zero `<label>` elements** in the file,
no `aria-label`, no `aria-labelledby`, and no `<form>`:

| Control | Computed accessible name | Name source |
|---|---|---|
| `#sName` | "Type your name" | **placeholder** |
| `#sEmail` | "you@email.com" | **placeholder** |
| `#sWhy` | "Tell me the real reason..." | **placeholder** |

The placeholder is the *only* label, and it disappears the moment the user types — so anyone who is
interrupted mid-entry, or who relies on magnification and cannot see the whole field at once, has no
way to recover what the field was for. The question text that gives each field its real meaning
("Nice. *Who are you?*", "Where do I *reach you?*") is in a sibling `<div class="q">` with no
programmatic association at all.

Related gaps in the same component: no `required`, no validation, no error messaging, and the success
panel (`#sent`, line 615) has no `aria-live` and does not move focus — a screen-reader user gets no
announcement that anything happened, while `window.location.href` is reassigned 700ms later.

- Fails **1.3.1 Info and Relationships** (A), **3.3.2 Labels or Instructions** (A), **4.1.2** (A).
  The missing status announcement fails **4.1.3 Status Messages** (AA).

## 4. Colour contrast — computed ratios

The palette's structural colours are strong. Everything that fails is the same two decisions repeated:
`--faint` used as a text colour, and `--blue` used at small sizes.

### Passing comfortably

| Pair | Usage | Ratio | Needs |
|---|---|---|---|
| `--ink-2 #2A3554` on `--paper #F3F5FA` | body copy, hero sub | **11.09** | 4.5 |
| `--ink #141D35` on `--paper` | all headings, `strong`, `.facts .v` | **15.32** | 4.5 |
| `--muted #515C78` on `--paper` | section notes, row descriptions, footer, nav | **6.11** | 4.5 |
| `--muted` on `--paper-2 #FFFFFF` | card body copy | **6.66** | 4.5 |
| `--muted` on `--paper-3 #E9EDF7` | capabilities column copy | **5.69** | 4.5 |
| `--ink-2` on `--paper-3` | capabilities stack rows (13.5px) | **10.33** | 4.5 |
| `--ink` on glass `#FAFBFD` | status feed line | **16.14** | 4.5 |
| `#fff` on `--ink` | `.btn:hover` | **16.71** | 4.5 |
| `--blue` on `--paper`, ≥24px | `h2 i`, hero `.ci`, `.about .hl` | **4.18** | 3.0 (large) |

### Failing — body text under 4.5:1

| Pair | Where | Size | Ratio | Verdict |
|---|---|---|---|---|
| `--faint #8089A1` on `--paper-3` | `.cap-col .stk em` ("core", "app", "data" …) | 11px | **2.98** | fail |
| `--faint` on `--paper` | `.brand .tag` ("SOFTWARE STUDIO") | 10px | **3.20** | fail |
| `--faint` on `--paper` | `.row .rnum` (01–10) | 13px | **3.20** | fail |
| `--faint` on glass `#FAFBFD` | `.status .yr` / `.now` | 11px | **3.37** | fail |
| `--faint` on glass | `.step-lab` ("Step 1 of 3"), `.step-back` | 11px | **3.37** | fail |
| `--faint` on glass | `.s-skip` ("skip") | 11px | **3.37** | fail |
| `--faint` on glass | **all three input placeholders** | 19 / 16.5px | **3.37** | fail |
| `--faint` on chip bg | `.step-sum .sc b` | 10.5px | **3.46** | fail |
| `--blue #CB4D14` on `--paper-3` | `.eyebrow` in the capabilities section | 12px | **3.89** | fail |
| `--blue` on `--paper` | `.eyebrow` ×5, `.row .go` ("Open ↗") | 12px | **4.18** | fail |
| `--blue` on `--paper` | `.contact .mail b` (the email address) | 14px | **4.18** | fail |
| `--blue` on `--paper` | `.btn.ghost:hover` label ("See the work") | 14.5px | **4.18** | fail |
| `#27B584` on `#FFFFFF` | `.sent .ring` check glyph | 27px | **2.62** | fail (needs 3.0) |

The placeholder failure compounds §3: the placeholders are the only labels *and* they are the lowest-
contrast text in the component.

### Failing worst — the portrait caption over the photograph

`.p-cap` sits on top of `assets/kazim.jpg`. Background pixels sampled directly from a render with the
caption hidden (7,000 px across the caption's box):

- Background range: darkest `rgb(107,104,99)` → lightest `rgb(227,224,219)`, mean `rgb(207,204,199)`.
- `.p-cap span` — `--blue #CB4D14`, **9px** ("FOUNDER" / "K13 SOFTWARE STUDIO"): **1.22:1 at worst,
  2.84:1 at the mean, 3.46:1 at best.** Fails 4.5:1 everywhere on the image.
- `.p-cap b` — `--ink`, 20px 600-weight (qualifies as large text): 3.01:1 worst / 10.43:1 mean.
  Scrapes past the 3:1 large-text bar at its worst point. Marginal, not broken.

The white `text-shadow` glow on line 188 helps visually but carries no weight under WCAG — contrast is
measured against the background colour, and a shadow cannot be relied on over arbitrary image content.

### Razor-thin passes worth naming

`#fff` on `--blue #CB4D14` = **4.55:1** against a 4.5 requirement. Every primary button, `.s-next`,
and `::selection` sits on a 0.05 margin. Any future darkening of the label, lightening of the orange,
or use of a lighter font weight breaks AA silently. `.card .num` (blue on white, 12px) is the same
4.55:1. Treat both as already-at-the-limit, not as headroom.

### Non-text contrast (SC 1.4.11, needs 3:1)

| Element | Composited pair | Ratio | Verdict |
|---|---|---|---|
| Input underline at rest (`--line-2` → `#C7CAD1`) vs field bg | the *only* thing that says "this is a field" | **1.58** | fail |
| `.opt` button border vs its background | boundary of an actionable control | **1.58** | fail |
| `.btn.ghost` border vs `--paper` | boundary of an actionable control | **1.58** | fail |
| `.step-prog i` "off" dot vs glass | progress state indicator | **1.58** | fail |
| `#27B584` status dot vs glass | 2.53 | **2.53** | fail (decorative — parent is `aria-hidden`) |
| Card / row separators (`--line` → `#D8DBE2`) | purely decorative | 1.27 | not applicable |
| Focus underline (`--blue`) vs field bg | **4.40** | pass |
| `.step-prog i.on` (blue) vs glass | **4.40** | pass |

## 5. Focus visibility

Good news first: **every link and button keeps the browser's default focus ring** (computed
`outline: auto 1px rgb(0,95,204)`). Nothing does the classic `*:focus{outline:none}`. That is better
than most sites of this kind.

The exception is line 247 — `.s-field input, .s-field textarea { outline: none }`. All three text
fields have `outline-style: none` confirmed at runtime. Their only focus indicator is
`.s-field:focus-within` changing a 1.5px underline from `#C7CAD1` to `--blue`. That transition is
4.40:1 against the field background, so it is perceivable and arguably satisfies 2.4.7 — but it is a
colour-change-only, 1.5px cue, it fails the WCAG 2.2 **Focus Appearance** metric (AAA, so advisory),
and it is fragile. There is no `:focus-visible` styling anywhere in the file.

## 6. Motion and `prefers-reduced-motion` — the strongest area of the page

The `@media (prefers-reduced-motion: reduce)` block at line 303 is real and it works. Verified by
loading the page in a Chromium context with `reduced_motion: 'reduce'`:

| Behaviour | Under reduced-motion | Result |
|---|---|---|
| `.card` transitions | `transition-duration: 0.001s` | neutralised |
| Status-dot `beat` keyframes | `animation-duration: 0.001s` | neutralised |
| `.rv` scroll reveals | all sampled elements `opacity: 1` | content shown immediately, not gated on scroll |
| Lenis smooth scroll | `html.lenis` never applied — **the library is never initialised** | correct (line 643) |
| Portrait parallax | early-return at line 764 | disabled |
| Count-up numbers | jump straight to final value (line 667) | correct |
| Rotating status feed | text unchanged after 6s | stopped |

This is a genuinely thorough implementation — the JavaScript honours the preference, not just the CSS,
which is the part most sites miss. It is also why the scroll-reveal pattern does not create a
content-visibility trap: with motion reduced, nothing depends on an IntersectionObserver firing.

**One motion gap remains — SC 2.2.2 Pause, Stop, Hide (AA).** The `#feed` line rotates through 49
strings every 3,800ms, indefinitely, starting automatically. Verified changing within 9 seconds at
default settings. There is no pause, stop or hide control anywhere on the page. `prefers-reduced-motion`
switches it off, which is a strong mitigation — but 2.2.2 asks for a mechanism the user can operate,
and a visitor who has not set that OS preference has no way to stop it. Mitigating factor: the whole
`<aside class="status">` is `aria-hidden="true"`, so screen readers are not disturbed by it; the issue
is confined to sighted users with attention or vestibular sensitivities. **Likely AA failure — worth a
second opinion before it is called one.**

## 7. Clean passes

These were checked and are genuinely fine.

- **`alt` text — full pass.** Two `<img>` in markup plus ten injected at runtime. The portrait carries a
  real, descriptive alt ("Kazim K., founder of K13 Software Studio, leaning on a plinth in front of a
  burnt-orange circle"). The hover-preview image (`#peekImg`) and the ten mobile thumbnails correctly
  use `alt=""` — they are decorative duplicates of link text that already names each project. Getting
  decorative-vs-informative right in both directions is the hard part, and this file does.
- **Heading order — pass.** `h1 → h2 → h3 h3 h3 → h2 → h2 → h3 h3 h3 → h2 → h3`. No skipped levels,
  exactly one `h1`, all headings are real heading elements (the `.eyebrow` labels are correctly *not*
  headings). Minor note, not a failure: the About section is the only one with no heading, so it
  cannot be reached by heading navigation.
- **`lang="en"`** present and correct on `<html>`. Descriptive `<title>` and `meta description`.
- **Zoom permitted** — `width=device-width, initial-scale=1.0` with no `maximum-scale` or
  `user-scalable=no`. Passes 1.4.4.
- **Reflow — pass.** Measured at 320, 360 and 1280px: `scrollWidth === clientWidth` at every width and
  zero elements overflow the viewport. No two-dimensional scrolling at 400% zoom. Passes 1.4.10.
  (`body { overflow-x: hidden }` exists but is not masking anything.)
- **Link text out of context — pass.** Each `.row` is a single link whose accessible name concatenates
  to "01 Tiger Hospitality One roof for a family of restaurant brands… Brand · Maps Open ↗". Verbose,
  but unambiguous when read from a links list. All ten carry `rel="noopener"`. New-tab behaviour is
  announced once at the section level ("Click to open the live site in a new tab") rather than per
  link — SC 3.2.5 is AAA, so this is advisory only.

## 8. Smaller findings

- **Target size (SC 2.5.8, AA, 24×24 CSS px).** At a 390px viewport: `‹ BACK` is 44×**18**, `SKIP` is
  28×**18**. Both are standalone buttons and do not qualify for the inline-text exception, so both
  fail. The mail link (23px), phone link (21px) and footer email (16px) are inline links within
  sentences and are covered by the exception.
- **`body { font-size: 18px }`** is a fixed pixel value, so the page ignores a user's browser
  default-font-size setting. Browser zoom still scales everything, so 1.4.4 passes — but users who
  raise their default font size instead of zooming get no benefit.
- **Anchor-jump offset.** The fixed header is 70px tall; `scrollTo` uses `offset: -30` and no
  `scroll-padding-top` is set, so an in-page jump leaves roughly the top 40px of the target section
  behind the header. Cosmetic here because the targets are sections with generous padding, not
  focusable elements — worth fixing when the header height next changes.
- **`data-c` attributes** appear on 20 elements and are referenced by nothing in the JavaScript.
  Dead markup, harmless, presumably left from the custom-cursor implementation the README describes.

---

# Privacy

## What the site does NOT do — verified, not assumed

Loaded in a fresh Chromium context and inspected after full page load:

- **Cookies set: zero.** Context cookie jar empty; `document.cookie` is `''`.
- **`localStorage`: empty. `sessionStorage`: empty.** No IndexedDB.
- **No analytics of any kind.** Source grep for `gtag`, `google-analytics`, `googletagmanager`,
  `analytics`, `plausible`, `fathom`, `umami`, `hotjar`, `clarity`, `fbq`, `pixel`, `segment`,
  `mixpanel`, `posthog`, `sentry`, `recaptcha` returns **zero matches**.
- **No data ever leaves the browser.** No `fetch`, no `XMLHttpRequest`, no `sendBeacon`, no WebSocket
  anywhere in the file. There is no backend.

**No cookie banner is required, because there are no cookies.** That is the correct outcome and it
should be preserved deliberately — it is the site's strongest privacy property.

## What it does do

**Three third-party hosts receive a request on every page load**, captured live:

| Host | What is fetched | What that host receives |
|---|---|---|
| `fonts.googleapis.com` | the CSS declaration | visitor **IP address**, User-Agent, Referer |
| `fonts.gstatic.com` | 4 `.woff2` files (Fraunces ×2, Inter, JetBrains Mono) | visitor **IP address**, User-Agent, Referer |
| `cdn.jsdelivr.net` | `lenis@1.1.18` | visitor **IP address**, User-Agent, Referer |

No consent gate, no notice. An IP address is personal data under GDPR Art. 4(1).

**Personal data collected by the contact stepper:** name, email address, and a free-text field
explicitly asking for something personal — *"Why does it matter to you? The honest version."* That
prompt invites disclosure well beyond contact details.

The redeeming detail, and it is a significant one: **there is no server**. `#sendIt` (line 816) builds
a `mailto:` string and hands it to the visitor's own mail client. Nothing is transmitted, stored or
processed until the visitor themselves presses send in their own mail app. That is a genuinely
privacy-preserving design and it should be stated as such if a notice is ever written — but it is
currently stated nowhere, so a visitor typing into that box has no way to know it.

**There is no privacy policy, no terms, no imprint, and no data-handling statement anywhere on the
site.** Inquiries are routed to `projects.k13@gmail.com`, a consumer Gmail account. A phone number,
`+1 949 306 6998`, is published in the clear.

`rel="noopener"` is set on all ten outbound links but not `noreferrer`, so each destination site
learns the visitor came from here. Minor.

## Flagged for human legal review — I am not deciding these

1. **Google Fonts / jsDelivr without consent.** Dynamically embedding Google Fonts was held to
   infringe GDPR Art. 6(1) in *LG München I, 3 O 17493/20* (20 Jan 2022), which triggered a large
   German warning-letter wave; the same reasoning applies to any CDN that sees visitor IPs, including
   jsDelivr. **Whether K13 is exposed depends on whether the site targets or serves EU/UK visitors —
   that is a legal determination about K13's market, not a technical one, and it is not mine to make.**
   Worth knowing while it is being decided: self-hosting the four font files and the one 
   library removes the question entirely, costs nothing, changes no pixel of the design, and makes the
   page faster. That is an engineering option, not legal advice.
2. **Whether a privacy notice is required at all**, given that the site sets no cookies and stores
   nothing. The third-party font/CDN requests are the only processing, and the answer turns on
   jurisdiction and on point 1.
3. **Business-inquiry PII in a personal Gmail account** — controller/processor posture, retention
   period, and whether that satisfies the obligations K13 takes on by inviting the "honest version"
   of why a project matters. Kazim's call with counsel.
4. **Accessibility exposure.** The site markets `WCAG · 508` as a paid capability (line 525) and
   describes accessibility as something K13 is "careful" about (line 398) while itself failing four
   Level A criteria. That is a marketing-claim question as much as a technical one — flagged, not
   judged.

---

# Out of scope but found

- **`pricing/tiger/index.html` — confidential client pricing on an unauthenticated path.** The file's
  own header comment reads: *"K13 PRIVATE · Tiger founding-partner pricing · unlinked, no auth yet.
  Do not add to nav/sitemap."* It contains a named client's commercial terms — partner discount
  percentages and dollar figures — protected only by being unlinked. `noindex, nofollow` keeps it out
  of search results; it does not keep anyone out of the file. If it deploys to the same host as
  `index.html`, the URL is the only thing standing between Tiger's negotiated pricing and anyone who
  has it. **Route to the security stage and to Kazim.** (Its accessibility posture is otherwise
  reasonable: `lang="tr"` correctly matches its Turkish content, `aria-label` on the quantity steppers,
  a `prefers-reduced-motion` block, one `h1`.)
- **`README.md` documents a design that no longer exists.** It specifies Ink `#15130F`, Paper
  `#F1ECE2`, Signal orange `#FF4F00`, muted `#6F6755`, Space Grotesk, plus a marquee, magnetic buttons,
  a custom cursor and a reviews section. `index.html` ships `#141D35` / `#F3F5FA` / `#CB4D14` /
  `#515C78` with Fraunces + Inter + JetBrains Mono and none of those components. The brief for this
  assessment used the README values, so I computed both: the README palette would fail *harder* —
  `#FF4F00` on `#F1ECE2` is **2.80:1** and white on `#FF4F00` is **3.30:1**, so the documented accent
  is unusable for text either way. **Every finding in this report is against the shipped `index.html`
  palette, which is the one that matters.** The README needs updating, but that is not a compliance fix.
- No `robots.txt`, no `sitemap.xml`, and no header configuration (`vercel.json` / `_headers`) in the
  repo. README says the host is Vercel, so security headers would be configured there — noted for the
  security stage.

# Suggested order of work

Roughly by severity per unit of effort. All are for a later stage — nothing was changed here.

1. Make inactive stepper screens truly hidden (`visibility:hidden`, or `inert` on non-active screens,
   or `display:none` once the transition ends) and hide `.step-back` the same way. Fixes eight Level A
   tab stops and the empty-send bug in one change.
2. Add real `<label>`s (visually hidden if the design requires) to `#sName`, `#sEmail`, `#sWhy`.
3. Wrap the page content in `<main id="main">` and add a skip link as the first focusable element.
4. Darken `--faint` and `--blue` for small text, or stop using them below 24px. `--faint` needs roughly
   `#6B7590` to clear 4.5:1 on `--paper`; `--blue` needs about `#B23F0E`. Fix `.p-cap span` first —
   at 1.22:1 it is the worst pixel on the site.
5. Give the input underline a 3:1 resting boundary, and restore a real focus indicator on the three fields.
6. Add a pause control to the rotating status feed (or accept the `prefers-reduced-motion` mitigation
   as a documented, reviewed decision).
7. Enlarge `SKIP` and `‹ BACK` to 24×24 minimum.
8. Self-host the fonts and Lenis — pending the legal decision, but it is the cheap answer either way.

## Status      NEEDS-REVIEW
## Summary     Verified in Chromium: 4 Level-A failures (8 invisible-but-focusable stepper controls in the tab order, no <main>/skip link, 3 form fields labelled only by placeholder) plus 13 computed contrast failures — worst is the 9px orange portrait caption at 1.22:1 over the photo. prefers-reduced-motion is thorough and genuinely good; alt text, heading order, lang and reflow all pass. Privacy is clean where it counts — zero cookies, zero analytics, zero storage, no backend — but Google Fonts + jsDelivr leak visitor IPs with no notice and there is no privacy policy anywhere.
## Files       Inspected: /Users/k13/Desktop/PROJECTS/K13_Website/index.html, /Users/k13/Desktop/PROJECTS/K13_Website/README.md, /Users/k13/Desktop/PROJECTS/K13_Website/pricing/tiger/index.html, /Users/k13/Desktop/PROJECTS/K13_Website/.gitignore. Created: /Users/k13/Desktop/PROJECTS/K13_Website/docs/handoffs/compliance_2026-08-09.md. Modified: none.
## Risks       The site sells "WCAG · 508" as a paid capability while failing four Level A criteria — an ADA demand letter would cite this page as evidence against its own marketing claim. White-on-#CB4D14 buttons pass at exactly 4.55:1, so any future tweak to the brand orange breaks AA across every CTA silently. pricing/tiger/index.html carries a named client's confidential pricing on an unauthenticated, unlinked path — obscurity is its only protection. README.md documents a superseded palette, so anyone trusting it will compute contrast against colours the site does not use.
## Next        security-auditor — for the unauthenticated pricing/tiger path, the missing SRI on the jsDelivr script, and the absent security-header configuration. Then frontend-engineer for the remediation list, re-running this assessment as the gate.
## Human gate  Kazim decides: (1) whether the site targets EU/UK visitors, which determines if the consent-free Google Fonts + jsDelivr embedding is a real GDPR exposure (LG München I, 3 O 17493/20) — a legal question I did not decide; (2) whether a privacy notice is required at all given zero cookies and no backend; (3) whether business-inquiry PII should keep landing in a consumer Gmail account; (4) whether pricing/tiger/index.html may deploy to a public host with no auth; (5) whether marketing "WCAG · 508" while the site fails Level A is acceptable until remediation ships. Items 1-3 warrant counsel, not an agent.
