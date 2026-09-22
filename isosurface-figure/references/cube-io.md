# Reading a cube file

A Gaussian cube is a simple format and a minefield: the two things that go wrong
(axis order, volume element) both produce numbers of a plausible magnitude, so neither
raises an error. Read this before parsing any grid.

---

## 1. Layout

```
line 1            title / comment
line 2            title / comment
line 3            natoms   origin_x origin_y origin_z          [bohr]
line 4            nx       ax_x ax_y ax_z                      [bohr]
line 5            ny       ay_x ay_y ay_z
line 6            nz       az_x az_y az_z
next natoms       Z  charge  x y z                             [bohr]
then              nx*ny*nz grid values                         [field units]
```

Notes that matter:

- **Coordinates in the header are in bohr.** Convert to Å with
  `BOHR = 0.52917721067` (CODATA-2018) before doing anything geometric.
- The three axis lines carry the *voxel vector*, not just a count. Read the vectors and
  keep them: a writer may emit a non-orthogonal or negative-step box, and a mesh built
  from the counts alone would be silently wrong. In practice these are orthorhombic, so
  the vectors reduce to `spacing_bohr = |a| / n`.
- The value block may be wrapped at an arbitrary count per line (Multiwfn wraps at 320,
  i.e. one x-row per line for a 320³ grid). Never assume a fixed values-per-line count.

## 2. Parsing: try fixed-width, then tokenise

CP2K/Multiwfn writers emit values as `%14.5E`-style **fixed-width 14-character
fields**, and a field can begin with a `-` immediately after the previous field's last
digit — `-0.123456789-0.987654321-...` — which a naive whitespace tokenisation reads as
one malformed token and which `np.fromstring(sep=" ")` may silently drop.

The robust pattern, in this order:

1. **Fixed-width.** Require every line to have the same length and the total to be an
   exact multiple of the field width (14); then `np.frombuffer(blob, dtype="S14")`.
   Accepted only under those conditions — these writers "occasionally emit a ragged
   line", and a fixed-width parse of a ragged file shifts every subsequent field.
2. **Tokenise** as the fallback (`np.fromstring(blob, dtype=np.float32, sep=" ")`).

Assert `arr.size == nx*ny*nz` after either, and raise otherwise. Record which parser
succeeded — if it was the fallback, a ragged line may still have shifted values, and
that is worth knowing before trusting a delicate number.

## 3. Axis order: the specification and reality disagree

The Gaussian cube *specification* says **x varies fastest**. Grids written by some
Multiwfn paths are stored **z-fastest**. So the array must be reshaped to `(nx, ny, nz)`
such that `array[ix, iy, iz]` maps index 0→X, 1→Y, 2→Z, and the values must be read in
that order — `data.reshape((nx, ny, nz))`, **not** `(nz, ny, nx)`.

**Always validate empirically; never trust the spec or the writer's docs.** Two
independent checks that worked on the reference data:

- an independently validated integration script produced the same numbers to 5×10⁻⁷
  e/bohr³ as a numpy subtraction of the constituent fields;
- integrating the difference field over a 4 Å sphere about a known guest reproduced the
  published accumulation/depletion to five decimals **only** under the z-fastest
  mapping. With the transposed mapping the sphere sat in an empty region and returned
  **exactly `0.0000`** — a literal zero is the signature of transposed axes, and it is
  worth asserting on: `assert abs(value) > 1e-6`.

Symptom in a figure: lobes render as detached blobs several Å away from the fragment
they belong to, with everything else looking fine.

## 4. Units: two volume elements, one bug

| quantity | unit | consequence |
|---|---|---|
| grid values of a density-based field | e/bohr³ (or e/Å³ for some writers — check) | a **charge** integral needs the **bohr³** volume element |
| mesh volumes, marching-cubes spacings, radii | Å | geometric quantities use the **Å** spacing |

Keep `dv_bohr3` and `dv_ang3` as two named values and never let a single `dv` serve
both. For a 0.145432 bohr grid the ratio is `(1/0.529177)³ ≈ 6.75` — small enough to
read as a plausible physical result rather than a bug.

**Wavefunction amplitudes are neither.** An orbital cube holds ψ in a.u.; integrating
ψ dV is meaningless and must not be reported as a charge. If your summary JSON has a
`charge` field for an orbital panel, that field is a bug.

## 5. Caching

A 320³ grid is ~460 MB of text. Parse once and write a `.npy` (float32) plus a small
`_meta.json` carrying the header (natoms, origin, counts, axes, and the atom list) next
to the cube, then load from the cache. `load_cube(path, force=True)` regenerates.

Practical sizing: a CP2K `&MO_CUBES` run can coarsen the grid with `STRIDE n` —
`STRIDE 2` gives a 160³ grid at ~0.29 bohr, ~55 MB per orbital in text and far less as
`.npy`. That is the knob for making a big cluster's orbital cubes tractable; if the
surface looks faceted, re-run the writer with `STRIDE 1` rather than smoothing harder.

## 6. The atom block is data too — and its order is not stable

The atom block is often the only place the geometry lives, so it is tempting to read
atoms from it and index them positionally. **Do not.**

A re-exported cube reordered the same atoms — the guest moved from the head of the list
to the tail while the metals moved to different positions — same
absolute frame, same chemistry, different order. A script that used `atoms[:3]` ("the
first three atoms are the guest") then integrated a sphere around a framework carbon,
returning a **plausible but wrong number** (values redacted) — the same order of
magnitude as the correct one, so nothing looked wrong, and the wrong number reached a
results write-up.

Two order-independent helpers solve it, and both are worth having:

```python
mol_atom_map(meta, mol_path)   # cube atom i -> its 1-based index in the mol file
                               # (matches on element + coordinates within 0.05 A, so it
                               #  also *validates* that the two files share a frame)
locate_co2(meta)               # a fragment centre found by geometry signature
                               # (a C carrying exactly two O at ~1.16 A), not by layout
```

Key per-atom outputs by an index that is stable across generations (here: the mol file's
index) so a table's site labels keep meaning the same thing.

**Related, and easy to miss:** the same system's `.mol` and grid file may disagree about
guest ordering in a *known* way — a guest listed first in the `.mol` and appended last in
the grid. If you mix the two files (mol for connectivity, grid for coordinates), permute
explicitly and assert element-by-element. Worked measurement:
`examples/reference-project/README.md` (max coordinate disagreement 4.77×10⁻⁷ Å after the
permutation).

## 7. Where the grids come from

- **Multiwfn** — the workhorse: real-space functions (density difference, IGM/IGMH δg
  and sign(λ₂)ρ, ESP, ELF/LOL, NCI) can all be exported as `.cub`. Its outputs are the
  ones this skill's parser was written against.
- **CP2K** — `&MO_CUBES` writes orbital cubes directly (with `NHOMO/NLUMO` and
  `STRIDE`), and the density can be written as a cube too.
- **Other codes** — VASP `CHGCAR`, Gaussian `.fchk`, ORCA `.gbw`: convert to cube with
  Multiwfn or the code's own tool before rendering; do not write a bespoke parser for a
  binary format when a converter exists.
