# Ceili Documentation

The engineering story behind Ceili: how a from-scratch C++ engine reaches the
performance tier of the industry's leading engines, and the design decisions that got
it there.

These documents double as **learning material** and a look under the hood. They
are written to be read by someone who builds engines, or wants to. Code and data
fragments are real, pulled from the engine itself. The source is not open yet;
these docs share the *how* and *why* ahead of the code.

> **Status:** the engine is in daily development; this documentation set is being
> written alongside the run-up to the open-source release. Pages marked *(draft)*
> are written and in review; expect polish and added media before launch.

<!-- MEDIA: a single wide banner image or a short montage GIF/video of the engine
     across a few subsystems (Studio editor, a material preview, the 100k flock)
     would sit well here. -->

---

## Start here

| Document | What it covers |
|----------|----------------|
| [Philosophy](Philosophy.md) *(draft)* | The founding bets: performance is king, the runtime *is* the tools, self-contained packages, plug-and-play backends, C+-, handles over objects, and a testing ethos. |

Where Ceili stands against the field (the commercial heavyweights, the
open-source peers, the data-oriented ECS stacks) is woven into the relevant
pages, and no separate scoreboard is kept.

## Core engine (`Core/`)

| Document | What it covers |
|----------|----------------|
| [Core](Core/Core.md) *(draft)* | The foundation: STL-shaped custom containers (contiguous by default), scope allocators, the `data` table store, metadata, delegates, resources, hot reload, threading and tasks, profiling, time, strong types, handles, and hashing. |
| [Script Generation](Core/ScriptGeneration.md) *(draft)* | One source of truth for C++, C#, and Lua: how the generator turns exported headers into interop bindings and reflection metadata. |
| [Component Architecture](Core/Components.md) *(draft)* | Swappable implementations behind interfaces, with instances authorable in C++, C#, or Lua, and the generic-CID + alias pattern that picks a backend at runtime. |
| [Metadata & Reflection](Core/Metadata.md) *(draft)* | Single Read/Write points as the foundation for the PropertyGrid, the JSON/binary/network serializers, and built-in undo/redo. |
| [Networking](Core/Networking.md) *(draft)* | One field-level change-capture layer feeding both real-time multiplayer and live collaborative editing: transports, reliable and unreliable channels, delta snapshots against acked baselines, interest management, and the star-relay host. |
| [The .NET Runtime on Core](Core/DotNetRuntime.md) *(draft)* | C# compiled ahead of time to C, on a vendored NativeAOT runtime and GC that depend on Ceili Core alone: the pipeline, the Core APIs the rule produced, the shadow-stack GC roots, and the phase gates. |

## Rendering (`Rendering/`)

| Document | What it covers |
|----------|----------------|
| [Rendering](Rendering/README.md) *(draft)* | The map of the renderer: its plug-and-play parts (backends by alias, passes as components, forward or deferred and PBR or legacy Blinn switched at runtime) and a guide to every rendering page. |
| [Render Devices & Backends](Rendering/RenderDevices.md) *(draft)* | One device interface over D3D12, Vulkan (Full or Compat tier), a console backend and a headless Null device; the frame clock, the record-time ring, and texture creation. |
| [Render Passes & Visibility](Rendering/RenderPasses.md) *(draft)* | `IRender` and its 46 implementations, pass types as the frame's order, the producer stage, runtime-switched forward and deferred, `IVisibility` providers that cull every frustum in one call, the spatial index backends, and the compiled world's selection. |
| [Surfaces & Draw Data](Rendering/Surfaces.md) *(draft)* | Surfaces as data, the hot-cold visibility rows, one generic refresh system, instance uploads, and render interpolation between fixed ticks. |
| [Compiled Worlds](Rendering/CompiledWorlds.md) *(draft)* | A ported level's kd-tree, BSP, areas and portals in one block; the portal walk; the indoor and outdoor classification and how it governs the sky's light, sky lights and weather indoors, while the sun is shadowed by geometry. |
| [Materials](Rendering/Materials.md) *(draft)* | Materials authored as Lua programs with embedded HLSL, inheritance, script-driven constants, and content-addressed shader/constant dedup. |
| [Lighting](Rendering/Lighting.md) *(draft)* | Three lighting paths in one frame: per-light forward, a clustered light grid, and a deferred G-buffer. The measured reason each exists, and the parity gates that hold them together. |
| [Shadows](Rendering/Shadows.md) *(draft)* | Cascaded directional shadow maps: a per-scene 2x2 atlas, the cascade fit and its quantisation, the multi-frustum cull, the bake siblings, two bias mechanisms, and a cache measured at a twelfth of the bake cost. |
| [Atmosphere](Rendering/Atmosphere.md) *(draft)* | A physical sky in real units: Hillaire's model, transmittance and multiple-scattering tables baked only when the air changes, a froxel volume that hazes the world with the sky's own integrator, a local dust layer, the time-of-day sun, and the horizon seal. |
| [Meshes & Animation](Rendering/MeshesAndAnimation.md) *(draft)* | glTF and legacy meshes, one entity with many drawable items, skeletons and clips, joint masks, an animation state machine authored as an asset, root motion, and skinning. |
| [Post-Processing & HDR](Rendering/PostProcessing.md) *(draft)* | The data-driven post chain (auto-exposure, bloom, depth of field, tonemapping, lens flare, a graded look), post-stage fusion that draws adjacent per-pixel stages as one pass, and SDR / HDR10 / scRGB output. |
| [Weather](Rendering/Weather.md) *(draft)* | One row of weather facts read by every effect: presets that leave the level's fields alone, the overhead depth map and the drifting weather map, clouds drawn per pixel or marched, rain, lightning as published facts, a Gerstner ocean that gameplay can query, and the scatter system that dresses the ground. |
| [XR and Quest](Rendering/Xr.md) *(draft)* | Stereo on a Meta Quest through OpenXR: a loader opened at runtime, a frame loop woven into record and replay, one camera's union view split into two eyes, simulated stereo on the desktop, a Java-free Android app, and measuring on the device. |

## Gameplay subsystems (`Gameplay/`)

Three young subsystems, stood up in the summer of 2026, documented with their
honest limits alongside what runs.

| Document | What it covers |
|----------|----------------|
| [Physics](Gameplay/Physics.md) *(draft)* | A trunk-plus-backend physics subsystem over a vendored deterministic library: a capsule character controller with collide-and-slide, step-up and jump, box / sphere / capsule / hull colliders with a static mesh, fixed and spherical joints, synchronous ray and shape queries, and a settings-toggled debug view. |
| [Audio](Gameplay/Audio.md) *(draft)* | Entity-authored sound over a trunk-plus-backend design: a Sound row that plays through miniaudio with looping, decibel volume, per-play variation, and 3D attenuation, plus the headless exit-deadlock that shaped its null-device path. |
| [Particles](Gameplay/Particles.md) *(draft)* | A world-space CPU particle simulation inside Graphics: a database-backed store sliced per emitter, colour and size ramps, finite one-shots, and a camera-facing billboard pass. |

## Tools and workflow (`Tools/`)

| Document | What it covers |
|----------|----------------|
| [Building Ceili](Tools/Building.md) *(draft)* | The engine builds itself: a frozen Bootstrap copy runs GENie inside the engine's own Lua VM, one configuration table drives the command line, the scripts and Studio's persisted Code Build document, and the check tiers and generated AI context map that serve humans and agents alike. |
| [Debugging](Tools/Debugging.md) *(draft)* | Failures that explain themselves: the per-run log, Standard-tier minidumps and hung-process stack capture, screen captures, GPU frame captures placed from code and surveyed with RenderDoc's Python, the D3D12 debug layer, and the device-removed report that names the stalled draw and its frame. |
| [Profiling](Tools/Profiling.md) *(draft)* | Two clocks kept apart, and one profiler backend per build (the internal profiler, or Tracy in the Profile build): the CPU zone tree over per-thread event queues, the GPU timestamp registry whose readback lags by frames in flight, the interleaved A/B harness that makes a comparison trustworthy, and the memory panel's scope, heap, container and GPU-arena views. |
| [Studio](Tools/Studio.md) *(draft)* | Why the editor *is* the runtime: build documents, text documents backed by real language servers, the PropertyGrid, cross-language plugins, the gizmo, and action-based undo/redo. |
| [Brush Editing](Tools/BrushEditing.md) *(draft)* | The plane-set solid brush editor: planes as the source of truth with geometry derived, CSG operations that are just plane arithmetic, and the content-hashed caches that make a thousand-brush level interactive. |
| [The Materials Viewer](Tools/MaterialsViewer.md) *(draft)* | The live material preview as a real self-ticking scene: one viewport-interaction rig shared with the 3D view, a thumbnail-grid render farm, and the file-watcher hot-reload round trip. |
| [AI Integration](Tools/AiIntegration.md) *(draft)* | How the engine was made AI-aware for its own development, and the in-process MCP server that lets an embedded agent drive the live engine. |

## Case studies (`CaseStudies/`)

| Document | What it covers |
|----------|----------------|
| [Performance: The Road to 100k Boids](CaseStudies/Performance_100kBoids.md) *(draft)* | Taking a flock from an 8k all-pairs wall to 100,000 fully-featured entities rendered live in 2D and 3D views at once. |
| [AI-Assisted Engine Development](CaseStudies/AI_Assisted_Engine_Development.md) | A case study of building a serious C++ codebase with Claude Code: conventions, autonomy, and what actually moved the needle. |

---

## How to read this

Each document stands on its own, but they build on a few ideas introduced in
[Philosophy](Philosophy.md): read that first if you read only one. The rendering
and tools pages assume the core concepts (handles, components, metadata) but link
back where it matters.

Each table names the folder its pages live in. The `Rendering/` folder opens on
its own map of the renderer, with a summary of every rendering page.

Diagrams render inline (GitHub draws Mermaid natively). Where you see a note about
an image or clip, that is a placeholder for media being gathered.

Built by [Instinct](https://instinct-ware.com).
