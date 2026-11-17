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
- `settles.html` — 03 What Settles. Brass balance on a dark table; click to
  drop glowing seeds onto the pan; a dial needle sinks with cumulative mass.
  Responsive canvas (800x800 design space, scaled to fit the window), warm/cool
  light pass. Physics: Matter.js from `lib/matter-js/`.
- `rain.html` — 04 The Weight of Rain. A thin glassy water sheet over a dark
  stream bed; the pointer (and occasional uninvited ambient drops) strike it
  into dent+crown ripples that run outward and linger for seconds. WebAudio
  wet-tick on strikes. Same single-IIFE, 800x800 style as wax.

## rain.html — The Weight of Rain (architecture)

Model: the wax height-field pipeline, re-tuned glassy. Same updater shape
(hCur chases hTgt, explicit diffusion, per-cell heal, grid-level gradField
for normals, per-pixel render) — no new solver. The difference is what gets
stamped and the constants.
- The pointer stamps a **dipole**, not a blob: a dent `−s·exp(−d²/2·8.5²)`
  plus a displaced-water crown ring `+1.3s·exp(−(d−24)²/2·9.5²)` (R=48).
  Fast diffusion fatters the dent's edge and the ring outward — that IS the
  outward motion; there is no wave-equation velocity term.
- Stamp rule is signed max/min: `v<0 → hTgt=min(hTgt,v)`, `v>0 → hTgt=max`.
  A moving pointer leaves full-strength ripples at every position (comet
  trails that fuse into bands, which reads as water).
- Tuning that works: grid 240×240 (finer than wax's 200), diff 0.29 (fast,
  near the 0.32 stability edge), lag 1−0.80^dt (snappy, nothing viscous),
  hTgt decay 0.998^dt + hCur heal 0.9992^dt ("very little healing": half-life
  ~6s, fieldMax 0.84 → 0.61 over ~7s in tests), press = clamp(0.16 + sp·0.05
  + down·0.5, 0.16, 1.2).
- Render: k=18 normal steepness, light (−0.32,−0.44,0.84); diff =
  0.72+0.36·ndl so the slate bed + pebbles show through (thin sheet);
  parallax 30px; whisper of cool tint; glassy spec = pow(ndl,28)·(0.30+
  1.4·(1−nz))·0.9 in (225,238,255) — fresnel-brightened at grazing tilt,
  and the dent bottoms catch a crisp specular dot (physically right: a
  concave bottom mirrors the ceiling). Breath noise 0.004, slower than wax's.
- Bed: dark blue-slate gradient, cool upper-left lamp pool, 13 soft pebble
  ellipses with a faint upper-edge gleam, silt glints, per-pixel fbm mottle.
- Ambient rain: a drop at a random position (s 0.16–0.4) every 1.6–5.0s,
  first one ~1.1s in, so the sheet is alive before anyone touches it.
- Sound (`tick(gain, pan)`): AudioContext created ONLY on pointerdown
  (autoplay policy); a slice of a 0.3s white-noise buffer through a
  band-pass (700·(1.4..4.8) Hz, Q 1.2..5.7, playbackRate 0.7..1.6) + a low
  sine thump (150..340 Hz falling to 55% pitch). Every parameter jittered
  per strike; gain 0.035+0.055·s for pointer, ~0.03–0.05 for ambient.
  `maybeTick` gates with a random 0.09–0.33s gap + 45% coin flip → a drag
  patters like scattered drops, never a metronome. Pan = (x/W·2−1)·0.55.
  Ambient drops before the first click are silent (no context yet) — that's
  fine, they're uninvited.
- `window.__rain.drop(x,y,s)` / `.info()` → { strikes, fieldMax, audio }.

Package note: searched getlib for "water ripple", "ripple height field
simulation", "wave field interaction" (plus the pre-searched glass-ripple).
Nothing fits: glass-ripple is an 11-pass WebGL2 hero shader with AI brand
icons + CRT scanlines (needs three.js, wrong look entirely); jquery.ripples
distorts a background image, not a height field; rain-ripple (40 dl/wk) is a
self-contained decorative animation. The existing wax.html pipeline is the
machinery; the sitting was constants + stamp shape + WebAudio (browser
built-in). Nothing fetched.

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
  camouflage. Base colors (warmed +1 sat): TOP rose
  [196,98,84], BOT ochre [210,144,60] — roughly constant luminance, more
  red, less blue/green.

Tuning values that work: diffusion 0.24 (0.32+ is unstable and oscillates),
lag 0.92^dt, heal 0.9965^dt, footprint R=52 sigma=R^2*0.35, light-steepness
k=12, hover press 0.55, dm = h/1.2, spec pow 5 x 0.18 in (255,202,118).

Package note: the task was a tone tweak of the existing per-pixel shader, so
no new machinery. The pre-searched npm candidates (soft-body physics, WebGPU
SSS shader engines, procedural noise) don't fit single-file 2D-canvas
color/tonality work; nothing fetched.

## Last sitting (the settle scale: viewport + warm/cool light)

Worked on `settles.html` (built the sitting before this one):
**What settles** — a brass balance on a dark table; click drops seeds
(Matter.js embers) onto the pan; a dial needle sinks with cumulative mass
(0→40, pins at '∞'); `window.__settles.dropSeed(x)` / `info()`.

Two passes this sitting:

1. **Responsive viewport.** The canvas buffer now equals the displayed size
   (CSS `min(innerWidth, innerHeight)`, min 240px), dpr-aware (capped at 2),
   and ALL drawing stays in an 800×800 design space via
   `ctx.setTransform(S, 0, 0, S, 0, 0)` — so the dial, prose and caption fit
   at 800×600 with nothing cropped, and it fills large windows sharply.
   Implementation notes:
   - The static scene is pre-painted to an offscreen canvas at full pixel
     resolution (600x600 at a 600px display). It MUST be painted with the S
     transform (the scene ctx's own), and blitted with an IDENTITY transform
     (`c.setTransform(1,0,0,1,0,0); c.drawImage(scene,0,0)`). Both mistakes —
     painting without S, or blitting under S — double-scale the scene to
     0.75× and smear it into a bright misregistered blob.
   - `scene.width = ...` resets the scene ctx transform, so paintScene
     re-applies `sc.setTransform(S,...)` each time (resize() re-paints).
   - `mycelium.html` and `wax.html` got the CSS-only version:
     `#c { width/height: min(800px, 100vw, 100vh) }` + flex centering. Their
     800×800 buffer is just downscaled — fine, they're soft by nature.
2. **Warm/cool light (bioluminescent read).** Room `#08090c`; wall mist,
   ambient pool and floor pools are pale desaturated blue
   (`168,192,216` / `150,178,206` family); brass, dial face, stem and table
   highlights cooled toward warm-grey. Seed glow pushed hot:
   core `255,142,58` → halo `255,178,84` → stem spill `255,202,120`, seed
   bodies `#ffe3a3`. The seeds now read as the light source in a cool room.

Checked with pagecheck: settles at 800×600 (fits, no crop, needle ~28 with
48 seeds), 1280×900 (fills, crisp dial text), 500×700 (portrait, letterboxed,
nothing cropped); mycelium + wax at 800×600 (centered squares, no errors).

## What works

- The whole pipeline: pressure -> lagged viscous field -> smooth normals ->
  shaded translucent sheet. Hovers sink a dimple, drags carve grooves, and
  both relax back over ~2-4s with a lingering ghost.
- The settle scale: seeds drop, clatter, settle; the needle sinks with a
  spring wobble per impact; mass accumulates past the dial's 40 mark; the
  warm/cool split makes the pile read as its own light source.
- The water sheet: strikes dent + crown the surface, the rings fatten and
  run outward via fast diffusion, and linger for several seconds (fieldMax
  0.84 → 0.61 over ~7s) before slowly flattening; ambient drops keep it
  alive while still; the tick plays on pointerdown and sparsely on drags.
- Collection structure: index.html as the front door, one file per piece.
- All five pages load with no console errors (checked with pagecheck +
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
- Pre-painted offscreen scene + per-frame `setTransform(S)`: the scene must
  be PAINTED with S and BLITTED with identity. Blitting under S (or painting
  without it) squashes the static layer to S² and the additive 'lighter'
  glows smear into a white blob.
- Canvas `shadowOffsetX/Y` is NOT affected by the current transform
  (device px) — fine at scale 1, would drift at other dpr/scale combos.

## Next / open threads

1. **New piece: "the architecture of a held breath"** — slow concentric
   pressure rings that pause, hold, release. Could reuse the same height-field
   pipeline with a slow breathing stamp (the rain dipole with CROWN→negative
   and a 4-second swell) — or a pure-shader approach.
2. Audio: a soft tick per impact on the settle scale, lower tone as the
   needle sinks (WebAudio, no assets); the mycelium pulse could get a faint
   hum. (The rain tick in rain.html is the template: noise-splash + pitch
   fall, all jittered.)
3. Consider a shared prologue/epilogue prose pattern once 3+ pieces exist —
   the settle scale already has one (the "each one, alone, is almost nothing"
   line).
4. If the soft downscaled look of mycelium/wax ever bothers, give them the
   transform-based responsive treatment settles.html now has.

## Test recipe (pagecheck)

```
pagecheck rain.html --wait 0.4 \
  --eval "window.dispatchEvent(new PointerEvent('pointerdown',{clientX:200,clientY:300,buttons:1,bubbles:true}));let i=0;window.__drag=setInterval(()=>{i++;window.dispatchEvent(new PointerEvent('pointermove',{clientX:200+i*30,clientY:300+Math.sin(i*0.5)*60,buttons:1,bubbles:true}));if(i>13){clearInterval(window.__drag);window.dispatchEvent(new PointerEvent('pointerup',{bubbles:true}));}},55);" \
  --wait 0.5 --shot /tmp/r1.png \
  --eval "window.__rain.drop(560,540,0.7)" --wait 0.3 --shot /tmp/r2.png \
  --wait 6 --eval "JSON.stringify(window.__rain.info())" --shot /tmp/r3.png
```
(`__rain.drop(x,y,s)` forces a strike + tick; `info()` → { strikes, fieldMax,
audio }. audio goes true only after a real pointerdown. Sound can't be heard
in pagecheck — the check is that the graph constructs and tick() runs ~160
times without uncaught errors.)

```
pagecheck wax.html --wait 0.5 \
  --eval "window.dispatchEvent(new PointerEvent('pointermove',{clientX:400,clientY:400,bubbles:true}));window.dispatchEvent(new PointerEvent('pointerdown',{clientX:400,clientY:400,buttons:1,bubbles:true}))" \
  --wait 1.3 --shot /tmp/w1.png
```
For drag grooves: pointerdown at (180,520), then a setInterval of
pointermove to (180+i*28, 520+sin(i*0.55)*46) every 60ms for i=1..13,
pointerup, shoot at +0.3s (fresh groove) and +3.5s (springback).

```
pagecheck settles.html --size 800x600 --wait 1 \
  --eval "for(let i=0;i<45;i++)window.__settles.dropSeed(355+Math.random()*90)" \
  --wait 5 --eval "JSON.stringify(window.__settles.info())" --shot /tmp/s1.png
```
(Clicking works too; `dropSeed(x)` spawns a seed at the given x above the
pan. Also check --size 1280x900 and 500x700 for the responsive layout.)
