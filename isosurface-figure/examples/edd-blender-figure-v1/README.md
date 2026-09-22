# Legacy snapshot: the original `edd-blender-figure` skill (v1)

This is the **verbatim** content of the single skill that preceded the split into
`mol-framework-figure` and `isosurface-figure`, kept as a worked end-to-end example and
as the record of what was known at the time. `references/` beside this file holds its
three reference documents unchanged.

> **Redacted.** Scientific result values (energies, charges, integrals, per-system
> trends, atom indices/coordinates of the reference system) have been removed from this
> snapshot; drawing and rendering parameters are retained. Series members are referred
> to as `SYS1 … SYS5`.

**What changed when this was split.** The content was distributed by concern:

| this snapshot's content | now lives in |
|---|---|
| grid parsing, isosurface extraction, isovalue, lobe colour, Fresnel shell | `isosurface-figure/` |
| atoms/bonds/radii, palette, materials, lights, Freestyle outline, two view layers, camera, render, QA, Blender environment | `mol-framework-figure/` |
| the EDD workflow end to end | `isosurface-figure/references/surface-types.md` §1 + `examples/reference-project/` |

Two things in here are **out of date** relative to the new skills, and are kept only
because this is a snapshot: the lobe colours are described as a single global pair
(there are now two pairs, one per figure family, and they are deliberately different),
and the file paths assume one figure generation (there are now four:
`result/`, `result2/`, `result3/render/`, `result3/render2/`). The frontmatter of the
original is preserved below unchanged.

```yaml
name: edd-blender-figure
description: Build publication-quality 3D electron-density-difference (EDD / 差分电荷密度 / 电子密度差) isosurface figures in Blender — molecule framework plus translucent positive/negative lobes — driven through blender-mcp. Use whenever the user mentions EDD, 差分电荷密度, 密度差, electron/charge density difference, isosurface or 等值面 rendering, 等值面图, a Multiwfn/CP2K .cub or .cube density grid that needs a 3D figure, or asks to render an SBU/CO2/cluster structure in Blender with tube or ball-and-stick atoms. Also use when extending an existing EDD figure series to more systems (SYS1, SYS2, SYS3, SYS4, SYS5), or when a Blender molecular render needs the seamless-white-background / three-point-light / colour-separation treatment.
```

---


# EDD isosurface figures in Blender

Produce a journal figure: an atom framework in **global ball-and-stick** style (the
default — every atom a sphere, every bond a thin stick), plus translucent EDD
isosurface lobes, on a seamless white background.

This skill is the distilled workflow of the origin project. The canonical,
working implementation lives in that project's `script/` directory — **reuse those
scripts, do not rewrite them**. Read `references/pitfalls.md` before touching any of
the four stages; every rule there was learned by getting it wrong first and has
measured evidence behind it.

## Hard requirements

These came from explicit user instruction. They are not preferences.

1. **Build through blender-mcp, in the running GUI Blender.** Modelling must land in
   the session the user has open, so they can see and hand-tweak it, and the project
   `.blend` must be saved. Headless (`blender -b`) is for batch re-computation and
   verification only — never as the way the deliverable is modelled.
   Entry point: `script/build_in_blender.py` fed to the socket via
   `script/blender_bridge.py`.
2. **Derive the bonding yourself, or use `structure/`.** Do not assume a mol file is
   authoritative. Perceive bonds geometrically
   (`d <= 1.25 * (r_i + r_j)`, Cordero radii) and **cross-check against the mol bond
   block**, reporting any disagreement. Write the result into the build JSON.
3. **Every atom and every bond is its own object**, with one shared material per
   element. Do not merge atoms of the same element into one mesh, and do not merge
   all bonds into one mesh.
4. **Global ball-and-stick is the default** (`frame_style="ballstick"`, chosen by the
   author after comparing both). Every atom is a sphere and every bond a thin stick,
   with radii from one rule — `r = 0.44 × Cordero covalent radius` — so the size
   ordering stays physical (H smallest, the metals largest). The metals therefore read
   as the subject of the figure (the science is metal substitution) by **size**
   (0.537 Å vs 0.334 Å for carbon), not by a style difference. `frame_style="tube"` is
   the retained alternative; see `build_blender_scene.py` for what each one changes.
5. **Isosurface lobe colours: yellow for accumulation, blue-violet for depletion.**
   The original brief excluded pink/purple; the author later explicitly requested this
   complementary pair because the earlier green/teal pair left the two signs hard to
   tell apart. Measured on isolated renders, the rendered lobe-to-lobe hue gap went
   from 43.8° (green/teal) to 157.5° (yellow/blue-violet). Pick lobe colours **by
   measurement**, never by taste or by a colour picker — see pitfall 5.
6. **The lobes are a Fresnel-alpha shell**, not a constant-alpha one: clear where the
   surface faces the camera, dense and saturated at the silhouette. A glossy
   coat/transmission "glass" finish was tried and rejected — the specular spread over
   the whole lobe and dissolved the shape into white glare.
7. **Surface finish is per-element**: metals are metallic (`Metallic=1`, low
   roughness, **no AO term** — the AO distance exceeds the metal sphere radius and
   merely darkens it), ligands stay matte.
8. **CO₂ is a ball-and-stick model with double bonds drawn as two parallel sticks.**
9. **In the `tube` variant only, H atoms and C–H bonds use the same gauge as the rest
   of the framework**, so the ligand reads as one tube-style object. Under the default
   ball-and-stick style H is the smallest sphere (0.136 Å), which is the conventional
   ball-and-stick reading.
10. **Shade with angle-based sharp edges ("Shade Auto Smooth")**, never blanket
    smooth shading. See pitfall 4 — blanket smoothing puts a dark ring at every
    bond junction.
11. **The framework is outlined; the isosurfaces are not, and the outline must survive
    *under* the translucent lobes.** Implemented with Freestyle, whose line set selects
    by *collection*, inside a render split into two **view layers**: `Framework`
    (framework only, Freestyle on) and `Lobes` (the lobes, with the framework as a
    per-layer *holdout* so depth order is preserved), alpha-composited then laid over
    white. A single layer cannot do this, and neither can the visibility filter — see
    pitfall 12 for why, and pitfall 9 before touching the collections.

## Workflow

### 1. Parse the density grid

```bash
python script/edd_grid.py result/EDD_cubes/<SYS>_EDD.cub
```

Caches a `.npy` next to the cube (first run ~100 s for a 464 MB / 320^3 grid). Two
things here are silent traps — read pitfall 1 and 2 before trusting any number:

- **Axis order.** These Multiwfn cubes are **z-fastest**; reshape to `(nx, ny, nz)`.
- **Volume element.** Grid values are e/bohr^3, so charge integrals multiply by the
  **bohr^3** volume element.

Always validate the parse against an independent number. The project ships
`result/region_edd_stats.csv`; for the reference system the r = 4 Å sphere about the CO2
must reproduce the independently computed accumulation/depletion values (redacted), and
the whole-box net charge must be ~0. If you get 0.0000 for the sphere, your axes are
transposed.

### 2. Extract the isosurface

```bash
python script/edd_isosurface.py result/EDD_cubes/<SYS>_EDD.cub --iso 0.0003
```

Writes `result/render/meshes/<stem>_<pos|neg>_iso<v>.npz` plus a `_summary.json`
carrying volume, component breakdown and a CO2-placement check.

- Marching cubes: skimage `method="lewiner"`.
- Taubin smoothing (λ=0.50, μ=−0.53, 15 passes). Not plain Laplacian — that shrinks
  the lobe every pass; Taubin holds volume to within ~1.4%.
- Drop connected components below 0.02 Å³ to declutter. Do **not** Gaussian
  pre-smooth the field: it biases lobes smaller (~30% at σ=1.5 grid units).

Isovalue choice, measured on the reference system (looser isovalue → larger lobes;
per-lobe volumes redacted):

| iso (e/bohr³) | note |
|---|---|
| 0.0004 | tighter, more conservative |
| **0.0003** | **project default / user-chosen** |
| 0.0002 | starts eating the ligand region |

### 3. Build the scene in the user's Blender

```bash
# run from the project root; blender_bridge.py lives in script/
python - <<'EOF'
import sys; sys.path.insert(0, 'script')     # blender_bridge.py is under script/
from blender_bridge import Blender
Blender().execute(open('script/build_in_blender.py').read())
EOF
```

`build_in_blender.py` loads `build_blender_scene.py` fresh from disk, calls `build()`,
saves the project `.blend` and the per-figure `.blend`, and prints a report (object
counts, bond cross-check, metal coordination, framing). Override by prepending
globals, e.g. `ISO="0.0004"` or `SAVE_COPIES=[]`.

`build_blender_scene.py` is also the single source of truth for every parameter —
read its `DEFAULTS` and module-level tables rather than hard-coding numbers
elsewhere. It works head-less too (`blender -b -P ... -- --iso 0.0003 --render`) for
batch work.

### 4. Render

Cycles, 1024 samples, OpenImageDenoise, OptiX GPU. The build sets:
`film_transparent=True` + a compositor that lays the render over pure white (this is
what makes the background *exactly* 255), `view_transform="Standard"` (AgX would grey
the white out and mute the palette), a **raking three-point rig** (`LIGHT_POS` /
`LIGHT_SIZE` tables in the builder) so the model is not lit flat-on, a
**top-bright/bottom-dark gradient world** that shapes the metal reflections, no ground
plane, no shadows, and a camera solved so the subject fills 80% with even margins.

Two exposure traps worth knowing before you "fix" a bright render:

- **Keep `exposure` at 0.0.** View-transform exposure is applied *after* the
  compositor, so any non-zero value tints the pure-white background grey. Dim the
  lights and the world instead.
- Metals read as **reflected** light, so a metal sphere's brightness tracks the world.
  A dim world plus a metallic material produces almost-black metal — and no light
  energy fixes it, because a metal has no diffuse term to brighten.

### 5. Verify numerically, then hand it over

Automated vision may be unavailable. Do not conclude "it looks right" — measure it:

```bash
python script/qa_render.py result/render/<file>.png
blender -b result/render/<file>.blend -noaudio \
    -P script/render_masks.py -- --out /tmp/masks --res 1000
python script/analyse_render.py --masks /tmp/masks --beauty result/render/<file>.png
blender -b result/render/<file>.blend -noaudio \
    -P script/audit_bonds.py
```

See `references/verification.md` for what each check proves and which failures are
real versus artefacts of the checker. Then state plainly that the final aesthetic
call is the user's — you have verified geometry and colour, not taste.

## Reference files

- `references/pitfalls.md` — the sixteen traps that actually bit, with measured evidence.
  **Read this before stage 1 or 3.**
- `references/verification.md` — the no-vision verification stack and how to read it.
- `references/environment.md` — Blender 5.2 API specifics, MCP socket workflow, and
  what is missing from Blender's bundled Python.
- The project's own `result/render/README_render.md` — parameters, measured palette,
  QA table, and the scientific caveat about near-nucleus cusp artefacts.
- `result/render/README_series.md` — the five-system series: outputs, uniform
  parameters, QA table, and the measured reason the Co panels are not comparable
  to SYS1 (pitfall 14).

## Reusing for the other systems

The five systems share one coordinate frame and differ only in metal identity, so
everything transfers:

```bash
python script/edd_grid.py        result/EDD_cubes/<SYS>_EDD.cub
python script/edd_isosurface.py  result/EDD_cubes/<SYS>_EDD.cub --iso 0.0003
```

**For the whole series, do not edit `SYS_NAME` / `MESH_STEM` in the builder by hand
and do not build one system at a time.** `script/build_series_in_blender.py` drives
the same builder once per system over the MCP socket (patching the module globals
per run), saves one `.blend` + `_build.json` per structure, and ends on `SYS1` so the
live session is left on the reference panel:

```bash
python - <<'EOF'
import sys; sys.path.insert(0, 'script')
from blender_bridge import Blender
Blender(timeout=7200).execute(open('script/build_series_in_blender.py').read())
EOF
python script/make_montage.py            # labelled 5-panel strip
```

Output layout: `result/render` is flat (the v1 convention), while `result2/render`
splits the images into `figures/` and the `.blend` + `.json` records into
`projects/`. `build_series_in_blender.py` detects which applies (`SPLIT_DIRS`) and
writes accordingly, so a rebuild lands beside the existing outputs; `make_montage.py`
resolves images in `figures/` and build records in `projects/`, falling back to a
flat tree, and names its default output after the dataset (`result2` ->
`EDD_v2_series_montage.png`) so the two generations cannot collide.

The Co colour is already defined, the yellow/blue-violet lobe pair keeps a 157.5°
rendered hue gap against every atom colour, and bond perception needs no changes.

**Camera: one camera for the whole series.** `fit_camera()` is solved once from the
union of all five systems and written into every scene, instead of per panel. The
frameworks are identical and the *lobes* are what differs, so a per-panel fit would
scale the smaller ones up until each filled the frame and would normalise away the
very difference the series exists to show. Measured result: fill 0.800 in all five
panels, margins within 1.9% of each other.

**Render the batch head-less** from the saved `.blend` (`script/render_blend.py`
with `blender -b`), not in the user's GUI session: the modelling hard requirement
covers *modelling*, and batch rendering an already-built scene should not occupy the
session for the length of five renders. Verified equivalent — the same scene
rendered head-less and in the GUI differs by mean |Δ| 0.27 levels with identical
content bbox and background fraction. **Give `--out` an absolute path**: Blender
resolves a relative render output path against the drive root rather than the shell's
cwd, so `--out result2/render/figures/x.png` reports success and writes to
`C:\result2\...` (pitfall 16) — and check the `Saved:` line rather than assuming.

**Before shipping a metal-substituted series, read pitfall 14.** A single isovalue is
only fair if the systems' fields are comparable, and in this project they are not:
the Co cubes carry a near-metal difference residual ~6-17x larger than the entire
adsorption signal. `script/diagnose_metal_blobs.py` and
`script/diagnose_metal_charge.py` measure it per metal site.

## Maintaining this skill

Documentation that drifts from the code is worse than no documentation, so the
project ships two checks. Run both from the project root after editing either the
skill or `build_blender_scene.py`:

```bash
python script/validate_skill.py         # frontmatter, structure, refs resolve, no global leak
python script/validate_skill_claims.py  # every number/path asserted here still matches the code
```

`validate_skill_claims.py` re-derives the isovalue default, all sphere/bond radii,
the lobe hex colours, the bonding factor, the 256-object count and the five-system
cross-check straight from the source, and fails if any disagrees with this file.
Update the test's expected values when you intentionally change a parameter.

This skill lives at `<project>/.agents/skills/edd-blender-figure/` and is
**project-local by design** — it hard-codes this project's paths and is only
meaningful inside this repository. Do not install it into a global skills
directory; if another project needs it, copy the directory into that
project's own `.agents/skills/` and adjust the paths.
