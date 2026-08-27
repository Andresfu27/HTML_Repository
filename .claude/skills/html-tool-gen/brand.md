# Hinicio Brand Defaults

Every HTML tool uses these brand defaults unless the user explicitly asks for something different for that specific tool.

## Colors — define as CSS custom properties on `:root` in the `<style>` block
```css
:root {
  --hinicio-navy: #175196;        /* primary brand color — headers, primary actions */
  --hinicio-navy-dark: #18427B;   /* darker accent — hover states, emphasis */
  --hinicio-blue: #115293;        /* secondary blue */
  --hinicio-cyan: #00ADEE;        /* signature accent — the "H" color, use for highlights/active states */
  --hinicio-cyan-bright: #4BD0FF; /* lightest accent — subtle backgrounds, chart fills */
  --hinicio-gray: #595959;        /* secondary text, muted labels */
  --hinicio-gray-light: #7F7F7F;  /* tertiary text, disabled states */
  --hinicio-text: #404040;        /* primary body text */
  --hinicio-bg: #FFFFFF;          /* base background */
}
```
Use `--hinicio-navy` as the dominant color (headers, primary buttons, key lines/borders) and `--hinicio-cyan` as the accent (active states, highlights, key data series in charts). Don't use more than these two as the dominant pair unless the tool needs a wider palette (e.g. multi-series charts), in which case fall back to the remaining variables in order before introducing any new color.

## Fonts — replace the Google Fonts `<link>` in the skeleton
```html
<link href="https://fonts.googleapis.com/css2?family=Jost:wght@400;500;600;700&family=Mulish:wght@400;600;700&display=swap" rel="stylesheet">
```
- **Jost** — headings, titles, nav labels, buttons (geometric sans, closest free match to the Hinicio logo's Century Gothic-style lettering)
- **Mulish** — body text, form labels, table data, chart tick labels (same geometric family, better legibility at small sizes)

## Logo — embed as a base64 constant, never hotlink
Define once near the top of the `<style>` block or as a JS constant, and reference it in an `<img>` tag or CSS `background-image` in the header/toolbar area:
```html
<img id="hinicioLogo" alt="Hinicio" />
```
```js
const HINICIO_LOGO = 'data:image/png;base64,' + '<PASTE_RAW_CONTENTS_OF_assets/hinicio_logo_base64.txt_HERE>';
document.getElementById('hinicioLogo').src = HINICIO_LOGO;
```
**Critical — two separate pieces, both required:**
1. The prefix `data:image/png;base64,` (literal string, must be typed exactly, not stored in the asset file)
2. The raw base64 payload, read in full from `assets/hinicio_logo_base64.txt` (320×226px, optimized PNG, ~17.8KB)

Concatenate them into ONE string with no space or line break between the comma and the payload. Before returning any tool, verify `HINICIO_LOGO` starts with exactly `data:image/png;base64,` followed immediately by base64 characters — if the prefix is missing, the image renders as a broken icon with only the alt text visible, which is a silent failure.

Place the logo top-left of the tool by default, sized to roughly 120–160px wide. This keeps the file 100% self-contained — no external image request, fully consistent with the "no external resources beyond Google Fonts" rule.

Never truncate, paraphrase, or partially quote the base64 payload — read the asset file and use its full contents verbatim. Only load `assets/hinicio_logo_base64.txt` when the tool actually needs the logo embedded — it's a ~17.8KB read, don't pull it in for tools that don't display it.
