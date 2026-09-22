# Example: reference project — molecular framework figures

> **Redacted.** Scientific result values and the specific cluster identity have been
> generalised in this example; drawing and rendering parameters are retained. The five
> members of the substitution series are referred to as `SYS1 … SYS5`.

The reference project this skill was distilled from. It is a worked instance: the rules
and measured numbers in the skill body are dataset-independent, but the *paths, scripts
and outputs* below are this project's, and are what a reader here will actually need.

The isosurface side of the same figures is in
`isosurface-figure/examples/reference-project/`.

---

## What is drawn

A **pentanuclear metal-cluster SBU** with truncated organic ligands, drawn as a
metal-substitution series: five members (`SYS1 … SYS5`) differing in which of the four
peripheral metal sites is substituted (the central site unchanged). In the adsorption
models the cluster additionally carries a single **CO₂ guest** in the binding pocket.

| item | value |
|---|---|
| scene objects | `natoms + nbonds + 2 + 1 + 3` per system — 251 for the bare SBU, 256 with the guest, in the origin project |
| structure files | `structure/translated/<SYS>.mol` (SBU), `<SYS>_CO2.mol` (with guest) |
| guest ordering | **guest first in the mol, last in the cube** — see `isosurface-figure/references/pitfalls.md` §7 |
| render | Cycles/OptiX, 2400², 640 spp, OIDN, Standard view transform, transparent film composited over pure white |

## Scripts

| script | role |
|---|---|
| `script/build_blender_scene.py` | **the builder** — every geometry, material, light, camera and compositing parameter; single source of truth |
| `script/build_in_blender.py` | drives the builder inside the GUI session over the MCP socket, saves `.blend` + build JSON |
| `script/build_series_in_blender.py` | the same, once per system, leaving the session on the reference panel |
| `script/build_mo_in_blender.py` | the builder repointed at orbital cubes (see the isosurface example) |
| `script/blender_bridge.py` | ~60-line TCP client for the blender-mcp addon |
| `script/render_blend.py` | head-less batch render from a saved `.blend` |
| `script/run_edd.py`, `run_mo_renders.sh` | batch drivers |
| `script/qa_render.py` | whole-image QA (background, framing, exposure, palette, contact shading) |
| `script/render_masks.py`, `script/analyse_render.py` | per-object masks and their analysis |
| `script/audit_bonds.py` | structural audit: per-block face indices, materials, sharp-edge counts |
| `script/make_montage.py`, `make_mo_montage.py`, `make_edd_curve_panels.py` | labelled montages and curve panels |
| `script/validate_skill.py`, `validate_skill_claims.py` | the two validators described in `SKILL.md` |

## Parameters actually used

Everything is the skill's default (they were chosen here): ball-and-stick at
`0.44 × Cordero`, one uniform 0.100 Å bond stick, the palette and per-element finishes in
`references/materials-lighting-outline.md`, the raking three-point rig, the gradient world
at 0.38, Freestyle silhouette+contour at `#141414` / 4 px at 2400, camera focal 85 mm
solved to fill 0.800.

## Figure generations

| directory | what | note |
|---|---|---|
| `result/render/` | v1: one system (SYS1), flat layout | the first EDD figure, plus its README/REPORT |
| `result2/render/` | v2: the five-system EDD series, `figures/` + `projects/` split | the near-metal residual was found here (pitfall 5) |
| `result3/render/` | frontier orbitals, **before** CO₂: 18 panels (HOMO+LUMO, α/β) | camera `(0.201, 0.897) Å`, d = 46.27 Å |
| `result3/render2/` | frontier orbitals, **after** CO₂: 9 panels (HOMOs only) | **reuses** the `result3/render` camera, measured pull-back 0% |
| `result3/bader/` | Bader/QTAIM charges for the same systems | the complementary number to the EDD integrals (values redacted) |

The camera reuse is the one decision here worth reading in full
(`references/camera-and-layout.md` §3): the "after" model is the "before" geometry plus a
guest — per-atom displacement below the reuse guard — so the same camera makes the two
rows of a before/after figure pixel-comparable, which is the whole point of drawing them.

## Reproduce

```bash
cd <root>

# build + render one system (modelling in the GUI session)
python - <<'EOF'
import sys; sys.path.insert(0, 'script')
from blender_bridge import Blender
Blender().execute(open('script/build_in_blender.py').read())
EOF
bash script/run_edd.sh                       # or the per-generation driver

# QA, masks, structural audit
python script/qa_render.py result/render/<fig>.png --build-json <fig>_build.json
blender -b <fig>.blend -noaudio -P script/render_masks.py -- \
    --out /tmp/masks --res 1000
python script/analyse_render.py --masks /tmp/masks --beauty result/render/<fig>.png
blender -b <fig>.blend -noaudio -P script/audit_bonds.py

# montage + validators
python script/make_montage.py
python script/validate_skill.py && python script/validate_skill_claims.py
```

## Where the rules came from

Every numbered pitfall in `references/pitfalls.md` names the symptom and the measurement
that pinned it, and each was found by getting it wrong here. Two of the twelve are worth
reading before anything else:

- **§5 (empty collection kills the outline)** — the outline was configured, enabled and
  correct in the UI, and contributed exactly zero pixels, because the line set was scoped
  by a *name lookup* into `bpy.data.collections` and resolved to the previous build's
  empty collection. Byte-identical renders with Freestyle on and off is the signature.
- **§10 (relative render output path)** — a head-less render with a relative `--out`
  reports success, echoes the requested path, creates the requested directory, and writes
  the image to the drive root. Check the `Saved:` line.
