# Value-mapped isosurfaces (ESP-type): a different animal

The rest of this skill draws **signed fields** as one or two single-colour lobes where
the colour is an *identity* (this lobe = depletion, that = phase B). A second family —
ESP/MEP painted on a molecular surface, sign(λ₂)ρ colouring on an IGM surface, any
"continuous scalar rendered as a colour map on one closed isosurface" — has **one
surface whose colour is the message**. Nearly every material and compositing rule of
the lobe pipeline inverts for it. This page is the measured counter-scheme, from a
Blender replication of Materials-Studio "ESP_iso" figures (ESP on the Connolly
molecular surface of a pentanuclear metal cluster, rendered as a three-system series).

Read this page **instead of** `lobe-material.md` when the colour carries a number.

---

## 1. The colormap is a scientific object, not a style

* **Diverging map, white anchored at value 0.** White means "uncharged / zero"; that is
  a physical claim. On an *asymmetric* window the white stop therefore does **not** sit
  at the midpoint: for −40..+80 kcal/mol it sits at 1/3 of the ramp. Anchoring white at
  the midpoint silently mislabels every colour.
* **Pure saturated endpoints** (pure blue / pure red were the author's pinned values,
  sRGB 0,0,255 / 255,0,0), interpolated **piecewise-linear in sRGB**, clamped outside
  the window.
* **Derive the window from the physics or the data**, then round for reading: the
  source tool's display window was −0.05..+0.10 Ha/e = −31.4..+62.8 kcal/mol, rendered
  as −40..+80; a later edition asked for −30..+60. Near-nucleus vertices of a metal
  cluster reach +10³–10⁴ kcal/mol — **clamping is expected, not a bug**; sanity-check
  the window against the surface-value quantiles and record the extremes in the build
  record.
* **Smooth the scalar, never the colours.** Grid-sampled values carry voxel noise that
  reads as speckle after mapping; neighbour-smooth the vertex values (3 iterations,
  λ = 0.5, edge-neighbour averaging) *before* mapping. The geometry itself is untouched
  (house rule 1 still holds).
* **Per-vertex colours, linear space.** Map the smoothed values per vertex, convert
  sRGB → linear, and write a `FLOAT_COLOR`/`POINT` colour attribute (this project:
  `esp_color`); the material is Principled with `Base Color ← Attribute` and the
  rasteriser interpolates. With the Standard view transform the authored sRGB values
  come back out of the PNG bit-faithfully.
* **One source of truth + a colourbar from the same function.** The map function is
  written once and consumed by both the mesh builder and a standalone colourbar PNG.
  A colourbar drawn by any other code *will* drift from the surface.
* **Rule:** window, endpoints and the white anchor go into the caption and the build
  JSON; "state the map range and units in the footer" (`surface-types.md` §5) is
  enforced by putting them in the build record first.

---

## 2. Clarity: uniform alpha beats the Fresnel shell here

House rule 4 — the Fresnel-alpha shell — is a *single-colour-lobe* rule: there the
shape is the message and the saturated silhouette carries the identity. On a
value-mapped surface **the interior colour is the message**, and a Fresnel shell
(front α = 0.32) clears exactly the area that carries it: measured, the map went pale
across the whole face while only the rim stayed saturated.

The measured ladder (identical scene, 2400 px, one system):

| variant | surface colours | structure behind | verdict |
|---|---|---|---|
| opaque | clearest | invisible | the pure-ESP panel |
| uniform transparency 0.6 (α 0.4) | washed | washed | both messages lost — the failure mode |
| **uniform transparency 0.4 (α 0.6)** | reads well | legible | **chosen** |
| Fresnel shell | rim-only | clearest | right for lobes, wrong here |

**Rule:** render the ladder instead of arguing. Each panel is minutes, and the choice
is then made from pixels. "Transparency" scales must be agreed in words — this
collaboration used *1 = fully transparent*, so transparency 0.4 = Blender α 0.6; record
the convention and the α in the build JSON, because "alpha" said aloud means the
opposite convention half the time.

Two structure-side changes measured to help the map survive the veil: **matte metals**
(Metallic 0, ligand-like finish) keep their authored hue under the veil where a
metallic ball reads as world-tinted grey-blue; and a **mid-grey carbon** (`#828282`)
stops competing with the map's warm end where the usual beige did not.

**Light the side the camera sees.** The raking side-key that shapes metal balls for the
lobe figures put the camera-facing hemisphere of the surface in shadow and the whole
map went dark. For a value-mapped surface the key moves to the **front upper quadrant**
(~35° off the view axis so form survives), fill on the opposite front side, rim behind.

---

## 3. Two compositing routes — pick by geometry, not preference

* **Interpenetrating** (lobes weave *between* atoms; per-atom depth order matters —
  EDD, orbitals, IGM): ray-traced single pass with the two-view-layer holdout split,
  `../mol-framework-figure/references/compositing.md`. Compositing two flat images
  destroys exactly the depth that makes the figure true.
* **Enclosing** (a molecular surface that wraps the whole framework — the ESP-type
  case): **render the layers separately and composite in post.** Structure-only render
  + opaque-surface render, both on transparent film, then alpha-over at any α.
  The author's conclusion after the ladder: separating the two figures beats tuning
  transparency inside the modelling software — transparency stops being a render
  decision, each layer is QA-able in isolation, and a transparency change costs
  seconds instead of a re-render.
* **The test:** does the surface ever dip between individual atoms? Enclosing → route
  B; interpenetrating → route A. Route A is always *safe*; route B is the convenient
  one, and only for enclosed layouts. This project shipped both: the holdout composite
  for the in-render ladder, and the pure-layer pair as the post-compositing deliverable.

---

## 4. Series rules

One view frame for the whole series (anchor the view on *named* atoms — a
Kabsch fit over near-symmetric metal skeletons picks wrong correspondences; name
matching fixed a 90° flip). Per-panel orthographic solve at fill 0.80 kept the three
scales within 1% (26.25 / 26.27 / 26.07), so panels are directly comparable. Same
isovalue, same window, same lights everywhere; one montage plus the shared colourbar.

---

## 5. Mechanics worth keeping (each measured once, the hard way)

* **Marching-cubes winding can come out inward.** Test on a sample: face normal ·
  (face centroid − nearest atom) > 0 should hold for ≫50% of faces; if not, flip the
  winding globally. An inward surface self-shadows and renders dark with no error.
* **Delivery PNGs get locked by the author's image viewer** (Windows): render to a
  temp name and `os.replace` with retries, or the save fails after the render.
* Pixel QA: corners ≥254 white (transparent film + compositor white), both saturated
  endpoint colours present, plus the framework outline check from
  `../mol-framework-figure/references/verification.md`.
* Render each figure size with the outline thickness rescaled from its reference
  resolution (2 px @ 2400 here), or previews look over-inked.
