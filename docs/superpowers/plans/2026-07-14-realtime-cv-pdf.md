# Real-time CV PDF + Contact De-duplication — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the redundant on-page contact links and let the site produce a clean, light, ATS-friendly PDF résumé from the *live* page (current content + selected language) via the browser's print-to-PDF.

**Architecture:** All changes live in the single static `index.html`. The bottom Contact section is repurposed into a "Download CV" call-to-action whose button calls `window.print()`. A single `@media print` stylesheet reflows the existing DOM into a résumé and reveals a print-only header carrying the contact line. No content is duplicated — the print layout renders the same nodes `setLang()` already fills, so it is inherently real-time and language-aware.

**Tech Stack:** Plain HTML/CSS/vanilla JS (no framework, no build). Verification uses `python3` (structural checks) and, where available, a Chromium/Chrome headless `--print-to-pdf` render checked with `pdftotext`/`pdfinfo`.

## Global Constraints

- Single static file `index.html`; must work on GitHub Pages with **NO build step and NO external/runtime dependency** (no JS or PDF library).
- **No change to on-screen appearance** beyond the contact→CV section change; every print rule is scoped to `@media print`.
- **No new content/facts** — the résumé reuses existing site content only.
- **Real-time & language-aware:** the PDF must reflect the current DOM and the currently selected language (it prints the live page).
- The four sidebar contact links (`index.html:590-595`) remain intact.
- Use the correct hyphenated Credly URL `credly.com/users/kirelos-namroud` in the print header (the old contact section used a wrong non-hyphenated variant).
- Verification scripts live in the session scratchpad and are **not committed** to this public portfolio repo. In every command below:
  `SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad`

---

## File Structure

- **Modify only:** `index.html`
  - `<style>` — remove the now-unused `.contact-grid`/`.contact-link` rules; add screen styles for `.cv-cta` / `.cv-download` / `.cv-print-header`; append the `@media print` résumé block.
  - `<main>` hero — insert the print-only header markup after the name.
  - `#contact` section (`index.html:765-774`) — replace duplicate links with the CV CTA + button.
  - i18n `it` / `en` dicts — rename `nav_contact` to `CV`; add `cv_download` and `cv_cta`.
- **Scratchpad (not committed):** `$SCRATCH/check1.py`, `$SCRATCH/check2.py`, `$SCRATCH/print_pdf_check.sh`.

---

## Task 1: Repurpose Contact section → CV download CTA (DOM + i18n + screen CSS)

**Files:**
- Modify: `index.html` — CSS block `index.html:498-522`; section `index.html:765-774`; i18n `it` `index.html:791-792`; i18n `en` `index.html:838-839`.
- Test: `$SCRATCH/check1.py` (structural, runs in this shell).

**Interfaces:**
- Produces: a screen-only `<button class="cv-download" onclick="window.print()">`; i18n keys `cv_download`, `cv_cta`; `nav_contact` value `'CV'` in both locales. Task 2's print CSS relies on the selectors `.cv-download`, `.cv-cta`, and `#contact`.

- [ ] **Step 1: Write the failing structural check**

Create `$SCRATCH/check1.py`:

```python
import re, sys, pathlib
html = pathlib.Path("index.html").read_text(encoding="utf-8")
fails = []

# Old duplicate contact-link anchors removed from the page body
if 'class="contact-link"' in html:
    fails.append("still has .contact-link anchors (duplicate contact links not removed)")
# The section is kept (now the CV CTA)
if 'id="contact"' not in html:
    fails.append("#contact section missing")
# CV button present and wired to window.print()
if not re.search(r'class="cv-download"[^>]*onclick="window\.print\(\)"', html):
    fails.append("cv-download button with window.print() missing")
# CTA text element present
if 'data-i18n="cv_cta"' not in html:
    fails.append("cv_cta CTA element missing")
# i18n keys present in BOTH locales (>=2 dict occurrences each)
for key in ("cv_download", "cv_cta"):
    if len(re.findall(rf'\b{key}:\s*', html)) < 2:
        fails.append(f"i18n key {key} not defined in both locales")
# nav_contact renamed to 'CV' in both locales
if len(re.findall(r"nav_contact:\s*'CV'", html)) < 2:
    fails.append("nav_contact not renamed to 'CV' in both locales")

print("PASS" if not fails else "FAIL")
for f in fails:
    print(" -", f)
sys.exit(1 if fails else 0)
```

- [ ] **Step 2: Run the check to verify it fails**

Run: `cd /home/kiro/Progetti/knamroud.github.io && SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad && python3 "$SCRATCH/check1.py"`
Expected: prints `FAIL` with lines including "still has .contact-link anchors", "cv-download button …", "cv_download not defined in both locales", "nav_contact not renamed …". Exit code 1.

- [ ] **Step 3: Replace the Contact section markup**

In `index.html`, replace the whole `<!-- Contact -->` section (`index.html:765-774`):

```html
    <!-- Contact -->
    <section id="contact">
      <div class="section-label" data-i18n="nav_contact"></div>
      <div class="contact-grid">
        <a class="contact-link" href="mailto:namroudkirelos04@proton.me">✉ Email</a>
        <a class="contact-link" href="https://github.com/knamroud" target="_blank" rel="noopener">⌥ GitHub</a>
        <a class="contact-link" href="https://linkedin.com/in/knamroud" target="_blank" rel="noopener">⇗ LinkedIn</a>
        <a class="contact-link" href="https://credly.com/users/kirelosnamroud" target="_blank" rel="noopener">◈ Credly</a>
      </div>
    </section>
```

with:

```html
    <!-- CV download -->
    <section id="contact">
      <div class="section-label" data-i18n="nav_contact"></div>
      <p class="cv-cta" data-i18n="cv_cta"></p>
      <button type="button" class="cv-download" onclick="window.print()" data-i18n="cv_download"></button>
    </section>
```

- [ ] **Step 4: Swap the screen CSS for the section**

In `index.html`, replace the Contact CSS block (`index.html:498-522`, the `/* ── Contact ── */` rules for `.contact-grid` and `.contact-link`):

```css
    /* ── Contact ── */
    .contact-grid {
      display: flex;
      flex-wrap: wrap;
      gap: 0.8rem;
    }

    .contact-link {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      padding: 0.55rem 1rem;
      border: 1px solid var(--border);
      border-radius: 4px;
      text-decoration: none;
      color: var(--text);
      font-size: 0.82rem;
      font-family: 'JetBrains Mono', monospace;
      transition: border-color 0.15s, color 0.15s;
    }

    .contact-link:hover {
      border-color: var(--accent);
      color: var(--accent);
    }
```

with:

```css
    /* ── CV download ── */
    .cv-cta {
      color: var(--text);
      font-size: 0.9rem;
      max-width: 480px;
      margin-bottom: 1.2rem;
    }

    .cv-download {
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
      padding: 0.6rem 1.4rem;
      border: 1px solid var(--accent);
      border-radius: 4px;
      background: var(--accent-dim);
      color: var(--accent);
      font-family: 'JetBrains Mono', monospace;
      font-size: 0.82rem;
      font-weight: 500;
      cursor: pointer;
      transition: background 0.15s, color 0.15s;
    }

    .cv-download::before { content: '↓'; }

    .cv-download:hover {
      background: var(--accent);
      color: var(--bg);
    }

    /* print-only header (revealed in @media print) */
    .cv-print-header { display: none; }
```

- [ ] **Step 5: Update the `it` i18n dict**

In `index.html`, replace the line (`index.html:792`):

```javascript
    nav_contact:    'Contatti',
```

with:

```javascript
    nav_contact:    'CV',
    cv_download:    'Scarica CV',
    cv_cta:         'Scarica una copia del mio CV in PDF, generato dal vivo da questo sito.',
```

- [ ] **Step 6: Update the `en` i18n dict**

In `index.html`, replace the line (`index.html:839`):

```javascript
    nav_contact:    'Contact',
```

with:

```javascript
    nav_contact:    'CV',
    cv_download:    'Download CV',
    cv_cta:         'Download a PDF copy of my CV, generated live from this site.',
```

- [ ] **Step 7: Run the check to verify it passes**

Run: `cd /home/kiro/Progetti/knamroud.github.io && SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad && python3 "$SCRATCH/check1.py"`
Expected: prints `PASS`. Exit code 0.

- [ ] **Step 8: Commit**

```bash
cd /home/kiro/Progetti/knamroud.github.io
git add index.html
git commit -m "feat: replace redundant contact section with real-time CV download CTA

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Print stylesheet + print-only header (the résumé)

**Files:**
- Modify: `index.html` — insert print-only header markup after the hero name (`index.html:608`); append the `@media print` block just before `</style>` (`index.html:570`).
- Test: `$SCRATCH/check2.py` (structural, runs in this shell).

**Interfaces:**
- Consumes from Task 1: `.cv-download`, `.cv-cta`, `#contact`, and the `data-i18n="role"` mechanism.
- Produces: `.cv-print-header` / `.cv-print-role` / `.cv-print-contact` markup shown only in print; a complete `@media print` résumé stylesheet.

- [ ] **Step 1: Write the failing structural check**

Create `$SCRATCH/check2.py`:

```python
import re, sys, pathlib
html = pathlib.Path("index.html").read_text(encoding="utf-8")
fails = []

if not re.search(r'@media print\s*\{', html):
    fails.append("@media print block missing")
if 'cv-print-header' not in html:
    fails.append("cv-print-header markup/style missing")
if not re.search(r'class="cv-print-role"\s+data-i18n="role"', html):
    fails.append("print role element (class=cv-print-role, data-i18n=role) missing")
if 'credly.com/users/kirelos-namroud' not in html:
    fails.append("correct hyphenated Credly URL missing from print header")
if not re.search(r'@page\s*\{', html):
    fails.append("@page rule missing")
# chrome hidden in print (each selector must have a display:none rule somewhere)
for sel in (r'\.sidebar', r'\.lang-toggle', r'\.cv-download', r'#contact', r'\.footer'):
    if not re.search(sel + r'[^{}]*\{[^}]*display:\s*none', html):
        fails.append(f"{sel} not hidden with display:none (print rules)")
if 'break-inside: avoid' not in html and 'break-inside:avoid' not in html:
    fails.append("break-inside: avoid missing")

print("PASS" if not fails else "FAIL")
for f in fails:
    print(" -", f)
sys.exit(1 if fails else 0)
```

- [ ] **Step 2: Run the check to verify it fails**

Run: `cd /home/kiro/Progetti/knamroud.github.io && SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad && python3 "$SCRATCH/check2.py"`
Expected: prints `FAIL` with "@media print block missing", "cv-print-header markup/style missing", "@page rule missing", etc. Exit code 1.

- [ ] **Step 3: Insert the print-only header markup**

In `index.html`, find the hero name line (`index.html:608`):

```html
      <h1 class="hero-name">Kirelos (Kiro)<br>Namroud<span class="cursor"></span></h1>
```

Insert immediately **after** it:

```html
      <div class="cv-print-header">
        <div class="cv-print-role" data-i18n="role"></div>
        <div class="cv-print-contact">namroudkirelos04@proton.me · github.com/knamroud · linkedin.com/in/knamroud · credly.com/users/kirelos-namroud</div>
      </div>
```

- [ ] **Step 4: Append the `@media print` résumé stylesheet**

In `index.html`, find the end of the reduced-motion block and the closing `</style>` (`index.html:567-570`):

```css
    @media (prefers-reduced-motion: reduce) {
      .cursor { animation: none; opacity: 1; }
    }
  </style>
```

Replace it with (adds the print block before `</style>`):

```css
    @media (prefers-reduced-motion: reduce) {
      .cursor { animation: none; opacity: 1; }
    }

    /* ── Print / CV résumé ── */
    @media print {
      @page { size: A4; margin: 1.4cm 1.5cm; }

      html {
        -webkit-print-color-adjust: exact;
        print-color-adjust: exact;
      }

      html, body {
        background: #fff !important;
        color: #1a1a1a !important;
        font-size: 10.5pt;
        line-height: 1.4;
      }

      /* Hide interactive / decorative chrome */
      .sidebar,
      .lang-toggle,
      .cv-download,
      .cv-cta,
      .cursor,
      .hero-eyebrow,
      .footer,
      #contact { display: none !important; }

      .layout { display: block; max-width: none; margin: 0; }
      main { padding: 0; max-width: none; }

      /* Reveal print-only header */
      .cv-print-header { display: block; margin: 0.2rem 0 0.6rem; }
      .cv-print-role {
        font-family: 'JetBrains Mono', monospace;
        font-size: 9pt; color: #444; letter-spacing: 0.02em;
        margin-bottom: 0.15rem;
      }
      .cv-print-contact { font-size: 8.5pt; color: #333; }

      /* Hero → résumé header */
      .hero { margin-bottom: 1rem; }
      .hero-name { color: #111 !important; font-size: 22pt; line-height: 1.05; margin-bottom: 0.3rem; }
      .hero-name br { display: none; }
      .hero-tagline { color: #333 !important; font-size: 10pt; max-width: none; margin-bottom: 0.3rem; }
      .hero-location { color: #555 !important; font-size: 8.5pt; }

      /* Sections */
      section { margin-bottom: 0.9rem; }
      .section-label {
        color: #111 !important; font-size: 9pt; letter-spacing: 0.1em;
        margin-bottom: 0.5rem; break-after: avoid;
      }
      .section-label::after { background: #ccc !important; }

      /* Experience timeline — strip dots/rails, tighten */
      .timeline { gap: 0.7rem; }
      .tl-item { padding-left: 0; break-inside: avoid; }
      .tl-item::before, .tl-item::after { display: none !important; }
      .tl-company { color: #111 !important; font-size: 11pt; }
      .tl-company a { color: #111 !important; }
      .tl-badge { color: #444 !important; background: none !important; border-color: #999 !important; font-size: 7pt; }
      .tl-role { color: #333 !important; font-size: 9.5pt; }
      .tl-period { color: #666 !important; font-size: 8pt; }
      .tl-desc { color: #222 !important; font-size: 9.5pt; }
      .tl-bullets li { color: #333 !important; font-size: 9pt; }
      .tl-bullets li::before { color: #666 !important; }

      /* Education */
      .edu-grid { gap: 0.5rem; }
      .edu-item { break-inside: avoid; }
      .edu-school { color: #111 !important; }
      .edu-degree { color: #333 !important; }
      .edu-period { color: #666 !important; }

      /* Skills → inline wrapped list */
      .skills-grid { display: block; }
      .skill-tag {
        display: inline; background: none !important; border: none !important;
        padding: 0; color: #222 !important; font-size: 9pt;
      }
      .skill-tag:not(:last-child)::after { content: ' · '; color: #999; }

      /* Homelab — compact */
      .homelab-card { background: none !important; border: none !important; padding: 0; break-inside: avoid; }
      .hl-header { margin-bottom: 0.3rem; }
      .hl-title { color: #111 !important; }
      .hl-subtitle { color: #555 !important; }
      .hl-badge { color: #444 !important; background: none !important; border-color: #999 !important; }
      .hl-desc { color: #222 !important; font-size: 9pt; border-top-color: #ccc !important; padding-top: 0.4rem; margin-bottom: 0.4rem; }
      .hl-chip { background: none !important; border: none !important; padding: 0; color: #333 !important; font-size: 8.5pt; }
      .hl-chip:not(:last-child)::after { content: ' · '; color: #999; }

      /* Certifications — compact */
      .cert-list { gap: 0.3rem; }
      .cert-item { background: none !important; border: none !important; padding: 0.1rem 0; break-inside: avoid; }
      .cert-icon { display: none; }
      .cert-name { color: #111 !important; font-size: 9.5pt; }
      .cert-issuer { color: #555 !important; font-size: 8pt; }

      a { text-decoration: none !important; }
    }
  </style>
```

- [ ] **Step 5: Run the check to verify it passes**

Run: `cd /home/kiro/Progetti/knamroud.github.io && SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad && python3 "$SCRATCH/check2.py"`
Expected: prints `PASS`. Exit code 0.

- [ ] **Step 6: Commit**

```bash
cd /home/kiro/Progetti/knamroud.github.io
git add index.html
git commit -m "feat: add @media print résumé stylesheet + print-only header for real-time CV PDF

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: End-to-end verification (PDF render + manual acceptance)

**Files:**
- Test: `$SCRATCH/print_pdf_check.sh` (browser-conditional; skips cleanly with no browser).

**Interfaces:**
- Consumes: the finished `index.html` from Tasks 1–2. Produces: no code — a verified deliverable.

- [ ] **Step 1: Re-run both structural checks (regression)**

Run: `cd /home/kiro/Progetti/knamroud.github.io && SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad && python3 "$SCRATCH/check1.py" && python3 "$SCRATCH/check2.py"`
Expected: both print `PASS`, exit 0.

- [ ] **Step 2: Write the browser-conditional PDF render check**

Create `$SCRATCH/print_pdf_check.sh`:

```bash
#!/usr/bin/env bash
set -u
HTML="$(pwd)/index.html"
OUT="${SCRATCH:?set SCRATCH}/cv.pdf"
BROWSER=""
for b in chromium chromium-browser google-chrome google-chrome-stable brave; do
  command -v "$b" >/dev/null 2>&1 && { BROWSER="$b"; break; }
done
if [ -z "$BROWSER" ]; then
  echo "SKIP: no Chromium/Chrome on PATH — do the manual print check in Step 4 instead."
  exit 0
fi
rm -f "$OUT"
"$BROWSER" --headless=new --disable-gpu --no-pdf-header-footer \
  --virtual-time-budget=2500 --print-to-pdf="$OUT" "file://$HTML" 2>/dev/null
[ -f "$OUT" ] || { echo "FAIL: no PDF produced"; exit 1; }
echo -n "Rendered: "; pdfinfo "$OUT" | grep -E '^Pages:'
TXT="$(pdftotext "$OUT" - 2>/dev/null)"
fail=0
for needle in "Namroud" "Calitera" "Django"; do
  echo "$TXT" | grep -q "$needle" || { echo "FAIL: '$needle' missing (rasterized/empty?)"; fail=1; }
done
# Screen-only CV button label must NOT be in the PDF → proves @media print applied
echo "$TXT" | grep -q "Scarica CV" && { echo "FAIL: CV button leaked into print"; fail=1; }
# Print-only contact line SHOULD be present
echo "$TXT" | grep -q "github.com/knamroud" || { echo "FAIL: print contact line missing"; fail=1; }
[ "$fail" -eq 0 ] && echo "PASS: real PDF, selectable text, print CSS applied."
exit "$fail"
```

- [ ] **Step 3: Run the PDF render check**

Run: `cd /home/kiro/Progetti/knamroud.github.io && SCRATCH=/tmp/claude-1000/-home-kiro-Progetti-knamroud-github-io/5b2184dc-4658-4955-9dcb-daf89451a120/scratchpad && bash "$SCRATCH/print_pdf_check.sh"`
Expected (this shell, no browser): prints `SKIP: no Chromium/Chrome on PATH …`, exit 0.
Expected (machine with a browser): prints `Rendered: Pages: 2` (or similar) then `PASS: real PDF, selectable text, print CSS applied.`, exit 0.

- [ ] **Step 4: Manual acceptance (the layout gate — run in a real browser)**

Serve and open, then print-preview:

```bash
cd /home/kiro/Progetti/knamroud.github.io && python3 -m http.server 8765
# open http://localhost:8765/ in a browser
```

Confirm, in the browser's Print preview (Ctrl/Cmd+P → "Save as PDF"):
- [ ] With **IT** selected: résumé is light (white bg, dark text), ~1–2 pages, no sidebar / nav / language toggle / download button / blinking cursor; name, role, and contact line appear at the top; Experience/Education/Skills/Homelab/Certifications are present and tidy; no experience/education/cert item is split across a page break.
- [ ] Toggle **EN**, print again: same layout, all text now in English (proves language-awareness / real-time).
- [ ] Close the preview: the on-screen dark site is visually unchanged; the sidebar still shows Email/GitHub/LinkedIn/Credly; the CV section shows the CTA line + "Scarica CV / Download CV" button; the nav item reads "CV".
- [ ] Stop the server (Ctrl+C).

---

## Self-Review

**1. Spec coverage:**
- Contact de-duplication → Task 1 (removes `.contact-link` block, keeps sidebar). ✓
- Repurpose `#contact` as CV CTA + nav rename → Task 1. ✓
- `window.print()` button + i18n (`cv_download`, `cv_cta`) → Task 1. ✓
- `@media print` résumé, print-only contact header, hide chrome, page-break rules, font fallback (system stack inherited from `body`) → Task 2. ✓
- Real-time & language-aware (renders live DOM) → inherent; verified in Task 3 Step 4 (IT+EN). ✓
- Correct hyphenated Credly URL → Task 2 markup + check2. ✓
- No new content, no deps, screen unchanged → constraints + Task 3 Step 4 last check. ✓
- Objective PDF verification (headless Chromium + selectable text) → Task 3 Steps 2–3. ✓

**2. Placeholder scan:** No TBD/TODO; every code and command step is complete. ✓

**3. Type/selector consistency:** `.cv-download`, `.cv-cta`, `.cv-print-header`, `.cv-print-role`, `.cv-print-contact`, `#contact`, i18n keys `cv_download`/`cv_cta`/`nav_contact` are used identically across Tasks 1–3 and the checks. ✓
