# CLAUDE.md — K13 Software Studio (the shopfront site)

> House rules for this repo. Full org conventions live in the War Room
> (`/Users/k13/Desktop/PROJECTS/K13-WarRoom/starter-kit/CONVENTIONS.md`); Kazim's full operating
> character is one read away in `K13-WarRoom/K13_GENOME.md` — start there on a fresh session.

## Identity
- **Author/owner:** Kazim (K13 / Kazimiro). This is Kazim's own project — the studio's shopfront.
- Collaborators Halil, Memo (MCS), Volkan are **EDISYN-only** — not involved here.

## Project
- **What:** K13 Software Studio's marketing site — a single-page pitch plus a per-client pricing
  subpage. It is the studio's front door, so its own quality is the argument.
- **For:** K13 itself (self-marketing).
- **Shortcode:** `site13`   ·   **Deploy:** Vercel (live at **k13projects.com**)
- **Repo:** https://github.com/k13-projects/K13_Website

> Note: the port registry in `CONVENTIONS.md` records the shortcode as `site`, but every branch in
> this repo's history uses **`site13`** (`site13_jul03_v4`, `site13_aug10_v1`). The repo's own
> precedent wins; the registry row is the thing that's drifted.

## Run locally
- **Assigned dev port: `9130`** (K13 dev-port registry — one fixed port per project, for life).
  Static site, no build step:
  ```sh
  python3 -m http.server 9130 --bind 127.0.0.1
  ```
- Open: **http://localhost:9130/** · macOS: never use 5000/7000 (AirPlay squats them).

## Stack
Static HTML/CSS/JS — **no build step, no `package.json`**. One `index.html` carries its own inline
CSS and JS. Lenis for smooth scroll (jsDelivr, SRI-pinned). Google Fonts. Contact is a client-side
`mailto:` composer, not a backend form.

Because there is no bundler, **`index.html` is the whole application** — treat edits to it with the
care you'd give a source tree, and expect concurrent agents to collide there. Partition by file.

## Security & deploy
- `vercel.json` carries the CSP and 5 security headers. **The CSP allowlist is derived from captured
  network requests** (`fonts.googleapis.com`, `fonts.gstatic.com`, `cdn.jsdelivr.net`) — if you add
  an external resource you MUST update it, and prove it by serving locally with those exact headers
  and confirming zero CSP violations. A wrong CSP takes production down on the next deploy.
- `script-src` currently needs `'unsafe-inline'` because the JS is inline. Externalising that script
  would let it go — no build step required. Known follow-up.

## Accessibility — this one is not optional here
The site **markets "WCAG · 508" as a paid capability**, so shipping a11y failures is a credibility
problem, not just a defect. As of 2026-08-11 it passes axe-core with zero violations. Keep it there:
- Inactive contact-stepper steps must be genuinely `inert` — `opacity:0; pointer-events:none` does
  **not** remove elements from the tab order, and that exact mistake shipped a form that told
  visitors their message was sent when nothing had been.
- Never claim an action succeeded unless you can verify it did.
- Contrast floors: 4.5:1 body text, 3:1 large text and non-text boundaries. Several pairs sit near
  4.5 — recompute before changing any colour token.
- Gate all motion behind `prefers-reduced-motion` (currently the page's strongest area).

## ⚠️ Testing the contact stepper — stub `mailto:` first

The stepper ends by handing a `mailto:` to the OS. **Chromium delegates that to the real mail
client even in headless mode**, so every end-to-end walk-through opens a Mail compose window on
Kazim's Mac. A QA pass on 2026-08-11 walked it repeatedly and buried his desktop in identical
drafts.

Before testing this flow, intercept the navigation instead of letting it reach the OS:

```js
// Playwright — capture the attempt, assert on it, never hand it to macOS
const mailtos = []
await page.route('mailto:**', r => { mailtos.push(r.request().url()); r.abort() })
await page.evaluate(() => { window.__mailto = []
  document.addEventListener('click', e => {
    const a = e.target.closest('a[href^="mailto:"]')
    if (a) { e.preventDefault(); window.__mailto.push(a.href) }
  }, true)
})
```

This is also the *better* test: it lets you assert on the encoded subject and body, which no pass
has actually verified yet.

## How Claude/Cursor should work here
- Plan mode for non-trivial tasks; use subagents liberally, one task each.
- **Never commit directly to `main`** — every change goes branch → PR → real merge
  (`gh pr merge --merge`, keep branches). Data refreshes included.
- Branch naming: `site13_<monDD>_v<N>` (e.g. `site13_aug11_v1`).
- **Hail Mary:** `hm` → new branch + documented grouped-bullet commit + `git push -u`;
  `hm-1` commits on the current feature branch; `hm++` also PRs and merges.
- **No em dashes in visitor-facing copy** (house rule) — that includes OG/Twitter meta, which shows
  up in share cards. Rewrite with commas, colons or periods. House docs like this one are exempt.
- Verify before done: build, logs, diffs, and actually run it in a browser.
- Delivery-pipeline work goes to its named agent (see `K13-WarRoom/starter-kit/ORG.md`);
  handoff artifacts land in `docs/handoffs/<stage>_<YYYY-MM-DD>.md` and feed the War Room Org tab.

## Settled decisions (dated — do not re-litigate)
- **2026-08-18 · Brand canon is the shipped site, and the square mark is dead.** Kazim saw the
  rounded-square glyph mark (dark tile, cream "1" stem, `#FF4F00` angular "3" — `k13-mark.svg`
  and friends) surface in a client deliverable, did not recognize it as his logo, and ordered it
  destroyed. All `brand/logo/` mark/glyph/lockup files and the mark-based `brand/print/`
  collateral (business card, letterhead, previews) were removed from the working tree that day.
  **The official K13 identity is what `index.html` ships:** the typographic nav lockup
  (Fraunces "K" in ink `#141D35`, "13" in the orange accent, JetBrains Mono "SOFTWARE STUDIO"
  tag) and the Kazim-approved typographic `favicon.svg`. Never regenerate, reference, or
  recreate the square mark in any project.
- **2026-08-19 · Brand triage.** Kazim reviewed the full brand inventory card by card and ruled:
  keep the typographic identity, the root `favicon.svg`, `assets/kazim.jpg`, all ten Work shots,
  the 13 live color tokens, and the Fraunces/Inter/JetBrains Mono trio. Removed for good: the
  duplicate `brand/favicon.svg`, `assets/kazim 2.jpg`, the orphaned root `Kazim Image.png`,
  Space Grotesk (old README canon), and the entire quarantined mark family (purged, no restore).
  The `brand/` folder is gone; `README.md` was rewritten the same day to match this canon.

## Open decisions (do not decide these yourself)
1. **`pricing/tiger/`** — a named client's confidential pricing, untracked with no git backup.
   Verified not leaked (404 on production, never pushed). Whether it belongs in this repo is
   Kazim's call; it has been deliberately excluded from every commit so far.
2. **Branch protection on `main`** is not enabled, so the never-commit-to-main rule is honour-only.

## Known follow-ups
- ~2.5MB of oversized `assets/shots/*.jpg` with no WebP or `srcset`; plus an orphaned 1.9MB
  `Kazim Image.png` and an unused `assets/kazim 2.jpg`.
- `scripts/` has never been committed — `hm.sh`/`ship.sh` exist only on Kazim's Mac.
- Self-XSS `innerHTML` sink in the stepper summary chips (confirmed non-exploitable: no backend,
  no URL-param feed).


---

<!--K13_BROADCAST_START · managed by War Room — do not hand-edit-->
## 📡 War Room Broadcasts (org-wide rules)
> Synced from the K13 War Room. Each entry is a house rule that applies to every K13 project. Managed automatically — edit the rule in the War Room, not here.

<!--bc:2026-06-29-agent-agency-org-->
### 2026-08-13 · Team K13: named departments, the handoff contract & the autonomy contract
**K13 runs as team K13 — a controlled delivery pipeline, not a swarm.** Each AI specialist owns one repeatable stage, emits a predictable artifact, and hands off cleanly to the next. The **main Claude session is the GM (James)** — the only layer that sequences work (the hierarchy is flat: subagents don't spawn subagents, so agents never hand off to each other directly). **Jessica** runs Kazim's desk.

- **Roster + status legend:** `starter-kit/ORG.md` (War Room). Lean 7 to build first: Selma (`solutions-architect`) → Valentina (`brand-dna-designer`) → Natalia (`frontend-engineer`) → Olga (`qa-test-engineer`) → Irina (`security-auditor`) → Kate (`release-engineer`) → Gabi (`report-writer`). Human names are display labels; the functional `name:` is the routing key.
- **Handoff contract + Definition of Done:** `starter-kit/AGENT_HANDOFF_PROTOCOL.md`. Every delivery agent ends with the handoff block (Status / Summary / Files / Risks / Next / Human gate) and writes its artifact to `docs/handoffs/<stage>_<YYYY-MM-DD>.md` (same-day re-run → `_v2`, never overwrite).
- **Delegation is not optional.** James does not do a pipeline stage's work himself and call it done — every stage gets its named agent actually invoked (Task tool, `subagent_type` matching the agent file), even on a small project. **No artifact = the work never happened**: the War Room Org tab reads only `docs/handoffs/`, so skipping the artifact makes team K13 invisible on the board.
- **Autonomy contract — don't drip questions at Kazim.** Agents proceed by default. Only `Human gate` items come back to him: irreversible/destructive steps, money, real scope changes, anything that leaves for a client. Every other decision gets made, then **recorded in the handoff** instead of asked. Questions that genuinely survive are batched at the end of a run — never one at a time.
- **Parallel work:** sequential by default; James may fan out several agents **concurrently for independent work** (QA dimensions, security + a11y, research) and relay findings between them — each still writes its own handoff.
- **Agent vs skill:** token-heavy + isolatable → agent; in-context checklist/workflow → skill (compliance-checklist, media-generation).

<!--K13_BROADCAST_END-->
