# Particles

A particle system in Ceili is an entity that spawns camera-facing billboards: a
fire, a spark shower, a dust cloud, a swinging swarm of motes. It is young. It
went in during the summer of 2026, and the code still reads as a first pass with
its scope written down where the reader can see it.

The subsystem does not have its own package. It lives inside
[Graphics](../Rendering/README.md), because a particle is a thing that draws and its
whole cost is the draw. The simulation, the storage, and the billboard build all
sit under `Pkg/Engine/Graphics`.

One decision shapes everything else on this page: particles simulate in **world
space**. A particle is placed in the world once, at birth, and never moved into a
new frame after that. So an emitter that travels leaves its particles behind it,
and the exhaust of a moving rocket trails the rocket instead of riding along with
it. That choice sets how the storage works, what the renderer has to do, and
where the cost lands.

---

## The shape of a frame

An entity carries one `ParticleSystem` row. The row names an effect. The effect
resolves to an array of emitters, and each emitter owns a slice of one shared run
of live particles, keyed by the entity. Every frame does two things to that run:
it spawns new particles into world space, and it integrates the live ones.

```mermaid
flowchart TD
    ROW["ParticleSystem row<br/>(names an effect)"] --> RES["ResolveParticleSystem<br/>hash compare once settled"]
    RES --> EMIT["EmitterArray<br/>N emitter Desc"]
    EMIT --> STEP["StepEmitter, per frame"]
    STEP --> SPAWN["SpawnParticleWorld<br/>place in WORLD space at birth"]
    STEP --> INT["StepParticles<br/>semi-implicit Euler, compaction"]
    SPAWN --> TAB["Particle table<br/>one run per entity, sliced per emitter"]
    INT --> TAB
    TAB --> QUAD["ParticleBuildCorner, vertex shader<br/>camera-facing billboard"]
    QUAD --> PASS["kParticle pass<br/>depth test, no depth write"]
```

The effect name is resolved every frame rather than once at insert. When the row has
settled the resolve is a single hash compare, and paying it each frame is what
lets an effect register after its row already landed, rather than never.

---

## Storage: a database table

Particle storage is a plain table in the scene's own database, one that holds
many `Particle` items per entity key. There is no bespoke store beside it. The
private header states the reason:

```cpp
// Src/ParticleSimulationInternal.h
// The word "pool" is deliberately gone from this API. The row type IS Particle and the table is
// literally "ceili::graphics::particle::Particle", so "pool" was only ever an informal alias -- and
// once an entity carries N emitters, each owning a SLICE of one shared run, it became an actively
// wrong one. There is no pool per emitter.
//
// An entity's particles are a MULTI-ITEM RECORD in the scene's own database, keyed by the entity's
// key -- the BrushFace / BrushMaterial shape.
```

Two properties fall out of using the table store described in
[Core](../Core/Core.md#data-the-table-store-the-engine-is-made-of). The particles of a scene live in one
storage block, so each emitter's slice stays contiguous and reads back as a plain
array. And the table is scoped to the scene's memory arena, so destroying the
database drops the particles with it. There is no destroy hook to write and no
teardown path to get wrong.

A `Particle` is flat plain-old-data, chosen so the store can recycle a dead slot
by moving the last live particle over it:

```cpp
// Include/ParticleSimulation.h
// Age is carried as `life` counting DOWN plus the `lifeSpan` it started with, rather than as
// a normalized 0-1 age, because the retirement test is then `life <= 0` -- exact at the
// boundary -- while normalized age stays a derived value (AgeFraction below).
struct Particle
{
    // WORLD space, both of them. Placed once at birth by SpawnParticleWorld and never
    // re-transformed, which is what leaves a moving emitter's particles behind it as a trail.
    math::Point3 position{0.0f, 0.0f, 0.0f};
    math::Point3 velocity{0.0f, 0.0f, 0.0f};
    float        life{0.0f};     // seconds remaining
    float        lifeSpan{0.0f}; // seconds this particle started with
```

Each emitter clamps its own live count to a hard ceiling of 4096
(`kMaxActiveParticlesLimit`), so a mis-authored spawn rate is bounded rather than
free to allocate. The default per-emitter ceiling is 64. The table starts at
8192 items, sized so a whole scene of emitters fits one block. A scene with no
emitter creates no table and pays nothing.

---

## World space is what makes a trail

A particle is born in the emitter's local frame, then transformed into world
space once and stored there. The spawn function carries the design note:

```cpp
// Src/ParticleSimulationInternal.h
// THIS IS WHAT MAKES A TRAIL. Particles are transformed ONCE, at birth, and then live in world
// space, so an emitter that moves LEAVES THEM BEHIND. Simulating in the emitter's local frame and
// transforming at draw instead makes a moving emitter carry its whole cloud along -- a rocket then
// wears its exhaust like a blob instead of trailing it, which is what this engine used to do and
// what legacy never did (CEmitter::spawnParticle births at the emitter's current world position and
// never re-transforms).
//
// It is also the cheaper half of that trade: the transform was previously paid per LIVE particle
// per FRAME at draw, and is now paid once per particle for its whole life.
//
// The velocity is rotated as a DIRECTION (no translation row), so a rotated emitter throws its
// particles the way it is actually facing.
CE_API Particle SpawnParticleWorld(const Desc& Desc, uint32_t& RandomState, const math::Mat4& EmitterWorld);
```

The position is transformed as a point, so the emitter's translation places it.
The velocity is transformed as a direction, so a rotated emitter throws its
particles the way it faces. After that the particle is on its own. Because the
particle already holds a world position, the renderer reads no emitter transform
at draw time, and the per-frame matrix multiply that a draw-time transform would
cost is gone.

The applied force follows the same rule. Force is world-space acceleration, so an
authored gravity points down the world rather than down the emitter's tilted axis.

The axis launch is world space too. It adds a speed along one axis to the random
throw, which is what lifts a chimney's smoke up out of its stack:

```cpp
// Include/Particles.h
// Initial speed along ONE axis, added to the throw above: legacy's SpawnImpulse_Axis, which is what launches a
// chimney's smoke up out of its stack. WORLD space, as legacy's was and as `force` is, and the axis is NOT
// normalized: legacy scaled the speed by the authored vector, and the corpus relies on it (a few author
// (0, 0.05, 0)). The axis defaults to +Y, as legacy's did. Zero speeds turn it off and consume no random draw, so
// an emitter without it spawns exactly the particles it always did.
math::Point3 spawnAxis  CE_DESC("Axis of the launch, in world space; its length scales the speed") = {0.0f, 1.0f, 0.0f};
float spawnAxisSpeedMin CE_DESC("Slowest launch along the axis")                                   = 0.0f;
float spawnAxisSpeedMax CE_DESC("Fastest launch along the axis")                                   = 0.0f;
```

The launch is added after the emitter transform, so a tilted emitter still sends
its smoke up the world's +Y.

---

## The step

`StepParticles` runs once per emitter per frame. It ages each particle, retires
the dead ones, and integrates the rest:

```cpp
// Src/Particles.cpp
        // Semi-implicit Euler: velocity first, then position from the UPDATED velocity. Stable
        // under a constant force where explicit Euler drifts. Wobble rides the same accumulator
        // as the authored force -- it IS an acceleration, unlike damping below.
        particle.velocity.x += (Desc.force.x + particle.wobble.x) * DeltaSeconds;
        particle.velocity.y += (Desc.force.y + particle.wobble.y) * DeltaSeconds;
        particle.velocity.z += (Desc.force.z + particle.wobble.z) * DeltaSeconds;

        // Damping is an OPERATOR, applied to the integrated velocity rather than folded into the
        // accumulator -- a force cannot express "scale whatever speed you ended up with".
        particle.velocity.x *= damping_scale;
        particle.velocity.y *= damping_scale;
        particle.velocity.z *= damping_scale;

        particle.position.x += particle.velocity.x * DeltaSeconds;
        particle.position.y += particle.velocity.y * DeltaSeconds;
        particle.position.z += particle.velocity.z * DeltaSeconds;
```

Gravity has no field of its own. The `force` accumulator points down to make it.
Damping scales the speed the particle ended up with, which a force cannot
express, so it is applied as a separate operator after integration. Wobble is a
held random kick that lasts an authored interval, which reads as turbulence,
where a fresh random kick every step would average out to jitter.

Retirement uses swap-with-last compaction. When a particle's `life` reaches zero,
the last live particle is copied over it and the live count drops by one:

```cpp
// Src/Particles.cpp
        particle.life -= DeltaSeconds;
        if (particle.life <= 0.0f)
        {
            // Swap-with-last compaction: O(1) per death and no shuffle. Particle ORDER is
            // therefore not stable across a step, so nothing may key off an index between
            // steps. Do NOT advance i -- the element just swapped in has not been stepped.
            --live;
            particle = pParticles[live];
            continue;
        }
```

The cost of retiring a particle is constant and there is no shuffle. The price is
that particle order is not stable across a step, so nothing downstream may key off
a particle's index between frames. The step also clamps its delta to 0.1 seconds
(`kMaxStepSeconds`), so a stalled frame from a scene load or a shader recompile
does not integrate a whole second of motion in one go.

---

## Colour and size over life

A particle changes colour and size as it ages. Both use a three-key ramp over the
particle's normalized age, sampled at spawn, half life, and death:

```cpp
// Include/Particles.h
struct ColorRamp
{
    color::ColorF32 startColor  CE_DESC("Colour at spawn")     = {1.0f, 1.0f, 1.0f, 1.0f};
    color::ColorF32 middleColor CE_DESC("Colour at half life") = {1.0f, 1.0f, 1.0f, 1.0f};
    color::ColorF32 endColor    CE_DESC("Colour at death")     = {1.0f, 1.0f, 1.0f, 0.0f};
};
```

`AgeFraction` gives the age: 0 at spawn, 1 at death. The ramp is piecewise
linear in two segments, from the start key to the middle key over the first half
of life and from the middle key to the end key over the second. A caller does not
pre-clamp, because the sampler clamps outside the range.

`SizeRamp` has the same three-key shape for the quad's world-space size. One
primitive covers both colour and size, and neither needs an on-off flag, because
three equal keys already spell a constant. There is deliberately no separate
constant-colour or constant-size field, so there is no second field to ask which
one wins.

---

## Finite one-shots

An emitter is continuous by default. An explosion, a puff, a single spark burst
are not: they emit a fixed count and then stop. Two authored fields carry that,
`infiniteParticles` (default true) and `numberToEmit` (default 100):

```cpp
// Include/Particles.h
// FINITE EMISSION. An emitter that is not infinite spawns numberToEmit particles IN TOTAL and
// then stops for good -- legacy's Emitter_Spawning_InfiniteParticles and NumberToEmit, which
// 111 corpus emitters author. Without it every one-shot effect in the corpus ports as a
// continuous one: an explosion emits smoke for ever at its impact point, and no effect can be
// given a "destroy when finished" lifetime, because none of them ever finishes.
//
// NOT a burst, despite what a one-shot looks like: the count is emitted AT particlesPerSecond,
// so the legacy explosion's 4 particles at 10/second take 0.4 seconds. Legacy spawns from the
// same per-step loop whether or not the emitter is finite; only the exit condition differs.
```

A finite emitter does not burst all at once. Its count is emitted at the same
`particlesPerSecond` rate as a continuous one, so four particles at ten per
second take four tenths of a second to appear. The spawn count each frame is
clamped to the budget first, then to the free room in the slice:

```cpp
// Src/Particles.cpp
    // BUDGET first. A finite emitter that has spent its count is done, and stays done -- there is
    // no path that resets numEmitted short of the storage being rebuilt.
    if (!Emitter.infiniteParticles)
    {
        const uint32_t remaining = (State.numEmitted >= Emitter.numberToEmit) ? 0 : (Emitter.numberToEmit - State.numEmitted);
        if (requested > remaining)
        {
            requested = remaining;
        }
    }

    // ROOM second, and it does NOT consume budget: a particle the full pool refuses is one the
    // one-shot still owes. Legacy's decrement sits inside the same guard.
    if (requested > Room)
    {
        requested = Room;
    }
```

The order matters. A particle the full slice refuses does not count against the
budget, so a one-shot still owes it once room frees up. An entity can set
`destroyOnFinish`, and when every emitter has spent its budget and no particle
is still alive, the system destroys the entity. So a one-shot cleans itself up.

---

## Billboards

Every particle draws as a quad that faces the camera, and the vertex shader
builds the quad. The renderer writes one 32-byte record per live particle into
the view's instance buffer, and draws each emitter over a static quad list. For
each draw it reads the camera's right and up axes from the view matrix and binds
them as vertex constants:

```cpp
// Render/Base.cpp
void BindParticleRunConstants(const material::Pass& Pass, const ParticleRun& Run, const draw::ViewDesc& View)
{
    if (!Pass.hVsConstants.isValid())
    {
        return;
    }
    // World-space camera right / up, from the view matrix's basis.  Mat4LookAtLH stores the view with column 0 = right
    // and column 1 = up, and in the flat row-major float[16] a column is a stride-4 read.  Taking the ROWS reads the
    // transpose, and every quad then faces a direction that sweeps as the camera turns.
    const float* const vm = View.viewMatrix;
    const math::Vec4   camera_right{vm[0], vm[4], vm[8], 0.0f};
    const math::Vec4   camera_up{vm[1], vm[5], vm[9], 0.0f};
```

The shader scales the two axes by the particle's half size and adds them to the
world centre at each of the four corners:

```hlsl
// Resources/Shaders/particle.hlsli
// Size is the FULL width and height, so the corner sits half of it along each axis.
const float half_x = live ? ParticleCornerX(corner) * size.x * 0.5f : 0.0f;
const float half_y = live ? ParticleCornerY(corner) * size.y * 0.5f : 0.0f;

ParticleCorner c;
c.centre = p.positionAge.xyz;
c.offset = float3(right_x, right_y, right_z) * half_x + float3(up_x, up_y, up_z) * half_y;
c.world  = c.centre + c.offset;
```

An emitter can align its quads to the camera plane or to the particle's
velocity, and each particle carries its own spin so two born together do not
turn in lockstep.

The particles draw in a dedicated pass, `kParticle`, which tests depth but does
not write it:

```cpp
// Render/Particles.cpp
    // One draw per emitter. The pass depth-TESTS but does not depth-WRITE (particle.material), so
    // solid geometry occludes particles while two overlapping particles both contribute to the
    // blend -- which is also why no sort against the scene is needed here. Particles WITHIN an
    // emitter are submitted in pool order, which swap-with-last compaction does not keep stable;
    // for the additive-ish ramps the corpus authors that is invisible, and a proper back-to-front
    // sort belongs with the per-emitter sort mode a later slice adds.
```

Solid geometry occludes a particle because the pass tests depth, and two
overlapping particles both add to the blend because it does not write depth. So
the particle pass needs no sort against the rest of the scene.

A particle takes the scene's haze
([aerial perspective](../Rendering/Atmosphere.md#aerial-perspective-the-air-in-front-of-the-world)), and the
blend decides how. An alpha-blended particle writes over the background, so it
takes both the transmittance and the in-scatter. An additive particle adds to a
background that already carries its haze, so it takes the transmittance only:

```hlsl
// Resources/Materials/particle.material
// An alpha-blended particle WRITES its own colour over the background, so it takes both
// terms once. An ADDITIVE one (self-lit: fire, a flash) adds to a background that already
// carries its haze, so it takes the transmittance only, as the lit additive passes do.
// The in-scatter added there lifted a flash texture's black border to the haze colour,
// and every quad showed as a pale rectangle (DirtBox's drill debris). RGB only -- the
// alpha drives the blend and must not be attenuated.
const float4 ap = AerialPerspectiveFetch(sampler0, In.AerialUv, fsc.aerialVolumeIndex);
c.col.rgb       = (fsc.selfLit.w > 0.0) ? AerialApplyAdditive(c.col.rgb, ap) : AerialApplyBase(c.col.rgb, ap);
```

Ported effects name their materials the way the legacy content did. The legacy
material converter writes each legacy billboard material (`BaseSFX`,
`BaseSFXTrans`, `BaseSFXLight`, `BlackBlend`) onto `base/particle`, and each
`BaseSFXRefract` material onto `base/particle/refract`, under its own legacy path.
An emitter's `materialName` is then that path, for example
`dogfighterv1/textures/particles/cloud`, with no lookup table between the two. The
template converter fails on a name that no package defines as a particle
material, and on a name longer than 63 characters.

**A particle system can stand in for a fog volume.** Legacy levels fake low mist
with particle emitters. The legacy importer writes a fog volume beside each
emitter that a game registers (`env::fog::RegisterLegacyStandIn`), and it sets the
emitter's `fogStandIn` fact. On a view that draws fog volumes, the particle draw
skips that emitter. On any other view (a Quest, a legacy view such as Studio's
compare, a device with no fragment storage buffers) the emitter draws as
authored. The simulation steps a stand-in on every view, because one simulation
feeds every view of the scene. See
[Weather: fog volumes](../Rendering/Weather.md#fog-volumes) for the volume itself.

<!-- MEDIA: a short clip of a single emitter dragged across the viewport in
     Studio, showing the live particles staying put in the world while the
     emitter moves, so the trail is visible. Capture the same effect once
     stationary and once moving, side by side if it fits. -->

---

## A particle that lights the scene

The billboard pass draws a particle. A second, opt-in pass makes every live
particle of an emitter add light to the [deferred G-buffer](../Rendering/Lighting.md):

```cpp
// Include/Particles.h
// PARTICLE LIGHTS: every live particle of this emitter also ADDS LIGHT to the deferred
// G-buffer, which is the capability the legacy engine's lightPrePassParticles pass had and
// this one has not until now.  Appended here for the reason the two fields above were - the
// struct is an FFI cdef, so scalars land at the end of the scalar region and materialName
// stays last.
//
// OPT-IN, and it must stay opt-in: turning every particle into a real light is a different
// cost class entirely, and no existing content expects it.  The draw is skipped whole when
// this is clear, so an emitter that does not ask pays nothing.
bool lightEnabled CE_DESC("Every live particle also lights the scene through the G-buffer") = false;

// How far the light reaches, as a MULTIPLE of the particle's sprite size.  The light quad is
// the sprite quad widened about its centre by this, so 1.0 lights only what the sprite covers
// and the default carries a firefly-sized sprite a useful distance.
float lightRadiusScale CE_RANGE_DESC(1.0f, 64.0f, "Light reach as a multiple of the particle's sprite size") = 8.0f;

// Scales the per-particle ramp colour into a light intensity.  Separate from the colour so an
// emitter can stay visually dim and light brightly, or the reverse.
float lightIntensity CE_RANGE_DESC(0.0f, 64.0f, "Multiplier on the particle's colour when it lights the scene") = 1.0f;
```

Two dials keep the light separable from the look. The radius is a multiple of the
sprite size, so a large soft sprite lights further than a spark without anyone
authoring a world distance. The intensity scales the ramp colour independently of
what the sprite draws, so an emitter can read dim and light brightly, or the
reverse.

Reach is a view flag as well as an emitter opt-in. The engine's split-compare can
then hold particle lights on one side and off the other, on the same frame of the
same scene.

A particle's light is a local light, so it takes the same calibration as a
ported scene's torches, on a PBR view only. A legacy (Blinn-Phong) view also draws
the particle-light pass, and there the light keeps its authored intensity:

```cpp
// Render/Particles.cpp
// A particle's light is a local light, so a ported scene's take the same calibration as its torches: the
// local light gain and the view's row scale, and the ramp colour decoded from a legacy display code. On a
// PBR view only, as light::ApplyLocalLightGain and render::ParticleSelfLit do: a Blinn-Phong (legacy)
// view draws this pass too, with lightBlinn, and keeps the light as authored.
const graphics::SceneLighting& lighting  = graphics::GetSceneLighting();
const float                    intensity = blinn ? entry.lightIntensity : entry.lightIntensity * lighting.localLightGain * light::RowScale(lighting);
const float                    decode    = (!blinn && lighting.albedoDisplayCoded) ? 1.0f : 0.0f;
```

A lit particle takes the scene's ambient. A self-lit particle (an additive pass:
fire, a flash) gives off its own light, so the ambient stays at one for it. A
particle can also refract, drawn by the pass that draws refracting surfaces.

The feature also produced a measuring lesson worth repeating. The first gate for
it passed while the feature was inert, and what fixed that was a fixture with a
ratio and **two negative controls**: one run with the emitter's opt-in cleared and
one with the view flag cleared. A gate that cannot fail reads as coverage. The
engine now has a general mechanism for that check, described in
[Core](../Core/Core.md#three-primitives-a-defect-audit-paid-for).

---

## What is not built yet

The particle files carry no `TODO` markers. The scope is drawn in prose and in
the shape of the code, so the honest list is what the source does not do:

- **Billboards only.** Every particle is a camera-facing quad. There is no mesh
  particle, and no ribbon or trail built from connected geometry. A trail here is
  the world-space smear of separate billboards rather than a connected strip.
- **The simulation runs on the CPU.** The step and the spawn run in one scene
  system (`ParticleUpdateSystem`) once a frame, and only the quad build runs on
  the GPU. There is no compute-shader or
  GPU simulation path.
- **No collision.** A particle does not test the world. It follows its force and
  its damping until it ages out.
- **No sub-emitters.** A particle cannot spawn an effect of its own, and there is
  no event-driven or child-effect spawn.
- **No animated texture.** A particle carries one static texture coordinate. There
  is no atlas flipbook and no scrolling UV.
- **No per-emitter sort yet.** Particles inside an emitter draw in storage order,
  which compaction does not keep stable. For the additive-leaning ramps that
  effects author this is invisible, and a back-to-front sort waits on a
  per-emitter sort mode.
- **No culling for emitters yet.** The renderer walks the emitter rows directly
  rather than through the sorted surface path, so an emitter is not stamped by the
  cull system. Culling an emitter needs world bounds recomputed from the live
  particles each frame, which is a later slice.

The list is short because the subsystem is. What is here is a world-space CPU
simulation with colour and size ramps, finite one-shots, a billboard pass, and an
opt-in light per particle, built on the scene's own table store so it inherits
that store's lifetime rules for free.

Next: [Render Passes & Visibility](../Rendering/RenderPasses.md) for the pass machinery, or
[Lighting](../Rendering/Lighting.md) for the G-buffer a particle light writes into. Back to the
[documentation index](../README.md).
