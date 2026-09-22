# Verification without vision

The model running this workflow may have **no image input at all** — `Read` on a PNG
can return the file as an attachment that the provider then strips. Do not interpret
that as "the image looks fine". Substitute measurement, and at the end tell the user
plainly which judgements are still theirs (taste, whether the metal spheres are big
*enough*, whether the lobes suit the story).

Two levels, because one is not enough:

1. **Whole-image QA** (`script/qa_render.py`, system Python + PIL) — background,
   framing, exposure, palette coverage.
2. **Per-object masks** (`script/render_masks.py` inside Blender, then
   `script/analyse_render.py` in system Python) — is each element legible against
   white, is the guest still readable under the translucent lobes.

Plus a structural audit (`script/audit_bonds.py`, inside Blender) for the scene graph
itself, which pixels cannot tell you.

---

## What each check proves

| check | proves | how |
|---|---|---|
| `background_solid_white` | no grey tint, vignette or shadow | border ring + corners are ≥250 on every channel |
| `background_exactly_white_outside_subject` | the white is uniform *behind* the subject too | dilate the subject mask, then require the rest to be pure white |
| `framing` | 80% fill, centred, nothing cropped | computed from the recorded camera projection, not from pixels |
| `exposure` | not blown out, not muddy | luminance percentiles; near-white highlight fraction ≤0.25 |
| `palette_fraction` / `palette_present` | every element and lobe colour actually appears | hue classification of subject pixels against the authored sRGB |
| `all_elements_legible` | nothing vanishes into the white | median per-element distance from white ≥12/255 |
| `co2_legible_under_lobes` | the guest survives lobe coverage | median distance from white, fraction >32 |
| `contact_shading` | the form reads as 3D | fraction of subject pixels in deep shadow ≥2% |
| `audit_bonds.py` | object structure, materials, sharp edges | per-block index assertions, no pixels involved |

**The outline needs its own check, because "no outline" fails silently.** Freestyle
being enabled, with a line set that looks right, can still contribute exactly nothing
(pitfall 9). Do not trust the settings — render the frame twice, Freestyle off and on,
and require the images to differ:

```python
d = np.abs(on.astype(int) - off.astype(int)).max(2)
assert (d > 10).sum() > 1000        # the outline changed a substantial pixel set
```

That is what caught the empty-collection bug: with Freestyle on the images were
byte-identical. Worth re-running after any change to collections or line sets.

The render is split into two view layers (`Framework`, `Lobes` — see pitfall 12), so a
change confined to the outline should also be **confined to the lobe region**. Check the
magnitude away from the lobes as well, otherwise AA/denoiser jitter in a difference map
looks alarming when nothing has actually moved:

```python
# in a window with no lobes at all
d = np.abs(new - old).max(2)
assert d.mean() < 1.0 and np.percentile(d, 95) <= 4
```

Reference measurement for the SYS1 figure at iso 0.0003, outline at 4 px (2400-px
reference) rendered at 1400 px:

| quantity | value |
|---|---|
| pixels changed vs outline-off | 54 444 (single layer) |
| on the framework / inside the lobe silhouette | 51 719 / 2 725 |
| newly drawn lines after the layer split | 2 194 total, 1 525 inside the lobe silhouette |
| mean abs diff in a lobe-free 300×270 window | 0.21 (p95 = 2) |

Reference values for the SYS1 figure (iso 0.0003, raking rig, yellow/violet lobes):
fill 0.80, margins ≈10–12% per side, luminance p5/25/50/75/95 ≈ 31/85/131/184/246,
near-white highlight fraction 0.02, deep-shadow fraction of subject 0.27, subject
coverage ≈13.4%, background border min 254.

Exposure sanity for this rig, measured on the subject pixels only:

| quantity | healthy | meaning |
|---|---|---|
| `min(RGB) ≥ 250` fraction | ≤ ~0.04 | only the metal specular cores clip; non-metals should not clip at all |
| highlight colour | still tinted | a blown highlight that has gone pure white means the base colour is lost |

---

## False failures seen in practice

Knowing these saves a wasted repair loop. Check the checker before "fixing" the render.

**Framing measured from pixels is unreliable.** Thin bonds and faint lobe edges
disappear against white, and a handful of stray pixels at the frame edge wreck the
bounding box. The build records the exact camera projection (`framing` in the build
JSON: NDC extents, margins, `fully_inside`, `centred_ok`); trust that. Use the pixel
silhouette only for a sanity range.

**Background checks trip on anti-aliased silhouette edges.** A few pixels at the
subject boundary are legitimately tinted. Dilate the subject mask before asserting
that the rest is pure white, otherwise you get a spurious failure. (The genuine
version of this problem — a grey wash across the whole frame from a denoised white
*world* background — was fixed by compositing a transparent film over pure white
instead.)

**Emit the masks at the beauty render's resolution, or the dilation must scale.**
`render_masks.py --res` is independent of the beauty pass. A mask at 600 px upscaled
to a 2400 px beauty covers 4×4 beauty pixels per mask pixel, so a fixed 5-px dilation
kernel is far too small and the subject's anti-aliased rim falls outside the grown
mask — `background_exactly_white_outside_subject` then reports ~717 "tinted" pixels
with a max distance of 41 and fails, even though the background is genuinely pure
white. `analyse_render.py` now records `mask_upscale` and scales the kernel with it
(verified: res 600 → kernel 20, res 1000 → kernel 12, both report
`fraction_pure_white = 1.000000`). If you see this failure, check `mask_upscale` in
the output before suspecting the render.

**Element saturation looks diluted if the mask includes edge pixels.** Measuring a
mask with anti-aliased boundary included dropped Zn's apparent saturation to 0.50
versus an authored 0.685; eroding the mask by ~2 px recovers ≈0.58–0.69. When a colour
fidelity number looks off, erode before believing it.

**Do not audit bond materials by nearest-cylinder sampling.** Two traps: a
`cylinder()` emits one **full-length** quad per side, so every side quad's centre sits
at the half's midpoint (a window at 5–40% along the half finds nothing on thin H
bonds); and reading `polygon.center` via `foreach_get("center", ...)` yields garbage.
Use per-block index assertions instead — that is what `audit_bonds.py` does, and it is
exact.

**Blender's bundled Python lacks PIL, scipy and skimage.** Do not write image analysis
inside Blender. Either have Blender *write* masks/PNGs and analyse them in system
Python, or read pixels through `bpy.data.images.load(...).pixels` (linear float —
convert to sRGB before comparing with authored colours).

---

## Triaging a failure

- `FileNotFoundError` for an isosurface mesh → regenerate that isovalue:
  `python script/edd_isosurface.py <cube> --iso <v>`. Object count 254 instead of 256
  means the same thing.
- Object count off by a lot → the builder loaded a stale module; `build_in_blender.py`
  force-reloads, but confirm you edited the file it actually loads.
- Lobe position wrong (far from the guest) → pitfall 1, axis order.
- Charge integral ~0 or 6.75× off → pitfalls 1 and 2.
- Dark rings at bond junctions → pitfall 4.
- Bond hue mismatch at sampled pixels → is the mesh wrong (pitfall 3) or is the
  sampling window wrong (above)? The exact audit distinguishes them.
