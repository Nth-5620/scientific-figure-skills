# Extraction: level set → mesh

The pipeline is short and every step has a reason and a measured effect. Run it as one
code path for every surface family — an EDD lobe and an orbital lobe should differ in
what they *mean*, not in how they were drawn.

```
grid (nx, ny, nz) float32
  └─ marching cubes at |field| = iso, per sign   → verts, faces
       └─ drop connected components < min_volume
            └─ Taubin smooth (λ, μ, n passes)
                 └─ .npz (verts float32, faces int32) + a JSON summary
```

## 1. Marching cubes

`skimage.measure.marching_cubes(field_signed, level=iso, spacing=(dx, dy, dz),
method="lewiner", allow_degenerate=False)`.

- **`method="lewiner"`** — the topologically correct variant; the default
  (`lorensen`) leaves ambiguous-face artefacts that show up as pinholes in a translucent
  surface.
- **`spacing` must be the Å spacing, in (x, y, z) order matching the array axes.** If
  the array is `(nx, ny, nz)` with index 0→X, passing the spacings in that order is
  correct and the returned vertices are already in `(x, y, z)`. Do **not** permute the
  result afterwards — that is where a transposed field turns into a correctly-shaped but
  misplaced mesh.
- **Extract each sign separately** by extracting at `level = iso` from `+field` and from
  `−field`. Two meshes, two objects, two materials.
- **Fail loudly on an empty extraction.** If the isovalue is outside the field's range
  (or the field has no values past it), you get zero vertices — a silent, lobe-less
  figure. Return an explicit `empty: True` plus the field's max, and let the caller
  raise.
- A **near-nucleus cusp artefact** is normal in a difference field: `ρ(AB) − ρ(A) − ρ(B)`
  is a difference of huge numbers near a nucleus, so basis-set superposition and the
  cube's finite precision leave a shell within ~0.2–0.4 Å of each nucleus. At a
  0.0003 e/bohr³ isovalue that shell is usually invisible; it is *not* invisible when the
  two sides of the subtraction were not converged to the same solution (see the EDD
  section of `surface-types.md`, and pitfall 5). A "core exclusion" mask (zero the field
  inside a per-element radius) is provided for diagnosis, and it removes only ~1–2% of
  the lobe volume in the reference case — so it is a diagnostic, not a fix.

## 2. Drop tiny components

A real field carries a long tail of isolated blobs. Suppress them **by component
volume**, after meshing:

| `min_volume` | effect |
|---|---|
| 0.02 Å³ (density-difference default) | keeps almost everything; declutters only specks |
| 0.15 Å³ (orbital default) | removes a few percent of total volume; visibly cleaner for ψ fields, which fragment into many lobes |
| 0 | no filtering |

Report what was removed, always: `n_components`, `n_kept`, `removed_volume_A3`,
`removed_frac_pct`, and the sorted component volumes. A filter whose effect is not
reported is indistinguishable from a bug the day someone asks why a lobe is missing.

Filter **components**, not vertices or faces — a component is a connected run of faces,
computed once and reused as a label array. Implementation note: after filtering, remap
the surviving vertices and rebuild the face indices; do not leave unreferenced vertices
in the mesh (the framework skill's `expect_all_used` guard catches that class of error).

## 3. Smooth, with Taubin, not Laplacian

Marching-cubes output is faceted at the voxel scale — visible in a translucent surface
as a corrugated rim, which reads as noise in a publication figure.

**Taubin smoothing** alternates a positive and a negative Laplacian pass (λ = 0.50,
μ = −0.53), 15 iterations. Plain Laplacian smoothing **shrinks** the surface at every
pass — for these lobes that means a systematic volume loss; Taubin is a low-pass filter
that holds volume to within ~1.4% over the same number of passes. Use Taubin.

**Do not Gaussian-smooth the field before meshing.** It was measured to bias the lobes
**~30% smaller** at σ = 1.5 grid units, because it attenuates the peak before the level
set is taken. If you need a smoother surface, smooth the *mesh*, or extract from a finer
grid (re-run the writer with a smaller `STRIDE`).

## 4. The mesh summary is part of the deliverable

Per lobe, record at least:

| field | why it earns its place |
|---|---|
| `volume_after_A3` | the comparable physical number across a series |
| `n_verts`, `n_faces` | lets the verifier confirm the mesh on disk is the one summarised |
| `components.n_kept` / `n_components` | how much filtering happened |
| `min_dist_to_metal_A` | is the lobe actually on the metal, or merely near it |
| `metal_vertex_share_r1.2` | share of surface vertices within 1.2 Å of a metal nucleus |
| per-element share (`zn_share`, `co_share`, `co_fraction_of_metal_share`) | tells a ligand orbital from a metal-d state — see below |

`|ψ|` is an **amplitude**, so the lobe volume is not a charge and the same isovalue
means the same thing for every orbital **of one basis set and one grid box** — which is
what makes one isovalue fair across an orbital series (and why the "share" numbers are
only meaningful within that series).

**The "share on a metal" metric is a geometric proxy, not a population analysis.** It
is the fraction of marching-cubes vertices lying within a radius of a metal nucleus;
vertices are distributed over the surface roughly in proportion to area, so it is a
good proxy for the share of the *surface* on the metal — and it is not Mulliken/Löwdin
d-orbital character. Say "geometric proxy" in any report. When a series substitutes one
metal for another, report the **per-element split**, not the total: "the orbital has
metal character" is a different claim from "the orbital has *Co* character", and only
the split can tell them apart.

## 5. Isovalue: start here, then measure

Starting values per surface type are in `surface-types.md` (they span five orders of
magnitude between a density difference and a wavefunction amplitude — that is not a
mistake, they are different quantities).

Whatever you choose, **record it and put it in the figure footer**. Two further
obligations:

- **A sweep is cheap and settles arguments.** Extract at three or four isovalues and
  tabulate volume, component counts and the metal share. The origin project ships
  this as one sweep command and its
  record includes a coverage table (on the reference cluster, the covered-atom fraction
  ran from ~60% at 0.004 a.u. down to ~20% at 0.02 a.u.). "Why doesn't the lobe cover
  the whole molecule?" is answered by that table, not by prose — ψ has nodal surfaces
  and decays away from the atoms that dominate it, so the surface only exists where
  |ψ| exceeds the threshold.
- **One isovalue across a series is only fair if the fields are comparable.** Measure
  that before shipping (pitfall 5). If they are not comparable, the honest move is to
  say so and quote a field-integral number that *is* comparable, not to switch to
  per-panel isovalues.

## 6. Uniform across a panel set

Give every panel: the same isovalue, the same smoothing, the same component filter, the
same lobe radii/lighting/material, and — crucially — the **same camera** (see
`../mol-framework-figure/references/camera-and-layout.md`). Then a difference between two
panels is a difference in the field. Any knob that varies per panel is a difference you
must be able to defend.
