# AI-Assisted Engine Development: Lessons from Building a Game Engine with Claude

*How investing in conventions, documentation, and tooling - and then in autonomy - turns AI from a novelty into a genuine development multiplier.*

*Dermot Gallagher, [Instinct](https://instinct-ware.com). Updated July 23, 2026 - the 22-week mark. The original edition covered the first 8 weeks; this edition extends through 22 weeks and adds the workflow changes that, as much as anything, drove the later work. Part of the [Ceili engine](https://github.com/instinct-ware/ceili) project.*

## The Setup

Ceili is a C++17 game engine targeting Windows, with D3D12 and Vulkan backends, C#/.NET and Lua scripting, and a custom studio/editor. When the first edition of this post was written it was around 60K lines (strictly code, without blanks and comments).
Twenty-two weeks in, a line counter reports **228,047 hand-authored lines** across more than two dozen packages - C++, C#, Lua, the material and entity authoring DSLs, and HLSL - with blanks and comments excluded and generated code excluded. Counting the generated interop bindings as well takes it to **397,178**, and there are a further 19,682 lines of documentation and 47,127 lines of session prompt/design notes alongside it. It ships with its own embedded Claude agent, reachable over MCP, that can drive the engine's scripting API; it has a real-time multiplayer layer backed by a real-UDP transport; and it now has a full plane-set brush editor with CSG. It is built primarily by one developer with Claude Code as a regular collaborator.

This post covers what's worked, what hasn't, and what you can do to get the most out of AI-assisted development on a serious codebase. The first half - conventions, documentation, build verification - is the foundation, and it held up. The newer lesson, and the throughline of the second 8 weeks, is autonomy: giving the AI the tooling to verify, navigate, and debug its own work without a human in the loop for every cycle.

## CLAUDE.md Is Your Most Important File

Claude Code reads a `CLAUDE.md` file at the root of your project for context. Most people treat it as a brief note, "this is a React app, use yarn." That's leaving most of the value on the table.

For Ceili, CLAUDE.md is the project's primary developer documentation. It covers:

- Architecture philosophy and design decisions
- Package dependency rules
- Naming conventions (a full table mapping element types to patterns)
- Code style rules (brace style, section headers, include ordering)
- File organization rules (declaration order in headers vs. implementations)
- Error handling patterns
- Component system registration and lifecycle
- Container types and their STL equivalents
- Key macros and their usage (including quirks like semicolon rules)
- Build system commands and known workarounds

This isn't documentation written *for* the AI. It's documentation that happens to be in a file the AI reads automatically. The same document serves any human contributor equally well.

### The Dual Benefit

By investing in AI-assisted development, you're forced to write down things that most small-team projects never document. Naming conventions, macro semantics, parameter ordering rules, dependency boundaries, these usually live in one developer's head. Making them explicit for the AI makes them explicit for everyone.

It also keeps the documentation honest. If CLAUDE.md says "Allman braces, two blank lines between functions" and the AI follows it but your existing code doesn't match, the mismatch surfaces immediately. The AI becomes a continuous compliance checker against your documented conventions.

### What to Put in CLAUDE.md

Prioritize information that prevents wrong-on-first-attempt mistakes:

**High value:**
- Naming conventions with concrete examples
- File/directory structure and where things go
- Dependencies and package boundaries ("don't add cross-package includes without asking")
- Build commands the AI can run to verify its work
- Your type system mapped to familiar equivalents (custom containers to STL, etc.)
- Architecture patterns ("common code goes in the trunk, not the backends")

**Moderate value:**
- Code style rules (the AI will mostly infer these from existing code, but explicit rules prevent drift)
- Error handling patterns
- Macro usage and quirks

**Lower value:**
- Detailed rationale for *why* decisions were made (the AI needs to know *what* to do, not necessarily *why*)
- Historical context about the project

### Keep It Concise

CLAUDE.md works best as a dense reference, not a narrative. Use tables for conventions. Use short bullet points for rules. Link to deeper docs rather than inlining everything. The AI reads this on every conversation, make it scannable.

## Convention Rigidity Is a Feature, Not a Bug

Game engines are large, complex codebases. The natural tendency is toward flexibility, let each subsystem do what makes sense locally. For AI-assisted development, the opposite is true: **the more rigid and predictable your conventions, the better the AI performs.**

Ceili enforces:

- Every module follows the same directory layout (Module.h, ModuleFactories, Include/, Src/, Tests/)
- Every header follows the same declaration order (types, structs, interfaces, accessors, functions)
- Every implementation follows the same declaration order (types, globals, internal functions, accessors, functions)
- Section headers use consistent `//-----` delimiters with labels
- Naming uses predictable prefixes (CID_ for components, IID_ for interfaces, m_ for members, g_ for globals, k for constants)

Once the AI understands one module, it understands them all. This dramatically reduces the context needed per task. When the AI needs to add a new component, it doesn't have to study how *this particular module* does things, the answer is the same everywhere.

### The Custom Type Problem

One real tension: Ceili uses no STL. Custom containers, custom strings, custom allocators, custom type traits. Every one of these is a departure from the AI's training data, which is dominated by STL patterns.

The solution was adding a container mapping table to CLAUDE.md:

```
| Ceili     | STL equivalent                | Notes                    |
|-----------|-------------------------------|--------------------------|
| Array     | std::vector                   | Dynamic or fixed size    |
| FlatMap   | std::flat_map                 | Keys sorted, values in   |
|           |                               | separate contiguous mem  |
| MapArray  | std::map / std::unordered_map | Key/value sorted on key  |
| String    | std::string                   |                          |
| Pool      | (none)                        | Push/pop, avoids realloc |
```

This eliminates an entire class of "reflex errors" where the AI reaches for std::vector and has to be corrected.

**If your project has custom alternatives to standard library types, document the mapping explicitly.** This is one of the highest-value additions you can make to CLAUDE.md.

## Build Verification Is Essential

An AI writing code it can't compile is guessing. An AI that can build and test after every change is iterating.

For Ceili, this meant creating helper scripts that work from Claude Code's bash environment:

- `Claude_Build_Debug.cmd` - compile and check for errors
- `Claude_UnitTests.cmd` - run the test suite
- `Claude_Generate.cmd` - regenerate build projects (when files are added/removed)
- `Claude_Clean.cmd` - full clean (when builds fail unexpectedly)

This sounds trivial, but there were real obstacles. The engine's main executable is a GUI subsystem app, its stdout isn't capturable from bash. The solution was a separate console-subsystem executable that runs the same code. These kind of environment-specific workarounds are worth documenting in CLAUDE.md so the AI doesn't rediscover them every session.

**The ideal is a single fast-check command** that builds and runs tests without cleaning or regenerating. For Ceili this converged on one script, `Claude_FastCheck.cmd`: it builds the Debug configuration and, only if that succeeds, runs the unit-test suite - which includes scripting tests that exercise the C++, C#, and Lua bindings in one pass. Most AI changes don't add or remove files, so skipping generation saves significant time. The faster the feedback loop, the more confidently the AI can work.

### The C++ Tax

Compared to Python or JavaScript where you get near-instant feedback, C++ compilation latency is a real cost. A full rebuild takes minutes. This means:

1. Getting things right on the first attempt matters more
2. Convention documentation pays for itself faster (fewer compile-fix-compile cycles)
3. Incremental builds are important, structure your project so a change in one module doesn't rebuild everything
4. Unit tests for the components the AI is most likely to modify are disproportionately valuable

## Closing the Loop: From Verification to Autonomy

Build verification was the lesson of the first 8 weeks. The lesson of the second 8 was what surrounds it: the difference between an AI that can verify its work *when asked* and one that verifies, navigates, and diagnoses *on its own* is the difference between a fast assistant and an autonomous collaborator. Three changes did most of that work.

### LSP, Not Grep

Claude Code can navigate a codebase with text search, and for a while that was the only option. The bigger unlock was wiring it to the Language Server Protocol - the same machinery an IDE uses. Instead of grepping for a function name and guessing which hit is the definition, the AI calls `goToDefinition`, `findReferences`, `workspaceSymbol`, and reads `hover` types directly. Before renaming a function it pulls the real reference list; after an edit it checks diagnostics instead of waiting for a full compile.

This matters more on a codebase like Ceili than on a typical one. With no STL and a wall of custom containers, allocators, and macros, text search is noisy - half the "matches" for a common name are unrelated. Symbol-aware navigation cuts straight through that. It also catches the expensive class of mistake early: a signature change that breaks six call sites shows up in `findReferences` in seconds, not three minutes into a build.

There is a pleasing recursion here. One of the features built during these 16 weeks was the engine's own LSP *client* - a from-scratch Language Server Protocol implementation in C++ so the Studio editor gets Go To Definition and hover for Lua, C#, and inline HLSL. Claude Code, meanwhile, independently uses LSP to navigate the very code that implements it.

### Logs and Minidumps: Let the Program Explain Itself

The single biggest enabler of autonomous debugging was unglamorous: making every executable write down what happened.

Every binary that runs now produces a per-run text log under `Logs/` - the full transcript, including the entire boot sequence, flushed to disk after every write so the tail survives a crash. And every binary installs a top-level exception filter that writes a minidump under `Dumps/` on an unhandled exception; assertions and critical-log calls write one inline at the failure site too. A `-noassertdialog` flag makes a tripped assert write its dump and exit instead of popping a modal dialog, so headless and scripted runs never hang waiting on a human.

The payoff: when a test fails, the AI doesn't ask the developer to reproduce it. It lists the newest files in `Dumps/` and `Logs/`, greps the log for the errors leading up to the exit, and loads the dump into the debugger for a stack trace - all from the failure that just happened. The log shows what led up to the crash; the dump shows the stack at the crash. That pair is the standard autonomous-debugging primitive, and it turns "it crashed, can you run it again with more logging?" into a diagnosis in a single turn.

If you take one thing from this section: **make your program explain its own failures.** Persistent logs and on-crash dumps are worth more to an autonomous agent than almost any amount of cleverness, because they convert "reproduce and observe" - the slow, human-in-the-loop step - into "read a file."

### Offload the Heavy Lifting to a Cheaper Model

Long sessions on a C++ codebase accumulate context: build logs, stack traces, search results, file dumps. There is a real cost to letting that pile up in the main conversation - on every subsequent turn, the model re-reads it. A 50KB build log sitting in the primary context is paid for again and again.

The fix is delegation. Routine, verbose work - running the fast-check build and tests, triaging the newest dump-and-log pair, running the code generators, fanning out a multi-file search - is handed to a Sonnet subagent. The subagent does the work in its own context and returns only a verdict: passed, or failed with the specific evidence. The 50KB log never touches the main thread. The more capable, more expensive model stays in the main loop for the things that genuinely need the whole session in view: planning, architecture decisions, sweeping cross-package refactors, and the kind of debugging that needs the full history.

The same pattern covers research. Before designing a subsystem it's often worth surveying how the problem is solved elsewhere - how other engines structure their render graph, how the literature handles HDR tonemapping - and that research is verbose and ephemeral. Run it on a subagent; keep the conclusion, discard the transcript. This very document was assembled that way: while the main thread wrote, Sonnet subagents mined the full commit log and the session-transcript statistics in parallel, returning compact tables rather than raw logs.

## The Loop Closes: The Engine Gets Its Own Agent

There is a fitting endpoint to all of this. During these 16 weeks the engine grew its own AI integration. An `Agent` package defines an `IAgent` interface; an `AgentClaude` backend spawns the `claude` CLI as a subprocess and reads its streaming responses; and the engine hosts an MCP server in-process that exposes its own scripting API as callable tools. The tool catalogue is generated automatically - the same script generator that produces the C, C#, and Lua bindings also emits a per-package API catalogue, with parameter names and doc-comments, that the agent reads to discover what it can call.

So you can ask the in-engine assistant to preview the PBR materials, adjust the atmosphere, or tweak the post-process chain, and it executes Lua against the live engine to do it. Claude Code builds the engine; the engine ships with Claude inside it.

This had an unexpected design benefit. Exposing the API to an embedded agent is a forcing function for exposing it well: the catalogue that tells the agent what each function does is the same documentation a human reads. Making the engine legible to its own agent made it more legible, full stop.

## Architecture Decisions That Help (And Hinder) AI

### C-Like Public APIs

Ceili uses C-like function APIs with opaque handles at its public boundary, with C++ implementation behind them. This has an interesting effect on AI-assisted development.

The C-like API doesn't reduce the language complexity the AI encounters when writing implementation code, that's still full C++ with templates, inheritance, and custom allocators. But it **constrains where the complexity lives**. The complex C++ is scoped behind clean function boundaries rather than leaking across the entire surface area.

More importantly, the C API forces explicit, documented interfaces. A C++ class can accumulate methods and implicit conversions until its actual interface is unclear. A C-like API makes the boundary unambiguous, the AI can look at the exported functions and know exactly what a subsystem offers.

### Package Dependency Rules

This is a lesson learned the hard way. During an ImGui multi-viewport implementation, the AI tried introducing a dependency from the graphics backends (GraphicsD3D12, GraphicsVk) to the UI package. The dependency graph already had an established route, shared UI code lives in the Ui package, which depends on Graphics, not the other way around.

The fix was adding an explicit rule to CLAUDE.md:

> **Do not introduce new inter-package dependencies without asking.** The package dependency graph is well established. If you need functionality from another package, there is likely an existing interaction route.

The AI is good at finding the shortest path to a solution. Sometimes the shortest path violates architectural boundaries that exist for good reasons. **Documenting these boundaries prevents the AI from taking shortcuts that create long-term problems.**

### The Component/Manager Pattern

Ceili uses a trunk-and-leaf pattern: common code lives in a manager package (e.g., Graphics), which forwards to backend-specific implementations (e.g., GraphicsD3D12, GraphicsVk). Adding this pattern to CLAUDE.md means the AI knows to put shared logic in the trunk rather than duplicating it across backends.

This is a general principle: **any architectural pattern that determines *where* code goes should be documented.** The AI can write correct code, but it needs guidance on correct *placement*.

## What the AI Is Good At

Based on experience across many sessions:

- **Following established patterns.** Give it one example component and it can create more that follow the same structure perfectly.
- **Mechanical refactoring.** Renaming across files, updating signatures, propagating changes through bindings.
- **Reading and navigating large codebases.** It can grep, read, and cross-reference faster than manual browsing.
- **Writing boilerplate.** Component registration, factory definitions, script export markers, the repetitive infrastructure that's easy to get wrong by hand.
- **Implementing well-specified features.** When the requirements are clear and the patterns are documented, the AI can produce production-quality code on the first attempt.

## What the AI Struggles With

- **Architectural decisions.** It will find *a* solution, but not necessarily the one that fits your architecture. It needs documented boundaries and patterns.
- **Custom type semantics.** Without the STL mapping table, it regularly reaches for std::vector patterns that don't apply to custom containers.
- **Cross-module implications.** A change in one package may affect others through interfaces the AI hasn't read. Explicit dependency documentation helps.
- **Build system nuances.** GENie/Premake/CMake configurations, platform-specific compilation flags, DLL export macros, these are under-represented in training data for custom engines.
- **Knowing when to ask.** Left to its own devices, the AI will make a reasonable-looking decision rather than asking. Rules like "don't add dependencies without asking" convert silent mistakes into conversations.

## Practical Recommendations

If you're starting your Claude Code journey with a serious codebase:

1. **Write CLAUDE.md first, and write it well.** This is your highest-leverage investment. Treat it as the developer guide you wish existed. The AI reads it every session, everything in there compounds.

2. **Document your conventions as rules, not suggestions.** "Prefer PascalCase" is weaker than a table mapping every element type to its naming convention with examples.

3. **Map custom types to familiar equivalents.** If you have custom containers, strings, or error types, create an explicit mapping table to their standard library counterparts.

4. **Create fast build/test scripts the AI can run.** The difference between "write code and hope" and "write code and verify" is enormous. Make the verification path as fast as possible.

5. **Document architectural boundaries explicitly.** Package dependencies, the trunk/leaf pattern, where shared code goes, anything that determines code *placement* needs to be written down.

6. **Add rules based on real mistakes.** When the AI makes an architectural error, don't just fix it, add a rule to CLAUDE.md that prevents it from happening again. Your CLAUDE.md should grow from experience.

7. **Keep module structure predictable.** The more consistent your directory layout and file organization, the less context the AI needs per task. Rigid conventions are a feature.

8. **Invest in unit tests for AI-touched code.** Tests are how the AI validates its own work. Focus coverage on the components and patterns the AI is most likely to modify.

9. **Use CLAUDE.md as living documentation.** It's not a one-time setup. Update it as conventions evolve, as you discover new failure modes, and as the codebase grows. The AI and your future self both benefit.

10. **The AI is a collaborator, not an autopilot.** It excels at implementation when given clear architectural guidance. The developer's role shifts toward design decisions, boundary enforcement, and quality review, with the AI handling the mechanical work at a pace no human can match.

Once that foundation is in place, the next increment is autonomy:

11. **Make your program explain its own failures.** Persistent per-run logs and on-crash minidumps are the highest-leverage investment for autonomous debugging. They convert "reproduce and observe" into "read a file" - which is the step the AI can do without you.

12. **Wire the AI to LSP, not just text search.** Symbol-aware navigation - go-to-definition, find-references, diagnostics - is far more reliable than grep on a large codebase, especially one with custom types that pollute text matches. It also catches signature-change fallout before you spend a build finding it.

13. **Offload verbose, routine work to a cheaper model.** Builds, test runs, crash triage, code generation, broad searches, background research - delegate these to a subagent that returns a verdict, not a transcript. The main session stays lean and fast, and you only pay for the heavy output once.

## 22-Week Results: The Numbers

Ceili began using Claude Code on February 14th, 2026. The first edition of this post covered the 8 weeks through April 12th. This edition extends the count through July 23rd - the 22-week mark.

### Throughput

Lines touched (insertions + deletions) across source files (C++, HLSL, Lua, C#), excluding:
- Generated script bindings (ScriptingC, ScriptingDotNet, ScriptingLua) and their refreshed copies under the frozen Bootstrap
- 3rdParty additions/removals and vendored tooling (ImGui binding replacement, the bundled LSP servers, the netcode library, stb_image, etc.)
- Large mechanical changes (the IE4 -> Ceili engine-wide rename in the first period)

Every row below was re-measured for this edition with one uniform command, so the
figures differ slightly from the previous edition's, which mixed two methods. Where
a single commit was a vendored drop or an engine-wide rename it is excluded whole,
which costs the first block some genuine work and is the conservative direction to
err in.

| Period | Days | Lines touched | Per week |
|--------|------|--------------|----------|
| Dec 6 - Dec 31 (pre-Claude baseline) | 26 | 4,883 | ~1,315/wk |
| Jan 12 - Feb 10 (pre-Claude baseline) | 30 | 7,769 | ~1,812/wk |
| Feb 14 - Mar 15 | 30 | 65,921 | ~15,381/wk |
| Mar 16 - Apr 14 | 30 | 106,732 | ~24,904/wk |
| Apr 15 - May 14 | 30 | 97,517 | ~22,754/wk |
| May 15 - Jun 13 | 30 | 127,630 | ~29,780/wk |
| Jun 14 - Jul 13 | 30 | 156,113 | ~36,426/wk |
| Jul 14 - Jul 23 (partial block) | 10 | 122,651 | ~85,856/wk |
| **Total (Feb 14 - Jul 23)** | **160** | **676,564** | **~29,599/wk avg** |

Five complete 30-day blocks, and after the first one the trend is monotonic: every
block beats the one before it. Against the stronger of the two pre-Claude baselines
(~1,812 lines/week), the run averages about **16x**, opening at roughly 8x while the
conventions and build tooling were still being built out and reaching **20x** in the
most recent full block. The acceleration has been sustained rather than spiky.

The 10-day tail is an outlier and should be read as one. It covers a codebase-wide
refactor audit (described below) that ran at 130 to 173 commits a day, so its
~86k/week rate reflects a mechanical sweep across many files rather than a new
steady state. It is reported here because omitting it would be worse, not because
it is a rate anyone should extrapolate.

### Sessions

Session transcripts older than about a month rotate off disk, so a clean 22-week count isn't recoverable. The window still on disk is 31 days (June 23 to July 23) and holds **243 sessions** for this project alone - against 139 counted across the entire first 8 weeks. Roughly eight sessions a day, sustained. Sessions skew heavily toward the large end: the multi-hour architecture and implementation sessions account for most of the output, while a long tail of tiny transcripts represents quick probes and questions that never grew into a full working session.

### Commits

**3,511 commits** since February 14th. The first 8 weeks produced 423; the last ten days alone produced 971. The jump reflects both pace and granularity: the refactor audit ran as one commit per audited item plus its verification, and the networking milestone before it paired phased code commits with per-session documentation commits recording hypotheses, root causes, and gotchas. The small-commit discipline held throughout: build-and-verify after each language boundary keeps changes bisectable and individually meaningful.

### What Got Built: Weeks 1-8 (Feb 14 - Apr 12)

The first eight weeks laid the foundation the rest of the engine stands on: the shader and material pipelines, an editor with real language-server support, and the first physically based rendering. The pace built through the block as the conventions and build tooling settled into place.

#### Foundation and the first material system (Feb 14-22)

The opening sprint cleared pre-existing debt before the velocity kicked in: the engine-wide IE4 -> Ceili identifier rename across every file, a round of PropertyGrid work (multidimensional array support, container insert and remove, undo/redo action descriptions carrying the full parent chain, and the crash fixes that came with them), and optional generation bits on the handle type. It also laid down the first data-driven material system, a handle-based API replacing the old approach, and with it the first Lua-authored materials.

#### The shader pipeline (Feb 22 - Mar 1)

The DirectX Shader Compiler was set up from scratch as a 3rdParty dependency, and a shader-compiler public API put the DXIL and SPIR-V backends behind one interface, with compilation running as part of the project build rather than by hand. Vertex and fragment stages were combined into a single `.shader` source file. The largest single move was replacing the ImGui binding stack: cimgui, ImGui.NET, and LuaJIT-ImGui (about 134k lines of generated bindings) came out, and roughly 8k lines of hand-written C++ wrappers went in.

#### Material compiler and an LSP from scratch (Mar 1-9)

DDS texture loading landed with explicit texture indices and sampler clamping, and shader constants became data-driven, defined in Lua material scripts instead of hardcoded. The material compiler was integrated into the build, firing before script generation, emitting Materials.hlsli for shaders to consume, and writing through a temp-file swap. The headline was a Language Server Protocol client implemented from scratch in C++ (JSON protocol, process API, pipe communication) and wired to clangd, OmniSharp, and a Lua server, with compile_commands.json generated from the project build. Go To Definition, hover, colour highlighting, and auto-complete all worked inside the engine's own text editor.

#### Editor and rendering foundation (Mar 9-21)

Inline shaders arrived: HLSL written directly inside `.material` Lua scripts, compiled on save, with LSP colour highlighting and Go To Definition working inside the inline blocks. The rendering side got initial render targets, a first draw-command-buffer API, and `draw::IRender`, the first proper render interface, with Ui as its initial implementation. A live material preview panel appeared in the Studio with its own camera, lighting, and geometry selection. Underneath, a lock-free profiler (SPSC recording with RDTSC timestamps) and a logging rewrite (a single circular buffer with lock-free reads) went in, and a ShaderToy example package made rapid shader prototyping easy.

#### Material system maturity (Mar 22-28)

Materials gained multiple render passes, each with its own textures and isolated constants. Shader constant storage became copy-on-write and ref-counted, so derived materials share a parent's constants until they diverge, and per-material constant defaults got PropertyGrid reset support. An annotation system surfaced range, description, and semantic metadata on shader constants in the grid. Hot reload now covered materials, shaders, and constants, refreshing metadata when a constant is renamed, and a `Resources<>::ExecuteLoad` helper made resource loads non-blocking.

#### PBR rendering (Mar 29 - Apr 12)

The first physically based rendering landed: a context-based light system (projection-vector computation, multi-light additive layering, per-light colour and specular power) feeding Cook-Torrance forward lighting on a metallic/roughness workflow. Image-based lighting used octahedral environment maps with GPU irradiance convolution and a separated IBL prepass, integrated with a day-sky atmosphere (exposure control, presets, sun-direction conventions). A material Flags system made the HDR toggle data-driven from scripts, and a depth pre-pass brought layered lighting to the preview.

### What Got Built: Weeks 9-16 (Apr 15 - Jun 13)

Where the first 8 weeks built the material and rendering foundation, the next eight built the systems around it: memory infrastructure, a full post-process chain, serialization, an entity/scene architecture, an AI agent inside the engine itself, and - as the capstone - a dependency-driven system scheduler with a live showcase that puts the whole stack under load. The work ran in several overlapping arcs rather than tidy weekly chunks.

#### Memory, scope allocators, and async loading (Apr 13 - May 3)

The arena-allocator strategy got real. A scope allocator with checkpoint/rewind and finalizers, a linked-storage mode for the core containers (blocks chained rather than one contiguous buffer), and a per-frame thread-local allocator together cut frame-scope allocation churn from ~13.9 GB/frame to ~60 MB/frame. Resource loading went fully asynchronous - textures, shaders, scripts, and .NET assemblies all load off-thread, with a lock-free slot pool and de-duplication of concurrent loads of the same file. Memory, materials, and texture analytics panels were added to the Studio to make all of this visible.

#### Crash and log infrastructure (May 2)

The logs-and-minidumps machinery described earlier landed here: the top-level exception filter, inline dumps at assertion and critical-log sites, per-run log files flushed to disk, and `-noassertdialog` for headless runs. It is worth listing twice - it's both a feature and the thing that made the rest of the half go faster.

#### The HDR post-process pipeline (May 3 - May 11)

A complete, data-driven post-process chain: bloom (mip-chain), depth of field, vignette, chromatic aberration, a lens flare with ghosts, a rotated starburst and lens dirt, and a runtime-baked 3D LUT for color grading (with `.cube` file override). Output modes for SDR, HDR10, and scRGB were threaded through both the D3D12 and Vulkan backends, including swapchain recreation and per-mode pipeline-state dispatch. Chain entries cross-fade via per-entry constant blending. The Studio got a post-process chain editor with per-entry constant pickers and full undo/redo, and the whole thing is covered by 23 unit tests with ~1,200 assertions.

#### An AI agent inside the engine (May 12 - May 20)

The most novel addition. New `Agent`, `AgentClaude`, `Network`, and `Mcp.Net` packages give the engine its own Claude integration: `AgentClaude` spawns the `claude` CLI as a subprocess and reads its streaming JSON; the engine hosts an MCP server that exposes the scripting API as tools; and a `ScriptingApi` package gathers the per-package API catalogues (with doc-comments and parameter names harvested from the headers) that the agent reads to discover what it can call. The agent executes Lua directly in the live engine VM, so it can manipulate lights, surfaces, atmosphere, and the post-process chain. A Studio "AI Assistant" panel drives it. (This is the dogfooding loop described above.)

#### Serialization from scratch (May 22)

A `Serialize` package, built in a single focused day: an `IByteStream` abstraction, a `SerializerBase` walker that traverses composite types and containers, and JSON and binary backends sharing a per-type leaf-encoder registry. Leaf encoders cover the math types, strings, enums (via their underlying integer), nested containers, and time values; fields opt out with a `CE_NOSERIALIZE` flag. This is the spine of the current milestone - the property grid, serializers, and undo/redo - and it immediately fed scene save/load and the build-document format.

#### Scene, entities, and an ECS-style system architecture (May 23 - Jun 6)

The largest architectural arc. A data layer (a columnar store of typed-row tables joined by views, with a sparse-set record index, scope-allocated storage, key recycling, and a schema cache hardened for concurrent reads) underpins an entity system: entity *templates* authored in Lua, spawned into database rows, with deferred off-thread create/destroy drained on the main thread. Entity handles became a packed `{scene, key}` value, and scene rows (surfaces, lights, render targets) became RAII wrappers that own their handle.

On top of that sits an ECS-style ordered-system architecture, `scene::system::ISystem`. The Cull and Draw passes became ordered systems reading from the scene; a `Contract<T...>` mechanism declares each system's read/write set at compile time; a band scheduler runs independent systems in parallel by default; and systems can be implemented in C# or Lua directly, not just C++. Along the way the old `Scene` package was dissolved - its core relocated into `core::scene`, its graphics-coupled parts into `graphics::scene`, and the entity-template pipeline moved from a compile-time bake to runtime registration - templates now register from their `.entity` files the way materials register from theirs. Supporting all of it, `math` gained quaternion and transform primitives.

#### Studio and build-system polish (Apr 30 - May 22)

Exit codes now propagate cleanly from the .NET host through the app result out to the shell, so CI and scripted runs get a real pass/fail. The Studio gained native file dialogs (via COM), a most-recently-used document list, a single-file `.buildtasks` document format with full round-trip persistence, and JSON/Markdown viewing in the Explorer.

#### The system scheduler grows up - and a showcase to prove it (Jun 7 - Jun 13)

The ordered-system architecture got a real scheduler. The first version grouped systems into parallel *bands* separated by barriers: every system in a phase finishes before the next phase starts. This week that was replaced by a dependency-DAG scheduler. Instead of phase barriers, it derives a dependency graph directly from each system's declared read/write contract - the same `Contract<T...>` from the previous arc - and runs each system the moment its inputs are ready, overlapping work across what used to be hard phase boundaries. The band barrier was deleted outright. Systems now also dispatch off the main thread, frustum culling runs wide via a new index-sliced `ParallelFor`, and a frame-overlap mode runs the pre-cull work on a worker concurrently with the main-thread update. A snapshot/restore primitive captures scene data on entering Play mode and rolls it back on exit, so live simulation never corrupts the authored scene.

To exercise all of it there is now a boids showcase - thousands of agents flocking in real time. The flocking system is implemented three times over, in C++, C#, and Lua, and a new Studio "Systems" window swaps the live implementation with a single click: the same scene driven by any of the three languages through the identical `ISystem` interface. Rendering thousands of near-identical agents drove a GPU-instancing pipeline: a (material, primitive) batch cache, shared meshes collapsed by a refcounted geometry hash, per-draw UV transforms, and instance rings on both the D3D12 and Vulkan backends, with instanced forward-lit and IBL-prepass paths. Supporting the scale, per-frame visibility moved from table inserts to in-place row flags, batch building dropped from O(n^2) to O(n log M), and redundant per-surface material resolves collapsed from thousands per frame to about one per material.

This is the first workload that puts the data layer, the entity system, the scheduler, the cross-language scripting, and the renderer under load at once - and the clearest single demonstration of why the previous fifteen weeks of architecture were built the way they were.

### What Got Built: Weeks 17-20 (Jun 14 - Jul 13)

The theme of this block is multiplayer: a real-time, real-UDP co-op session for DogFighterX, built from the transport layer up to the in-game player prediction. The boids showcase from the previous block was already running; this block added a player-controlled plane alongside the flock, then made that plane work across two real machines over a live UDP connection.

#### Real-UDP transport and session manager (Jun 14 - Jun 21)

The `Network` package gained a real-UDP backend built on the mas-bandwidth/netcode library (bundled as a static lib with libsodium; the vendor drop itself is excluded from the line count as 3rdParty). A `NetworkNetcode` leaf package wraps the library behind the existing `INetTransport` interface, so the session manager - publish/connect/pump - runs identically whether the transport underneath is in-process loopback or real UDP sockets. The Studio got a C# "Co-op" panel for publishing and connecting scenes. A cross-process smoke test guards the publish/connect handshake, and Play-state synchronisation (server Play/Stop propagates to clients, client Play button locks while not hosting) landed alongside it.

#### Play-mode multiplayer - spawn, input, prediction (Jun 22 - Jul 4)

The multiplayer game layer built out in five phases over the following two weeks:

- **Phase 2 S1-S3:** `PlayerNetSystem` skeleton; server spawns a plane for each connecting client and streams the host's plane state; clients receive and mirror remote planes.
- **Phase 2 S4:** Server processes each client's uploaded `InputMsg` and applies it to the owned plane via `SetPlayerInputForKey`, giving the host authority over all planes.
- **Phase 2 S5 teardown:** On Play-to-Design mode transition, the server despawns all client planes (broadcasting the despawn to peers) and resets spawn state, so the next Play starts clean from the authored scene.
- **Phase 3 S1-S3 (client prediction):** The player entity gained a `count=2` component multiplicity substrate: a `kDisplaySlot` item that the client writes locally for zero-latency feel, and a `kAuthoritativeSlot` item that receives the server's truth. The client reconciles the display slot toward the authoritative slot each frame (`ReconcileToward`), and remote planes are interpolated onto their own display slot. `ApplyDelta`/`Restore` route the server state to the last item slot generically, leaving the display item untouched.
- **Phase 3 S4:** Client prediction fully closes the loop: the client writes its local input onto the owned plane's display `PlayerControl` so the global `PlayerControlSystem` predicts the display this frame (zero input latency), while the same input still streams up as an `InputMsg` for the authoritative slot. A unit test asserts the display leads the authoritative during the input-response transient, then converges after a zero-input settle.
- **Phase 3 S5:** `BoidsCameraSystem` now prefers the local player key over the lowest-key plane, so the trailing camera follows the plane this peer drives rather than whichever host plane happens to have the smallest key.

#### Entity template multiplicity (count=N) (Jun 19)

The double-buffer substrate above required entity templates to support spawning more than one item per component. A `count=N` attribute on a template component now spawns N identical items into the data row at spawn time, applied by broadcasting the one resolved default. A new `entity::GetItem<T>(hEntity, Index)` slot accessor exposes individual items; `Get<T>` remains equivalent to `GetItem<T>(.,0)`.

#### Generic VisibilityRefreshSystem (Jun 24)

Before this change, every entity type that moved needed its own `SystemVisibilityRefresh` companion to keep the render `worldMatrix` and cull `Visibility.world` in sync with the live `Transform`. The player plane, as a static surface with no such companion, rendered frozen at its spawn pose while the live Transform flew. The fix is one globally-scoped, change-detected system: `VisibilityGeometry` gains a `cachedTransform` (non-serialized); the system compares it against the live Transform each frame and recomposes only when they differ, so statics pay only a compare. New moving surfaces need no new companion - the single system covers all of them. The stop-gap per-entity companions were deleted.

#### Narrow-phase entity selection (Jun 26)

Studio entity selection previously stopped at the AABB broad phase. This block added a narrow phase: candidate entities are sorted by ray/frustum distance, then tested triangle-by-triangle against the actual mesh geometry. A bounded per-entity mesh cache avoids re-reading geometry every frame. Selection now reliably picks the clicked surface even when multiple entity AABBs overlap.

#### Per-mesh local Transform via nested .entity overrides (Jul 3-9)

The final plane-authoring arc resolved a longstanding gap: mesh orientation inside an entity could not be set from a `.entity` file because a nested struct sub-field was not reachable through the override walk. Two things landed together: `ApplyOverride` in `EntityCompiler.cpp` was extended to follow dotted field paths into nested structs, and `VisibilityGeometry` gained a `localTransform` field (offset, rotation, scale) that is composed at the single `ComposeWorldFromMatrix` funnel with an identity fast-path when untouched. The player plane nose now authors to point forward through a data-driven `VisibilityGeometry.localTransform.rotation` nested override rather than requiring a code change. The kd-tree re-seat was also hardened: moving a surface that was previously static correctly evicts and reinserts its kd-tree leaf, fixing a plane-vanishing bug on the first frame after a Transform update.

#### Server-authoritative swarm D9 replication (Jul 10-12)

With the player plane working in co-op Play mode, the boids flock followed. The D9 milestone adds a full server-authoritative replication and hit-detection pipeline for the swarm, built in slices:

- **Slice 1 - server kill by offset identity:** The server owns kill authority. A plane fires; the server determines which boid key to kill using an offset-identity match (position at time of fire rather than live position), removes it from the authoritative flock, and broadcasts the kill. Clients apply the kill on receipt. A determinism guard (`c373b28a3`) verified the wide-grid flock is partition-independent at 1,000 boids before live sync was wired.
- **Slice 2 - lag compensation and fire loop:** A per-server position-history ring buffers boid transforms at a configurable depth. On a fire event, the server rewinds to the client's reported fire tick, resolves the hit against the rewound state, and authors the kill. The Studio fire input closes the loop end-to-end in a single session.
- **Slice 3 - interest-scoped boid correction feed:** The client's local predictor drifts from the server's authoritative flock over time. A boid-correction stream replicates server positions back to each client in priority order, scoped by interest (distance/LOD tier). A temporal-delta + zigzag-varint codec (`d6337c0f2`) compresses the feed; per-key LOD tiers adjust reconcile rate and snap threshold; in co-op Play the owned client's population is frozen to the server's authoritative count on join.
- **Swarm desync fix (`1ea0efeb1`):** A subtle clock bug in the reconcile path compared the client's `Update.tick` against the server's `baseTick`, causing an unsigned underflow that skipped all reconcile targets. Fixed with a stale-check guard; the resync retry and count-scaled interest tolerances landed alongside it. Live-verified with two clients.
- **Reserved key pools and spectator mode:** The plane key and flock key spaces were colliding. Key 0 is the null sentinel (the wire delta encoding decodes it to -4096); plane keys are now pooled at [1, 32], flock keys start above that range. A spectator mode was added to the co-op button: spectators do not spawn a plane, the server is aware (via a `HelloMsg.spectator` flag), and the camera trails the first player's display slot. The `isConnected()` gate ensures the spectator head-count is deferred until after the Hello handshake completes.

### What Got Built: Weeks 21-22 (Jul 14 - Jul 23)

Ten days, 971 commits, and a different character to the work. Where the previous
block added a subsystem, this one finished several and then turned the tooling on
the codebase itself.

#### The brush editor, finished (Jul 14-15)

The plane-set brush editor closed out its remaining phases: destructive CSG (Cut,
Hollow, Merge) and the vertex and edge modeling tools, all wired to the ribbon.
The vertex tool is the interesting one, because it needs no topology code at all:
the operation is "move the point, re-hull the point set," and the behaviours a
user expects (a face fanning into triangles, a corner vanishing when pushed
inside, two corners fusing) fall out of taking the convex hull of a point set. A
precision bug that corrupted any non-box brush during Merge was found by a
dedicated probe pass before the tool went to runtime confirmation. The full story
is in [Brush Editing](BrushEditing.md).

#### Scene save, and the key-identity bugs behind it (Jul 22-23)

End-to-end scene saving did not fully exist before this block. A `.scene` writer
plus a registry deciding what persists was built and then hardened through a run
of identity bugs: scene handles gained a generation counter, because recycled
handle slots were being compared by raw index; entities now reload at the key
their rows were saved under; a failed save no longer destroys the scene it failed
to write, and it names the entity that aborted it.

#### Legacy level import (Jul 15-16)

Scene loading moved behind format providers, and a minimal XML reader plus a
Studio-side provider now imports levels from a previous in-house engine (brushes
and lights) directly. Two correctness passes followed contact with a real 86-brush
level: a missing-material checkerboard so an unresolved material name renders
loudly instead of vanishing, and a faithful reproduction of the old engine's
texture-coordinate math, where a missed unit-scale factor had been making imported
textures 50 times too large.

#### Brush rendering, and GPU memory (Jul 20-22)

Thousands of small convex solids is a pathological renderer input: one test level
emitted 774 draws from 322 brushes. Brushes now form persistent merged batches per
(spatial cluster, material), cutting draw count by roughly 38x. The follow-on work
was allocator-shaped rather than algorithmic: pooling D3D12 buffer resources
instead of asking the driver for each one took the merge upload from **231 ms to
39 ms**, and moving static geometry into a single VRAM arena uploaded by one
batched staging copy took it to **18 ms**.

#### The materials grid, and two dead ends (Jul 17-20)

The material picker became a live thumbnail grid of lit spheres, and then went
through several rearchitectures chasing a scroll-pop and lighting-flicker bug.
Two of those were abandoned: a transparent-background trick that clobbered
depth-of-field alpha and leaked full-screen materials, and a scroll-delta blit
compensation. What shipped was simpler than both: render each material's thumbnail
once into a cache and 2D blit it while scrolling. That took the panel from 17 to
45 FPS, and runtime-confirmed at 150 to 180. It is described in
[The Materials Viewer](MaterialsViewer.md).

It is worth being honest that this arc was three attempts, not one design. The AI
did not converge on the right architecture by reasoning about it; it converged by
building, measuring, and being told the flicker was still there.

#### Co-op fixes and collaborative editing (Jul 14-19)

A cluster of multiplayer bug hunts: a GPU device hang traced to a use-after-free
on a persistent instance buffer released while in-flight draws still referenced
it; a 1 Hz stutter that turned out to be a mislabelled swarm checksum; a slow join
whose transfer was idle rather than bandwidth-limited; and a co-op camera drift
where the owned plane's rotation reconciled its display but not its input
accumulator. Two collaborative-editing features landed alongside: remote peers'
selections now render in grey, and property grid field edits replicate, not just
gizmo drags. See [Networking](Networking.md).

#### The refactor audit (Jul 17-23)

The dominant story of the block, and the most interesting one for this post.

The co-op, networking, and determinism code was put through a generated audit: 19
dimensions (single source of truth, strong types, const correctness, reuse, test
gaps, algorithmic issues, scheduler parallelism, and so on), each swept
independently, with every finding re-read and adversarially verified by a second
agent before it was allowed onto the list. That produced roughly 470 verified
findings out of a larger candidate set, each citing a file and line, each with a
priority and an effort estimate. It was then worked to completion as a checklist,
closing at **472 of 472 resolved**.

"Resolved" is doing real work in that sentence, and it is the part worth
transferring. Roughly 64 items closed as *already satisfied*: the finding was
stale, or other work had overtaken it. At least 17 more were **declined** after
being implemented, measured, and rejected. Two are worth stating outright:

- One item prescribed converting a per-tick join to a cached view. On one system
  that produced a 22% frame-time win. On a second, nearly identical-looking system
  the same conversion cost **35%**, and was reverted. The audit document now
  carries the conclusion in bold: the conversion is not a blanket win, measure per
  site.
- Another prescribed removing a scheduler barrier. It was declined as unsafe: that
  barrier was the only thing ordering a shared spatial grid's cross-package
  readers, and doing it properly needed a three-package change out of the item's
  scope.

There was also a case where the audit's prescribed fix would not have caught the
bug class it was aimed at. The finding was correct (a simulation clock and a frame
clock were the same raw integer type, with no compiler protection against mixing
them, and that had already caused two measured desyncs). The prescribed fix,
wrapping them in the engine's strong-type template, would have changed nothing,
because that template had a non-explicit conversion operator and defined no
operators of its own: it decayed to the raw integer at every comparison. The real
defect was one level down, in the strong type itself, and it had to be fixed first.

**The lesson: a generated audit produces hypotheses, not fixes.** The finding is
usually right about *where* to look and frequently wrong about *what to do*. An
audit worked by an agent that treats each item as an instruction will
enthusiastically land regressions. One that treats each item as a claim to be
verified against the code, measured where it touches performance, and recorded as
declined when it fails, is a genuinely powerful instrument. The difference is
entirely in whether declining is an acceptable outcome, and that is set by how the
work is framed, not by the model.

Two other things this block cost, listed because a case study that only reports
wins is not evidence of anything: a client-side prediction harness for the flock
was built and then turned off by default, because the flock turned out to be
deterministic without it; and an intermittent heap-corruption crash stopped
reproducing after a dozen unrelated commits without a culprit ever being named.
The investigation notes say so explicitly rather than claiming a fix.

## The Takeaway

The projects that get the most from AI-assisted development aren't the ones with the simplest code. They're the ones with the most disciplined conventions, the clearest documentation, and the fastest feedback loops. Every minute spent on CLAUDE.md, on consistent module structure, on build scripts, and on unit tests pays dividends across every future AI interaction.

The first 8 weeks proved that foundation matters. The next twelve added a corollary: once the foundation is solid, the next multiplier is autonomy. Give the AI the tooling to verify, navigate, and debug its own work - fast build checks, symbol-aware navigation, persistent logs and crash dumps, and a way to offload the verbose work - and it spends far more of each session moving forward instead of waiting on you.

The last two weeks added a third: the AI is at its best not when it is trusted, but when it is instrumented to distrust itself. The refactor audit worked because declining a finding was a first-class outcome and a wrong fix that regressed a benchmark was reverted on the record. An agent that is asked to *resolve* items will resolve them; an agent that is asked to *verify* them will tell you which ones were wrong. That framing is the whole difference, and it is yours to set.

The AI amplifies whatever is already there. If your codebase has clear patterns, the AI will follow them. If it has documented boundaries, the AI will respect them. If it has fast build verification and can read its own crash dumps, the AI will catch and diagnose its own mistakes.

Invest in the foundation, give the AI room to close its own loops, and it becomes genuinely productive.
