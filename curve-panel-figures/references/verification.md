# Verification without looking

The finished figure is the only thing the reader sees, so check the **PNG**, not the code
that drew it. The origin project's `script/verify_curve_panels.py` does exactly that and
prints one line per check with the measured number; the numbers below are that project's,
so a future run can tell "broken" from "slightly different".

Rule of thumb: a check that reads the build record back is a tautology; a check that reads
pixels and compares against an *independent* prediction is evidence.

---

## What each check proves

| check | proves | how |
|---|---|---|
| panel pixels reproduce the camera map | the map used for the register is right | predict the drawn subject's row extent from mol + meshes + sphere radii through the recorded camera; compare with the inked rows (measured: 268…2178 predicted vs 266…2179 actual, ≤2 px) |
| crop holds the whole drawn subject | nothing was cut by the crop | crop rows vs ink rows (margins 18/11 px) |
| axis limits (0 at the bottom) | the 0-based axis and the window agree | re-derive the limits from the frame's centre rows and compare with the record (0.0000 / 16.2264 Å) |
| box data rows == panel band rows | both panels cover the same rows | 40…2440 both |
| left render displayed at the crop's transform | the paste+scale arithmetic is right | ink rows 62…2427 vs 62…2426 predicted |
| drawn subject's top height on the axis | the register is right in *physical* units | 16.078 Å displayed vs 16.057 Å from the camera (3.0 px, the off-plane residual) |
| the curve is the data (two-sided) | right file, right column, right scale | median distance 0.25 px; every one of 2356 rows carries ink; no ink >1 line width off the path |
| thumbnail footprint == the guest extent | the in-plot model has the panel's size | 292×431 px measured vs 290×431 predicted from the projection × the crop's 1.236 |
| thumbnail centre row == the guest's row | the in-plot model has the panel's height | −0.6 px |
| thumbnail under the curve | the layer order the author asked for | rebuild with the thumbnail on top: 1336 px differ |
| gradient = white → metal tint | the wash is the right colour and direction | top 254, bottom within 1 level of the computed tint |
| frame / digit ratios | the reference's proportions were reproduced | frame 0.01307 (target 0.01351), digit 0.05289 (target 0.05405) |
| the text really is bold | the bold face is in use | bold probe 2626 ink px vs regular 1605 (+63.6%) |
| overlay: each colour is its own series | the legend cannot lie | per colour, ink lies on that series' path (median 0.42–0.70 px, 0 outside) |
| overlay: legend swatch order | the entries are in the advertised order | swatch rows ascend: 168, 281, 395, 509, 623 |

---

## False failures seen in practice

Check the checker before "fixing" the figure.

**"Median distance small but frac>25px large" on a curve.** Geometry, not an error: the
two flanks of a sharp peak merge inside the line width (pitfall 7). Use the two-sided test,
or restrict to rows whose ink run is short.

**"Overlay: the SYS1 curve is not SYS1's data" with 16% of pixels flagged.** The flagged
pixels are the *legend swatches* — straight segments of that exact colour sitting far from
the curve. Locate the legend block and exclude it.

**"Legend swatches run SYS1 → SYS5" failing with an odd order.** The mask for one series
caught anti-aliased pixels of a neighbouring, similar colour; the mean row then drifts.
Identify swatches by a long horizontal run instead of by "all pixels of that colour in the
region".

**Frame detection returning None on the landscape overlay.** The detector required a run
longer than 80% of the canvas height, which only holds for the portrait panel. Threshold
relative to the longest run found.

**Axis limit off by ~0.09 Å.** The frame's spine is 21 px wide; using the outer ink row
instead of the spine's centre line shifts the derived limit by half a spine. Measure the
spine on a row inside the box.

**A dark pixel count on a column of 230 "frame thickness".** That is the whole box height
(one column of the spine), not its width. Measure the spine across, not along.

---

## Sanity thresholds that hold for this class of figure

- register: |measured − predicted| ≤ 4 px; the off-plane model ≤ 0.15 Å (≈3 px).
- curve trace: median ≤ 2 px, no row without ink, ≤3% of ink outside one line width.
- model thumbnail: size within 8 px of the projection, centre row within 6 px.
- ratios: within 0.0025 absolute of the reference's frame/box and 0.006 of its digit/box.
- bold: the bold face's ink at least 4% above the regular face's.

State in the hand-over which judgements remain the author's — the aspect ratio, how heavy
the line looks, whether the legend placement is pleasing. Measurement does not decide
taste, and saying so is part of reporting honestly.
