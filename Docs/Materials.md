# Materials

Most engines treat a material as data: a fixed shader plus a bag of parameters
you fill in. Ceili treats a material as a **program**. A `.material` file is a
Lua script that builds the material (its passes, its render states, its
constants) and embeds its HLSL shader source inline. Materials inherit from one
another, constants can be driven by live script, and identical shaders and
constant blocks are de-duplicated automatically.

It is worth seeing one in full.

<!-- MEDIA: the Studio material preview panel showing a PBR sphere, ideally a
     short clip cycling through a few materials (chrome, iron, dielectric) and one
     of the live script-driven materials pulsing. This is the money shot for this
     page. -->

---

## A material is a Lua program

Here is the shape of a real one. `baseLightPBR.material` declares an inline
vertex shader as a value, then creates materials from it:

```lua
local M  = ceili.material
local P  = M.Pass
M.include("lightCommon")

local dirVertexShader = M.inlineShader([[
    #include "lightCommon.hlsli"
    // #ifdef SKINNED ... (skinning branch elided)
#elif defined(INSTANCED)
    PUSH_CONST(LightDirInstancedVSConstants, vsc);
    INSTANCE_BUFFER(instanceTransforms);
    LightDirInterpolants main(in LightVertex In, uint InstanceId : SV_InstanceID)
    {
        float4x4 world         = instanceTransforms[InstanceId];
        float4x4 worldViewProj = mul(world, vsc.viewProj);
        return LightDirVS(In, worldViewProj, world, world, ...);
    }
#else
    ...
#endif
]])

M.create("base/light/pbr", { flags = { lit = true, ... },
                             [P.kClusteredForward] = { ... } })
```

The HLSL lives right next to the Lua that uses it. The `#ifdef INSTANCED` is a
real preprocessor branch: the same source compiles to an instanced or a
per-surface variant depending on the defines a pass requests, and (as we will
see) both variants are de-duplicated if their `(source, defines)` pair matches
something already compiled.

> **Shader convention.** Ceili's `math::Mat4` is row-major with row-vector
> semantics, so the canonical multiply order in shaders is `mul(v, M)`, vector
> first. `o.Position = mul(pos4, WorldViewProj);`. `mul(M, v)` compiles fine and
> silently transposes your matrix; if a projection lands off-screen, check this
> first.

## The compiler VM

Calling a `.material` file "a program" is literal: the file is executed in a Lua
VM. The material compiler boots a minimal VM that loads the Core and Graphics
bindings and the material DSL itself, then runs each `.material` file so its
`M.create` calls register the material:

```lua
-- ceili_compiler.lua: minimal Lua VM startup for the material compiler.
ceili._resolveOnly = true
-- Load the material DSL, then run every .material file against it.
ceili.material = dofile(path.join(ceili.packagesPath, "Graphics",
                                  "Resources", "Materials", "core.material"))
```

Despite the name, `core.material` is the DSL: `M.create`, `M.inlineShader`,
`M.include`, the `PBR.*` helpers, the pass and binding tables, all of it is Lua
defined in `core.material`, so authoring a material is calling into a small
library, not filling in a fixed schema. The same DSL runs at runtime for hot
reload, which is why editing a `.material` and saving shows up in the running
editor the next frame (the hot-reload path described in [Core](Core.md)).

## Inheritance

A material can derive from another by naming it as the second argument to
`M.create`. The child inherits the parent's passes and shaders and overrides only
what differs. `baseLightPBR.material` derives its metal and legacy variants this
way, but the preprocessor define that picks a variant cannot simply live on the
child's `flags.definitions`: a material-level `definitions` string joins onto
*every* pass, and `base/light/pbr`'s passes get inherited into other lighting
models too, so the define would leak onto shaders that do not understand it. The
fix is a small per-pass helper, `pbrVariant`, that stamps the define onto only the
PBR passes:

```lua
M.create("base/light/pbr", { flags = { lit = true, ... }, ... })

local function pbrVariant(Definitions, Overrides)
    local def = Overrides or {}
    for _, pass_type in ipairs(kPbrLitPasses) do
        def[pass_type]             = def[pass_type] or {}
        def[pass_type].definitions = Definitions
    end
    return def
end

M.create("base/light/pbrMetal", "base/light/pbr", pbrVariant("PBR_METAL", {
    flags = { metal = true },
}))

M.create("base/light/pbrLegacy", "base/light/pbr", pbrVariant("PBR_LEGACY", {
    flags = { pbrLegacy = true },
}))
```

Concrete materials then derive from those bases and supply textures and constant
values through helper functions, so authoring a new material is a couple of
lines:

```lua
M.create("pbr/test/iron_smooth", "base/light/pbrMetal",
         PBR.metalWithTextures("iron", { metalRough = "predefined://mr_metal" }, 0.1))
```

## Constants driven by live script

A material constant does not have to be a static value. Its `binding` can be an
**expression**: a Lua function evaluated with a per-frame context. This material
pulses its colour every frame with a sine wave, entirely from the `.material`
file, no C++ involved:

```lua
fragmentShaderConstants = {
    { name = "pulseColor", valueType = VT.Float4, semantic = S.Color,
      binding = B.Expression,
      value = function(ctx)
          local t     = ctx and ctx.time or 0
          local pulse = 0.5 + 0.5 * math.sin(t * 2)
          return { pulse, 0.5, 0.5, pulse }
      end },
    { name = "textureIndex", valueType = VT.UInt, binding = B.Texture },
},
```

Other bindings cover the common cases: `B.User` (a value the property grid
edits, with a `default`), `B.Texture`, and `B.Expression` (the live function
above). The same declarative constant list drives editable parameters, texture
slots, and animated values alike.

<!-- MEDIA: a short clip of the pulse material animating in the preview, side by
     side with the ~8 lines of Lua that produce it. Shows "authored as data,
     alive at runtime" in one glance. -->

## Dedup: shaders and constants are content-addressed

An engine with hundreds of materials would drown in duplicate shader binaries and
constant layouts if each material compiled its own. Ceili avoids that by making
the **content** the identity.

A shader's identity is the pair `(source, defines)`. The first pass to claim a
pair owns the compiled file; every later pass or material sharing that pair reuses
it, even across different passes:

```lua
-- core.material (the material DSL)
-- Identity is (source, definitions): same pair produces identical compiled
-- output, so the first PASS claiming a pair owns the compiled shader file and
-- later passes / materials sharing the pair reuse that file.
inl.sourceHash = bit.bxor(hash32(pd.fragmentShader.source), fs_defs_hash)
```

On the C++ side the runtime registry is a set of hash-to-handle maps with
refcounts, so a state block or shader shared by 500 materials is stored once and
released when the last reference drops:

```cpp
// Material.h: namespace registry
MapArray<uint32_t, material::shader::Handle> shaderHashToHandle;
MapArray<uint32_t, material::Handle>         materialHashToHandle;

struct BlendSlot : core::slot::RefCounted<states::blend::Handle> {
    states::blend::Desc desc;
    uint32_t            descHash{0};
    Array<uint32_t, 0, uint16_t> nameHashes;  // reverse-alias bookkeeping
};
```

The identity even works *across* passes. An instanced pass whose fragment-shader
`(source, defines)` pair matches its per-surface sibling resolves to the
sibling's already-compiled shader, so the two variants share one binary. Each
inline shader records its source hash and its defines as the key:

```cpp
// Material.h: an inline shader's identity is its source hash plus its defines.
struct InlineSource {
    String                 materialName;
    String                 passName;
    material::shader::Type stage{material::shader::Type::Vertex};
    ConstStr                source{nullptr};
    uint32_t                sourceHash{0};
    uint32_t                sourceLine{0}; // line in material file where inline source starts
    String                  definitions;   // e.g. "PBR_METAL;PBR_LEGACY"
};
```

Sharing raises a lifetime question: if 500 materials share one blend state, when
is it freed? Each registry entry is reference counted and keeps the list of name
hashes that alias it, so releasing the last material that references a state block
drops it in O(K), without scanning the whole registry.

## The pipeline, end to end

```mermaid
flowchart TD
    A[".material file<br/>(Lua + inline HLSL)"] --> B["Material compiler VM<br/>(ceili_compiler.lua)"]
    B --> C{"(source, defines)<br/>seen before?"}
    C -- "no" --> D["Compile HLSL -> DXIL / SPIR-V<br/>record owner by content hash"]
    C -- "yes" --> E["Reuse the existing<br/>compiled shader"]
    D --> F["Runtime registry<br/>hash -> handle + refcount"]
    E --> F
    F --> G["material::Desc<br/>passes, constants, state handles"]
    G --> H["Draw: bind PSO,<br/>upload constants, render"]
```

At the bottom sits the C++ data model. A `Desc` is a named list of passes; a
`Pass` carries its compiled shader and state handles plus the declared constant
elements; a `ConstElement` describes one constant and its authoring metadata:

```cpp
struct ConstElement {
    String                       name;
    shader::constants::ValueType valueType;
    shader::constants::Binding   binding;      // User / Texture / Expression / ...
    bool                         hasExpression;
    String                       annotation;   // range / enum / description, from Lua
    Array<uint8_t>               defaultValue;
};

struct Pass {
    String                            name;
    states::blend::Handle             hBlend;
    states::depth::Handle             hDepth;
    shader::Handle                    hVertexShader, hFragmentShader;
    Array<ConstElement, 0, uint8_t>   vsConstants, fsConstants;
    // ...
};
```

## Baking the registration away

Running a Lua program for every material at every start is honest and it is not
free. A build-time pass now writes what each material registers as plain bytes,
so a shipped build can skip the VM:

```cpp
// Include/Materials/Recipe.h
// A material RECIPE is the plain data one material registers: its passes with their resolved shader names, state
// descs, constants layouts and defaults, its flags and texture infos. core.material computes it from a definition and
// writes it as bytes; Build and Update read those bytes and make every registration a material needs, in the order the
// Lua build used, so handle values do not move.
//
// Bytes rather than a cdata struct: the baked recipe file holds these same records, so one writer (core.material)
// and one reader (Recipe.cpp) serve both the runtime call and the file. The layout is documented beside the reader.
```

One writer and one reader serve both the live path and the file. The live
registration and the baked registration cannot drift, because they are the same
two functions reading the same bytes. Registrations also happen in the order the
Lua build used, so handle values do not move between a baked run and a live one.

Two settings ride on top, and both are off by default:

```cpp
// Include/Materials/Recipe.h
// Whether material::LoadScripts builds materials from the baked MaterialRecipes.bin files the material compiler writes,
// instead of running the .material files. It does so only when every file is fresh; otherwise it logs why and runs
// them. Off by default: a baked material has no Lua definition yet, so hot reload, Studio's definition queries and
// expression-bound constants do not work with it on. Set from config.lua, before materials load.
CE_API void SetBaked(const bool Enabled);
```

The default is off for a plain reason: a baked material has no Lua definition, so
hot reload, Studio's definition queries and expression-bound constants stop
working. The authoring loop keeps the VM; a shipped build can drop it.

A second setting makes a baked material build when a lookup first asks for it,
rather than at load. That one needed a companion, because a tool that lists every
material rather than looking one up would otherwise stall the frame that opens a
panel. A background budget in milliseconds per frame lets Studio's menus fill in
over a few seconds instead.

A material edited since the bake runs alone, through the VM, rather than forcing
the whole set back onto the slow path.

## Authored once, used everywhere

Look again at `ConstElement`: alongside the GPU-facing `valueType` and `binding`
it carries an `annotation`, a range or enum or description piped straight from the
Lua declaration. That one field is why the **property grid builds itself** from a
material. When the editor inspects a material it walks the constant elements, and
each one already knows its widget, its bounds, and its tooltip, so the property
grid needs no separate editor schema and no per-material inspector code.

So a single constant declaration in a `.material` file drives four things at once:
the GPU constant binding, the value the serializer saves, the value the network
layer replicates, and the widget the property grid renders. That is the
[Metadata](Metadata.md) philosophy (one declaration, one Read/Write funnel, many
consumers) reaching all the way into graphics.

One practical corollary worth knowing: a surface references its material by
*name*, and an unregistered name resolves to an invalid handle, which the render
passes skip silently. If a surface renders nothing, an unregistered material name
is the first thing to check. Name resolution is device independent, so this is
catchable headless (see [Rendering & Visibility](Rendering.md)).

## Output and post-processing

Materials render into an HDR scene target, which the tonemapper and post chain
turn into the final image, in SDR, HDR10, or scRGB depending on the display:

```cpp
enum class OutputMode : uint8_t { Ldr = 0, Sdr = 1, Hdr10 = 2, Scrgb = 3 };
```

The post-processing chain (bloom, depth of field, tonemap, lens flare, 3D-LUT
grading) is itself authored as a list of `.material` passes wired together by
their input/output slots. That is its own story:
[Post-Processing & HDR](PostProcessing.md).

---

## Why this is unusual

Pulling the pieces together: a material in Ceili is a **live, inheriting,
de-duplicated program** whose declaration simultaneously drives the GPU pipeline,
the serialized data, the network representation, and the editor UI, from a
single Lua file with the HLSL inline. Most engines split those into a shader
file, a material asset format, a separate parameter UI schema, and a C++ binding
layer. Collapsing them into one authored artifact is what makes iteration fast:
editing the `.material` file and hot-reloading shows the result immediately.

Next: [Post-Processing & HDR](PostProcessing.md) for the chain that reads the HDR
target, or [Lighting](Lighting.md) for the passes a lit material declares. Back
to the [documentation index](README.md).
