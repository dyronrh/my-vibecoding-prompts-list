# Mouse-Reveal Scanner Effect

A cursor-following spotlight reveals a hidden overlay image on top of a background. Moving the mouse opens a circular "scanner" window; leaving the area closes it. The overlay is tinted and has a soft bloom glow.

**Stack:** HTML5 Canvas 2D · GSAP 3

## How it works

Two images are needed:
- **Background** — full-viewport photo via CSS `background-size: cover`
- **Overlay** — line art or illustration PNG (white or transparent bg), e.g. `public/images/overlay.png`

## Build the component

Create a `'use client'` React component that mounts a `<canvas>` absolutely positioned over the container (`pointer-events: none; z-index: 16`). The component reads `canvas.parentElement` to size itself and attach listeners — no props needed.

### On load / resize: preprocess two offscreen canvases

**Cover-scale formula** (mirrors CSS `background-size: cover`):
```js
const scale   = Math.max(canvasW / img.naturalWidth, canvasH / img.naturalHeight)
const offsetX = (canvasW - img.naturalWidth  * scale) / 2
const offsetY = (canvasH - img.naturalHeight * scale) / 2
ctx.drawImage(img, offsetX, offsetY, img.naturalWidth * scale, img.naturalHeight * scale)
```

**Sharp layer** — draw overlay with cover-scale, then per-pixel:
```
lum      = 0.299*R + 0.587*G + 0.114*B
darkness = 255 - lum
if darkness < 20 → alpha = 0          (remove near-white bg)
else             → tint pixel + set A = min(255, darkness * 2.8)
```

**Glow layer** — draw the sharp canvas into a second offscreen canvas with `ctx.filter = 'blur(8px)'`.

### rAF render loop

```js
ctx.clearRect(0, 0, w, h)
if (r > 0.5) {
  ctx.globalAlpha = 0.55; ctx.drawImage(glowCanvas, 0, 0)   // bloom
  ctx.globalAlpha = 1.0;  ctx.drawImage(sharpCanvas, 0, 0)  // sharp

  // circular mask
  const grad = ctx.createRadialGradient(x, y, 0, x, y, r)
  grad.addColorStop(0,   'rgba(0,0,0,1)')
  grad.addColorStop(0.7, 'rgba(0,0,0,0.95)')
  grad.addColorStop(1,   'rgba(0,0,0,0)')
  ctx.globalCompositeOperation = 'destination-in'
  ctx.fillStyle = grad; ctx.fillRect(0, 0, w, h)
  ctx.globalCompositeOperation = 'source-over'
}
```

### GSAP mouse interaction

```js
// const spot = useRef({ x: 0, y: 0, r: 0 })

onMouseMove  → spot.x = mx; spot.y = my
               gsap.killTweensOf(spot)
               gsap.to(spot, { r: SPOT_R, duration: 0.38, ease: 'power2.out' })

onMouseLeave → gsap.killTweensOf(spot)
               gsap.to(spot, { r: 0, duration: 0.55, ease: 'power2.in' })
```

The rAF loop reads `spot.current` directly — no React re-renders.

### Usage

The parent container must have `position: relative; overflow: hidden`. Drop the component in as a direct child.

---

## Tunable constants

| Constant | Default | Effect |
|---|---|---|
| `SPOT_R` | `160` | Spotlight radius (px) |
| Bloom opacity | `0.55` | Glow layer strength |
| Blur radius | `8px` | Bloom softness |
| `darkness` threshold | `20` | Near-white cut-off |
| `darkness * 2.8` | multiplier | Line density / opacity |
| Gradient stops | `0.7 / 1.0` | Spotlight edge hardness |

## Tint presets (per-pixel color shift)

| Tint | R | G | B |
|---|---|---|---|
| **Gold** | `R*1.25 + 35` | `G*1.05 + 5` | `B*0.25` |
| **Cyan** | `R*0.3` | `G*1.1 + 20` | `B*1.3 + 40` |
| **White** | `255` | `255` | `255` |
| **Red** | `R*1.4 + 40` | `G*0.3` | `B*0.2` |
