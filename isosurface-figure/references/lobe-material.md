# The lobe material: a Fresnel-alpha shell

The isosurface must read as a translucent electron cloud: you can see the atoms behind
it, its own shape is legible, and its colour is unmistakable. Every naive way to get
that look fails in a specific, measured way.

**Scope: single-colour lobes** (EDD, orbitals — colour = identity). For a
*value-mapped* surface (ESP-type, colour = a number) the measured answer is the
opposite — uniform alpha, front lighting, separate-layer compositing — see
`value-mapped-surfaces.md`.

---

## 1. Drive **alpha** with Fresnel; keep reflection weak

```python
LOBE_FINISH = dict(roughness=0.38, specular=0.18,        # Specular IOR Level in 5.x
                   alpha_center=0.32, alpha_edge=0.97,
                   fresnel_ior=1.35, fresnel_gain=0.35,
                   coat=0.0, transmission=0.0)
```

- Alpha is driven by a Fresnel term through a Map Range: **0.32 where the surface faces
  the camera, 0.97 at grazing angles**. The front-facing area stays clear (atoms visible
  through it) while the silhouette is dense and saturated — the silhouette carries the
  colour and the shape, which is how a translucent molecular surface is conventionally
  read.
- **A uniform alpha cannot do this.** It either obscures what is behind the lobe or
  washes out the silhouette; there is no single value that does both jobs.
- `fresnel_gain = 0.35` (a power below 1) **widens the dense rim**, because the raw
  Fresnel curve only rises very near true grazing and otherwise leaves a hairline edge.
- Reflection terms are deliberately weak: low Specular IOR Level, **no Coat, no
  Transmission**.

## 2. Why not a glossy "glass" finish

Raising the reflection terms (low roughness + Coat + a little Transmission) to look
"jewel-like" spreads a specular highlight across the entire lobe. The lobes turn into
white glare, their form stops reading, and a guest behind them disappears into the glare.

The cause is shape, not material taste: a lobe is a **smooth, convex, low-curvature**
surface, which is the worst case for specular — the highlight is broad rather than a
tight hotspot, so it covers the lobe instead of describing it. The lobe reads as glossy
through the **tight edge falloff**, not through a mirror finish.

## 3. Verify the Fresnel direction empirically

Blender's Fresnel node convention is an implementation detail that could change. It was
verified rather than assumed: render a sphere with alpha driven directly by
`Fresnel.Fac`, read the alpha channel back → **centre 0.080, rim 0.492**, i.e. `Fac → 1`
at grazing angles and `0` facing the camera. If a future Blender changes that, this test
catches it before the figure does.

## 4. Lobes are not outlined

The framework gets a Freestyle outline; the isosurfaces deliberately do not. A hard line
on a translucent volume reads as a solid body and fights the "electron cloud" reading the
surface exists to convey.

That asymmetry is exactly what forces the **two-view-layer** render: Freestyle's
visibility test is geometric, so a translucent lobe occludes the framework's lines behind
it and the framework loses its outline wherever a lobe covers it. The fix — a framework
view layer with the lobes excluded, and a lobes view layer with the framework as a
per-layer holdout to preserve depth — is in
`../mol-framework-figure/references/compositing.md`. Read it before adding lobes to a scene
for the first time.

## 5. Object structure

- **One object per lobe** (`MO_pos`, `MO_neg`, `EDD_pos`, `EDD_neg`, …), never one merged
  mesh: the two signs need separate materials, and the user may want to hide or recolour
  one of them in the session.
- Lobes go into their **own collection**, separate from the framework collection, because
  the view-layer split and the Freestyle line set both select by collection.
- Name the lobe objects so the sign is obvious in the outliner. The build record should
  list, per lobe: object name, vertex/face count, volume, and the material colour.
- Mesh from the extractor: `verts` float32, `faces` int32, plus the JSON summary
  (`extraction.md` §4). Keep the mesh coordinates in the **same frame as the atoms** —
  subtract the same origin the builder subtracts from the atoms, or the lobes will float
  in the right shape at the wrong place, which no error message will tell you.

## 6. Occlusion and depth order

Because the lobes interpenetrate the framework (partly in front of the atoms, partly
behind), the depth order between them must be preserved by a **single ray-traced pass**
with the framework as a holdout in the lobe layer — not by compositing two rendered
images. A two-image composite destroys exactly the interpenetration that makes the figure
truthful about where the electron density sits.

## 7. Sanity numbers

For the reference lobes at 0.0003 e/bohr³, a lobe spans 2–5 Å³ of volume and lands within
~0.05–0.1 Å of a metal nucleus when the field is metal-localised. A "lobe" of tens of Å³,
or one whose minimum distance to the relevant fragment is several Å, is a sign the
isovalue is wrong or the field is not what you think it is — check the summary before
rendering.
