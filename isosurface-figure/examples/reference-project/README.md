# Example: reference project — isosurface figures

> **Redacted.** Scientific result values (energies, charges, integrals, per-system
> trends) have been removed from this example; drawing and rendering parameters are
> retained. The five members of the substitution series are referred to as
> `SYS1 … SYS5`.

Two surface families have been drawn in the reference project, and they are a good
illustration of the split this skill insists on: same cluster, same scene, same render
settings — different **field**, different **isovalue**, different **colour pair**, and
therefore different caption claims.

The framework side of the same figures is in
`mol-framework-figure/examples/reference-project/`.

---

## Family 1 — EDD (density difference), `result2/render/`

| item | value |
|---|---|
| field | `Δρ = ρ(complex) − ρ(SBU) − ρ(CO₂)`, fragment subtraction, from GTH valence density cubes |
| cube size | 320³ (Multiwfn output, z-fastest) |
| isovalue | **0.0003 e/bohr³** (project default; sweep guidance in `references/surface-types.md` §1) |
| colours | yellow `#DFE22C` = accumulation, blue-violet `#8A2BE2` = depletion |
| lobes | one mesh per sign, `min_volume = 0.02 Å³`, Taubin 15 passes |
| panels | five systems (SYS1 … SYS5), one `.blend` + build JSON each, plus a montage |

**Calibration / validation.** Integrating the difference field over an r = 4 Å sphere
about the guest must reproduce the independently computed accumulation/depletion values
(redacted), and the whole-box net charge must be ~0. That integral is also
the axis-order check: the transposed mapping returns exactly `0.0000`
(`references/pitfalls.md` §1).

**The finding that shaped the rules.** The five-system series had to ship with a caveat:
at one uniform isovalue the substituted panels' lobes were several times larger than the
closed-shell reference's, wrapped around a subset of the substituted metal sites, because
the complex and its SBU failed to cancel each other's near-nucleus density at those sites
(their field extremes ran orders of magnitude above the reference's). Full evidence in
`references/pitfalls.md` §5 and `references/surface-types.md` §1. The comparable number
turned out to be a **metal-excluded interfacial integral** — the same geometric window
for every system, with the metal neighbourhoods removed — which formed a smooth
monotonic trend across the series; the raw guest-sphere integrals, by contrast, spread
severalfold only because they silently included part of each metal's coordination
sphere. (Values redacted.)

Its complementary numbers: Bader/QTAIM charges (`result3/bader/`) — the redacted values
showed the net guest-related transfer to be essentially zero, i.e. the interaction does
not rearrange charge between guest and framework.

## Family 2 — frontier orbitals, `result3/render/` and `result3/render2/`

| item | value |
|---|---|
| field | CP2K `&MO_CUBES` wavefunction amplitude, a.u. (**not** a density) |
| cube size | 160³ (`STRIDE 2`) |
| isovalue | **0.02 a.u.** (the reference paper for the same chemistry used 0.015 a.u.) |
| colours | magenta `#FF2469` = ψ>0, green `#1DCC94` = ψ<0 — deliberately **not** the EDD pair |
| lobes | one mesh per sign, `min_volume = 0.15 Å³`, Taubin 15 passes |
| panels | `render/`: 18 (HOMO+LUMO, α and β, five systems) — before CO₂ · `render2/`: 9 (HOMOs) — after CO₂ |

**Two design points worth copying:**

1. **One camera, reused across the two datasets.** The "after" model is the "before"
   geometry plus a guest (per-atom displacement below the 0.05 Å reuse guard), so the
   same camera gives the same projection and the before/after rows are pixel-comparable.
   Verified: pull-back **0.0%**, union margins identical, framework fill 0.800 across
   all 27 panels of both sets. Guarded by the tolerance with a fallback to solving.
2. **The two spin channels are not a nuisance, they are the physics.** The fundamental
   gap's two edges sat in different channels for different members of the series, so the
   figures are not organised by spin; each panel's channel lives in the build record,
   and the tables carry all four orbitals.

**Independent verification of the orbital indices.** The indices are *derived* from the
frontier CSV (`index(HOMO) = n_occ` within a spin channel), so two external checks stand
behind them: CP2K's own `i.e. HOMO - 0` in the cube header, and the **count of occupied
eigenvalues** it prints per spin channel in the `.out` (every channel agreed, energy to
10⁻⁶ Ha). Neither reads the project's own tables.

**What the figures said** — the measured table (frontier energies, level shifts on
adsorption, lobe volumes before → after, metal share, lobe-to-metal distances, per
panel) is redacted. The three checks it supported are worth copying as a discipline:

1. compare frontier-level shifts *and* lobe-volume changes between the bare and
   adsorption models, and claim only what both support;
2. measure the HOMO surface's minimum distance to the guest's centroid and compare it
   with the guest's van der Waals envelope — "the orbital does/doesn't overlap the
   guest" is then a measurement, not an impression;
3. in a metal-substitution series it is the *substitution*, not the adsorption, that
   moves the orbitals: report per-element, per-spin metal character (the tables carry
   all four orbitals) so the redistribution across members is visible.

**The comparison to make explicit in a caption** if this figure sits next to a
literature one: in a *chemisorbed* system the guest appears **on** the complex's HOMO
(the classic `[Cluster + N₂/O₂] HOMO` picture, where the gas LUMO and the metal HOMO
form a bonding state), and a non-interacting guest's panels look unlike that. A set of
"no change on adsorption" panels is a physical result, not a rendering failure.

## Reproduce

```bash
cd <root>

# Family 1 (EDD)
python script/edd_grid.py       result/EDD_cubes/<SYS>_EDD.cub
python script/edd_isosurface.py result/EDD_cubes/<SYS>_EDD.cub --iso 0.0003
python script/build_series_in_blender.py          # via the MCP socket
python script/make_montage.py

# Family 2 (orbitals), before and after adsorption
python script/mo_isosurface.py --all --dataset sbu     --iso 0.02
python script/mo_isosurface.py --all --dataset complex --iso 0.02
python - <<'EOF'
import sys; sys.path.insert(0, 'script')
from blender_bridge import Blender
Blender(timeout=14400).execute(open('script/build_mo_in_blender.py').read())   # sbu
Blender(timeout=14400).execute('DATASET = "complex"\n'
                               + open('script/build_mo_in_blender.py').read())  # complex
EOF
TREE=result3/render  bash script/run_mo_renders.sh
TREE=result3/render2 bash script/run_mo_renders.sh
python script/make_mo_montage.py --layout gap                 # sbu series
python script/make_mo_montage.py --dataset complex --layout homo
python script/make_mo_montage.py --layout beforeafter         # the pair

# verification (nine checks each)
python script/verify_mo_figures.py --dataset sbu
python script/verify_mo_figures.py --dataset complex
```

`script/mo_spec.py` is the single parameter/orbital-table source for both orbital
datasets: `set_dataset("sbu" | "complex")` rebinds the member suffix, the orbital table,
the output tree and which panels are drawn, so the whole family switches from one place
and the default (`sbu`) keeps the older behaviour byte-identical.

## Not yet drawn here

- **IGMH / IGM / NCI** weak-interaction surfaces — the algorithmic path (two cubes, the
  `sign(λ₂)ρ` vertex colouring) is described in `references/surface-types.md` §3–4, but no
  figure in this project uses it yet, so it carries no measured numbers here.
- **ESP mapped on a density surface** — this project has a separate documented Multiwfn
  pipeline and renderer for it (the `multiwfn-esp` skill in `~/.agents/skills/` and an
  `esp_fig` renderer), so it deliberately does not go through Blender.
