# Materials, lighting, outline

The goal is a figure whose **form reads without shadow ambiguity**: no ground plane,
no cast shadows, a subject lit so that every sphere shows a bright side and a dark
side, and a palette in which every element is distinguishable from every other
element *and* from any isosurface lobe that will sit on top of it.

All values below are the origin project's, re-checked there against the builder by its
`script/validate_skill_claims.py` claim validator.

---

## 1. Element palette

Authored sRGB. The rendered hue is what matters, and it is *not* the authored hue —
the scene lighting and the metallic finish move it, so any colour decision that
matters (see `../isosurface-figure/references/colour-schemes.md`) is made from isolated
renders, never from the hex value.

| element | hex | rendered hue | note |
|---|---|---|---|
| Zn | `#4A9EEB` | ~206° | bright blue; metallic |
| Co | `#EE74A8` | ~332° | pink; metallic; the substituted site in a series |
| C | `#E6CB94` | ~40° | beige, not grey — a warm neutral that stays out of the way |
| H | `#F0F0F0` | — | white; needs a strong AO term to show form against the white background |
| O | `#DE1A1A` | ~0° | pure red |
| N | `#2020E0` | ~240° | pure blue |

Two constraints this palette imposes on any lobe colour pair: the blues are spent
(Zn ~206, N ~240) and so is red (O ~0). A pair of lobe colours must clear all of
them, which is why the measured pairs are yellow/violet and magenta/green rather
than the conventional green/blue.

## 2. Per-element finish

| element | metallic | roughness | specular | coat | AO |
|---|---|---|---|---|---|
| Zn, Co | **1.0** | 0.22 | 0.5 | 0.15 | **0.0** |
| C | 0.0 | 0.72 | 0.10 | — | fallback |
| H | 0.0 | 0.70 | 0.12 | — | fallback (strongest) |
| O | 0.0 | 0.70 | 0.12 | — | fallback |
| N | 0.0 | 0.70 | 0.12 | — | fallback |

Two entries carry real reasoning:

- **Metals are metallic and matte ligands stay matte.** The metallic sheen is what
  makes a substituted metal the subject of the figure; a matte base under the pale
  ligands keeps the palette legible and stops small atoms competing with the metal
  highlights.
- **Metals carry no AO term (`ao = 0.0`).** The AO node is a *fake contact shadow*
  that multiplies the base colour. On a metal the base colour *is* the reflection
  tint, so the node does not shade an occlusion — it just makes the sphere darker.
  Worse, the metal sphere is 0.42 Å in radius while the AO distance is 1.2 Å, so the
  whole ball sits inside the occlusion range and goes uniformly dark (measured: Zn
  authored `(74,158,235)` rendered `(65,118,168)`, and no light energy fixes it,
  because a metal has no diffuse term to brighten). Metals are shaped by the
  **world** instead. See pitfall 3.

Blender 5.x note: the BSDF input is `Specular IOR Level`, not `Specular`. Probe
`[s.name for s in bsdf.inputs]` rather than assuming.

## 3. Lighting: a raking three-point rig

Three area lights, positioned as **multiples of the subject's half-span about its
centre**, so the rig is scale-independent — the same numbers work for a 10 Å cluster
and a 60 Å one.

| light | position (× half-span) | size (× half-span) | energy |
|---|---|---|---|
| Key | `(-1.90, 1.50, 1.45)` | 1.15 | 7000 W |
| Fill | `(2.20, -1.50, 0.35)` | 2.60 | 1250 W |
| Rim | `(0.55, 1.15, -1.55)` | 1.40 | 3200 W |

- The key is deliberately **raking** — well off to one side and above, not frontal. A
  frontal key lights every sphere from the camera direction, which is exactly the
  flat, shadowless look this rig exists to replace.
- The fill is broad and low-energy: it opens the shadow side without introducing a
  second shadow direction.
- The rim is behind and above: it draws a bright edge around the metal so the
  silhouette separates from the white background.

## 4. World: a lighting device that never appears

A **top-bright / bottom-dark gradient** world at strength 0.38. It acts like a studio
ceiling: each metal gets a bright upper face and a dark underside, and that vertical
falloff is what reads as polished metal. This matters more than it sounds, because
with a metal there is nothing else to shade it.

**No ground plane, no shadows.** A ground plane would put a horizon and a contact
shadow into a figure whose background must be pure white.

The world can be shaped freely because the film renders **transparent** and the
compositor supplies the white background (`references/compositing.md`). That is the
separation to hold onto: the world is lighting, the background is a composite.

## 5. Outline (Freestyle)

| setting | value |
|---|---|
| edge types | **Silhouette + Contour** (Border and Crease off) |
| colour | `#141414` |
| thickness | 2.0 px authored **at 2400 px render width**, rescaled by the actual resolution |
| selection | by **collection** (the framework collection only) |
| isosurfaces | **not** outlined |

- **Why Freestyle at all**: a line set selects by collection, so one render pass can
  treat the framework and the lobes differently *without splitting the pass* — which
  matters because lobes interpenetrate the framework and any pass split would destroy
  the depth ordering between them.
- **Why not the alternatives**: an inverted-hull shell needs one shell on every one of
  ~250 objects and outlines every atom-to-bond junction (far busier than a silhouette)
  while multiplying the ray-traced geometry; compositor edge detection is
  screen-space, so line width is locked to pixels and cannot track camera distance
  (its author reports thickness moving only a couple of pixels before the filter
  breaks); the Line Art modifier emits Grease Pencil strokes, an EEVEE-first feature.
- **Why not on the lobes**: a hard line on a translucent isosurface reads as a solid
  body and fights the "cloud" reading the surface is there to convey.
- **Thickness is authored in pixels, so rescale it by resolution.** Otherwise a
  1100 px preview looks over-inked compared with the 2400 px final.
- Freestyle's line set is scoped by a **name lookup** into `bpy.data.collections`,
  which is how a stale empty collection silently kills the outline entirely
  (pitfall 5). Pass the collection *object* around and assert its membership count.

Freestyle + translucent overlays is the one place this gets subtle: the lobes are
geometric occluders, so framework edges behind a see-through lobe get dropped. The
fix is the two-view-layer split in `references/compositing.md`, which you must apply
whenever this skill is used with an isosurface.

## 6. Render settings

| setting | value |
|---|---|
| engine | Cycles, `compute_device_type="OPTIX"` (enable the OPTIX/CUDA devices) |
| resolution | 2400 × 2400 for the final; 1100–1400 for previews |
| samples | 640 (1024 was the original; 640 is the delivered default) |
| denoiser | OpenImageDenoise |
| view transform | **Standard** — AgX compresses pure white to grey and mutes the palette |
| exposure | **0.0** — non-zero greys the white background (pitfall 4) |
| film | `film_transparent = True`, background supplied by the compositor |

The first render after Blender starts pays a kernel-compile cost (~1 min for a small
test frame); afterwards a 2400²/1024-sample render is on the order of 10 s. Do not
read the first run's wall time as the real cost.
