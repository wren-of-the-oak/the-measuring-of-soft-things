# The Measuring of Soft Things

A collection of small interactive sketches and prose that attempt to render
the intangible: the weight of rain, the glow of fungi, the architecture of a
held breath. Built a sitting at a time. Each piece is one sitting, kept small
and quiet on purpose.

## Structure (how it's put together)

Plain HTML/CSS/JS, one file per piece, no build step, no CDN, runs offline
and on GitHub Pages. No shared JS yet — the pieces are siblings, not a
system; keep them siblings.

- `index.html` — the collection index: quiet dark page, one numbered line per
  piece with a short italic description, links to each piece. **This is the
  front door; new pieces get a line here.**
- `mycelium.html` — 01 Mycelium Map. 800x800 canvas, single IIFE. Seeded value
  noise/fbm, 140x140 coarse lattice of mycelial roots wandering upward,
  glowing tips, a "memory" of passed points rendered in two passes
  (soft shadow, then core). Space to pulse, drag to grow a new root.
- `wax.html` — 02 The Yield. 800x800 canvas, single IIFE. A semi-translucent
  ochre/rose wax sheet you press with the pointer.

## wax.html — The Yield (architecture)

Model: a coarse height field, not particles.
- 200x200 grid (4px cells), `hCur` (wax depth, 0=unpressed) chases `hTgt`
  (pressure target) with viscosity, plus explicit diffusion and slow healing.
- `hTgt` is a **max-stamped gaussian footprint** of the current smoothed
  pointer pressure; it decays slowly (0.99^dt) after the hand passes, so
  drags leave grooves that linger. Pressure = 0.55 hover base + speed*0.13 +
  0.9 while mouse-down, clamped 0.55–2.4, smoothed over time (0.82^dt).
- `gradField()` precomputes smooth grid-space gradients (gxArr/gyArr) each
  frame — do NOT take per-pixel finite differences of the coarse field, it
  aliases into feathery spikes.
- Render is per-pixel into ImageData (800x800 ~2-3ms). Shading: normal from
  (gx,gy) tilted toward upper-left light, diffuse ±, plus a separate soft
  warm rim sheen: spec = pow(light-dot, 5) * 0.18, color (255,202,118) —
  soft and amber, not a hard white spike. Parallax offset of the base
  texture by 24px*nx,ny (the dent reveals the ground below), thin-wax depth
  darkening toward umber, faint amber glow at deep press.
- `groundTex` (captured once after painting) is the dark umber ground: base
  fill + three soft radial "lamp" pools + fine grain + a few hairline
  cracks. Composed over the ground at alpha ~0.91, then vignette.
- Pre-rendered 800x800 rose->ochre texture: 3 fbm mottle scales (140/230/26px)
  + per-pixel grain * 0.97 * a diagonal lamp factor (upper-left +, lower-right
  -). Keep the mottle amplitudes low; the breathing shimmer is a slow 40px
  valNoise added to the normals (coeff ~0.006) — too high and it reads as
  camouflage. Base colors (this sitting, warmed +1 sat): TOP rose
  [196,98,84], BOT ochre [210,144,60] — roughly constant luminance, more
  red, less blue/green.

Tuning values that work: diffusion 0.24 (0.32+ is unstable and oscillates),
lag 0.92^dt, heal 0.9965^dt, footprint R=52 sigma=R^2*0.35, light-steepness
k=12, hover press 0.55, dm = h/1.2, spec pow 5 x 0.18 in (255,202,118).

Package note (this sitting): the task was a tone tweak of the existing
per-pixel shader, so no new machinery. The pre-searched npm candidates
(soft-body physics, WebGPU SSS shader engines, procedural noise) don't fit
single-file 2D-canvas color/tonality work; nothing fetched.

## What works

- The whole pipeline: pressure -> lagged viscous field -> smooth normals ->
  shaded translucent sheet. Hovers sink a dimple, drags carve grooves, and
  both relax back over ~2-4s with a lingering ghost.
- Collection structure: index.html as the front door, one file per piece.
- All three pages load with no console errors (checked with pagecheck +
  screenshots; caption text verified present via pixel readback).

## Pitfalls hit (don't repeat)

- Per-pixel `fieldAt(h, x±2)` finite differences on a 4px grid = aliasing
  spikes. Precompute the gradient at grid level, bilinear-sample it.
- Accumulative stamping (hTgt += strength per frame) makes moving pointers
  leave only faint trails, because each cell is visited for ~1 frame. Max
  stamping gives full-strength grooves that linger.
- Diffusion > 0.3 on this grid runs an unstable checkerboard mode: it looks
  like the field is "breathing" and never settles.
- A second `putImageData` on the ground canvas clobbers the ground paint —
  capture it into a canvas element instead of redrawing.

## What's next (concretely)

1. DONE (this sitting): dimple rim now has a soft warm sheen (spec pow 5 x
   0.18, amber 255,202,118) and the wax base colors are slightly warmer / more
   saturated (TOP [196,98,84], BOT [210,144,60]). Verified with pixel stats:
   rim avg R-B went +94 -> +115, flat wax +87 -> +109, luminance roughly
   held.
2. A third piece when the mood is right — candidates from the opening brief:
   the weight of rain (drops that land and spread on a water sheet, could
   reuse wax.html's height-field pipeline with different material constants),
   the architecture of a held breath (slow concentric pressure rings that
   pause, hold, release).
3. When there are 3+ pieces, consider a small shared prologue/epilogue
   pattern per piece (a single italic line of prose that appears after
   interaction starts) — decide then, don't build it now.

## Test recipe (pagecheck)

```
pagecheck wax.html --wait 0.5 \
  --eval "window.dispatchEvent(new PointerEvent('pointermove',{clientX:400,clientY:400,bubbles:true}));window.dispatchEvent(new PointerEvent('pointerdown',{clientX:400,clientY:400,buttons:1,bubbles:true}))" \
  --wait 1.3 --shot /tmp/w1.png
```
For drag grooves: pointerdown at (180,520), then a setInterval of
pointermove to (180+i*28, 520+sin(i*0.55)*46) every 60ms for i=1..13,
pointerup, shoot at +0.3s (fresh groove) and +3.5s (springback).
