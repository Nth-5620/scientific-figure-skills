# Environment

Machine specifics from the reference project. Re-probe rather than assume if the setup
changes — most of these are cheap to check and expensive to guess.

## Blender

- **Blender 5.2.1 LTS** (Windows, Git Bash shell; install path is machine-specific).
- Cycles with **OptiX** on an RTX 4070 Laptop GPU; the builder sets
  `compute_device_type="OPTIX"` and enables the OPTIX/CUDA devices. The first render
  after start-up pays a kernel-compile cost (~1 min for a small test frame); after that
  a 2400×2400 / 1024-sample render is on the order of 10 s. Do not read the first run's
  wall time as the real cost.
- Blender's Python has **numpy and bmesh**, but **no PIL, scipy or skimage**. All image
  analysis happens in system Python (3.13, numpy/skimage/scipy/PIL present).
- `bpy.ops.object.shade_auto_smooth` / `shade_smooth_by_angle` exist; the code sets
  sharp edges on mesh data via bmesh instead, so it needs no operator context and works
  head-less.

## blender-mcp

The addon and its MCP server are published at
<https://github.com/ahujasid/mcp-for-blender>. The addon listens on a TCP socket; the MCP
server is only a proxy, so a plain client works:

```python
from blender_bridge import Blender                # ~60 lines, stdlib only
Blender().execute("import bpy; print(bpy.app.version_string)")
```

Protocol: send JSON `{"type": "execute_code", "params": {"code": "..."}}` to
`127.0.0.1:9876`, read back `{"status": "success", "result": {"result": "<stdout>"}}`.
`get_scene_info` is available; the viewport screenshot endpoint requires a filepath and
is not needed for this workflow.

In an MCP-capable agent harness, the blender-mcp MCP tools (`execute_blender_code`,
`get_scene_info`,
`get_viewport_screenshot`) drive this same addon — prefer them when they are
available, and keep the raw-socket client for long batch feeds, where its explicit
timeout is under your control. The importlib note below applies to either transport:
it is the interpreter that caches, not the socket.

Notes:

- Modelling **must** go through this path (see `SKILL.md`, house rule 1). The user's
  open session is the deliverable's home; head-less runs are for batch rendering and
  verification.
- **`importlib` caches by module name**, so re-feeding a builder that has been edited
  silently reuses the old revision — and the *GUI process keeps its interpreter between
  runs*, so this bites twice: once for the builder and once for any spec/parameter
  module the builder imports. Pop the `sys.modules` entries (or `importlib.reload`) for
  every project module before loading, every time. If a change seems to have no effect,
  this is why. Measured instance: a new function added to a spec module raised
  `module has no attribute ...` inside Blender while the same call worked in system
  Python.
- Save explicitly (`bpy.ops.wm.save_as_mainfile`) — executing code persists nothing. Use
  `copy=True` for extra copies so the session keeps its own file.

## Blender 5.x API differences that bit

**Compositor.** Blender 5 exposes it as a node *group* on the scene:

```python
ng = bpy.data.node_groups.new("Composite", "CompositorNodeTree")
scene.compositing_node_group = ng
rl   = ng.nodes.new("CompositorNodeRLayers")
over = ng.nodes.new("CompositorNodeAlphaOver")     # sockets: Background, Foreground
ng.interface.new_socket("Image", in_out="OUTPUT", socket_type="NodeSocketColor")
out  = ng.nodes.new("NodeGroupOutput")             # NOT CompositorNodeComposite
```

`scene.node_tree` does not exist in 5.x, and `CompositorNodeComposite` is not a valid
type. Index `AlphaOver`'s sockets **by name** — `inputs[1]` is the foreground, not the
background.

**Principled BSDF** input names are the 4.x/5.x ones: `Base Color`, `Roughness`,
`Specular IOR Level` (not `Specular`), `Sheen Weight`, `Alpha`, `IOR`. Probe
`[s.name for s in bsdf.inputs]` before assuming.

**Alpha blending.** `material.blend_method` does not exist in 5.x; guard those
assignments with `hasattr` so the same builder runs on older versions.

**Colour management.** Set `view_settings.view_transform = "Standard"`. The AgX default
compresses pure white to grey and mutes the palette; with an alpha-over-white composite,
Standard is what guarantees the background lands on exactly 255.

## Paths (origin project — not global assumptions)

The origin project's layout, kept so `examples/` and the pitfalls stay readable.
These paths mean nothing outside that project; re-derive the equivalents elsewhere.

- Project root: `<project-root>` (redacted)
- Reusable scripts: `<root>/script/` — grid reader, isosurface extractor, scene builder,
  QA, audits, montage
- Figures and scenes: `<root>/result/render/`, `result2/render/`, `result3/render*/
  (one directory per figure generation)
- Grid cache: next to each `.cub`, regenerable

The machine-independent rule this section exists for: **render output is always an
absolute path** (pitfall 10).
