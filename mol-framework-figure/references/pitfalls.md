# Pitfalls — framework and scene

Twelve traps that cost real debugging time in the reference project. Every one is
**silent**: it produces a plausible-looking result rather than an error, so the rule
and the *evidence* both matter. Do not "simplify" these away.

The isosurface-side traps (grid axes, volume elements, lobe colours, extraction) live
in `../isosurface-figure/references/pitfalls.md`.

---

## 1. Bond mesh face indices (scrambled bonds)

**Symptom:** bonds appear to connect the wrong atoms; most bond geometry is misplaced
while a few look correct.

**Cause:** `cylinder()` returns faces with **local** vertex indices. Concatenating
cylinders requires rebasing those indices by the number of vertices already emitted —
and there are **two** places to do it: (1) between cylinders *within* an element block,
(2) between element blocks when concatenating. The first version got (2) wrong by using
`len(bverts)` — the count of *arrays* in the list, not of vertices — and omitted (1)
entirely. `from_pydata` accepts the result without complaint, so most faces silently
bound to the wrong vertices.

**Guard:** `new_mesh(..., expect_all_used=True)` asserts no vertex is left
unreferenced by any face. That check caught the second omission immediately after the
first was fixed; a hand-written audit based on polygon centres had missed both because
the sampling windows were wrong. Keep the guard on any mesh built by manual index
arithmetic.

**Verify with** a geometry-independent audit: because a cylinder is a fixed-size
vertex/face block, assert per block that every face references only that block's
vertices and carries that half's material. Expect 0 stray vertices and 0 wrong
materials.

---

## 2. Blanket smooth shading (dark ring at every junction)

**Symptom:** with "Shade Smooth" the cylinder end caps shade like domes and each bond
junction shows a dark ring. "Shade Auto Smooth" does not.

**Cause:** geometry, not a Blender quirk. A 24-sided cylinder's side facets meet at
only 15°, but the **rim between the side and the end cap is 90°**. Blanket
`use_smooth` averages vertex normals across that rim, so the rim vertex ends up with a
single tilted normal.

**Evidence** (loop normals at one rim vertex, angle to the cylinder axis):

| shading | side loop | cap loop |
|---|---|---|
| blanket smooth | 47.2° | 47.2° |
| angle-based sharp | **90.0°** | **0.0°** |

**Fix:** set `use_smooth` on all polygons, then mark edges sharper than the limit
(~35°) as sharp — on mesh data via bmesh, so it needs no operator context and behaves
identically in GUI and head-less. With the fix each bond carries only its cap rims as
sharp edges (2 halves × 1 cap rim × 24 sides = 48 for a single bond), spheres have 0,
and dark pixels in a bond-only render drop from 51.5% to 41.7%.

---

## 3. AO on metallic atoms darkens the whole sphere

**Symptom:** the metals come out almost black — Zn authored `#4A9EEB` rendering as
roughly `(65,118,168)` — and no amount of extra light energy fixes it.

**Cause:** the fake contact-shadow term (`Ambient Occlusion` node, distance 1.2 Å)
multiplies the **base colour**. On a metal the base colour *is* the reflection tint, so
the node does not shade an occlusion — it just scales the sphere darker. Worse, the
metal spheres are 0.42 Å in radius while the AO distance is 1.2 Å, so nearly every
point on the sphere is inside the occlusion range and the whole ball goes uniformly
dim. Compounding it: a metal has no diffuse component, so its brightness comes
entirely from reflected environment, and a dim world leaves it nothing to reflect.

**Fix:** give the metals `ao = 0.0` and let a bright, *directional* world shape them
(the top-bright/bottom-dark gradient in `materials-lighting-outline.md` §4). The world
can be shaped freely because the film is transparent and the compositor supplies the
background — **the world is a lighting device that never appears in the output.**

---

## 4. View-transform exposure greys the white background

**Symptom:** dimming the render with `view_settings.exposure = -0.25` also drops the
seamless-white background from 255 to about 240, breaking the one requirement the
compositor exists to guarantee.

**Cause:** exposure is applied **after** the compositor, so it scales the composite's
white too.

**Rule:** keep `exposure = 0.0` and dim the scene with the light energies and the
world strength, which act on the render *before* the composite.

---

## 5. A leftover empty collection silently kills the whole outline

**Symptom:** Freestyle is configured, enabled, and its line set looks correct in the
UI, yet the render comes out **byte-identical** to the same render with Freestyle
switched off. No warning, no error.

**Cause:** a stale collection. The scene clear removed objects, meshes, materials,
lights, cameras, worlds and node groups — but **not `bpy.data.collections`**. So on
every rebuild the builder created `SBU_framework` again, Blender renamed the new one to
`SBU_framework.001` (a name clash is resolved by suffixing), and the objects went into
`.001`. The line set, however, was scoped with
`bpy.data.collections["SBU_framework"]` — a *name lookup*, which resolved to the
previous build's collection, now empty. Freestyle dutifully selected no objects.

Measured, by rendering the same small frame per configuration and counting dark pixels
against a Freestyle-off baseline of 3114:

| line-set configuration | dark px | vs baseline |
|---|---|---|
| everything at defaults | 42609 | +39495 |
| silhouette + contour only | 42609 | +39495 |
| + visibility filter | 42609 | +39495 |
| **+ collection scoping** | **3114** | **+0** |

The last row is the bug: the correctly-scoped line set contributes *nothing*, because
the collection it names is the previous build's empty one.

**Fix, two parts — do both:**
1. the scene clear also clears `bpy.data.collections`;
2. pass the collection **object** around (the organizer returns it, the outline setup
   takes it) instead of looking it up by name;
3. assert the membership counts, so a silently empty grouping raises.

**Rule:** any time the deliverable depends on a *name lookup* into `bpy.data`, a
rebuild can silently bind to the previous revision's datablock. Prefer passing the
datablock object, and make the pattern fail loudly.

---

## 6. Interior end-caps make Freestyle draw a seam across every bond

**Symptom:** with the Contour edge type enabled, every bond shows a thin line across
its midpoint, as if each bond were two segments glued together.

**Cause:** a bond is two half-cylinders meeting at the midpoint, and each half was
**capped at both ends**. The midpoint caps are coincident *interior* disks: invisible
to the renderer, but real geometry to Freestyle, whose Contour edge type picks up their
rims.

Isolating it by edge type on the same frame: Silhouette only → clean; Contour only →
seams; both → seams; Border only → nothing (the meshes are closed).

**Fix:** give the cylinder primitive `cap0`/`cap1` switches and build each half open at
the midpoint. The halves abut exactly, so the union is visually unchanged, but there is
no interior disk to detect. Verified by rendering the same crop before/after: only 108
pixels changed at 1000 px, all of them the seams, with no contour lines lost.

**Keep the sharp-edge count in sync:** a half is now `2*SIDES + 1` vertices with **one**
cap rim, so the expected sharp-edge count per single bond is `2*SIDES` (48), and for a
double bond 96 — not 96/192.

---

## 7. Freestyle needs visible render geometry

**Symptom:** a diagnostic render made with every mesh's `hide_render` set shows no
lines at all, which looks like proof that Freestyle is broken.

**Cause:** Freestyle builds its view map from the geometry in the render. Hiding the
geometry removes the input, not the output — this is expected and says nothing about
whether Freestyle works.

**Rule:** to test whether Freestyle draws anything, compare **Freestyle on versus off
on the same visible frame** and count the differing pixels. Do not hide the objects
first.

---

## 8. Freestyle treats translucent overlays as occluders

**Symptom:** the framework loses its outline wherever an isosurface lobe covers it. The
lobes are see-through, so you can plainly see the atoms behind them — but those atoms
have no line, which reads as a hole in the drawing.

**Cause and fix:** in `references/compositing.md` §2–3 (two view layers, per-layer
holdout). Do not try to solve it with the visibility filter or a two-pass composite;
both were measured and both fail.

---

## 9. A missing mesh renders silently

**Symptom:** the figure renders with **no lobes at all** and no error. Easy to miss if
you are checking a QA script rather than the object count.

**Cause:** a cold rebuild of one isovalue deleted the other isovalue's `.npz`, and the
builder treated a missing mesh as a warning (`print` + `continue`).

**Fix:** raise `FileNotFoundError` naming the missing file and the command to
regenerate it. Object count is a cheap tripwire too:
`natoms + nbonds + 2 × n_lobes + camera + lights` — compute it from the inputs; a
count short by exactly the lobes means they are missing.

**Rule:** when a missing input would silently degrade the deliverable to something
that still looks finished, fail loudly.

---

## 10. A relative render output path lands at the drive root, not in the cwd

**Symptom:** a head-less render launched from the project root with a *relative* `--out`
reports success and writes the image somewhere else:

```
render           | Saved: 'C:\result2\render\figures\CO2_ballstick.png'
[render] CO2_ballstick.blend  2400x2400  samples=640  device=GPU  layers=['Framework']
RENDERED result2/render/figures/CO2_ballstick.png          <- the path that was asked for
```

Nothing errors, `RENDERED` even echoes the requested path, and the directory the caller
*did* name is created anyway — which is what disguises it. It surfaces only when the
figure is missing from `figures/` at hand-over time.

**Cause:** Blender resolves a **relative render output path** (`scene.render.filepath`,
which a `--out` CLI override assigns) against the **root of the current drive**. It does
not use the process working directory, and Python *inside the same process* does not
agree with Blender: `os.getcwd()` and `os.path.abspath()` return the correct project
path. Relative paths **do** resolve correctly for `-b <file>.blend` and `-P script/...`
arguments, because Blender's command-line parser resolves those — which is exactly what
makes the split easy to miss: the run demonstrably found its inputs.

**Rule:** pass an **absolute** path for the render output, and read the `Saved:` line
back rather than assuming the directory that was asked for. A saved `.blend` is safe
either way (the builder sets `scene.render.filepath` from an absolute join), so a GUI
`F12` writes where the scene says while a relative CLI override does not — that
asymmetry is the trap.

---

## 11. Text annotation: a wrapped line changes the canvas, a missing glyph draws a box

**Symptom:** a delivered montage's canvas height/width changes after an unrelated edit;
or a caption reads `CO▯` where it should read `CO₂`.

**Cause, part one:** the montage's canvas size is computed from how many lines the
title and footer wrap to, and wrapping depends on the exact strings. Editing a shared
title helper therefore re-lays-out figures that were already delivered.

**Cause, part two:** Arial and Arial Bold have no `U+2082`; PIL draws its
missing-glyph box. Measured: `U+2082` is absent from both `arial.ttf` and `arialbd.ttf`
on this machine (fontTools cmap check), so it affected a delivered footer.

**Fix:** keep dataset-specific text out of the path used by already-delivered figures,
pin the expected canvas size in the verifier, and either check glyph coverage before
using a character or composite the glyph (smaller `2`, lowered baseline, same font).

**Rule:** a figure that was delivered is *pinned*; a shared helper must not change its
pixels as a side effect of serving a new dataset.

---

## 12. Two datasets, one camera — verify before reusing

**Symptom class:** a before/after pair where the two rows are at different zoom levels.
The difference the figure exists to show is then indistinguishable from the change in
framing.

**Fix and the measurement that legitimises it:** reuse the camera only when the two
frameworks coincide, and *measure* that. Worked case: the "after" model was the "before"
geometry plus a guest, per-atom displacement **0.000 Å**, so the same camera put the
framework in the same place at the same size (pull-back 0%, union margins identical to
the single-dataset run). Guard it with a tolerance (~0.05 Å) and fall back to solving a
second camera, recording which happened.

**Related trap in the same operation:** the guest atom ordering differs between the two
input files of an adsorption model — see
`../isosurface-figure/references/pitfalls.md` (guest order) and
`references/geometry.md` §1.
