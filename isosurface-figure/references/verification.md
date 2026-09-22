# Verifying an isosurface figure without vision

The whole-image and per-object checks (background, framing, exposure, element
legibility, structural audit) are in `../mol-framework-figure/references/verification.md`.
This file adds the checks that are specific to the fact that the figure carries a
*scientific field*: they are the ones that catch a figure which is beautiful and wrong.

Everything here is a number, because the failure modes are silent.

---

## 1. The field itself

| check | how | what a failure means |
|---|---|---|
| parse sanity | the grid array has `nx·ny·nz` values, no NaN, and a plausible range | a ragged line shifted the values |
| **axis order** | integrate a known region and compare with a known value; assert `\|value\| > 1e-6` | a literal 0 means transposed axes (pitfall 1) |
| **volume element** | a charge integral from a density field must match an independent calculation to ~5×10⁻⁷ e/bohr³ | off by 6.75× means Å³ vs bohr³ (pitfall 2) |
| isovalue inside range | `min(field) < iso < max(field)` per sign, with a non-empty extraction | an out-of-range isovalue yields zero vertices and a silently lobe-less figure |
| a fragment located by geometry | the guest centre found by signature, not by index | a positional assumption is broken (pitfalls 6–7) |

## 2. The mesh

| check | how |
|---|---|
| both signs present | a non-empty mesh per sign, positive volume each |
| counts match the summary | `len(npz['verts'])` and `len(npz['faces'])` equal the recorded values — proves the mesh on disk is the one that was summarised |
| component filter declared | `removed_frac_pct` recorded and small (a few %) |
| position | every vertex lies inside the grid box, and the lobe's minimum distance to the fragment you claim it sits on is small and *reported* |
| frame agreement | the meshes and the atom coordinates were centred by subtracting the **same** origin (compare the mesh centroid against the atom constellation, not against an assumption) |

## 3. The panel set

| check | how | why it matters |
|---|---|---|
| one isovalue | read the build records | a per-panel isovalue is a per-panel claim |
| one camera | `camera_xyz` identical in every record | otherwise panels are not comparable and differences may be framing |
| one set of lobe colours | the hex in every record | a family must not drift panel to panel |
| the camera *contains* the lobes | predicted projection vs the recorded framing; and from the pixels | lobes are content, so they may exceed the framework's box — but they must not leave the frame |
| comparable fields across a series | per-system extremes vs the closed-shell reference, and the per-site diagnostics | the near-metal residual trap (pitfall 5): a uniform isovalue can be unfair |

## 4. The delivered image

- **Lobe colour presence.** Classify subject pixels by hue and require **both** lobe
  colours above a share floor (measured floors used here: 4% of subject pixels for the
  positive lobe, 2% for the negative). This is what catches a lobe that is present in the
  scene but invisible in the pixels — occluded, washed out by a highlight, or too small.
- **Nothing touches the frame edge**, for the composite figures especially: a caption or
  a lobe running off the canvas is the failure that does not raise.
- **Canvas size matches the pinned value** for every delivered composite, with a comment
  recording why the pin was last changed (pitfall 8).
- **The lobe colours are not confusable with the elements** in this projection — re-run
  the adjacency check if the camera or the palette changed.

## 5. The claim

Finally, check the caption against the numbers, because this is where a correct figure
becomes a wrong paper:

- does the caption state the field, the isovalue **with units**, and what each colour
  means (`surface-types.md` §7)?
- if the figure is a difference density, is the sign convention stated (`+` =
  accumulation) and is the quoted transfer a **net** number, with the gross
  accumulation/depletion beside it so a reader can see whether it is transfer or
  polarisation?
- if it is an orbital, does the caption avoid claiming charge movement? ψ is a phase.
- if it is IGMH, are the fragments and the δg variant stated, and the colour range of
  sign(λ₂)ρ?
- if two panels are compared, is the claim one the shared camera and shared isovalue
  actually support?

State plainly at hand-over which judgements remain the user's: whether the isovalue
shows the right amount of structure, whether the lobes suit the story they want to tell,
and whether the palette reads the way they want on a printed page.
