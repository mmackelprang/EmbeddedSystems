# Bench Labs — brand guide

**Proposed display name:** Bench Labs
**Tagline:** *Hands-on labs for real silicon.*

## Why this name

The repo is lab courseware — student sheets plus teacher's guides — for real vintage parts (an
ATF16V8 GAL, an Intel 8032, a Cypress PSoC). **Bench Labs** says where the learning happens: at
the bench, chips in hand, not in a simulator.

**Alternates considered:** *Chip School* (friendly, younger than the material), *Solder School*
(these labs are as much logic as solder), *The 8032 Papers* (too narrow).

## The mark

A breadboard with a jumper wire mid-flight between two tie points. The breadboard is the one
object every lab in the series shares, whatever the chip du jour.

## Palette

| Color | Hex | Role |
|---|---|---|
| Lab Navy | `#26547C` | Background / primary brand color |
| Board Cream | `#EFEBE0` | Breadboard, cards, text on dark |
| Jumper Amber | `#FFD166` | Wire, highlights, callouts |

## Voice

Teaching voice: numbered steps, "you should now see…", explicit checkpoints. Student sheets and
teacher's guides could carry the mark with a small `STUDENT` / `TEACHER'S GUIDE` badge in
Jumper Amber.

## Files in this directory

| File | Use |
|---|---|
| `logo.svg` | Full lockup (mark + wordmark + tagline) for README headers and docs |
| `favicon.svg` | Square app mark, scales from 16px to full size |
| `favicon.ico` | Legacy multi-size favicon (16/32/48) for browsers that want `.ico` |
| `favicon-32.png` | 32px PNG favicon |
| `apple-touch-icon.png` | 180px iOS home-screen icon |
| `icon-512.png` | Large raster for app manifests, social cards, stores |

### Wiring the favicon into a web page

```html
<link rel="icon" href="/branding/favicon.svg" type="image/svg+xml">
<link rel="icon" href="/branding/favicon.ico" sizes="16x16 32x32 48x48">
<link rel="apple-touch-icon" href="/branding/apple-touch-icon.png">
```

### README header

```markdown
<p align="center"><img src="branding/logo.svg" alt="Bench Labs" width="520"></p>
```

## Typography

Wordmark: **Montserrat Bold** (falls back to Segoe UI / system sans). Body text: the platform
default sans. For code-adjacent surfaces, any monospace at hand — the brand doesn't pin one.

The logo's wordmark is live SVG text, so it renders with whatever sans is installed; if you want
it pixel-identical everywhere, convert the text to outlines in any SVG editor and re-save.

## Dark and light backgrounds

The tile carries its own background, so both `logo.svg` and `favicon.svg` work unchanged on
light or dark pages. The wordmark in `logo.svg` is dark ink — on a dark page, either rely on the
tile alone (use `favicon.svg`) or restyle the two `<text>` fills to `#F0F2F5`.

---
*Generated as a proposal — names, colors, and marks are suggestions to accept, tweak, or reject.*
