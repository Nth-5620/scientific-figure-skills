# Pitfalls — isosurfaces and grids

Eight traps that cost real debugging time in the reference project. Every one is
**silent**: a plausible-looking result rather than an error. The framework-side traps
(outline, materials, camera, annotation) are in
`../mol-framework-figure/references/pitfalls.md`.

---

## 1. Cube axis order (wrong lobes, no error)

**Symptom:** lobes render as detached blobs ~8 Å from the guest instead of wrapping it.
Everything else looks fine. Nothing crashes.

**Cause:** the Gaussian cube *specification* says x varies fastest; these Multiwfn
outputs are stored **z-fastest**. The data must be reshaped to `(nx, ny, nz)` so
`array[ix, iy, iz]` maps index 0→X, 1→Y, 2→Z. Assuming the spec transposes X and Z.

**Evidence, two independent checks:**
- the project's own validated integration script reshapes with `data.reshape(dims)`
  where `dims = [nx, ny, nz]`, and Multiwfn's subtraction was separately verified against
  numpy to 5×10⁻⁷ e/bohr³;
- integrating the difference field over a 4 Å sphere about the guest reproduces the
  independently computed accumulation/depletion values (redacted) **only** under
  z-fastest. The
  x-fastest mapping puts the sphere in empty space and returns exactly `0.0000`.

**Rule:** after parsing, integrate a known region and compare against a known value. A
literal zero means transposed axes — assert `abs(value) > 1e-6`.

---

## 2. Mixing Å³ and bohr³ volume elements (6.75× error)

Grid values are e/bohr³, so a *charge* integral multiplies by the box volume in
**bohr³**. Geometric quantities (mesh volumes, spacings for marching cubes) use the
**angstrom** spacing. For the 0.145432 bohr grid the ratio is `(1/0.529177)³ ≈ 6.75` —
small enough to look like a plausible physical result rather than a bug.

**Rule:** keep `dv_bohr3` and `dv_ang3` as separate named values; never let a single `dv`
serve both.

**Related, same family of error:** a wavefunction cube holds an **amplitude**, so there is
no charge to integrate at all. If an orbital panel's summary reports a `charge`, that
field is meaningless — remove it rather than quoting it.

---

## 3. A missing isosurface mesh renders silently

**Symptom:** the figure renders with **no lobes at all** and no error. Easy to miss if you
are checking a QA script rather than the object count.

**Cause:** a cold rebuild of one isovalue deleted another isovalue's `.npz`, and the
builder treated a missing mesh as a warning (`print` + `continue`).

**Fix:** raise `FileNotFoundError` naming the missing file and the command to regenerate
it. Object count is a cheap tripwire: `natoms + nbonds + 2 × n_lobes + 4` (camera + 3
lights) — the reference figure is `114 + 136 + 2 + 1 + 3 = 256`, so 254 means the lobes
are missing.

**Rule:** when a missing input would silently degrade the deliverable to something that
still looks finished, fail loudly.

---

## 4. Lobe colours chosen by hue number instead of by rendered pixels

**Symptom:** a "cyan" negative lobe that looks fine in a colour picker turns out to be
nearly indistinguishable from the Zn atoms on the figure. Separately: two lobes that are
different colours on paper still read as "two greens" in the render.

**Cause — two compounding errors:**
- measuring hue from a **full-scene** render, where occlusion and a translucent lobe over
  atoms contaminate the mean, instead of from **isolated** renders;
- comparing hue *differences* in the abstract instead of asking whether the two things
  are **spatially adjacent** in the projection. They were: 5.8% of the negative lobe's
  pixels sat directly against a Zn atom, so that pair had to be separable.

The full measurement table, both chosen pairs and the selection procedure are in
`references/colour-schemes.md`.

**Rule:** pick lobe colours from isolated-render measurements, score the pair on both the
lobe-to-lobe gap and adjacency to every atom colour, and never ship pink/purple unless the
author asks for it.

---

## 5. A near-metal difference residual that makes a substituted series incomparable

**Symptom:** at one uniform isovalue the metal-substituted panels render with lobes an
order of magnitude larger than the closed-shell reference, wrapped around the substituted
metals. Measured at iso = 0.0003 across the reference series, the positive lobe volume
grew severalfold from the closed-shell member to the fully substituted one, and the
lobe's share of the rendered subject grew with it (values redacted). The reference panel
looks correct; the substituted panels look like those
metals are accumulating enormous charge. Worse, it switches on and off per site, so the
strip reads as a chemistry that depends on which atom you point at.

**Cause:** the difference is taken between two *independent* SCF calculations, so each
metal sits on both sides of the subtraction and its own density must cancel. For a
closed-shell d¹⁰ metal it does (the two runs reproduce each other's near-nucleus density
to ~10⁻⁴ e/bohr³). For open-shell UKS + DFT+U systems it does not, at a subset of sites,
by ~three orders of magnitude. The affected region is **far larger** than the extreme-value
voxels — a "the extremes are 0.2 Å from the nucleus and don't affect visualisation" note
was measured to be wrong: the above-isovalue region reached ~2 Å from the nucleus,
far beyond the rendered ball.

**Evidence** (per site, all measured — full protocol in `surface-types.md` §1): contiguous
above-iso radius 0 Å for every healthy site vs ~2 Å at affected sites;
accumulation within r = 2.0 Å an order of magnitude larger at affected sites with
net ≈ 0 (a pure rearrangement); angular power
l = 4 (cubic) dominated, so it is a grid-signature artefact and not a d-orbital shape.

**What does not fix it:** masking the metals at a radius just under the rendered ball
removes only 1–2% of the lobe volume — a diagnostic, not a fix; and per-panel isovalues
hide the finding (the spread in the isovalue needed is itself the result).

**Rule:** before building a substituted series, check each system's field extremes against
the closed-shell reference and run the per-site diagnostics. If affected sites exist, a
uniform isovalue does **not** make the panels comparable: say so, and quote a
metal-excluded interfacial integral as the number that *is*. Fix it at the source (make
the two runs converge to the same solution — e.g. seed one from the other's wavefunction —
and re-measure).

---

## 6. A re-exported cube reorders the atom block

**Symptom:** a quantity that was correct in the first generation of a dataset comes out
*plausible but wrong* in the second. Here: the guest-sphere integral for the reference
system — which barely changes
between generations — appeared to drop severalfold, and the drop was written up as
evidence of a
methodological fix. Nothing raised an error anywhere. (Values redacted.)

**Cause:** the two generations hold the same atoms in the same absolute frame but in a
different ORDER, so any positional assumption silently points at different chemistry:

| | v1 cube | v2 cube |
|---|---|---|
| guest atoms | the head of the list | **the tail (last three)** |
| the metals | scattered mid-list | **different positions** |

**Evidence:** with the true guest centre a 4 Å sphere gives the correct value; centred
instead on the mean of the cube's first three atoms (a framework C/H/C
trio) it gives a **plausible but wrong** number, matching the (wrong) published CSV to
five digits (values redacted). A sphere around a
framework carbon returns a number of the same order as the real one, which is why nothing
looked wrong. It bit twice: an upstream script that fell back to "the first three atoms
are the guest" and this project's own analyser that used `atoms[:3]` directly.

**Fix:** never index. Match on element **plus coordinates** to map cube atoms to a stable
index (`mol_atom_map`, tolerance 0.05 Å — which also *validates* that the two files share
a frame), and locate a fragment by its geometric signature (a C carrying exactly two O at
~1.16 Å) rather than by layout. Key per-atom results by the stable index so site labels
mean the same thing across generations.

**Rule:** recomputing or re-exporting a dataset is a reason to re-validate every
positional assumption about it, not just its numbers — and a fallback that assumes an
input layout should fail loudly, because it will otherwise keep returning a *different*
answer with the same units.

---

## 7. The guest is first in one input file and last in the other

**Symptom:** a model built from two files (a `.mol` for connectivity, a cube for
coordinates) draws or analyses the guest in the wrong place, or an assertion about atom
order fails for a reason that looks like a data problem.

**Cause:** the two files disagree about ordering **in a consistent, knowable way**: the
mol lists the guest first (atoms 1–3) and the cube appends it last (the tail), with
the framework atoms otherwise in the same order in both.

**Fix:** do not "fix" either file. Permute explicitly — build the map between the two
orders (framework atoms shift by the guest count, the guest moves to the tail), then
**assert element-for-element and coordinate-for-coordinate** that the permutation is
right before using it. Measured after the permutation: maximum disagreement 4.77×10⁻⁷ Å,
i.e. the same geometry to the cube's own precision. Carry the bond list through the same
permutation (including bond orders) and verify the bond count is unchanged.

**Rule:** guest ordering is a property of the file, not of the chemistry; make the
mapping explicit, assert it, and record it in the build record (`guest_atom_indices`).

---

## 8. Text and figures that were already delivered

**Symptom class:** a delivered figure changes because a *shared* helper was edited for a
new dataset.

Two measured instances, both on the caption/footer side: appending an adsorption-state
note to a shared title helper re-wrapped the title and changed a delivered canvas from
1872×1338 to 1872×1394; and fixing a missing-glyph box for `CO₂` changed a footer's width
from 4722 to 4713 px (both were caught because the verifier pins the expected canvas
size).

**Fix:** keep dataset-specific strings out of the code path used by already-delivered
figures, and pin each delivered canvas size in the verifier so a re-layout fails a check
instead of shipping. When a size *must* change, update the pinned value **with a comment
saying why** — the pin exists to catch accidental drift, not to forbid intentional
change. Details in `../mol-framework-figure/references/pitfalls.md` §11.

**Rule:** regenerating a figure that was already handed over is a decision to make
explicitly and report, not a side effect of adding a feature.
