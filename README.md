# K13 Software Studio: Brand & Website Kit

*Software with an edge. Crafted with care.*

## The idea

The mark is the letter **K built from a "1" and an angular "3"**: the whole name locked
into a single glyph. One mark, two readings. Built like a 13, we make our own luck.

## Palette

| Role | Hex |
|---|---|
| Ink (primary dark) | `#15130F` |
| Paper (cream) | `#F1ECE2` |
| Paper deep | `#E9E1CF` |
| Cream on ink | `#EFE9DC` |
| Signal orange (accent) | `#FF4F00` |
| Muted text | `#6F6755` |

## Type

Two families, two voices, nothing else:

- **Space Grotesk** does all the work: headlines (700), body (400), and small
  tracked-uppercase labels (500).
- **Fraunces Italic** appears only as a deliberate accent: the word "edge." in the hero,
  single accent words in section titles, review quotes, and taglines.

Both are free on Google Fonts.

## Contact

All inquiries route to **projects.k13@gmail.com** (form and mailto links are wired to it).
The public domain is **k13projects.com**.

## What's in this kit

```
index.html                      One-page site in a single file, zero build step.
                                Lenis smooth scrolling, scroll-driven reveals,
                                parallax "K13" backdrop, animated underline,
                                marquee, magnetic buttons, custom cursor,
                                inquiry form that opens the visitor's mail app
                                (no backend needed), reviews, mobile responsive.

brand/logo/                     SVG masters and PNG exports of the mark,
                                cream/dark/mono variants, horizontal lockup.

brand/print/k13-letterhead.pdf      Ready-to-use A4 letterhead.
brand/print/k13-business-card.pdf   3.5 x 2 in card, front and back pages.
brand/print/*-print.html            Print MASTERS with the exact webfonts.
                                    Open in a browser, then File > Print >
                                    "Save as PDF" for pixel-perfect type.
                                    (The .pdf files use the closest system
                                    fonts; the HTML masters are exact.)
brand/print/previews/               PNG previews of the print pieces.
```

## Hooking up k13projects.com

The site is one static file, so any host works:

1. **Vercel (current setup):** the repo is imported as the `k13-website` project;
   `main` serves production and branch subdomains (b1, b2, ...) serve stakeholder previews.
2. **Any shared host:** upload `index.html` to the web root. Done.

To change the contact address later, search for `projects.k13@gmail.com` in
`index.html` and replace all occurrences.
