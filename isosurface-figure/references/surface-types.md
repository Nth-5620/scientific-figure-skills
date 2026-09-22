# Surface types: what each field means, and how to label it

Pick the row for your field **before** extracting. The isovalue, the meaning of the two
signs, the colour scheme and the number you quote all follow from the field's
definition, and two of these rows are routinely confused with each other: a density
difference and a wavefunction are both "±" surfaces, but `+`/`−` means
accumulation/depletion in one and **phase** in the other.

Measured numbers are marked *(measured)*; the rest is standard practice in the field,
stated so you know what to check rather than what this project verified.

---

## 1. Electron density difference — EDD / Δρ / 差分电荷密度

**Definition.** Fragment subtraction: `Δρ = ρ(AB) − ρ(A) − ρ(B)`, where A and B are the
two parts (e.g. a cluster and an adsorbate) evaluated in the AB geometry. Values in
e/bohr³ (or e/Å³ — check the writer).

**Signs.** `+` = electron **accumulation**, `−` = **depletion**. A genuine charge
*transfer* shows up as a net integral over a region; a large accumulation balanced by an
equal depletion elsewhere is **polarisation**, not transfer. Report both.

**Isovalue.** 2–4 × 10⁻⁴ e/bohr³. *(measured)* Sweep on the reference system (looser
isovalue → larger lobes; volumes redacted):

| iso (e/bohr³) | note |
|---|---|
| 0.0004 | tighter, more conservative |
| **0.0003** | project default |
| 0.0002 | starts eating the ligand region |

**Colour.** The EDD pair: yellow = accumulation, blue-violet = depletion
(`colour-schemes.md`).

**The one thing that must be checked before shipping a series.** Δρ is a difference of
two *independent* SCF calculations, so anything that differs between them shows up as
signal. In the reference project, open-shell UKS + DFT+U systems failed to cancel
each other's near-nucleus density at a subset of metal sites, by ~three orders of
magnitude,
producing a false lobe whose above-isovalue region reached ~2 Å from the nucleus —
against a rendered metal ball of 0.537 Å radius. Nothing about the figure looked broken.

The measurement protocol, per metal site (values redacted; the *shapes* of the
measurements are the transferable part):

| measurement | what it tells you |
|---|---|
| whole-grid extremes vs the closed-shell reference | whether the fields are comparable at all |
| contiguous above-iso radius from the nucleus | whether a false lobe wraps the metal (healthy: 0 Å) |
| accumulation within r = 2.0 Å, and its net | a pure rearrangement has net ≈ 0 |
| ⟨\|Δρ\|⟩ radial peak and its radius | where the residual lives |
| angular power decomposition | l = 4 (cubic) dominance = grid-signature artefact, not a d-orbital shape |

If affected sites exist: **a uniform isovalue does not make the panels comparable.**
Do not report the extra lobe volume as metal-dependent charge transfer, do not mask it
away (masking the metals removed only 1–2% of the lobe volume *(measured)* — it is a
diagnostic, not a fix), and do not switch silently to per-panel isovalues (the spread
in the isovalue needed is itself the finding). Quoting a **metal-excluded
interfacial integral** — the same geometric window for every system, with the metal
neighbourhoods removed — gives a number that *is* comparable: *(measured on the
reference series)* it formed a smooth monotonic trend, where the raw
guest-sphere numbers spread severalfold only because they silently included
part of each metal's coordination sphere. (Values redacted.)

**Complementary numbers** to quote alongside: Bader (QTAIM) charges per metal and per
guest, and the ratio of net transfer to gross accumulation at the interface.

---

## 2. Frontier orbitals — HOMO / LUMO, 轨道图

**Definition.** The orbital's **wavefunction amplitude** ψ in a.u., from a single-point
calculation at the adsorption geometry. CP2K writes them with `&MO_CUBES`; any code's
orbitals can be exported as cubes.

**Signs.** `+`/`−` are **wavefunction phase** — they mean nothing physical on their own
(the overall sign of ψ is arbitrary, and the relative phase of two lobes is what carries
the bonding/antibonding information). **There is no accumulation and no depletion here,
and no charge may be integrated from these cubes** — ∫ψ dV is meaningless. Getting this
wrong is the single most common mislabel in this family of figures.

**Isovalue.** 0.015–0.02 a.u. is the conventional window (GaussView/Multiwfn default
0.02; the AIChE J. 2026 reference paper used ρ = 0.015 a.u. for its HOMO/LUMO diagrams).
*(measured)* A sweep on the reference cluster shows what the threshold does to coverage:
loosening the isovalue from 0.004 to 0.02 a.u. shrank the covered-atom fraction from
~60% to ~20%. Use that kind of table, not prose, to answer "why doesn't the lobe cover
the molecule?"
— ψ has nodal surfaces and decays away from the atoms that dominate it.

**Which orbitals to draw.**

- Bare system: HOMO and LUMO.
- Adsorption model: the **complex's** HOMO (and LUMO) — this is what shows whether the
  adsorbate participates. Check the minimum distance from the lobe's surface to the
  guest's centroid; if it stays outside the guest's van der Waals envelope, the orbital
  does not overlap the guest. *(measured:* the lobe stayed outside the envelope in every
  panel, while adsorption shifted the level by only meV — i.e. the orbital does not bond
  the guest; values redacted.*)
- **Open-shell systems: α and β are different orbitals** (different energies and
  shapes), so there are two HOMOs and two LUMOs. The level that *brackets the fundamental
  gap* may sit in different spin channels for different members of a series — verify
  which, from the data (`index(HOMO) = n_occ` within a spin channel), and say which
  channel each panel shows in the record. Whether to label α/β in the figure is a
  presentation choice; keeping the channel out of the panel labels and in the table is
  the convention here.

**A qualitative orbital-interaction diagram** (no energy axis, no eigenvalues) is a
useful companion panel, and one the reference paper used: three columns — the isolated
guest's HOMO and LUMO (only the guest's ball-and-stick model between them), the bare
cluster's HOMO, and the complex's HOMO — with coloured arrows from the guest orbitals to
the cluster HOMO, one colour per guest species, and an **✗ on any channel that is
symmetry-forbidden**. It communicates the interaction logic without pretending to
quantitative level alignment, which a single isovalue cannot support.

**Eigenvalues: mind the level of theory.** A PBE/Γ-point orbital energy is not comparable
to an experimental or hybrid-functional value, and absolute values differ between codes.
What *is* meaningful is the shift of a level between two structures computed the same
way. The independent check that a chosen cube really is the HOMO is the code's own
statement — CP2K writes `i.e. HOMO - 0` into the cube header and prints the occupied
eigenvalue count per spin channel in its `.out`; both are external to your table and both
are cheap to assert.

---

## 3. IGMH / IGM — weak-interaction surfaces, 弱相互作用可视化

The modern default for showing *what holds two fragments together*: hydrogen bonds,
halogen bonds, π-stacking, dispersion. Unlike NCI it needs no low-density cutoff, and
unlike a difference density it is not a subtraction of two SCF runs, so it does not
inherit the cancellation problem of §1.

**Definition.** The independent gradient model compares the true density gradient with a
reference built from the *absolute* atomic gradients:

```
δg(r) = |∇ρ_IGM(r)| − |∇ρ(r)| ,      ∇ρ_IGM = Σ_i |∇ρ_i|
```

`δg ≥ 0` everywhere and is large in regions where neighbouring atoms' gradients *oppose*
— i.e. where an interaction exists. The variants differ only in the partition used to
define the atomic densities ρ_i:

- **IGM**: promolecular reference (superposition of free-atom densities). Cheap, but the
  result depends on the reference and thus on the basis set.
- **IGMH**: the partition is **Hirshfeld / Hirshfeld-I**, using the actual calculated
  fragment density. This is the recommended variant for weak interactions because the
  descriptor becomes essentially basis-set independent.

**δg vs δg_inter.** `δg` includes both intra- and inter-fragment contributions (it is
dominated by the covalent structure of each fragment). For an interaction figure, export
**`δg_inter`** — the inter-fragment cross terms only — so the surface shows the
interaction and nothing else. Ask for a separate `δg_intra` only if you want to show
intramolecular strain.

**Signs.** δg is **one-sided** (a magnitude): there is no `+`/`−` pair and no second
colour-coded lobe. The colour comes from a *different* field.

**Isovalue.** δg_inter at ~0.005–0.02 a.u., commonly 0.01 a.u. *(convention — not
measured in the reference project; sweep 0.005/0.01/0.02 and pick by what appears, since
the right value depends on how strong the interaction is).*

**Colour: sign(λ₂)ρ, blue → green → red.** The surface is coloured by
`sign(λ₂)·ρ` (λ₂ = the second eigenvalue of the density Hessian, i.e. the sign of the
density curvature perpendicular to the bond path), over roughly ±0.05 a.u.:

| colour | sign(λ₂)ρ | reading |
|---|---|---|
| blue | large negative | strong attraction — H-bond, halogen bond, strong electrostatic |
| green | ≈ 0 | weak/dispersive — van der Waals, π-stacking |
| red | positive | steric repulsion / ring strain |

This is *the* standard IGM/NCI colour convention and it must be labelled in the footer
(it is not self-evident, and readers expect it).

**Implementation: two fields, not two signs.** This is the mechanism to build when the
family is implemented — the reference implementation in this project does **not** yet do
it (it supports one field with a two-colour ± split):

1. export two cubes from the wavefunction: **A = δg_inter** and **B = sign(λ₂)ρ**;
2. extract the isosurface of A at the chosen isovalue (single-sided, one mesh);
3. for every mesh vertex, **trilinearly interpolate B** at that vertex's Å coordinates
   (same grid, so this is a straight index computation — and the same axis-order/unit
   rules as `cube-io.md` apply);
4. map the B values through the diverging blue–green–red map over the stated range
   (clamp outside);
5. write a per-vertex colour attribute into the mesh and let the material read it
   (Blender: a `color_attribute` / vertex colour layer plus an `Attribute` node feeding
   Base Color). Keep the alpha Fresnel shell from `lobe-material.md` so the surface stays
   translucent;
6. record in the summary: the isovalue of A, the colour range of B, and the fraction of
   vertices clamped at either end (a large clamped fraction means the range is wrong).

The **scatter plot** of δg against sign(λ₂)ρ is the quantitative companion (spikes in the
low-gradient region are the interactions); export the plane data rather than eyeballing
the surface.

**Fragment definition is part of the result.** For `δg_inter` you must say which atoms
are fragment 1 and which are fragment 2 (e.g. the adsorbate vs the framework, or one
monomer vs the other). Different partitions give different surfaces; state it in the
caption.

---

## 4. NCI / RDG

The older, still common weak-interaction surface; complementary to IGMH.

**Definition.** Reduced density gradient `RDG = |∇ρ| / (2 (3π²)^{1/3} ρ^{4/3})`, plotted
(vs sign(λ₂)ρ) as a scatter, and mapped in real space as an RDG isosurface restricted to
**low density** (`ρ < 0.05 a.u.`) so the covalent bonds do not swamp it.

**Isovalue.** RDG ≈ 0.3–0.5 a.u. with the density cutoff at 0.05 a.u. *(convention.)*

**Colour.** Same sign(λ₂)ρ blue–green–red as IGMH (above). The colouring is what makes
NCI interpretable; an uncoloured RDG surface is nearly meaningless.

**When to prefer IGMH.** NCI shows intra- and inter-molecular regions together and its
appearance depends on the density cutoff; IGMH's `δg_inter` isolates the interaction and
has no cutoff. If the question is "what holds the adsorbate to the framework", IGMH.

---

## 5. ESP on a molecular surface

Not a signed pair: a **mapped colour on a single closed surface**. The surface is an
electron-density isosurface (the conventional "molecular surface", ρ = 0.001 a.u.) and
the colour is the electrostatic potential evaluated on it.

**Semantics.** Colour = ESP value; the *surface* itself carries no sign meaning. State
the map range and its units in the footer (`kcal/mol` and `a.u.` are both used, and the
sign convention is *not* universal — some conventions paint negative red).

**Typical scale.** ±20 kcal/mol for a diverging map, for organic/coordination systems;
always report the actual extremes the renderer measured.

**Quantitative companions** (usually what the figure is *for*): the surface-area
distribution over ESP bins, per-atom surface ESP statistics (min/max/mean per atom on the
surface), and σ-holes / lone-pair maxima. This project has a documented Multiwfn pipeline
and a dedicated ESP renderer for these — use them rather than rebuilding it in Blender,
since an ESP surface needs diverging-map colouring (the two-field mechanism in §3) plus
the statistics. For the Blender path the value-mapped rules — colormap construction,
the uniform-alpha clarity ladder, separate-layer compositing — are in
`value-mapped-surfaces.md`.

---

## 6. Charge density, spin density, ELF/LOL, and other single fields

The same machinery applies; decide three things first:

| field | signed? | isovalue | colour |
|---|---|---|---|
| total electron density ρ | no | 0.001–0.01 a.u. (0.001 = "molecular surface") | single, or a mapped quantity (§5) |
| spin density ρ_α − ρ_β | **yes** | 0.001–0.005 a.u. | two-colour ± pair |
| deformation density | yes | ~0.001–0.01 | two-colour ± pair |
| ELF / LOL | no | 0.5–0.8 | sequential |
| a difference of two same-basis runs | yes | as §1 | two-colour ± pair |

Two rules:

- **A one-sided field has no second lobe.** Do not invent a `−` surface for a magnitude,
  and do not reuse the accumulation/depletion colours for a magnitude — a sequential or
  diverging map is the honest encoding.
- **Colour carries meaning, so two families must never share a pair** unless the figure
  says so explicitly. This is why the EDD pair and the orbital pair are different: they
  appear side by side in one paper, and a reader must be able to tell "polarisation" from
  "orbital phase" at a glance (`colour-schemes.md`).

---

## 7. Footers: the minimum a reader needs

Every isosurface figure states, in the figure or its caption:

1. the **field** (and for a difference or IGMH, the definition/partition);
2. the **isovalue with units**;
3. the **meaning of each colour** — including whether `+` is accumulation or phase;
4. for a mapped surface, the **colour range with units**;
5. that one isovalue and one camera were used across the panels, if they were.
