---
name: mol-framework-figure
description: Build publication-quality 3D molecular framework figures in Blender — every atom a sphere, every bond a stick, per-element palette with metallic metals, raking three-point rig, seamless pure-white background, Freestyle outline solved to survive under translucent overlays, one camera per series, rendered with Cycles. Use whenever the user wants a molecule, SBU, cluster, MOF fragment or a small-molecule guest rendered as a ball-and-stick or tube model, mentions 球棍模型, 分子框架, 骨架/结构渲染, 团簇模型, 金属取代系列, molecular visualization, ball-and-stick, tube style, or needs the opaque structural panel of a figure that will also carry isosurfaces (差分电荷密度/EDD, 轨道, IGMH). Also use when such a render already exists and needs its outline, lighting, camera, framing or QA fixed.
---

# Molecular framework figures in Blender

Produce the **opaque** half of a molecular figure: the atom framework in global
ball-and-stick (the default) or tube style, on a seamless white background, with a
crisp outline that stays correct wherever a translucent isosurface overlaps it.

The translucent half — electron density difference, orbitals, weak-interaction
surfaces — is a separate skill, `../isosurface-figure/`. The two share one scene: this
skill owns everything that is *not* an isosurface (geometry, palette, materials,
lighting, camera, outline, compositing, render, QA), and the isosurface skill adds
lobes into that scene and owns their extraction, isovalue, colour and material.
**Read this skill first when building a scene from scratch, then hand the scene to
the isosurface skill.**

**Provenance.** This skill, `../isosurface-figure/` and `../curve-panel-figures/` were
distilled from the author's research-figure workflow; a redacted worked instance lives
in `examples/`. Every rule and measured number
in them is a property of Blender and of the drawing conventions and transfers to any
project; project-specific scripts (scene builder, bridge, QA, audits,
validators) and output trees do not. In a project that has those scripts, reuse
them; in one that does not, build the equivalent from the recipes in `references/`.

## The deliverable

One square PNG per panel, the subject centred and filling ~80% of the frame, pure
white background (border min ≥ 254/255), the framework outlined in `#141414`, plus:

- a `.blend` project per figure, saved in the user's **open GUI Blender session**;
- a `*_build.json` build record next to it: object counts, bond perception and its
  cross-check, per-element radii actually used, camera position and the solved
  framing, render settings;
- optionally a labelled montage strips the panels together.

The build record is not bureaucracy: it is what makes the figure auditable without
looking at it (see `references/verification.md`), and the camera/framing numbers in
it are the only trustworthy statement of the framing (pixels are not).

## House rules

These were chosen by the author of the reference project after comparing the
alternatives, and every later round inherited them. Treat them as the default and
deviate only on explicit instruction.

1. **Model through blender-mcp, in the running GUI Blender.** Modelling must land in
   the session the user has open, so they can see and hand-tweak it, and the `.blend`
   must be saved. Head-less (`blender -b`) is for batch re-computation and
   verification only — never as the way the deliverable is modelled.
2. **Derive the bonding yourself, or cross-check the source that provides it.** Do
   not assume a `.mol`/structure file is authoritative: perceive bonds geometrically
   (`d <= 1.25 * (r_i + r_j)`, Cordero radii) and compare against the file's bond
   block, reporting any disagreement in the build record. See `references/geometry.md`.
3. **Every atom and every bond is its own object**, sharing one material per
   element. Do not merge atoms of the same element into one mesh, and do not merge
   all bonds into one mesh — per-object structure is what lets the user re-colour or
   move one atom in the session.
4. **Ball-and-stick is the global default.** Every atom is a sphere and every bond a
   thin stick, radii from one rule (`r = 0.44 × Cordero`) so the size ordering stays
   physical (H smallest, metals largest). Metals read as the subject by **size**, not
   by a style difference. The tube alternative exists and is documented; it costs the
   size legibility, so prefer ball-and-stick unless asked.
5. **Surface finish is per element**: metals metallic (Metallic 1, low roughness,
   **no AO term** — see pitfall 3), ligands matte.
6. **Shade with angle-based sharp edges** ("Shade Auto Smooth", ~35°), never blanket
   smooth shading — blanket smoothing puts a dark ring at every bond junction.
7. **The framework is outlined; isosurfaces are not, and the outline must survive
   *under* a translucent overlay.** Freestyle, selecting by collection, inside a
   render split into two view layers with the framework as a per-layer holdout. One
   layer cannot do it. See `references/compositing.md`.
8. **Keep `exposure` at 0.0** and shape the light with light energies and the world
   strength. Exposure is applied after the compositor, so it greys the white
   background (pitfall 4).
9. **One camera for a whole series**, solved once and written into every scene — not
   fitted per panel, and reused across datasets when the frameworks coincide
   (`references/camera-and-layout.md`).
10. **Give every render an absolute `--out` path** and read the `Saved:` line back.
    A relative output path lands at the drive root (pitfall 10).

## Workflow

### 1. Get the structure in one coordinate frame

You need, for each atom: element symbol and XYZ in Å. Two sources must agree if you
use two of them (e.g. a `.mol` for bonds and a `.cub` for coordinates) — assert that
they list the same atoms in the same order, element for element, within 0.01 Å, and
fail loudly otherwise. **Never index into an atom list positionally**
(`../isosurface-figure/references/pitfalls.md` §6).
`references/geometry.md` has the parser contract and the two files that must be
reconciled in an adsorption model.

### 2. Perceive bonds, build one object per atom and bond

The origin project's builder is a single parameterised scene-builder script fed to
the GUI session. In a project that ships it, drive it as documented there; otherwise
write the builder against `references/geometry.md` (the perception rule, the radius
tables, the open-ended bond halves, the `expect_all_used=True` mesh-index guard) and
feed it to the session — through the blender-mcp MCP tools (`execute_blender_code`,
`get_scene_info`, `get_viewport_screenshot`) or over the raw socket:

```python
# raw-socket fallback; the addon protocol is in references/environment.md
from blender_bridge import Blender          # a ~60-line TCP client
Blender().execute(open('build_in_blender.py').read())
```

In every case: one object per atom and per bond, materials from
`references/materials-lighting-outline.md`, and the build record filled from the
builder's own output — never hand-typed.

### 3. Materials, lighting, outline

`references/materials-lighting-outline.md` holds the element palette with its
measured rendered hues, the per-element finish table, the raking three-point rig
expressed in units of the subject's half-span, the gradient world, and the Freestyle
line set. `references/compositing.md` holds the two-view-layer split, the holdout
that preserves depth order, and the alpha-over-white compositor.

### 4. Camera, then render

`references/camera-and-layout.md`: solve the camera once for the whole series,
record the projection in every build record, and render the batch head-less from the
saved `.blend`.

### 5. Verify numerically, then hand it over

Automated vision may be unavailable, so "it looks right" is not a measurement.
`references/verification.md` is the stack: `qa_render.py`-style whole-image checks,
per-object masks, `audit_bonds.py`-style structural assertions, and the
render-twice outline difference test. Then state plainly that the aesthetic call is
still the user's — you have verified geometry and colour, not taste.

## Reference files

| file | read it when |
|---|---|
| `references/geometry.md` | parsing structures, perceiving bonds, radii, bond/atom meshes |
| `references/materials-lighting-outline.md` | the palette, finishes, lights, world, Freestyle |
| `references/compositing.md` | overlays, view layers, holdout, white background |
| `references/camera-and-layout.md` | framing, series cameras, montages, annotation text |
| `references/pitfalls.md` | **before** touching stage 2 or 3 — 12 measured traps |
| `references/verification.md` | before claiming a render is correct |
| `references/environment.md` | Blender/MCP versions, Blender 5.x API changes, paths |
| `examples/reference-project/` | a worked instance of this whole skill |

## Maintaining this skill

Installed globally at `~/.agents/skills/mol-framework-figure/`, alongside its
siblings `../isosurface-figure/` and `../curve-panel-figures/`; cross-references
between them are `../<skill>/references/...` paths.

The origin project ships two validators, run from that project root
after editing either skill or the scene builder:

```bash
python script/validate_skill.py         # frontmatter, structure, refs resolve, no global leak
python script/validate_skill_claims.py  # every number/path asserted still matches the code
```

`validate_skill_claims.py` re-derives the radii, finishes, outline settings and
object counts straight from the builder source and fails on any disagreement.
Outside that project there is no such validator, so the discipline is manual: when
you intentionally change a parameter, update the matching line here **and** in any
builder you wrote, in the same edit — a skill whose numbers drift from the code is
worse than none.
