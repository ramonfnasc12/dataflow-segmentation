# Skill: Create a Presentation

Use this skill when the user asks you to create a presentation, slide deck, or keynote-style page for this repository.

## Overview

Presentations in this repo are self-contained HTML files using inline SVG slides. The reference implementation is `presentations/frankfurt-presentation.html`. Match its structure, style, and interaction patterns exactly.

---

## File Structure

```
presentations/<topic-slug>.html   ← create here
```

After creating the file, add a card to `index.html` in the `presentations/` folder section and update `folder-count`.

---

## HTML Shell

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[Presentation Title]</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body {
      width: 100%; height: 100%;
      background: #00111A;
      overflow: hidden;
      display: flex; align-items: center; justify-content: center;
    }
    #deck {
      position: relative;
      width: min(100vw, 177.778vh);   /* 16:9 */
      height: min(56.25vw, 100vh);
      overflow: hidden;
    }
    .slide {
      position: absolute; inset: 0;
      opacity: 0; pointer-events: none;
      transform: translateY(32px);
      transition: opacity .6s cubic-bezier(.4,0,.2,1), transform .6s cubic-bezier(.4,0,.2,1);
    }
    .slide svg { width: 100%; height: 100%; display: block; }
    .slide.active { opacity: 1; pointer-events: all; transform: translateY(0); }
    .slide.prev   { opacity: 0; transform: translateY(-32px); transition: opacity .38s ease, transform .38s ease; }
    #ctrl {
      position: fixed; bottom: 14px; right: 20px;
      display: flex; align-items: center; gap: 10px;
      background: rgba(0,17,26,.92); border: 1px solid rgba(255,255,255,.07);
      border-radius: 28px; padding: 7px 16px; backdrop-filter: blur(20px); z-index: 999;
      font-family: 'Inter', system-ui, sans-serif;
    }
    .cb {
      width: 32px; height: 32px; border-radius: 50%;
      background: rgba(0,237,100,.07); border: 1px solid rgba(0,237,100,.22);
      color: #00ED64; cursor: pointer; font-size: 14px;
      display: flex; align-items: center; justify-content: center;
      transition: background .2s; user-select: none;
    }
    .cb:hover { background: rgba(0,237,100,.17); }
    #ctr { font-size: 11px; color: #889397; letter-spacing: 1.2px; width: 56px; text-align: center; white-space: nowrap; }
  </style>
</head>
<body>
<div id="deck">

  <!-- slides go here -->

</div>

<!-- Navigation controls -->
<div id="ctrl">
  <div class="cb" id="prev">&#8592;</div>
  <span id="ctr">1 / N</span>
  <div class="cb" id="next">&#8594;</div>
</div>

<script>
  const slides = document.querySelectorAll('.slide');
  let cur = 0;
  function go(n) {
    slides[cur].classList.remove('active');
    slides[cur].classList.add('prev');
    setTimeout(() => slides[cur].classList.remove('prev'), 400);
    cur = (n + slides.length) % slides.length;
    slides[cur].classList.add('active');
    document.getElementById('ctr').textContent = (cur + 1) + ' / ' + slides.length;
  }
  document.getElementById('next').addEventListener('click', () => go(cur + 1));
  document.getElementById('prev').addEventListener('click', () => go(cur - 1));
  document.addEventListener('keydown', e => {
    if (e.key === 'ArrowRight' || e.key === ' ') go(cur + 1);
    if (e.key === 'ArrowLeft')                   go(cur - 1);
  });
</script>
</body>
</html>
```

---

## Slide Template

Every slide is a `<div class="slide">` wrapping a **1400×788** SVG. Each SVG has its own `<defs>` with IDs prefixed by the slide number (e.g. `s1_bg`, `s2_glow`) to avoid collisions.

```html
<!-- ===================== SLIDE N: TITLE ===================== -->
<div class="slide" id="sN">
<svg viewBox="0 0 1400 788" xmlns="http://www.w3.org/2000/svg"
     font-family="'Inter','SF Pro Display',system-ui,sans-serif">
<defs>
  <linearGradient id="sN_bg" x1="0" y1="0" x2="1" y2="1">
    <stop offset="0%"   stop-color="#001E2B"/>
    <stop offset="100%" stop-color="#00111A"/>
  </linearGradient>
  <radialGradient id="sN_vignette" cx="50%" cy="50%" r="72%">
    <stop offset="60%"  stop-color="transparent"/>
    <stop offset="100%" stop-color="rgba(0,8,14,.7)"/>
  </radialGradient>
  <filter id="sN_glow">
    <feGaussianBlur stdDeviation="6" result="b"/>
    <feMerge><feMergeNode in="b"/><feMergeNode in="SourceGraphic"/></feMerge>
  </filter>
  <filter id="sN_blur"><feGaussianBlur stdDeviation="28"/></filter>
  <pattern id="sN_grid" x="0" y="0" width="32" height="32" patternUnits="userSpaceOnUse">
    <circle cx="16" cy="16" r=".85" fill="#4A9B8E" opacity=".055"/>
  </pattern>
</defs>

<!-- Background layers (always include all three) -->
<rect width="1400" height="788" fill="url(#sN_bg)"/>
<rect width="1400" height="788" fill="url(#sN_grid)"/>
<rect width="1400" height="788" fill="url(#sN_vignette)"/>

<!-- Ambient glow blobs (1–2 per slide, very low opacity) -->
<ellipse cx="1050" cy="394" rx="450" ry="420" fill="#00ED64" opacity=".03" filter="url(#sN_blur)"/>

<!-- Content here -->

</svg>
</div>
```

The first slide must have `class="slide active"`. All others just `class="slide"`.

---

## Slide Types & Patterns

### Title Slide
- Large centred title (font-size 72–90, font-weight 800, fill `#E3FCF7`, letter-spacing -2)
- Subtitle in MongoDB green (`#00ED64`), font-size 24–28, lighter weight
- Event/date line in muted colour (`#889397`, font-size 14, letter-spacing 4)
- Pulsing green horizontal rule or decorative line
- Animated glowing dot grid particle orbiting the slide

### Agenda / Contents Slide
- "What we'll cover" heading (font-size 44, fill `#E3FCF7`)
- Numbered item cards using `<g transform="translate(x, y)">`:
  - `<rect>` with accent colour fill at 6% opacity, matching stroke at 20% opacity
  - Left accent bar: `<rect x="0" y="5" width="4" height="56" rx="2" fill="[colour]" opacity=".7"/>`
  - Number label (font-size 10, letter-spacing 2, font-weight 700, opacity .65)
  - Title (font-size 17, font-weight 600, fill `#E3FCF7`)
  - Subtitle (font-size 12, fill `#889397`)
  - Staggered entrance: `<animate attributeName="opacity" values="0;1" dur=".5s" begin="Xs" fill="freeze"/>`

### Section Break Slide
- Full-bleed accent ellipse glow
- Section number badge (`SECTION 0N`, font-size 11, letter-spacing 3.5, font-weight 600)
- Section title split across two lines — first line white, second line in accent colour with `filter="url(#sN_glow)"`
- Left accent bar (full height, pulsing opacity animation)
- Horizontal divider rule below title

### Content Slide (cards / pillars)
- Section label top-left (small caps, muted colour, letter-spacing 3)
- Page title (font-size 36–46, fill `#E3FCF7`)
- 2–4 content cards in a row using `<g transform="translate(x, y)">`:
  - Rounded rect with accent fill at 6% and matching stroke
  - Emoji or icon (font-size 26–32)
  - Label (font-size 12–13, font-weight 700, letter-spacing .5)
  - Divider line
  - Body text lines (font-size 12–13, fill `#E3FCF7` for primary, `#889397` for secondary)
  - Pulsing stroke-opacity animation on hover cards: `<animate attributeName="stroke-opacity" values=".3;.7;.3" dur="4s" repeatCount="indefinite"/>`

### Closing / Q&A Slide
- Pulsing concentric rings centred on slide
- Large "Questions?" or closing statement (font-size 70–80, font-weight 800, glow filter)
- Contact card with border `#4FC3F7`, photo circle, name, title, email, LinkedIn

---

## Colour Palette

| Use | Value |
|---|---|
| Background dark | `#00111A` |
| Background mid | `#001E2B` |
| Body text | `#E3FCF7` |
| Muted / meta | `#889397` |
| MongoDB green (primary) | `#00ED64` |
| Blue (secondary) | `#4FC3F7` |
| Teal (tertiary) | `#26C6A6` |
| Amber (warning/highlight) | `#FFB300` |
| Dot grid fill | `#4A9B8E` at opacity `.055` |

Keep accent colours consistent per topic: use one accent per section.

---

## Typography Rules

- Font stack: `'Inter','SF Pro Display',system-ui,sans-serif`
- Heading sizes: 80 (hero), 46–52 (section title), 36–40 (slide title), 18–22 (card title)
- Body: 12–14
- Labels/eyebrows: 10–11, letter-spacing 2–4, font-weight 600–700, ALL CAPS
- Never use `<foreignObject>` in presentation SVGs — all text must be SVG `<text>` elements

---

## Animation Guidelines

- Use SVG `<animate>` and `<animateMotion>` — no CSS animations in presentations
- Entrance animations: `values="0;1"`, short `dur` (.4s–.8s), `fill="freeze"`, staggered `begin`
- Ambient loops: pulsing opacity (`values=".3;.7;.3"`), slow `dur` (4s–10s), `repeatCount="indefinite"`
- Particle orbits: `<animateMotion path="M …" dur="12s" repeatCount="indefinite"/>` on a small circle
- Scan lines: animate `y1`/`y2` vertically across a region
- Keep animations subtle — they should add life, not distract

---

## Recommended Slide Order

1. **Title** — topic, event name, speaker, date
2. **Agenda** — 4–6 numbered items
3. **Section 01** — first topic introduction (section break style)
4. **Content slides** for section 01 (1–3 slides)
5. **Section 02** — next topic
6. **Content slides** for section 02
7. … repeat for each section …
8. **Key Takeaways** — 3–5 bullet points or cards
9. **Q&A / Closing** — questions + contact card

Aim for 10–14 slides total. Each slide should convey one clear idea.

---

## index.html Card

After creating the file, add to the `presentations/` section:

```html
<a class="card" href="presentations/[filename].html">
  <div class="card-tag"><span class="dot"></span>Presentation</div>
  <div class="card-title">[Title]</div>
  <div class="card-desc">
    [One sentence describing the topic and audience.]
  </div>
  <div class="card-arrow">
    Open presentation
    <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
      <path d="M2 7H12M8 3L12 7L8 11" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </div>
</a>
```

Update `folder-count` on the presentations section.

---

## PDF Export (Print Support)

Every presentation must include `@media print` styles so the user can export to PDF via the browser's print dialog (Ctrl+P / Cmd+P). Add this block at the end of the `<style>` section, after the `#ctr` rule:

```css
@media print {
  @page {
    size: landscape;
    margin: 0;
  }
  html, body {
    width: 100%; height: auto;
    overflow: visible;
    background: #00111A;
    display: block;
  }
  #deck {
    width: 100%; height: auto;
    overflow: visible;
    display: block;
  }
  #ctrl { display: none !important; }
  .slide {
    position: relative;
    opacity: 1 !important;
    pointer-events: all;
    transform: none !important;
    transition: none;
    display: block;
    width: 100%;
    height: 100vh;
    page-break-after: always;
    break-after: page;
  }
  .slide:last-child {
    page-break-after: avoid;
    break-after: avoid;
  }
  .slide svg {
    width: 100%;
    height: 100%;
  }
  .slide svg * {
    animation: none !important;
    transition: none !important;
  }
}
```

**What this does:**
- Sets landscape orientation with no margins (full-bleed slides)
- Shows all slides stacked instead of only the active one
- Inserts a page break after each slide so each becomes its own PDF page
- Hides the navigation controls
- Disables all SVG animations for clean static output

---

## Common Pitfalls

Real issues encountered and fixed — generalised as rules.

### Badge / pill sizing
Always calculate rect width before writing it. For a pill containing uppercase letter-spaced text:

```
available_width = rect_width - (text_x - rect_x) - right_padding (≥ 12)
required_width  = char_count × (font_size × 0.65 + letter_spacing)
```

Use `font-size="10"` with `letter-spacing="2"` as the default for uppercase badge labels. At those values, each character is ~8.5px wide, so a 30-character string needs ~255px of text space. **Never guess; always compute.** A badge rect that is too narrow will silently clip text.

### Inline text when mixing `<text>` with `<animate>` children
Never put the text content on an indented line inside a `<text>` element that also contains an `<animate>` child. SVG's `xml:space="default"` does not reliably strip the leading whitespace text node, causing the rendered text to start with a visible gap or sit at the wrong x position.

**Wrong:**
```svg
<text x="52" y="57" font-size="10" fill="#00ED64">
  SOME LABEL
  <animate attributeName="opacity" values="0;1" fill="freeze"/>
</text>
```

**Correct:**
```svg
<text x="52" y="57" font-size="10" fill="#00ED64">SOME LABEL<animate attributeName="opacity" values="0;1" fill="freeze"/></text>
```

### Connecting arrows between cards
Arrows that connect horizontally laid-out cards must be:
1. **Horizontally centered** in the gap: `cx = (left_card_right_edge + right_card_left_edge) / 2`
2. **Vertically aligned** to any connecting line (not the card midpoint unless no line exists)

Never hardcode arrow positions without deriving them from the card coordinates.

### Uniform card heights in list panels
When stacking N cards in a panel (e.g. a right-side list), all cards must be the same height. Calculate before writing:

```
available_height = 788 - header_bottom_y - bottom_margin (≥ 28)
card_height      = floor((available_height - (N - 1) × gap) / N)
```

Use `gap = 14` as the default. Internal content positions are then relative offsets from each card's `y`, keeping every card identical in structure.
