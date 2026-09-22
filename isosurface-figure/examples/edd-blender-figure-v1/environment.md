# Environment

Machine specifics from the origin project. Re-probe rather than assume if the
setup changes — most of these are cheap to check and expensive to guess.

## Blender

- **Blender 5.2.1 LTS** (Windows, Git Bash shell; install path is machine-specific).
- Cycles with **OptiX** on an RTX 4070 Laptop GPU; the build sets
  `compute_device_type="OPTIX"` and enables the OPTIX/CUDA devices. The first render
  after start-up pays a kernel-compile cost (~1 min for a small test frame); after
  that a 2400×2400 / 1024-sample render is on the order of 10 s. Do not read the first
  run's wall time as the real cost.
- Blender's Python has **numpy 2.3.4 and bmesh**, but **no PIL, scipy or skimage**.
  All image analysis happens in system Python (3.13, numpy/skimage/scipy/PIL present).
- `bpy.ops.object.shade_auto_smooth` / `shade_smooth_by_angle` exist; the code sets
  sharp edges on mesh data via bmesh instead, so it needs no operator context.

## blender-mcp

The addon listens on a TCP socket; the MCP server is only a proxy, so a plain client
works. `script/blender_bridge.py` implements it:

```python
from blender_bridge import Blender
Blender().execute("import bpy; print(bpy.app.version_string)")
```

Protocol: send JSON `{"type": "execute_code", "params": {"code": "..."}}` to
`127.0.0.1:9876`, read back `{"status": "success", "result": {"result": "<stdout>"}}`.
`get_scene_info` is available; the viewport screenshot endpoint requires a filepath
and is not needed for this workflow.

Notes:
- Modelling **must** go through this path (see SKILL.md hard requirements). The user's
  open session is the deliverable's home; head-less runs are for batch verification.
- `importlib` caches by module name, so re-feeding a builder that has been edited
  silently reuses the old revision. `build_in_blender.py` pops `sys.modules` entries
  before loading, every time. If a change seems to have no effect, this is why.
- Save explicitly (`bpy.ops.wm.save_as_mainfile`) — executing code does not persist
  anything. Use `copy=True` for the extra copies so the session keeps its own file.

## Blender 5.x API differences that bit

**Compositor.** Blender 5 exposes it as a node *group* on the scene:

```python
ng = bpy.data.node_groups.new("EDD_Composite", "CompositorNodeTree")
scene.compositing_node_group = ng
rl   = ng.nodes.new("CompositorNodeRLayers")
over = ng.nodes.new("CompositorNodeAlphaOver")     # sockets: Background, Foreground
ng.interface.new_socket("Image", in_out="OUTPUT", socket_type="NodeSocketColor")
out  = ng.nodes.new("NodeGroupOutput")             # NOT CompositorNodeComposite
```

`scene.node_tree` does not exist in 5.x, and `CompositorNodeComposite` is not a valid
type. `AlphaOver`'s sockets are named `Background` / `Foreground` — index them by
name, since `inputs[1]` is the foreground, not the background.

**Principled BSDF** input names are the 4.x/5.x ones: `Base Color`, `Roughness`,
`Specular IOR Level` (not `Specular`), `Sheen Weight`, `Alpha`, `IOR`. Probe with
`[s.name for s in bsdf.inputs]` before assuming.

**Alpha blending.** `material.blend_method` does not exist in 5.x; guard those
assignments with `hasattr` so the same builder runs on older versions.

**Colour management.** Set `view_settings.view_transform = "Standard"`. The AgX
default compresses pure white to grey and mutes the palette; with an alpha-over-white
composite, Standard is what guarantees the background lands on exactly 255.

## Paths

- Project root: `<project-root> (redacted)`
- Reusable scripts: `<root>/script/` (grid, isosurface, builder, QA, audits)
- Figures and scenes: `<root>/result/render/`
- Grid cache: `result/EDD_cubes/cache/` (regenerable; `--force` to rebuild)
- **Render output: always an absolute path.** Blender resolves a *relative*
  `scene.render.filepath` (i.e. `render_blend.py --out`) against the drive root, not the
  shell's working directory, so `--out result2/render/figures/x.png` reports success and
  writes to `C:\result2\...`. Python in the same process disagrees (`os.getcwd()` and
  `os.path.abspath()` are correct), and `-b <file>.blend` / `-P` resolve relative paths
  fine — measurement and the full trap in `pitfalls.md` #16.
