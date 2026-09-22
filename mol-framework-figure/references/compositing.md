# Compositing: pure white background, and an outline that survives an overlay

Two jobs, both of which fail silently if done the obvious way:

1. the background must be **exactly 255 white** everywhere outside the subject;
2. the framework outline must survive **under a translucent isosurface** that
   interpenetrates it.

---

## 1. Transparent film, then alpha-over white

Do **not** paint a white world and use it as the background: a denoiser will leave a
grey wash across it. Instead:

- render with `film_transparent = True` (alpha channel on),
- composite in the compositor: `AlphaOver(Background = pure white,
  Foreground = Render Layers)`.

Blender 5.x specifics (`references/environment.md` has the full API list):

```python
ng = bpy.data.node_groups.new("EDD_Composite", "CompositorNodeTree")
scene.compositing_node_group = ng          # scene.node_tree does not exist in 5.x
rl   = ng.nodes.new("CompositorNodeRLayers")
over = ng.nodes.new("CompositorNodeAlphaOver")   # sockets: Background, Foreground
ng.interface.new_socket("Image", in_out="OUTPUT", socket_type="NodeSocketColor")
out  = ng.nodes.new("NodeGroupOutput")           # NOT CompositorNodeComposite
```

Index `AlphaOver`'s sockets **by name**: `inputs[1]` is the foreground, not the
background. And keep `view_transform = "Standard"` + `exposure = 0.0`, or the white
lands somewhere near 240 (pitfall 4).

Measured result: border minimum 254/255 on every panel at 2400 px (Blender's 8-bit
dither is ±1 level), with zero pixels below 250.

## 2. The overlay problem

Freestyle's visibility test is **purely geometric**. A lobe is a closed surface, so
every framework edge behind it is classified as occluded and dropped — exactly as if
an opaque atom were in front. The lobes are see-through, so you can plainly see the
atoms behind them, and those atoms have **no line**, which reads as a hole in the
drawing.

The obvious fixes all fail, and it is worth knowing why before spending a day on them:

| attempt | result |
|---|---|
| line set visibility filter = `VISIBLE` (default) | lines behind lobes dropped — the bug |
| visibility filter = `HIDDEN` / `RANGE` (QI) | also draws lines hidden behind **opaque atoms**, painting outlines on top of nearer atoms, i.e. it changes the parts that were already correct |
| split into two **render passes** by object and composite | the lobes interpenetrate the framework (partly in front, partly behind), so compositing a "lobes" image over a "framework" image destroys the depth order between them |
| global `object.is_holdout` on the framework | not expressible: the framework must be a holdout in one layer and normal in the other |

## 3. The fix: split by *view layer*, hold the framework out of the lobe layer

| view layer | contents | Freestyle |
|---|---|---|
| `Framework` | the framework collection; the lobe collection **excluded** | **on** |
| `Lobes` | the lobes; the framework collection as a **per-layer holdout** | off |

- In `Framework`, the lobes are absent, so nothing occludes the framework and every
  edge is judged against real opaque geometry only — lines hidden behind atoms stay
  hidden, which is the whole point.
- In `Lobes`, the holdout keeps the framework present **for depth** (a lobe is still
  cut wherever an atom is genuinely in front of it) while contributing **no colour**.
- The compositor alpha-overs the lobe layer onto the framework layer, then the
  result over white.

`layer_collection.holdout` is a **per-view-layer** setting, which is what makes this
expressible at all.

**Cost:** both view layers render at the full sample count, so render time roughly
doubles. Worth knowing before blaming the machine for a slow render.

**Measured on the reference figure** (1400 px, outline at 4 px-equivalent): 2 194
pixels turned newly dark relative to the single-layer render, 1 525 of them inside the
lobe silhouette — the lines that were missing. Away from the lobes the change is below
noise (mean |Δ| 0.21, p95 = 2 levels over a 300×270 lobe-free window), which is the
check that confirms the rest of the figure was untouched.

## 4. Verifying it (both halves)

Neither half is visible in a passing glance, so measure both:

```python
# the outline contributes something at all: render twice, Freestyle off and on
d = np.abs(on.astype(int) - off.astype(int)).max(2)
assert (d > 10).sum() > 1000

# and the change is confined to the lobe region: away from the lobes, nothing moved
d = np.abs(new.astype(int) - old.astype(int)).max(2)     # lobe-free window only
assert d.mean() < 1.0 and np.percentile(d, 95) <= 4
```

Render the outline with the correct **absolute** `--out` path and read the `Saved:`
line back (pitfall 10).
