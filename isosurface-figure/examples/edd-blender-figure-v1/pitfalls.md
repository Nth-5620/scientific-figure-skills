# Pitfalls

Sixteen traps that cost real debugging time in the origin project. Each one is
silent — it produces a plausible-looking result rather than an error — so the rule
and the *evidence* both matter. Do not "simplify" these away.

---

## 1. Cube axis order (wrong lobes, no error)

**Symptom:** lobes render as detached blobs ~8 Å from the guest instead of wrapping
it. Everything else looks fine. Nothing crashes.

**Cause:** the Gaussian cube *specification* says x varies fastest, but these
Multiwfn outputs are stored **z-fastest**. The data must be reshaped to
`(nx, ny, nz)` so `array[ix, iy, iz]` maps index 0→X, 1→Y, 2→Z. Assuming the
spec (reshaping to `(nz, ny, nx)`) transposes X and Z.

**Evidence, two independent checks:**
- The project's own validated integration script (`result/scripts/region_edd_stats.py`)
  reshapes with `data.reshape(dims)` where `dims = [nx, ny, nz]`, and Multiwfn's
  subtraction was separately verified against numpy to 5e-7 e/bohr³.
- Integrating EDD over the r = 4 Å sphere about the CO2 reproduces the independently
  computed accumulation/depletion values (redacted) **only** under z-fastest. The
  x-fastest mapping puts the sphere in empty space and returns exactly `0.0000`.

**Rule:** after parsing, integrate the CO2 sphere and compare against the known
value. A literal zero means transposed axes.

---

## 2. Mixing Å³ and bohr³ volume elements (6.75× error)

Grid values are e/bohr³, so a *charge* integral multiplies by the box volume in
**bohr³**. Geometric quantities (mesh volumes, spacings for marching cubes) use the
**angstrom** spacing. For the 0.145432 bohr grid the ratio is
`(1/0.529177)³ ≈ 6.75`, which is small enough to look like a plausible physical
result rather than a bug.

**Rule:** keep `dv_bohr3` and `dv_ang3` as separate named values; never let a single
`dv` serve both. Square brackets in `edd_grid.py` already do this.

---

## 3. Bond mesh face indices (scrambled bonds)

**Symptom:** bonds appear to connect the wrong atoms; most bond geometry is
misplaced while a few look correct.

**Cause:** `cylinder()` returns faces with **local** vertex indices
(`0 .. 2*sides+1`). Concatenating cylinders requires rebasing those indices by the
number of vertices already emitted — and there are **two** places to do it:
1. between cylinders **within** an element block,
2. between element blocks when concatenating.

The first version got (2) wrong by using `len(bverts)` — the count of *arrays* in
the list, not of vertices — and omitted (1) entirely. `from_pydata` accepts the
result without complaint, so most faces silently bound to the wrong vertices.

**Guard:** `new_mesh(..., expect_all_used=True)` asserts that no vertex is left
unreferenced by any face. That single check caught the second omission immediately
after the first was fixed; a hand-written audit based on polygon centres had missed
both because the sampling windows were wrong. Keep the guard on any mesh built by
manual index arithmetic.

**Verify with** `script/audit_bonds.py`, which is geometry-independent: because a
cylinder is a fixed-size vertex/face block, it asserts per block that every face
references only that block's vertices and carries that half's material. Expect
`0` stray vertices and `0` wrong materials.

---

## 4. Blanket smooth shading (dark ring at every junction)

**Symptom:** with "Shade Smooth" the cylinder end caps shade like domes and each
bond junction shows a dark ring. "Shade Auto Smooth" does not.

**Cause:** geometry, not a Blender quirk. A 24-sided cylinder's side facets meet at
only 15°, but the **rim between the side and the end cap is 90°**. Blanket
`use_smooth` averages vertex normals across that rim, so the rim vertex ends up with
a single tilted normal.

**Evidence** (loop normals at one rim vertex, angle to the cylinder axis):

| shading | side loop | cap loop |
|---|---|---|
| blanket smooth | 47.2° | 47.2° |
| angle-based sharp | **90.0°** | **0.0°** |

47.2° is the dome normal. With the fix each bond carries only its cap rims as sharp
edges (2 halves × 1 cap rim × `sides` = 48 for a single bond; the midpoint end is
open, see pitfall 10), spheres have 0, and dark pixels in a bond-only render drop
from 51.5% to 41.7%.

**Fix:** `apply_angle_smooth(me, 35.0)` in `build_blender_scene.py` — sets
`use_smooth` on all polygons then marks edges sharper than the limit as sharp. Done
directly on mesh data via bmesh so it needs no operator context and behaves the same
in GUI and head-less. `bpy.ops.object.shade_smooth_by_angle` is the operator
equivalent; `smooth_angle` is exposed in `DEFAULTS`.

---

## 5. Lobe colours chosen by hue number instead of by rendered pixels

**Symptom:** a "cyan" negative lobe that looks fine in a colour picker turns out to
be nearly indistinguishable from the Zn atoms on the figure. Separately: two lobes
that are different colours on paper still read as "two greens" in the render.

**Cause:** two compounding errors.
- Measuring hue from a **full-scene** render, where occlusion and the translucent lobe
  over atoms contaminate the mean. Measure from **isolated** renders (hide everything
  else, 1 object visible).
- Comparing hue *differences* in the abstract instead of asking whether the two things
  are **spatially adjacent**. They were: 5.8% of the negative lobe's pixels sit
  directly against a Zn atom in this projection, so the pair must be separable.

**Measured** (isolated renders, scene lighting, initial round):

| candidate | rendered hue | Δhue vs Zn |
|---|---|---|
| cyan `#0FB0CE` | 194° | **12°** — too close |
| teal `#12A79B` | 173° | **33°** — minimum workable |
| violet `#D05BD8` | 283° | 61°, but reads as **pink** |
| indigo `#5A5AD6` | 240° | 34°, but collides with N |

The palette already spends the blues (Zn ~206, N ~240) and red (O ~0), which is why
the free space looked like green/teal rather than the conventional green/blue.

**Then the second symptom showed up.** Green/teal passed the Zn-adjacency test but
still failed the actual goal: the two lobes were only **43.8°** apart as rendered and
read as one colour family. The fix is to score candidate pairs on *both* axes at once
— lobe-to-lobe gap, penalised by proximity to any atom colour:

| pair | rendered hue | lobe gap | vs nearest atom |
|---|---|---|---|
| green / teal (original) | 129.0 / 172.8 | **43.8** | 79 / 36 |
| yellow / cyan (literature pair) | 45.8 / 195.6 | 149.8 | **6 / 13** — both collide |
| lime / steel blue | 81.7 / 199.2 | 117.5 | 42 / **9** — neg hits Zn |
| yellow-green / teal | 83.5 / 175.0 | 91.5 | 44 / 33 |
| **yellow / blue-violet** | 61.8 / 264.3 | **157.5** | 22 / 25 |

The winner is the **complementary** pair (yellow `#DFE22C` / blue-violet `#8A2BE2`),
requested by the author: a complementary partner to a yellow is forced to land near
241–271°, which is where the pure-blue N sits, and 271 is the only end of that window
that clears N at all. The violet also overrides the earlier "no pink/purple" rule,
deliberately and on explicit instruction.

**Rule:** pick lobe colours from isolated-render measurements, score the pair on both
the lobe-to-lobe gap and adjacency to every atom colour, and never ship pink/purple
unless the author asks for it. The sweep is scripted
(`script/lobe_color_sweep.py` drives Blender over the socket; `script/lobe_color_score.py`
scores in system Python) — re-run it rather than arguing from a colour picker.

---

## 6. A glossy "glass" lobe finish that dissolves the shape

**Symptom:** the isosurfaces are supposed to look translucent and jewel-like, but
raising the reflection terms (low roughness + Coat + a little Transmission) spreads a
specular highlight across the entire lobe. The lobes turn into white glare, their
form stops reading, and the guest behind them disappears into the glare.

**Cause:** the lobe is a *smooth, convex, low-curvature* surface, which is the worst
case for specular: the highlight is broad rather than a tight hotspot, so it covers
the lobe instead of describing it.

**Fix:** get the "translucent shell" look from the **alpha falloff**, not from
reflection. Drive Alpha with a Fresnel term so the silhouette is dense (0.97) and the
front-facing area is clear (0.32), and keep reflection weak (Specular IOR Level 0.18,
no Coat, no Transmission). A `fresnel_gain` power below 1 widens the dense rim, since
the raw Fresnel curve only rises very near true grazing and otherwise leaves a
hairline edge.

Fresnel's direction was **verified empirically** rather than assumed
(`script/_fresnel_test.py` renders a sphere with alpha driven directly by
`Fresnel.Fac` and reads the alpha channel back): centre 0.080, rim 0.492 — i.e.
`Fac → 1` at grazing angles, `0` facing the camera. If a future Blender changes that
convention, that test catches it.

---

## 7. AO on metallic atoms darkens the whole sphere

**Symptom:** the metals come out almost black — Zn authored `#4A9EEB` rendering as
roughly `(65,118,168)` — and no amount of extra light energy fixes it.

**Cause:** the material's fake contact-shadow term (`Ambient Occlusion` node,
`Distance = 1.2 Å`) multiplies the **base colour**. On a metal the base colour *is*
the reflection tint, so the node does not shade an occlusion — it just scales the
sphere darker. Worse, the metal spheres are 0.42 Å in radius while the AO distance is
1.2 Å, so nearly every point on the sphere is inside the occlusion range and the whole
ball goes uniformly dim.

Compounding it: a metal has no diffuse component, so its brightness comes entirely
from reflected environment. With `env_strength = 0.10` there was almost nothing to
reflect, which is why the first metallic attempt looked so dark.

**Fix:** give the metals `ao=0.0` (see `FINISH` in the builder) and let a bright,
*directional* world shape them. The world is a top-bright/bottom-dark gradient, which
acts like a studio ceiling and gives each metal a bright upper face and a dark
underside — that vertical falloff is what reads as polished metal. The world can be
shaped freely because the film is transparent and the compositor supplies the white
background; **the world is a lighting device that never appears in the output.**

---

## 8. View-transform exposure greys the white background

**Symptom:** dimming the render with `view_settings.exposure = -0.25` also drops the
seamless-white background from 255 to about 240, breaking the one requirement the
compositor exists to guarantee.

**Cause:** exposure is applied **after** the compositor, so it scales the composite's
white too.

**Rule:** keep `exposure = 0.0` and dim the scene with the light energies and the
world strength, which act on the render *before* the composite. `validate_skill_claims.py`
asserts `DEFAULTS['exposure'] == 0.0` for this reason.

---

## 9. A leftover empty collection silently kills the whole outline

**Symptom:** Freestyle is configured, enabled, and its line set looks correct in the
UI, yet the render comes out **byte-identical** to the same render with Freestyle
switched off. No warning, no error.

**Cause:** a stale collection. `clear_scene()` cleared objects, meshes, materials,
lights, cameras, worlds and node groups — but **not `bpy.data.collections`**. So on
every rebuild the builder created `SBU_framework` again, Blender renamed the new one
to `SBU_framework.001` (a name clash is resolved by suffixing), and the objects went
into `.001`. The line set, however, was scoped with
`bpy.data.collections["SBU_framework"]` — a *name lookup*, which resolved to the
previous build's collection, now empty. Freestyle dutifully selected no objects and
drew nothing.

Measured, by rendering the same small frame per configuration and counting dark
pixels against a Freestyle-off baseline of 3114:

| line-set configuration | dark px | vs baseline |
|---|---|---|
| everything at defaults | 42609 | +39495 |
| silhouette + contour only | 42609 | +39495 |
| + visibility filter | 42609 | +39495 |
| **+ collection scoping** | **3114** | **+0** |

The collection-scoped variants collapse to exactly the baseline, which is what
pointed at the object set rather than the edge-type filters.

**Fix, two parts — do both:**
1. `clear_scene()` also clears `bpy.data.collections`.
2. Pass the collection **object** around (`organize_collections` returns it,
   `add_freestyle_outline` takes it) instead of looking it up by name.
3. `organize_collections` now asserts the membership counts, so a silently empty
   grouping raises instead of producing an outline-less render.

**Rule:** any time the deliverable depends on a *name lookup* into `bpy.data`, a
rebuild can silently bind to the previous revision's datablock. Prefer passing the
datablock object, and make the pattern fail loudly.

---

## 10. Interior end-caps make Freestyle draw a seam across every bond

**Symptom:** with the Contour edge type enabled, every bond shows a thin line across
its midpoint, as if each bond were two segments glued together.

**Cause:** a bond is modelled as two half-cylinders meeting at the bond midpoint, and
each half was **capped at both ends**. The midpoint caps are coincident *interior*
disks: invisible to the renderer, but real geometry to Freestyle, whose Contour edge
type picks up their rims and draws them.

Isolating it by edge type on the same frame: Silhouette only → clean, no seams;
Contour only → seams; both → seams; Border only → nothing (the meshes are closed).

**Fix:** `cylinder()` gained `cap0`/`cap1`, and the bond halves are built open at the
midpoint (`cap1=False` since each half runs atom → midpoint). The two halves abut
exactly, so the union is unchanged visually, but there is no interior disk to detect.
Verified by rendering the same crop before/after: only 108 pixels changed at 1000 px,
all of them the seams, with no contour lines lost.

Side effect to keep in sync: a bond half is now `2*SIDES + 1` vertices with **one**
cap rim, so the expected sharp-edge count per single bond drops from `4*SIDES` (96) to
`2*SIDES` (48), and a double bond from 192 to 96. `audit_bonds.py` and
`validate_skill_claims.py` assert the new values.

---

## 11. Freestyle needs visible render geometry

**Symptom:** a diagnostic render made with every mesh's `hide_render` set shows no
lines at all, which looks like proof that Freestyle is broken.

**Cause:** Freestyle builds its view map from the geometry in the render. Hiding the
geometry removes the input, not the output — this is expected, and says nothing about
whether Freestyle works.

**Rule:** to test whether Freestyle draws anything, compare **Freestyle on versus off
on the same visible frame** and count the differing pixels. Do not hide the objects
first.

---

## 12. Freestyle treats the translucent lobes as occluders

**Symptom:** the framework loses its outline wherever a lobe covers it. The lobes are
see-through, so you can plainly see the atoms behind them -- but those atoms have no
line, which reads as a hole in the drawing.

**Cause:** Freestyle's visibility test is purely geometric. A lobe is a closed surface,
so any framework edge behind it is classified as occluded and dropped, exactly as if an
opaque atom were in front. Freestyle has no per-object "do not occlude" switch.

**Why the obvious fixes fail.** The line set's visibility filter cannot separate the
two cases:

| setting | result |
|---|---|
| `VISIBLE` (the default) | lines behind lobes dropped -- the bug |
| `HIDDEN` / `RANGE` (QI) | also draws lines hidden behind **opaque atoms**, i.e. outlines painted on top of nearer atoms, changing parts that were meant to stay as they were |

Splitting the render into two passes by object does not work either: the lobes
interpenetrate the framework (partly in front, partly behind), so compositing a "lobes"
layer over a "framework" layer destroys the depth order between them.

**Fix -- split by view layer, and use a holdout to preserve depth:**

| view layer | contents | Freestyle |
|---|---|---|
| `Framework` | the framework collection; `EDD_lobes` **excluded** | on |
| `Lobes` | the lobes; `SBU_framework` as a per-layer **holdout** | off |

The lobes are absent from the `Framework` layer, so nothing occludes the framework and
every edge is judged against real, opaque geometry only -- lines hidden behind atoms
stay hidden. In the `Lobes` layer the holdout keeps the framework present for depth (a
lobe is still cut wherever an atom is genuinely in front of it) while contributing no
colour. The compositor alpha-overs the lobes onto the framework, then over white.

`layer_collection.holdout` is what makes this work, and it is a **per-view-layer**
setting, so the framework can be a holdout in one layer and normal in the other. A
global `object.is_holdout` could not express that.

**Cost:** both view layers render at the full sample count, so render time roughly
doubles. Worth knowing before blaming the machine for a slow render.

**Measured on the reference figure** (1400 px): 2 194 pixels turned newly dark, 1 525 of them inside the
lobe silhouette -- the lines that were missing. Away from the lobes the change is below
noise (mean |delta| 0.21, p95 = 2 levels over a 300x270 lobe-free window), confirming
the rest of the figure is untouched.

---

## 13. Missing isosurface meshes render silently

**Symptom:** the figure renders with **no lobes at all** and no error. Easy to miss
if you are checking a QA script rather than the object count.

**Cause:** a cold rebuild of one isovalue deleted the other isovalue's `.npz`, and the
builder treated a missing mesh as a warning (`print` + `continue`).

**Fix:** the builder now raises `FileNotFoundError` naming the missing file and the
command to regenerate it. Object count is a cheap tripwire too: the complete scene is
`natoms + nbonds + 2 lobes + 1 camera + 3 lights` objects — compute it from the inputs;
a count short by exactly the two lobes means they are missing.

**Rule:** when a missing input would silently degrade the deliverable to something
that still looks finished, fail loudly.

---

## 14. A near-metal difference residual that makes a substituted series incomparable

**Symptom:** at one uniform isovalue the metal-substituted panels render with lobes
an order of magnitude larger than the closed-shell reference, wrapped around the
substituted metals. Measured at iso = 0.0003 across the reference series, the positive
lobe volume grew severalfold from the closed-shell member to the fully substituted one,
and the lobe's share of the rendered subject grew with it (values redacted). The
closed-shell panel looks correct; the substituted panels look like their metals are
accumulating enormous charge. Worse, the effect switches on and off per site — so the
strip reads as a chemistry that depends on which atom you point at, which is the
opposite of what a substitution series should show.

**Cause:** the EDD is a difference of two *independent* SCF calculations (the complex
and that system's SBU), so each metal sits on both sides of the subtraction and its
own density must cancel. For a closed-shell d10 Zn it does: the two runs reproduce
each other's near-nucleus density to ~1e-4 e/bohr^3. For the open-shell UKS + DFT+U
(Mulliken projection) systems it does not, at a subset of sites, by ~three orders of
magnitude. A "the |EDD| extremes sit at ~0.2 A from the nucleus and therefore do not
affect visualisation" note is **wrong** — the affected region is far larger than the
extreme-value voxels.

**Evidence, all measured on this project's cubes (values redacted; the *shapes* of the
measurements are the transferable part):**

| measurement | healthy site | affected site |
|---|---|---|
| whole-grid extremes | small, comparable to the closed-shell reference | orders of magnitude larger |
| contiguous above-iso radius from the nucleus | 0 A | **~2 A** (the measurement limit) in nearly all probe directions — far beyond the rendered ball |
| accumulation within r = 2.0 A | ~0.002-0.01 e | ~two orders of magnitude larger, net ~0 (net/acc = a few %) |
| <|EDD|> radial peak | ~1e-4 e/bohr^3 | orders of magnitude larger, at the same radius |
| angular power, l = 2 share | -- | small; the pattern is **l = 4 (cubic) dominated**, l = 0 and odd l ~ 0 |

The residual hugs the nucleus and decays by orders of magnitude well inside the
coordination shell at ~2 A. It is a pure rearrangement (accumulation = depletion), and
it is *not* a d-orbital shape -- so it is not an orbital re-occupation either, but a
mismatch in how the two runs represent the near-nucleus density, with the angular
signature of the cubic grid. The rendered metal ball is only 0.537 A in radius, so an
above-isovalue region reaching ~2 A is a plainly visible false lobe on the metal.

Confirmed by several independent checks, and the cheap fix ruled out:

- **Which sites:** the contiguous-radius and per-metal-integral diagnostics agree
  exactly, and the residual pattern at a given site is *identical* across systems
  (same radius, same magnitude) — which is what a property of the calculation pair
  looks like, not noise. (Affected-site lists redacted.)
- **The residual comes in tiers, and the weak tier is invisible.** At the strongly
  affected sites the radial peak is orders of magnitude above the closed-shell level;
  at the remaining sites it is much weaker, yet still above it. At iso = 0.0003 the
  weak tier *is* still above the isovalue, but the resulting shell hugs the nucleus
  and barely pokes outside the 0.537 A rendered metal ball, so it is invisible in the
  figure. Do not describe those sites as "clean" without checking: a low isovalue can
  wrap a site that carries almost no charge.
- **Not the coordination:** the strongly and weakly affected sites share the same
  coordination type, and their distance to the CO2 does not explain the split, so the
  strong/weak pattern cannot be read off the local chemistry — and no closed-shell
  site ever shows it at all.
- **The project's own CDC curves** are a cheap cross-check: along the adsorption axis
  all systems stay comparable, while perpendicular to it the affected systems diverge
  by orders of magnitude. The excess sits off the adsorption axis, at the metals.
- **The genuine interface signal is comparable across the series.** Integrating the
  CO2 sphere *with the metal neighbourhoods (r < 2.4 A) removed* — the same geometric
  window for every system — gives a smooth monotonic trend across the series, net ~ 0
  (`script/analyse_site_residuals.py`). The raw CO2-sphere numbers spread severalfold
  only because they silently include part of each metal's coordination sphere.
  (Values redacted.)
- **Masking does not fix it.** Masking only the metals at a radius chosen deliberately
  below the 0.537 A rendered ball so the mask boundary itself cannot appear removes
  just **1-2%** of the lobe volume (`script/test_metal_mask.py`).

**Rule:** before building a metal-substituted series, check each system's extremes
against the closed-shell reference, then run `script/diagnose_metal_blobs.py`,
`script/diagnose_metal_charge.py`, `script/analyse_site_residuals.py` and
`script/verify_angular_raw.py` per site. If affected sites exist, a uniform isovalue
does **not** make the panels comparable: say so plainly, and quote the metal-excluded
interfacial integral as the number that *is* comparable. Fix the residual at the
source (make the complex and the SBU converge to the same solution, e.g. seed the
complex from the SBU wavefunction, and check that the per-site residual drops to the
1e-3 e level). Do not report the extra lobe volume as metal-dependent charge
transfer, do not mask it away, and do not silently switch to per-panel isovalues —
an order-of-magnitude spread in the isovalue needed to match lobe sizes is itself
the finding.

---

## 15. A re-exported cube reorders the atom block, so `atoms[:3]` stops being the CO2

**Symptom:** a quantity that was correct in the first generation of a dataset comes out
*plausible but wrong* in the second. Here: the CO2-sphere integral for the reference
system — a system that barely changes between generations — appeared to drop
severalfold, and the drop was written up in a results README as evidence of a
methodological fix. Nothing raised an error anywhere. (Values redacted.)

**Cause:** the two generations hold the same atoms in the same absolute frame but in a
different ORDER, so any positional assumption silently points at different chemistry:

| | v1 cube | v2 cube |
|---|---|---|
| CO2 atoms | the head of the list | **the tail (last three)** |
| the five metals | scattered mid-list | **different positions** |

**Evidence:** with the true CO2 centre the 4 A sphere gives the correct value; centred
instead on the mean of the cube's first three atoms (a framework C/H/C trio) it gives a
*plausible but wrong* number, matching the (wrong) published CSV to five digits
(values redacted). A sphere around a framework carbon returns a number of the
same order as the real one, which is why nothing looked wrong. This bit twice: the
upstream script (which falls back to "the first three atoms are the CO2" when its
geometric test does not fire) and this project's own analyser, which used `atoms[:3]`
directly.

**Fix:** never index. `script/edd_isosurface.py` now provides

```python
mol_atom_map(meta)   # cube atom i -> its 1-based index in the mol file (element+coords)
locate_co2(meta)     # CO2 centre in angstrom, order-independent
```

`mol_atom_map` matches on element plus coordinates within 0.05 A, so it also *validates*
that the cube and the mol really share a frame (every atom resolves, or it reports how
many did not). `locate_co2` falls back to the geometry signature (a carbon carrying
exactly two oxygens at ~1.16 A) rather than to a layout, and `co2_placement_check` uses
it, so the per-lobe placement statistics stay correct for either ordering. Per-site
outputs are keyed by the **mol** index (per metal site, e.g. Zn#/Co#) so site labels
mean the same thing across generations.

**Rule:** recomputing or re-exporting a dataset is a reason to re-validate every
positional assumption about it, not just its numbers -- and a fallback that assumes an
input layout should fail loudly, because it will otherwise keep returning a
*different* answer with the same units.

---

## 16. A relative render output path lands at the drive root, not in the cwd

**Symptom:** a head-less render launched from the project root with a *relative* `--out`
reports success and writes the image somewhere else (log verbatim, with the last line
added by `render_blend.py` itself):

```
render           | Saved: 'C:\result2\render\figures\CO2_ballstick.png'
[render] outline thickness at 2400 px: 4.00 px
[render] CO2_ballstick.blend  2400x2400  samples=640  device=GPU  layers=['Framework']
RENDERED result2/render/figures/CO2_ballstick.png                 <- the path that was asked for
```

Nothing errors, `RENDERED` even echoes the requested path, and the directory the caller
*did* name was created anyway -- which is what disguises it. It surfaces only when the
figure is missing from `figures/` at hand-over time.

**Cause:** Blender resolves a **relative render output path** (`scene.render.filepath`,
which `render_blend.py --out` assigns) against the root of the current drive. It does not
use the process working directory, and Python *inside the same process* does not agree
with Blender: `os.getcwd()` and `os.path.abspath()` return the correct project path.
Relative paths **do** resolve correctly for the `-b <file>.blend` and `-P script/...`
arguments, because Blender's command-line parser resolves those -- which is exactly what
makes the split easy to miss: the run demonstrably found its inputs. `render_blend.py`
even calls `os.makedirs(os.path.dirname(os.path.abspath(args.out)))`, so the *correct*
directory is created by the caller and the wrong one by Blender's writer.

**Evidence** -- one process, Blender 5.2.1, launched from the project root. Verbatim, with
the annotations after `<-` added here; note that `os.path.abspath` of the very same
relative path is the *correct* destination, and that it is still empty afterwards:

```
render           | Saved: 'C:\result2\render\figures\_relpath_test.png'
TEST cwd            = '<project-root>'          <- correct
TEST abspath(rel)   = '<project-root>\result2\render\figures\_relpath_test.png'
TEST exists(abspath)= False      <- checked before the render, so not yet informative
TEST exists(C:root) = True       <- where the render actually went
TEST exists(proj)   = False      <- the abspath stayed empty
```

The transparent-background CO2 figure hit this first: the 2400 px render landed in
`C:\result2\...` and had to be re-run with an absolute `--out`.

**Rule:** pass `render_blend.py` (and any `-o` / `--render-output` override) an **absolute**
path, and read the `Saved:` line back rather than assuming the directory that was asked
for. The saved `.blend` is safe either way -- `build_blender_scene.py` sets `r.filepath`
from `os.path.join(ROOT, ...)` -- so a GUI `F12` writes where the scene says while a
relative CLI override does not; that asymmetry is the trap.
