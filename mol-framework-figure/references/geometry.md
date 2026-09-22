# Geometry: structures, bonds, atoms, bond meshes

Everything here is a property of the conventions, not of one dataset: the perception
rule, the radius rule and the mesh guards transfer to any molecule. The measured
numbers come from the origin project's builder, where a claim validator
(`script/validate_skill_claims.py`) re-checks them against the source.

---

## 1. Getting atoms into one coordinate frame

You need element + XYZ (Å) per atom. The trap is using **two** sources — typically a
`.mol` for connectivity and a `.cub` for coordinates — and assuming they agree:

- **Assert, do not assume.** Compare the two lists element for element and require
  the same length and the same order within 0.01 Å. The reference implementation
  measured 4.77×10⁻⁷ Å agreement, i.e. the files are the same geometry; anything
  larger means you are about to mix two orderings.
- **Never index positionally.** `atoms[:3]` was "the guest" in one generation of a
  dataset and a framework C/H/C trio in the next, because a re-export reordered the
  atom block (`../isosurface-figure/references/pitfalls.md` §6). Locate
  fragments by geometry (element pattern + bond lengths) or by an
  element-plus-coordinates map, and key per-atom results by an index that is stable
  across generations.
- When the coordinates come from a grid file, take them from the file's own atom
  block so the framework and any isosurface extracted from the same file sit in one
  frame by construction. See `../isosurface-figure/references/cube-io.md`.
- **Guest ordering differs between file formats of the same system.** In an
  adsorption model the guest (a CO₂, say) may be first in the `.mol` and last in the
  grid file's atom block. If you mix them, permute explicitly and assert the
  permutation element by element. Worked example: `examples/reference-project/README.md`.

## 2. Bond perception

```python
# Cordero et al. (2008) covalent radii, angstrom
COVALENT_RADII = {"H": 0.31, "C": 0.76, "N": 0.71, "O": 0.66, "Zn": 1.22, "Co": 1.26}

def perceive_bonds(atoms, factor=1.25):
    """atoms: [(element, xyz)] -> [(i, j)] pairs, i < j."""
```

A pair is bonded when `d(i,j) <= factor * (r_i + r_j)`. **factor = 1.25** is the
value to start from: it catches coordination bonds (metal–N/O) without inventing
bonds across a non-bonded contact. Two consequences worth knowing before you tune it:

- A **physisorbed guest** must stay unbonded. Measured case: a CO₂ whose nearest
  framework contact sits ~3 Å away — far beyond the C–N threshold of
  `1.25 × (0.76 + 0.71) = 1.84 Å` — so the guest floats free at every isovalue, and a
  spurious guest–framework bond never appears in the figure.
- A **metal–metal** contact in a cluster (e.g. 3.5 Å Zn···Zn) will be bonded by this
  rule if the radii say so; that is usually what you want for an SBU, and it is why
  the metal involvement in a bond is worth flagging in the record.

Always cross-check perception against the structure file's own bond block and write
the comparison into the build record: `{n_mol, n_perceived, agree, only_in_mol,
only_perceived}`. A disagreement is not automatically an error (the file may omit
coordination bonds), but it must be visible rather than silent.

**Bond order** is only used for one thing in these figures: drawing a small-molecule
guest's double bonds (two or three parallel sticks) so the guest is not mistaken for
a framework bond. Framework double bonds are deliberately drawn as single sticks —
element-coloured half-cylinders at these radii read as a bond either way, and the
licorice convention is what keeps the drawing legible at this scale.

## 3. Radii

| quantity | rule | value for C |
|---|---|---|
| ball-and-stick sphere | `BALLSTICK_SCALE × Cordero` | 0.44 × 0.76 = **0.334 Å** |
| metal sphere | same rule | 0.44 × 1.22 = **0.537 Å** (Zn), 0.554 (Co) |
| H sphere | same rule | 0.44 × 0.31 = **0.136 Å** |
| ball-and-stick bond | one uniform gauge | **0.100 Å** |
| guest single stick | ball-and-stick guest gauge | **0.085 Å** |
| guest double stick | thinner, pair straddles the axis | **0.060 Å**, offset **0.120 Å** |
| tube variant: ligand bond | `BOND_R2` | 0.150 Å |
| tube variant: metal bond | `BOND_R_METAL` | 0.175 Å |
| tube variant: metal/guest ball | `BALL_RADIUS` | Zn/Co 0.42, C 0.36, O 0.33 |

One rule for everything under the default style is the point: **the ordering stays
physical**, so the metals are the largest spheres (0.537 Å vs 0.334 Å for C) and read
as the subject of the figure by size alone, with no style change needed.

The tube variant keeps small SBU atoms with bonds nearly as thick as the atoms, so
the ligand framework reads as continuous tubes while metals and the guest are
ball-and-stick; the style contrast replaces the size contrast. Use it only when the
framework is too crowded for balls — it costs exactly the size legibility that makes
a metal-substitution series readable.

## 4. Atom and bond meshes

- **Spheres**: an icosphere, subdivision 3 (642 vertices). One object per atom.
- **Bonds**: a bond is **two half-cylinders meeting at the bond midpoint**, each
  carrying the element colour of the atom that owns it (the VMD-licorice reading).
  24 sides. One object per bond, with exactly the two material slots its halves need
  and a per-polygon `material_index`.
- **Each half is capped only at the atom end.** The midpoint end is left **open**
  (`cap1=False` for a half running atom → midpoint). This matters: capped halves put
  a coincident *interior* disk at every bond midpoint, which the renderer ignores but
  Freestyle's Contour edge type picks up, drawing a seam across every bond
  (pitfall 6). Open halves abut exactly, so the union is visually identical.
- **Double bonds**: two thinner parallel sticks offset `±DOUBLE_OFFSET` along an axis
  perpendicular to the bond, built from the same half-cylinder primitive. The
  perpendicular axis must be chosen so the pair does not collapse into one stick in
  projection; pick it from a vector not parallel to the view axis.
- **Guard every manually-assembled mesh.** Concatenating cylinders means rebasing
  face indices by the number of vertices already emitted — in two places, within an
  element block and when concatenating blocks — and `from_pydata` accepts a wrong
  answer silently. `new_mesh(..., expect_all_used=True)` asserts no vertex is left
  unreferenced, which caught the bug within seconds of it being introduced
  (pitfall 1). Keep that guard on anything built by hand.
- **Element → atomic number** is needed to read atom blocks out of grid files; the
  Z table used here is H 1, C 6, N 7, O 8, Co 27, Zn 30. Extend it rather than
  guessing when a new element appears — an unmapped Z raises instead of silently
  dropping an atom.

## 5. Object-count tripwire

Compute the expected complete scene as

```
natoms + nbonds + 2*n_lobe_surfaces + 1 camera + 3 lights
```

from the inputs, and check it in the build record. A count that is short by exactly the lobes
means the isosurface meshes are missing and the render will silently come out
without them (`../isosurface-figure/references/pitfalls.md` §3).
