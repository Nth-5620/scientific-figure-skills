# Colour schemes for isosurfaces

Two pairs are in use in the reference project, both chosen by **measurement** and both
recorded here with the numbers that justified them. A third pair must be chosen the same
way — not from a colour picker, not from a paper's figure, and not by taste.

The reason this is a measurement problem: a translucent surface under scene lighting
does not render as its authored hex. It sweeps a wide range of RGB values as the Fresnel
alpha and the shading vary across its silhouette, so only its **hue** is stable. And the
pair has to satisfy two geometric facts about the figure, which no hex value can tell
you: the two lobes must be far apart *as rendered*, and neither must sit near any atom
colour **in the pixels where they are adjacent**.

---

## 1. The two pairs in use

| family | positive / `+` | negative / `−` | meaning |
|---|---|---|---|
| **EDD** (density difference) | **yellow `#DFE22C`** | **blue-violet `#8A2BE2`** | accumulation / depletion |
| **Frontier orbitals** (ψ) | **magenta `#FF2469`** | **green `#1DCC94`** | wavefunction phase ψ>0 / ψ<0 |

They are deliberately **different pairs**, because the two families appear side by side in
one paper and the colours carry the meaning: a reader must never have to ask whether a
yellow lobe is "where charge went" or "where ψ is positive".

### EDD pair — measured

| candidate pair | rendered hue | lobe-to-lobe gap | vs nearest atom |
|---|---|---|---|
| green / teal (first attempt) | 129.0 / 172.8 | **43.8°** — "two greens" | 79 / 36 |
| yellow / cyan (a literature pair) | 45.8 / 195.6 | 149.8 | **6 / 13** — both collide |
| lime / steel blue | 81.7 / 199.2 | 117.5 | 42 / **9** — negative hits Zn |
| yellow-green / teal | 83.5 / 175.0 | 91.5 | 44 / 33 |
| **yellow / blue-violet (chosen)** | **61.8 / 264.3** | **157.5°** | 22 / 25 |

Isolated-render measurement (one object visible, real scene lighting), on a palette that
already spends the blues (Zn ~206°, N ~240°) and red (O ~0°).

Two caveats, measured and recorded so nobody re-litigates them blind:

- the violet sits **24.5°** from the pure-blue N (240°) — a complementary partner to a
  yellow *must* land near 241–271°, exactly where N lives, and 271 is the only end of that
  window that clears N at all;
- the yellow sits **21.8°** from the beige C (~40°), the warm end of the palette. It still
  separates in the render because the carbons are pale and thin while the lobes are large
  and saturated.

### Frontier-orbital pair — measured

| item | rendered hue | saturation |
|---|---|---|
| positive `#FF2469` | **345.4°** | 0.594 |
| negative `#1DCC94` | **156.6°** | 0.521 |
| Co atom `#EE74A8` | 332.0° | 0.335 |
| Zn atom | 209.1° | 0.446 |
| N atom | 239.9° | 0.462 |
| O atom | ~0° | 0.446 |
| C atom | 41.6° | 0.212 |

- lobe-to-lobe gap **171.2°** (the EDD pair gives 157.5°) — the two signs are
  unmistakable;
- the green clears every atom colour by ≥ **52°** (nearest is Zn);
- **the one honest problem:** the magenta sits only **13.4°** from the pink Co atom (and
  14.6° from O). It still reads because lightness and saturation differ sharply — Co
  renders as a muted dark plum (104, 69, 85), sat 0.34, while the lobe is bright magenta
  (221, 90, 122), sat 0.59. If a future figure needs more margin, move the **Co atom**
  colour toward darker purple rather than changing the lobe pair.

---

## 2. How to pick a new pair (the procedure)

Do not skip to "looks good in a picker". Run the sweep:

1. **Isolate.** Render each candidate colour as the *only* visible object, under the real
   scene lighting and world. A full-scene render contaminates the measurement with
   occlusion and with the lobe over atoms — a "cyan" lobe measured in a scene once came
   out nearly indistinguishable from Zn, while its isolated hue was fine.
2. **Measure the rendered hue and saturation** of the subject pixels (mean, or median
   after eroding the anti-aliased boundary — boundary pixels dilute saturation:
   measured, an authored 0.685 read as 0.50 with the rim included, 0.58–0.69 after
   eroding ~2 px).
3. **Score the pair on two axes at once:**
   - **lobe-to-lobe hue gap** — as rendered, and this is the dominant term; below ~90°
     two colours read as one family ("two greens");
   - **distance to every atom colour**, and weight it by *adjacency in the actual
     projection*: what matters is whether the two things touch. Measured: 5.8% of one
     lobe's pixels sat directly against a Zn atom in that projection, so a 12° gap to Zn
     was fatal there while a 33° gap to a non-adjacent colour would have been irrelevant.
4. **Prefer a complementary pair** if the palette allows it: the hue gap is then ~150–175°
   by construction, and only one arm of the pair has to be negotiated around the atom
   palette.
5. **Write the numbers into this file and into the builder's comments**, so the next
   person does not re-open a settled question and does not "tidy up" the values.
6. Keep both colours **overridable per run** (a `lobe_pos_color` / `lobe_neg_color`
   parameter), but make the default the measured pair.

---

## 3. Constraints any new pair must respect

- **Clears every atom colour.** Know your palette's occupied hue regions before
  choosing: with Zn ~206°, N ~240°, O ~0° and C ~40° spent, the free space for a pair is
  small — that is *why* the measured winners are magenta/green and yellow/violet rather
  than the conventional green/blue.
- **Distinguishable from every other figure family in the same paper.** If two families
  will sit side by side, give them different pairs and put the meaning in both footers.
- **Not confusable with a mapped colour scale.** If the figure also carries a diverging
  map (an ESP surface, or an IGMH sign(λ₂)ρ colouring), the ± pair must not reuse the
  map's endpoint colours.
- **Pink/violet is allowed only on explicit instruction.** An earlier brief in the
  reference project excluded pink/purple; the author later asked for yellow/blue-violet,
  and magenta/green for orbitals, overriding it. Record such an override where the colour
  is defined rather than silently tolerating it — a future editor needs to know the
  constraint was lifted deliberately, not forgotten.
- **Check the alpha shell, not just the hue.** The colours are seen through the Fresnel
  alpha (`lobe-material.md`): the front face is ~1/3 opacity over the atoms behind it, so
  a colour can be fine at the silhouette and muddy in the middle. Measure the rendered
  pixels, which is what step 1 does.

---

## 4. Where the numbers live in code

- Pairs: one table in the scene builder (`LOBE = {"pos": (hex, objname), "neg": ...}`),
  with the measurement table in the comment above it.
- Measurement harness: a colour sweep that drives Blender over the socket rendering one
  isolated object per candidate, plus a scorer in system Python that reads those PNGs and
  prints the two-axis table. Keep both — the next pair will need them.
- The verifier asserts the delivered pair, so a careless edit to either hex fails a check
  rather than shipping (`verification.md`).
