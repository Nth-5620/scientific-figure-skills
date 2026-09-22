# Camera, framing, series layout, annotation

Framing is the one thing in this workflow that a viewer notices instantly when it is
wrong and that pixels cannot measure reliably — so it is *solved*, *recorded*, and
*reused*, never eyeballed.

---

## 1. Solve the camera, do not eyeball it

A perspective camera, focal 85 mm on a 36 mm sensor, so

```
tan(half-FOV) = (36 / 2) / 85 = 0.2118
```

Solve it once per subject set: find the lateral centre `(cx, cy)` and distance `d`
that put the subject's projected bounding box at **fill = 0.80** with margins as even
as possible. The projection used for both the solve and the record is

```
depth = max(d - z, eps)
nx = 0.5 + 0.5 * (x - cx) / (depth * tan_half)
ny = 0.5 + 0.5 * (y - cy) / (depth * tan_half)
fill_fraction = 2 * max(|nx - 0.5|.max(), |ny - 0.5|.max())
```

with image row 0 at the **top** (`pixel_y = (1 - ny) * H`). Write
`fill_fraction`, the four margins, `centred_ok` and `fully_inside` into the build
record.

Two implementation notes that cost real time:

- **A fixed-point loop does not centre a subject with depth spread.** The obvious
  update `cx += (0.5 - centre) / (2 * median_depth * tan_half)` assumes all points sit
  at the median depth; these subjects span ~4 Å of depth, so the loop settles a couple
  of percent off centre. The projected bbox centre is monotone in the offset, so a
  **bracketed bisection** lands on it exactly. Use bisection.
- **`centred_ok` is a record, not a hope**: assert the projected bbox centre is within
  1% of the frame centre after the solve.

## 2. One camera for a whole series

Fit **once for the series**, not per panel. In a series the frameworks are identical
and the *contents* differ (a different metal, a different orbital, a different
adsorption state). A per-panel fit would scale the smaller content up until each panel
filled the frame — normalising away the very difference the series exists to show.

Solve it so that three properties hold at once, which pull against each other:

- **one** camera, so panels stay comparable;
- the **framework** fills the frame and is centred, because it is the part every panel
  has in common and the part that must look identical in all of them;
- **every piece of content must fit**, and the union's bounding box is set by the
  single most extended element of the whole set, which exists in one panel only.

The resolution: drive size and centre from the **framework**, and take a floor on the
distance from the **union** — if the framework's fit would push anything past a
minimum margin, pull the camera back. Fill reported for the union may then exceed the
target; the framework's does not. Measured on the reference series: framework fill
0.800 in all five panels, margins within 1.9% of each other.

Record the **framework's** framing in every panel's build record (not the
lobe-inclusive box). That makes the record comparable across panels and across
datasets; where a panel's own lobe reaches further, that is content, not framing —
record it separately as `framing_including_lobes`.

## 3. Reusing a camera across datasets

Two figure sets of the same system (before/after adsorption, two methods, two
functional) are most useful when they are **pixel-comparable**: then a difference
between the two rows is a real difference in the physics rather than a change of zoom.

The precondition is measurable, so measure it rather than assuming it:

- compute each structure's centre the same way (the mean of the metal positions is
  used here, so the frame is defined by the same atoms in both);
- if the frameworks coincide — the reference case measured **0.000 Å** per-atom
  displacement, because the "after" model was the "before" geometry plus a guest —
  the same camera gives the same projection and the reuse is exact. Verified: union
  margin identical to the single-dataset case, camera **pulled back 0%**;
- **guard it**: if the maximum centre shift exceeds a tolerance (~0.05 Å), fall back to
  solving a fresh camera for the second set and say so in the record. Silently
  mis-framing every panel of a set is the failure mode this guard exists to prevent.

Also verify the optics match before reusing: assert the recorded
`tan(half-FOV)` equals the current one, so a camera solved at a different focal length
cannot be borrowed.

## 4. Montages and annotation text

The rendered panels are assembled into labelled figures with PIL (system Python), not
in Blender: one square render per cell resized with LANCZOS, a bold name under each
panel, a grey detail line under that, a grey footer line stating the isovalue and what
the colours mean, and (for a grid) a rotated row title in a left gutter.

Three traps, all silent:

- **Canvas size is a function of the text.** A title or footer that wraps to one more
  line silently changes the delivered canvas height, and if that figure was already
  "finished" you have just re-laid it out. Keep the exact strings outside the current
  dataset's path (or pin the expected canvas size in the verifier and re-check it),
  and see the maintenance note in `SKILL.md` about pinning sizes.
- **A glyph the font does not have renders as a box.** Arial and Arial Bold have **no
  U+2082 (SUBSCRIPT TWO)**, so a caption reading `CO₂` is drawn as `CO▯` — in a
  delivered figure. Measure the glyph coverage before using a character, or composite
  the subscript: draw a smaller `2` at a lowered baseline from the same font (0.70×
  size, 0.16 em drop reads correctly), and measure string widths as the sum of the
  runs so wrapping stays correct. Other characters worth checking the same way: `Å`,
  `λ`, `δ`, `ρ`, `ψ`, `°`, en/em dashes.
- **Size a rotated gutter strip from the measured run width**, not from
  `textbbox(...)[2:]`: the bbox treats the ascender padding as ink, so the strip
  carries dead white and the canvas drifts. And when a figure contains subscripted
  text, `textlength` of the raw string measures the *box glyph's* advance, not the
  composited subscript's.

Leave the live Blender session on a **reference panel** at the end of a batch (the
most representative one), so the user opens onto the figure the set is judged by.
