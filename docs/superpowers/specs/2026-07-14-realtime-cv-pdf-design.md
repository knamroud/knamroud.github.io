# Real-time CV PDF + Contact Section De-duplication — Design

**Date:** 2026-07-14
**Scope:** `index.html` (single-file static site, GitHub Pages)
**Status:** Approved for planning

## Problem

1. The bottom **Contact section** (`index.html:765-774`) duplicates the four links
   already present in the sidebar (`index.html:590-595`): Email, GitHub, LinkedIn,
   Credly. It is redundant.
2. There is no way to obtain a CV/résumé from the site. The owner wants to
   **generate a PDF CV based on the live site, in real time** — i.e. the PDF must
   always reflect the current site content and the currently selected language,
   never a stale static file.

## Goals

- Remove the redundant contact links from the page body without losing any
  contact information (the sidebar keeps them).
- Add a one-action "Download CV" flow that produces a clean, light,
  recruiter/ATS-friendly PDF résumé from the live page.
- The PDF is generated from the same DOM/i18n data the site already renders, so
  it is inherently real-time and language-correct.
- No external dependencies, no build step (must work as a static file on GitHub
  Pages), no change to the on-screen appearance of the site.

## Non-goals

- No server-side PDF rendering, no bundled PDF/JS library, no pre-generated
  static PDF file.
- No visual redesign of the on-screen site beyond the contact→CV section change.
- No new content/facts — the résumé reuses existing content only.

## Chosen approach

**Print CSS over the existing DOM.**

A single `@media print` stylesheet reflows the live page into a clean light
résumé and `window.print()` lets the browser's "Save as PDF" produce the file.
The page already stores every fact exactly once (in the `i18n` object and the
section markup), so the print layout renders those same nodes — no duplication,
no drift, genuinely real-time and language-aware.

### Alternative considered (rejected)

A hidden `<div>` repopulated from `i18n` by JS, styled as a résumé. Gives finer
layout control but reintroduces content duplication — the exact problem being
removed from the contact section — and adds JS for something CSS can do. Rejected.

## Detailed design

### A. Contact section → CV section (`index.html:765-774`)

- Remove the four duplicate `.contact-link` buttons.
- Repurpose `#contact` as the CV download call-to-action: a short CTA line plus
  the download button.
- The four contact links remain in the sidebar (`index.html:590-595`), untouched
  — no information is lost.
- Rename the nav item that targets `#contact`: **Contatti → CV** / **Contact → CV**
  (`nav_contact` value in both locales). The href stays `#contact`.

### B. Download button + i18n

- One `<button class="cv-download" onclick="window.print()">`.
- New i18n keys in both `it` and `en`:
  - `cv_download` — button label: "Scarica CV" / "Download CV"
  - `cv_cta` — CTA line: e.g. "Scarica una copia del mio CV in PDF" /
    "Download a PDF copy of my CV"
- Styled like the existing `.contact-link` pill but with an accent fill so it
  reads as the primary action.
- Lives in the repurposed `#contact` (CV) section. Hidden in print.
- Screen-only; no effect on print output.

### C. Print stylesheet — the résumé itself

A single `@media print { ... }` block added to the existing `<style>`. It:

- **Hides** sidebar, `nav`, language toggle, the download button, the blinking
  `.cursor`, and decorative timeline dots/connector lines
  (`.tl-item::before` / `.tl-item::after`).
- **Recolors** to white background / near-black text; sets
  `print-color-adjust: exact` (and `-webkit-`) only where a restrained accent
  rule under section headings is intentionally kept, so browsers neither drop it
  nor re-tint the whole page.
- **Reveals** a print-only header block (`display: none` on screen,
  shown in print): **Name · role · location · contact line**
  (Email · GitHub · LinkedIn · Credly). Role reuses the existing `role` i18n
  value so it stays language-correct; location reuses the hero's static string
  ("Greater Milan Metropolitan Area", identical in both locales). This puts
  contact details on the printed page even though the on-screen contact buttons
  were removed.
- **Reflows** content compactly:
  - Hero tagline → résumé summary.
  - Experience / Education / Skills / Homelab / Certifications stack tightly.
  - Skills render as a wrapped inline list instead of large boxed tags.
  - Company/cert links render as plain text (no underlines/URLs needed on paper).
- **Page rules:** `@page { size: A4; margin: ~1.5cm; }`,
  `break-inside: avoid` on each experience/education/cert item,
  `break-after: avoid` on section headings. Target ~2 pages.
- **Font fallback:** system sans-serif stack behind the Google webfonts so an
  offline print still looks correct.
- The Homelab block is **included** in the PDF as a compact infrastructure entry
  (kept as a differentiator per owner decision).

### D. Data flow / real-time & language

No content is copied anywhere. The print layout renders the same DOM nodes that
`setLang()` already fills. Sequence: user toggles IT/EN (optional) → clicks
Download CV → `window.print()` → browser renders the `@media print` layout of the
current DOM → user saves as PDF. Result: a résumé in whatever language is active,
reflecting whatever is currently on the page. The site remains the single source
of truth.

### E. Error handling / edge cases

- `window.print()` is universally available; cancelling the dialog changes
  nothing.
- Offline / webfont failure → system-font fallback keeps layout intact.
- Content slightly exceeding 2 pages still prints correctly (layout is tuned to
  fit, not hard-capped).
- All print rules are scoped to `@media print`, so the on-screen site is
  provably unchanged.

## Testing / verification

- **Manual:** open the print preview in both IT and EN; confirm the light résumé
  layout, clean page breaks, and that sidebar / nav / toggle / button /
  animations are absent while the on-screen view is unchanged.
- **Automated/objective:** render `index.html` to an actual PDF with headless
  Chromium (`--print-to-pdf`) to confirm real output, not just an assertion.
  Spot-check that text is selectable (not rasterized) and that both language
  variants produce a correct résumé.

## Open decisions (defaults chosen, overridable)

- Homelab block **kept** in the PDF (compact). — default
- Download button lives **only** in the CV section (a sidebar copy is optional
  and not included by default). — default
