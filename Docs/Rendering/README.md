# Rendering

Ceili's renderer is built from parts that plug in. A device backend is chosen by
a one-line alias. A pass is a component that any package can register. The
lighting path, the lighting model, the output range and dozens of cost strategies
are runtime settings, so one build switches between them while the game runs and
measures each one against the others inside a single run. The pages below cover
each part in depth; this page is the map.

<!-- MEDIA: the Studio compare slider dragged across one scene: PBR HDR on the
     left, the legacy Blinn LDR game on the right, then the same split with
     forward on the left and deferred on the right. -->

## Plug and play

- **Backends behind one interface.** Calling code never names a graphics API.
  Direct3D 12, Vulkan (with a Full or a Compat tier, picked at device creation),
  a console backend and a headless `Null` device all sit behind `IDevice`, and a
  config line picks one.
- **Passes are components.** Each of the 46 passes in the frame implements
  `IRender` and registers under a pass type whose priority places it. The Ui
  package adds its own; a game package can too.
- **Visibility behind interfaces.** Each scene holds a priority-ordered set of
  `IVisibility` providers that fill the frame's visible sets, every frustum in
  one call. The rows provider queries a spatial index with a uniform grid and a
  kd-tree as its backends, and a compiled world picks its own items through a
  registered selector: its kd-tree, and a portal walk through its areas.
- **Forward or deferred, at runtime.** Five lighting paths, from one forward
  draw per light to clustered deferred, are one setting read on the next frame.
  The paths share a frame: a material with no G-buffer pass stays forward-lit
  inside a deferred view.
- **PBR or the legacy look, at runtime.** A ported material carries a Blinn-Phong
  pass and a PBR pass side by side, and the enabled lighting model picks one. With
  the 8-bit LDR output and the legacy view rules, the original game draws inside
  the new engine.
- **Strategies as settings.** Each cost decision (draw runs, detail levels, shader
  levels, cloud modes, the ocean's simulation) is a toggle with a recorded latch
  point, so a device profile is a list of settings and every choice can be
  measured on the device.

## What sets it apart

- **The editor is the runtime.** Studio's 3D and 2D views, the material preview
  and its thumbnails draw through the same renderer as the game, in the same
  frame.
- **Two renderers in one frame.** The compare slider splits the viewport by
  lighting model, by render path or by texture resolution, under one camera.
- **Physical units end to end.** Lights, the sky and the exposure are in real
  units, from a noon sun to a moonless night.
- **Authored as data.** Materials are Lua programs, and the post chain is a list in
  config that fuses adjacent stages into one pass.
- **One codebase from desktop to headset.** The same scenes run in stereo on a
  Meta Quest, fitted to its frame with geometry, draw and shader levels of detail.
- **Testable with no GPU.** The `Null` device runs the whole path headless, so a
  material name that resolves to nothing is a checkable no-draw.

## The rendering pages

| Page | What it covers |
|---|---|
| [Render Devices & Backends](RenderDevices.md) | One device interface and its backends, the Vulkan Full and Compat tiers, the frame clock and the record-time ring, and texture creation off the lock. |
| [Render Passes & Visibility](RenderPasses.md) | `IRender` and its 46 implementations, pass types as the frame's order, the producer stage, forward and deferred as runtime-switched passes, `IVisibility` providers and the one-call cull of every frustum, the spatial index backends, and the compiled world's selection. |
| [Surfaces & Draw Data](Surfaces.md) | Surfaces as data, the hot-cold visibility rows, the one generic refresh system, instance uploads, and render interpolation between fixed ticks. |
| [Compiled Worlds](CompiledWorlds.md) | The ported level as one block: the kd-tree, the BSP, areas and portals walked every frame with no PVS, the indoor and outdoor classification, and what it decides for the sun, the sky's ambient, sky lights and the weather. |
| [Materials](Materials.md) | Materials as Lua programs with embedded HLSL, inheritance, live constants, content-addressed dedup, pipelines built on demand on workers, refraction stand-ins, and PBR and legacy Blinn passes in one material. |
| [Lighting](Lighting.md) | Per-light forward, the clustered light grid and the deferred G-buffer in one frame, the runtime lighting path, physical light units, and the area ambient that keeps outdoor light out of a tunnel. |
| [Shadows](Shadows.md) | Cascaded sun shadows in one atlas, the cascade fit, the multi-frustum cull, two bias mechanisms, and the cascade cache. |
| [Atmosphere](Atmosphere.md) | A physical sky in real units, the lookup tables, the froxel volume that hazes the world, local dust, the daily sun and moon, and the horizon seal. |
| [Weather](Weather.md) | One row of weather facts, cloud and fog volumes, rain and lightning, a Gerstner ocean that gameplay can query, the indoor fact, and the scatter system. |
| [Meshes & Animation](MeshesAndAnimation.md) | glTF and legacy meshes, skeletons and clips, the animation state machine, root motion, and skinning. |
| [Particles](../Gameplay/Particles.md) | A world-space particle simulation, its store, ramps and one-shots, and the billboard pass. |
| [Post-Processing & HDR](PostProcessing.md) | The data-driven post chain, post-stage fusion, auto-exposure, the lens flare, and SDR, HDR10, scRGB and legacy LDR output. |
| [XR and Quest](Xr.md) | Stereo through OpenXR, controllers and hands, the overlay in the headset, phones, and the geometry, draw-run and shader levels of detail that fit the frame. |
| [The Materials Viewer](../Tools/MaterialsViewer.md) | The live material preview as a real scene, the thumbnail render farm, and the hot-reload round trip. |

Two tools pages cover the renderer too: [Profiling](../Tools/Profiling.md) for the GPU
zones each pass type gets, and [Debugging](../Tools/Debugging.md) for frame captures and
the device-removed report.

Next: [Render Passes & Visibility](RenderPasses.md) for how a frame is assembled,
or [Render Devices & Backends](RenderDevices.md) for where it is submitted. Back
to the [documentation index](../README.md).
