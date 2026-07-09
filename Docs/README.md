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
pages rather than kept in a separate scoreboard.

## Core engine

| Document | What it covers |
|----------|----------------|
| [Core](Core.md) *(draft)* | The foundation: STL-shaped custom containers (contiguous by default), scope allocators, the `data` table store, metadata, delegates, resources, hot reload, threading and tasks, profiling, time, strong types, handles, and hashing. |
| [Materials](Materials.md) *(draft)* | The standout: materials authored as Lua programs with embedded HLSL, inheritance, script-driven constants, and content-addressed shader/constant dedup. |
| [Script Generation](ScriptGeneration.md) *(draft)* | One source of truth for C++, C#, and Lua: how the generator turns exported headers into interop bindings and reflection metadata. |
| [Component Architecture](Components.md) *(draft)* | Swappable implementations behind interfaces, with instances authorable in C++, C#, or Lua, and the generic-CID + alias pattern that picks a backend at runtime. |
| [Metadata & Reflection](Metadata.md) *(draft)* | Single Read/Write points as the foundation for the PropertyGrid, the JSON/binary/network serializers, and built-in undo/redo. |
| [Networking](Networking.md) *(draft)* | One field-level change-capture layer feeding both real-time multiplayer and live collaborative editing: transports, reliable and unreliable channels, delta snapshots against acked baselines, interest management, and the star-relay host. |

## Graphics

| Document | What it covers |
|----------|----------------|
| [Materials](Materials.md) *(draft)* | (see above) |
| [Post-Processing & HDR](PostProcessing.md) *(draft)* | The data-driven post chain (bloom, depth of field, tonemapping, lens flare, 3D-LUT grading) and SDR / HDR10 / scRGB output. |
| [Rendering & Visibility](Rendering.md) *(draft)* | Pluggable render and visibility backends (D3D12 and Vulkan today), and how surfaces get culled and drawn. |

## Tools and workflow

| Document | What it covers |
|----------|----------------|
| [Studio](Studio.md) *(draft)* | Why the editor *is* the runtime: build documents, text documents backed by real language servers, the PropertyGrid, cross-language plugins, the gizmo, and action-based undo/redo. |
| [Brush Editing](BrushEditing.md) *(draft)* | The plane-set solid brush editor: planes as the source of truth with geometry derived, CSG operations that are just plane arithmetic, and the content-hashed caches that make a thousand-brush level interactive. |
| [The Materials Viewer](MaterialsViewer.md) *(draft)* | The live material preview as a real self-ticking scene: one viewport-interaction rig shared with the 3D view, a thumbnail-grid render farm, and the file-watcher hot-reload round trip. |
| [AI Integration](AiIntegration.md) *(draft)* | How the engine was made AI-aware for its own development, and the in-process MCP server that lets an embedded agent drive the live engine. |

## Deep dives

| Document | What it covers |
|----------|----------------|
| [Performance: The Road to 100k Boids](Performance_100kBoids.md) *(draft)* | Taking a flock from an 8k all-pairs wall to 100,000 fully-featured entities rendered live in 2D and 3D views at once. |
| [AI-Assisted Engine Development](AI_Assisted_Engine_Development.md) | A 20-week case study of building a serious C++ codebase with Claude Code: conventions, autonomy, and what actually moved the needle. |

---

## How to read this

Each document stands on its own, but they build on a few ideas introduced in
[Philosophy](Philosophy.md): read that first if you read only one. The graphics
and tools pages assume the core concepts (handles, components, metadata) but link
back where it matters.

Diagrams render inline (GitHub draws Mermaid natively). Where you see a note about
an image or clip, that is a placeholder for media being gathered.

Built by [Instinct](https://instinct-ware.com).
