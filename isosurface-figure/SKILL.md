---
name: isosurface-figure
description: Draw isosurfaces from a scalar grid in Blender — electron density difference (EDD / 差分电荷密度), frontier orbitals (HOMO/LUMO 轨道图), weak-interaction surfaces (IGMH / IGM / NCI / RDG / 弱相互作用可视化), ESP or charge density on a molecular surface. Covers reading .cub/.cube grids (axis order, bohr vs angstrom), marching-cubes extraction, Taubin smoothing, choosing the isovalue per surface type, the measured colour schemes, the Fresnel-alpha translucent lobe material, value→colour mapped surfaces (ESP-type: colormap rules, uniform-alpha clarity ladder, separate-layer compositing), and overlaying lobes on a framework with correct depth. Use whenever the user mentions EDD, 差分电荷密度, electron/charge density difference, isosurface/等值面, HOMO/LUMO/frontier orbital/轨道图, IGMH/IGM/NCI/RDG/弱相互作用/sign(λ2)ρ, cube/cub 文件渲染, translucent lobes over a molecule, ESP 着色面/静电势面, or asks to visualise a grid quantity in 3D.
---

# Isosurface figures

Turn a **scalar field on a grid** into a translucent 3D surface (or a pair of them for
a signed field), sitting on the molecular framework, on a seamless white background.

A scalar grid is not a picture. Three things have to be decided before anything is
drawn, and all three are scientific claims rather than styling:

1. **what the field means** — an amplitude, a density, a difference, a gradient-based
   descriptor; hence whether `+` and `−` are *phases* or *accumulation/depletion*;
2. **what isovalue is fair** — including across a series, where one value may not be
   comparable at all;
3. **what the colours mean** — and they must not collide with the element palette.

Get those wrong and the figure is confidently mislabelled. `references/surface-types.md`
is the menu; read the section for your field before extracting anything.

The **framework** — atoms, bonds, materials, lights, camera, outline, white background
— is the sibling skill `../mol-framework-figure/`. The lobes are added *into* that
scene: read `../mol-framework-figure/SKILL.md` first, and
`../mol-framework-figure/references/compositing.md` before overlaying anything
translucent.

**Provenance.** This skill and its siblings were distilled from the author's
research-figure workflow; a redacted worked instance lives in `examples/`. The rules and
measured numbers here
transfer to any project; project-specific scripts do not — in a project that has them, reuse
them; in one that does not, build the equivalent from the recipes in `references/`
(the parser contract and the extraction pipeline are specified there in full).

## The deliverable

Per panel: one square PNG with the framework plus its isosurfaces, plus the mesh files
and a build record. The meshes are the audit trail — a `.npz` of vertices and faces per
signed lobe, plus a JSON summary carrying, per lobe: volume, connected-component count
before/after filtering, minimum distance to a metal, and the share of the surface sitting
on each metal. Those numbers are how a reviewer checks the figure *is* what the caption
says, without looking at it.

## House rules

1. **Extract, do not sculpt.** The surface is a level set of the field at a stated
   isovalue. Report the isovalue in the figure's own footer, and keep one isovalue
   across a series unless you have measured that the fields are not comparable — in
   which case say so instead of silently switching (`surface-types.md`, EDD section).
2. **Signed field → two lobes with opposite sign.** One extraction per sign, separate
   meshes, separate objects, separate materials. Never one merged mesh.
3. **Lobe colours are chosen by measurement, never by taste or by a colour picker.**
   Render each candidate colour in isolation, under the real scene lighting, and score
   the pair on lobe-to-lobe hue gap and on distance to every element colour. Both pairs
   in use in the reference project were chosen that way and are recorded, with their
   measured hues, in `references/colour-schemes.md`.
4. **The single-colour lobe is a Fresnel-alpha shell**: clear where it faces the camera,
   dense and saturated at the silhouette. Never a uniform alpha, and never a glossy
   "glass" finish — both were tried and both destroy the shape
   (`references/lobe-material.md`). For a **value-mapped** surface (ESP-type, colour =
   a number) the measured answer is the *opposite* — uniform alpha, front-lit, and
   possibly separate-layer compositing (`references/value-mapped-surfaces.md`).
5. **Isosurfaces are not outlined.** A hard line on a cloud reads as a solid body.
6. **A missing mesh is a hard error**, not a warning: a figure that renders without its
   lobes still looks finished (pitfall 3).
7. **Integrals are in the field's own units, geometry is in Å.** A density grid's volume
   element is bohr³ while mesh volumes and spacings are Å (pitfall 2).

## Workflow

### 1. Read the grid

```bash
# origin-project form; the parser contract it implements is references/cube-io.md
python <project>/script/edd_grid.py path/to/field.cub   # parse once, cache .npy + meta
```

The parser contract, the axis-order trap, the unit trap and the caching scheme are in
`references/cube-io.md`. **Validate the parse against an independent number before
trusting any voxel** — the reference project ships a workflow that integrates a known
sphere and compares against a published value, and a literal `0.0000` means the axes
are transposed.

### 2. Choose the isovalue for this surface type

`references/surface-types.md` gives the per-type starting values, why they differ by
order of magnitude (a density difference is ~10⁻⁴ e/bohr³; a wavefunction amplitude is
~10⁻² a.u.), and the diagnostic that tells you a value is unfair across a series.

### 3. Extract the signed lobes

```bash
# origin-project form; the pipeline it implements is references/extraction.md
python <project>/script/edd_isosurface.py path/to/field.cub --iso 0.0003
```

Marching cubes (`skimage`, `method="lewiner"`) → connected-component filter →
Taubin smoothing. `references/extraction.md` has the parameters, the reason **not** to
Gaussian-pre-smooth the field, and the component-volume effect.

### 4. Material and colour

`references/lobe-material.md` (the Fresnel shell) and `references/colour-schemes.md`
(the two measured pairs and how to add a third). If the surface is **value-mapped** —
a continuous scalar rendered as a colour map on one closed surface (ESP on a molecular
surface, sign(λ₂)ρ colouring) — read `references/value-mapped-surfaces.md` instead:
the colormap rules, the clarity ladder and the compositing route all differ.

### 5. Put the lobes into the framework scene

Build the framework per `../mol-framework-figure/`, then add the lobe objects into the lobe
collection, turn the framework outline's view-layer split on, and re-check the object
count. The trap this avoids: Freestyle treats a translucent lobe as an opaque occluder,
so framework lines behind it disappear —
`../mol-framework-figure/references/compositing.md`.

### 6. Verify

`references/verification.md`: lobe statistics against the summary, position against an
independent atom reference, colour presence in the pixels, and the count tripwire.

## Which surface, and what it looks like

| field | signed? | what `+`/`−` mean | typical isovalue | colour convention |
|---|---|---|---|---|
| electron density difference (EDD / Δρ) | yes | accumulation / depletion | 2–4 × 10⁻⁴ e/bohr³ | measured complementary pair |
| frontier orbital (ψ) | yes | **wavefunction phase** | 0.015–0.02 a.u. | measured pair, ≠ the EDD pair |
| IGMH / IGM (δg, δg_inter) | no | — (one-sided) | 0.005–0.02 a.u. | blue→green→red by sign(λ₂)ρ |
| NCI / RDG | no | — (one-sided) | RDG 0.3–0.5, ρ cut 0.05 a.u. | same colouring as IGM |
| ESP on a density surface | no | — (a colour map) | ρ = 0.001 a.u. surface | diverging ESP map |
| charge / spin density | yes / no | ±/ magnitude | 0.001–0.01 a.u. | sequential or ± |

Details, semantics and caveats per row: `references/surface-types.md`. For the
value-mapped rows (ESP-type) the colouring/clarity/compositing rules are in
`references/value-mapped-surfaces.md`.

## Reference files

| file | read it when |
|---|---|
| `references/cube-io.md` | reading a `.cub`/`.cube`, or a parse gives a suspiciously round number |
| `references/extraction.md` | choosing an isovalue, smoothing, component filtering, mesh stats |
| `references/surface-types.md` | **before** drawing — what each field means and how to label it |
| `references/colour-schemes.md` | picking or changing any lobe colour |
| `references/lobe-material.md` | the translucent shell, alpha values, what not to do |
| `references/value-mapped-surfaces.md` | **value→colour mapped surfaces (ESP-type)**: colormap rules, uniform-alpha clarity ladder, separate-layer compositing |
| `references/pitfalls.md` | the eight measured traps on this side |
| `references/verification.md` | before claiming an isosurface figure is correct |
| `examples/reference-project/` | two worked families: EDD and frontier orbitals |
| `examples/edd-blender-figure-v1/` | the original single-skill version (historical) |

## Maintaining this skill

Installed globally at `~/.agents/skills/isosurface-figure/`; see
`../mol-framework-figure/SKILL.md` ("Maintaining this skill") for the shared
provenance and validator story: the origin project ships
`script/validate_skill.py` and `script/validate_skill_claims.py` — the latter
re-derives the isovalue defaults, the lobe hex colours, the smoothing parameters and
the object counts straight from the source and fails on any disagreement. Run them
from that project root after editing either skill or the scene builder. Outside it,
update the expected values **and** the matching lines here in the same edit — a
skill whose numbers drift from the code is worse than none.
