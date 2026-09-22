# Pitfalls

Traps that bit while building the curve figures in the origin project. Each one is silent —
the figure renders, the script exits 0, and the result *looks* finished — so the rule and
the measurement that established it both matter. Numbers are from this project's own runs.

---

## 1. Setting the font size to the measured digit height makes text ~1.5x too small

**Symptom:** the reproduced frame looks right but the text is visibly smaller relative to
the box than the reference's, and every automatic check still passes.

**Cause:** a reference measurement gives you *how tall the digits are*, while matplotlib's
`fontsize` is the **em**. A Times digit is only ~0.685 em tall, so using 8 px (measured)
as the font size gives 5.5 px digits.

**Fix:** measure the ratio by rendering a probe digit at a known size, then solve:
`em = target_digit_px / (probe_ink_height / probe_em)`. In this project the probe gives
0.685 em, so an 85.7 px digit needs a 125.2 px (30 pt at 300 dpi) em — the difference
between a figure that matches the reference's proportions and one that looks under-set.
Do not hard-code the 0.685: re-measure if the face changes.

**Rule:** derive *every* size from a measurement of the actual font, not from a table of
nominal ratios.

---

## 2. Asking for a bold font silently returns the regular one

**Symptom:** the axis text is not bold while the code says `weight="bold"`. matplotlib
logs only `findfont: Failed to find font weight bold, now using 400.` — easy to miss in a
long render log, and the figure looks plausible.

**Cause, two variants:**
- `findfont(FontProperties(family="serif", weight="bold"))` resolves through the rcParams
  *serif list*, which may not contain your face at all.
- `FontProperties(fname=r"...\timesbd.ttf")` does **not** read the weight out of the file —
  `prop.get_weight()` returns `normal` even for a genuine bold face, so an assertion built
  on it is either wrong or vacuous.

**Fix:** look the face up in the font index by name *and* weight and assert there:

```python
faces = [f for f in font_manager.fontManager.ttflist
         if f.name == "Times New Roman" and str(f.weight) in ("700", "bold")
         and f.style == "normal"]
font = font_manager.FontProperties(fname=faces[0].fname, weight="bold")
```

**Verify it from pixels, not from the object:** render the same digit with the bold and
the regular face and compare ink area. Measured here: bold 2626 px vs regular 1605 px
(+63.6%) — a one-line check that a regular-weight regression cannot pass.

---

## 3. mathtext silently uses a non-bold face — and logs four warnings per figure

**Symptom:** every figure logs `findfont: Failed to find font weight bold, now using 400`
even after the faces are pinned; the superscripts in `(10$^{-3}$ e bohr$^{-1}$)` are the
suspicious part.

**Cause:** with `mathtext.fontset = "custom"`, mathtext still constructs a *fallback* font
eagerly. The default fallback (`stixsans`) has no bold face registered, so it warns — even
though the fallback is never used, because every glyph in the label exists in Times.

**Fix:** `mathtext.fontset="custom"` + `mathtext.rm/it/bf` pinned to Times New Roman +
`mathtext.fallback="stix"` (STIX is the Times-compatible set and does have a bold face).
Set the rcParams **before** the font probes, so the probes measure what ships.

**Rule:** a warning that fires once per figure and mentions the very property you care
about (bold) must be chased, not tolerated.

---

## 4. Multi-line axis titles: wrong width, and a bottom margin that clips

**Symptom, part 1:** the canvas comes out too narrow and the first line of a two-line
title runs off the edge. **Part 2:** the second line is cut off at the bottom — and the
render reports success, because a clipped label is still a successful render.

**Cause:** `renderer.get_text_width_height_descent(s, prop, ismath=True)` does **not**
parse newlines, so measuring `"line one\nline two"` as one string gives a meaningless
width. And a bottom margin written for a one-line title (3 em here) cannot hold two.

**Fix:** measure the **widest line** (`max(measure_text_px(font, l) for l in
XLABEL.splitlines())`) and scale the margin with the line count
(`(1.0 + 0.45 + 1.30 * n_lines) * em`). Then *check the finished PNG* for ink in the last
rows/columns — the two clips in this project were both found that way.

---

## 5. Outward y ticks push their labels out of the canvas

**Symptom:** after switching the y ticks to outward (the author's convention, and the
usual journal look), the y tick labels or the rotated axis title are cropped, or they sit
on top of the neighbouring panel.

**Cause:** the tick label offset is measured from the *end of the tick*, so an outward
tick adds its full length (53.6 px here) to the space the label needs. The margin had been
computed for inward ticks.

**Fix:** add `TICK_MAJOR_PX` to the left margin when the ticks point outward. Keep the
tick styling in **one shared function** used by every figure type — the two copies in this
project drifted within a single change.

---

## 6. A tall plot box squeezes the value axis; the fix is not just "make it wider"

**Symptom:** in the reference-matching panel the value axis has only 100 px per
1×10⁻³ e/bohr, so five systems' curves differ by a few pixels and comparisons are
impossible.

**Cause:** the panel's axes box is locked to the render's row band (2400 px) for the
register, and its width came from the reference's own portrait box (148:224). That is
correct for a *panel* and wrong for a *comparison plot*.

**Fix:** for the standalone comparison figure, use a **4:3 box** (the author's convention)
and a wider value axis — 150 px per unit here, i.e. 1.5× the panel's horizontal
resolution. Keep the axis *ranges* identical to the panels so the curve shapes stay
comparable row for row; only the aspect changes. Say so in the record, because the shapes
on screen then differ from the panels'.

---

## 7. Verifying a curve by "the centre of the ink run in each row" fails at sharp peaks

**Symptom:** a correct curve fails the check with deviations of 26–35 px at the peaks
(0.02–0.17 of the rows), while the median is <1 px. Blaming the data is a mistake.

**Cause:** the line is 21 px wide. At a sharp peak the two flanks are closer together
than the line width, so a row's ink is a *single merged run* whose centre sits **between**
the flanks — up to ~35 px from the path. The estimator, not the drawing, is broken.

**Fix — a two-sided distance test instead:**

1. for every row of the axes, the data path point `x_ref(row)` must have ink within a
   tolerance (≈1 line width): median distance ≤ 2 px, ≤2% of rows outside;
2. every ink pixel must lie within the tolerance of *some* path point: ≤3% outside.

Two extra masks are needed for this to be exact:
- **exclude grey anti-aliased text**: a plain "blue-ish" test (`B>110 & R<120 & G<120`)
  also matches (115,115,115) pixels of the axis text; require `B - R > 60`.
- **exclude the legend block**: its colour swatches are straight segments far from any
  curve and read as "ink 200 px off the path". Locate the block from the swatches and mask
  it out.

---

## 8. Two ways to get the plot box wrong when measuring it back

**Symptom:** the box's data-area rows come out ~11 px off, or the detector returns `None`.

**Cause, two variants:**
- Counting *dark pixels per row* to find the frame also catches the x axis title (a row
  full of dark pixels). Use the longest **contiguous** dark run per row instead.
- Counting *dark pixels across the box* at the frame's own row returns **both** spines, so
  it doubles the frame width and mis-places the data area by half a spine. Measure the
  spine width on one row well inside the box.
- A criterion of "run > 80% of the canvas height" works only when the box fills the
  canvas (it does in the portrait panels, not in the 4:3 overlay). Use a threshold
  relative to the **longest run found**, since frame spines are the longest straight runs
  in the figure (curves reach a few hundred px at most).

---

## 9. numpy 2 removals that break a working verifier

`np.trapz` → `np.trapezoid`, and the `ndarray.ptp()` method → `np.ptp(arr)`. Both raise
`AttributeError` with a message that suggests a different name, so they are quick to fix
but only once you see them; a checker that crashes is a checker you will not run.

---

## 10. The crop's integer rows move the axis origin by half a pixel

**Symptom:** the recorded origin says "the box frame's 4.000 Å plane" while the axis, read
back from the PNG, starts at 4.0034 Å. Nobody notices; a reader measuring the figure sees
a 0.5 px inconsistency with the caption.

**Cause:** the window's lower edge is rounded to a round box coordinate, and the crop box
is then rounded again to integer pixel rows.

**Fix:** derive the axis limits from the rows *actually cropped* (`y_of_row(r_a)`,
`y_of_row(r_b)`) rather than from the nominal window, and record both numbers ("rounded to
4.000 Å before cropping, 4.0034 Å after"). Now the register is exact by construction and
the residual is documented instead of hidden.

---

## 11. A swapped legend or colour makes every reader wrong, silently

**Symptom:** none, ever. The figure looks perfect and says the opposite of the truth.

**Cause:** the colour list, the plot order and the label list are three separate places
that must agree; a refactor (e.g. sorting systems for a different reason) breaks the
mapping without touching the render.

**Fix — two cheap pixel checks:**
- **ink ↔ data**: for each series, mask the image by that series' exact colour
  (tolerance ~16 levels) and require the ink to lie on **that** series' data path
  (two-sided, pitfall 7);
- **legend order**: identify each entry by its *swatch* (a horizontal run ≥80 px of that
  colour in the legend corner) and require the rows to ascend in the order the systems are
  listed. Taking "the mean row of all pixels of that colour in the upper-left block"
  instead let anti-aliased pixels of a neighbouring curve drift the mean (it reported
  SYS2 *above* SYS1).

---

## 12. An Origin-style gradient fill: measure the direction and the strength

The author's reference has a faint wash inside the plot. Sampling its interior in a 3×3
grid gives the rule: essentially white at the top (242,250,251) fading to a ~25% tint at
the bottom (194,254,222). Reproduce it as white → tint with the tint ≈ 0.28 of a saturated
colour, drawn with `imshow(..., extent=(x0,x1,y0,y1), zorder=0.5)` so it sits under the
CO2 image and under the curve. For a multi-system overlay pick white (a blend of five
metal colours means nothing) and say why.

**Verify it from pixels:** median colour of the top and bottom strips of the plot interior
vs the computed tint. Measured agreement here: ≤1 level.

---

## 13. Naming the plotted quantity: `plane-integrated` ≠ `plane-averaged` ≠ `line-integrated`

The axis title is part of the result and it is easy to get wrong by one power of length.

**Multiwfn's own English** (the data's origin) names the three curves: *local integral
curve* = the integral over the plane ⟂ the chosen axis (e/bohr for a density);
*plane-averaged curve* = that divided by the box cross-section (e/bohr³); *integral curve*
= the cumulative integral over the coordinate (an electron count), known as the *charge
displacement curve* (CDC) when the integrand is a density difference. The manual states no
units — they follow from the definitions.

**Literature usage** (phrase counts in Europe PMC): `plane-averaged` / `planar-averaged`
(81/72) is the dominant phrase but denotes the **average**; `plane-integrated` (3–7) is the
correct form for the sum, with real precedent (e.g. *Nat. Commun.* 2018, PMC6086897, which
plots exactly this quantity against z); `laterally averaged` belongs to electrostatic
potential / work-function profiles (0 hits for a density difference); `line-integrated` is
taken by plasma diagnostics for a 1-D integral (e/Å²) and implies the wrong dimensionality.
Papers do mix the terms — one writes "planar-averaged" in the text and "plane-integrated"
in the caption for the same formula — so **the unit, not the adjective, is the
disambiguator**: print it, spell the quantity out (no bare acronym), and define the
integral in the caption.
