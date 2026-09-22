---
name: curve-panel-figures
description: Build publication-quality 2D data figures in Python (matplotlib + PIL) whose layout is measured off a reference image — plane-integral / charge-displacement-style curves beside a rendered model, and all-curves-in-one-frame overlays with a legend. Use whenever the task mentions drawing or assembling curve figures, 曲线图, 叠加图 / overlay, 图例, 排成参考图的版式, 与渲染图对齐, pixel-exact panel layout, tick direction / frame proportions / Times New Roman bold axis labels, or verifying a finished figure against its data by measuring the PNG. Covers the measured-calibration and verification discipline that goes with them; the 3D render side lives in the mol-framework-figure and isosurface-figure skills.
---

# Curve and overlay figures, laid out against a measured reference

Produces the two figure types built in the origin project:

| figure | what it is | origin-project example |
|---|---|---|
| **panel** | rendered model on the left, one data curve on the right, both covering the **same height window** row for row | `result2/render/figures/<SYS>_EDD_curve_panel.png` |
| **overlay** | every system's curve in one landscape frame with a legend, for comparing them | `result2/render/figures/EDD_v2_curve_overlay.png` |

The origin project's `script/make_edd_curve_panels.py` is the working implementation
and `script/verify_curve_panels.py` re-measures the finished PNGs. **In that project,
reuse them; do not rewrite them** — they are the source of truth for every number
quoted here. In another project there is no such script: build the equivalent from
the parameterisation in §4 and hold it to the same verification discipline (§5). The
recipe transfers; the file does not.

The 3D renders this skill puts beside a curve are the subject of the sibling skills,
`../mol-framework-figure/` (Blender/Cycles, one object per atom, Freestyle outline, the
shared camera) and `../isosurface-figure/` (the lobes). This skill picks up where those
end: the PNG exists, now it has to become a figure.

## Conventions the author has fixed

These came from explicit instruction. Do not "improve" them.

1. **All text is English, Times New Roman, BOLD.** Resolve the bold face by *name and
   weight from the font index* and pass it explicitly — see pitfall 2.
2. **Frame, curve and font sizes are RATIOS measured off the author's reference image**,
   reproduced at your own resolution. Measure the reference yourself (frame line, tick
   digit height, tick length) at one fixed darkness threshold; do not carry over the
   absolute numbers (a screenshot is a downscaled copy of the original).
3. **Tick direction is mixed:** x (value) ticks point **inward**, y (coordinate) ticks
   point **outward**. Outward y ticks push their labels out by the tick length, so the
   left margin must include it (pitfall 5).
4. **Frame closed, no gridlines**, one thin grey vertical line at zero. A gradient wash,
   when used, sits *under* everything (pitfall 12).
5. **The curve line is thick** — the reference's curve and frame are the same weight;
   for an *overlay* of five near-coincident curves, thin the curves to ~0.6 of the frame
   or they merge into one smear.
6. **A standalone data plot uses a 4:3 box.** The *panel* figures deliberately keep the
   reference's portrait box (≈0.66), because their row band is locked to the render's
   height for the register in §3.
7. **Everything is verified from the delivered pixels**, not from the code that drew it
   (§5). A figure that renders is not a figure that is right.

## Workflow

### 1. Audit the data's units before plotting anything

Derived curves (plane integrals, cumulative integrals, plane averages) are easy to
mislabel by one power of length. Read the exported files and check the algebra that
must hold — three checks, all cheap:

| check | what it proves |
|---|---|
| column 2 = column 1 × 0.529177 | which column is bohr and which is Å |
| `locint / planeavg` is a constant = the grid's cross-section **area** (bohr² or Å²) | the "integral" is the integral **over the plane**, not a line integral (a line integral would give a *length*) |
| `∫ locint d(coordinate)` = the exported cumulative curve's end value | the integral's unit is density × length² (e/bohr), the cumulative one's is an electron count |

Then label the axis with the quantity *and* its unit, in the reference's "value
(×10⁻ⁿ unit)" style, and put the defining integral in the caption. Pitfall 13 lists the
naming traps (`plane-integrated` vs `plane-averaged` vs `line-integrated` vs `CDC`).

### 2. Choose the height window once for the whole series

For panels: the window is the union of all systems' **drawn** extents (atoms + meshes +
sphere radii, measured from the PNGs — not predicted from atom positions, which is
20–60 px short), plus a small margin, with the lower edge rounded in the *box* frame so
the plotted zero lands on a round plane. One window for the series, for the same reason
the renders share one camera: identical frameworks must not be rescaled against each
other.

The plotted height coordinate is then **height above the window's lower edge**, and the
record must state what that is in the box frame; the y axis title alone cannot say it.

### 3. Register: make equal rows mean equal height

The render is a perspective projection, so "the same height on screen" needs a reference
depth. Use one linear map anchored at a chosen plane (the scene origin / metal plane):

    row(y) = H/2 - (y - cam_y) * 0.5 * H / (d * tan_half)      # H = render height

Crop the render to the window, rescale it into the left band, put the plot's axes box on
the **same canvas rows** with limits taken from that map (after the crop, from the rows
actually cropped, so the rounding cannot accumulate). Off-plane objects keep a residual:
the guest sits 4.30 Å nearer the camera and projects ~11 px lower than the map — half a
frame line width. Place the in-plot model thumbnail on the object's *actual* projected
row so the two images coincide, and record the residual.

### 4. Draw, then measure the result

The origin project's `make_edd_curve_panels.py` shows the shape of the implementation:
parameterised at the top — `X_LIM_1E3`, `X_MAJOR_1E3`, `Y_MAJOR`/`Y_MINOR`,
`WINDOW_MARGIN_A`/`WINDOW_ROUND_A`, `GRADIENT_TINT_MAX`/`TINT_ZN`/`TINT_CO`,
`OVL_BOX_W/H`, `OVL_CURVE_FRAC`, `OVL_LEGEND_SCALE`, and the `REF_*` reference ratios —
with everything else derived: the font em is calibrated from a rendered probe, the
margins from measured label widths, the canvas from the margins. Reproduce that shape:
explicit parameters at the top, derived values below, nothing hard-coded mid-file.

Then verify from the delivered PNGs only:

```bash
python make_curve_panels.py       # panels + overlay + a build/params JSON
python verify_curve_panels.py     # PASS/FAIL, ~80 checks, reads only the PNGs
```

Checks worth having in any figure like this (see `references/verification.md`):

- the camera/map prediction vs the panel's own ink rows (≤2 px), and the crop covering
  the whole subject;
- axis limits re-derived from the frame's pixel rows vs the record;
- the left crop's rows vs the render's ink rows inside the composite;
- **two-sided curve test** per curve: every data row has ink within a tolerance of the
  path, and every ink pixel lies within the tolerance of the path (not run centres —
  pitfall 7);
- the in-plot model's ink footprint vs the panel camera's own projection of it (size and
  centre row), plus "the thumbnail is under the curve", plus "the text really is bold";
- **colour ↔ series mapping**: each colour's ink against *its own* curve, and the legend
  swatch order top-to-bottom (a swapped legend is silent and misleads every reader).

## Reference files

- `references/pitfalls.md` — the traps that actually bit, with the measurements. Read it
  before writing or editing any figure code.
- `references/verification.md` — the checks, the thresholds, and which failures are real
  versus artefacts of the checker.
- The origin project's `result2/render/README_curve_panels.md` — what the delivered
  figures contain: parameters, the alignment residuals, the QA table.

## Maintaining this skill

Installed globally at `~/.agents/skills/curve-panel-figures/`, alongside its siblings
`../mol-framework-figure/` and `../isosurface-figure/`. The origin project
ships the panel builder and its verifier; elsewhere the recipe in §4
is the contract. If you add a numeric claim here, keep it with the measured evidence —
a convention without its measurement reads as a preference and will be "improved" away.
