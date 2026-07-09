# Component Architecture

The component system is how Ceili delivers on the "plug and play: swap any
implementation" bet from [Philosophy](Philosophy.md). It is a small, deliberate
piece of machinery: an interface, a factory that makes instances of it, and a
runtime alias layer that decides which concrete implementation answers when
someone asks for the interface. That is enough to run any number of graphics
backends off one call site, let an AI agent be a hardcoded stub in tests and a live subprocess
in the editor, and let a component be authored in C++, C#, or Lua without the
caller knowing the difference.

The governing rule is narrow, and worth stating up front because it is what keeps
the system from sprawling:

> A component exists when there are, or credibly will be, **multiple
> implementations of an interface**. The interface drives the abstraction, not the
> other way around. One implementation with no sibling on the horizon does not
> need a component; it needs a function.

Everything below is in service of that rule.

---

## The interface surface

Every component implements `IComponentBase`, the lifecycle-and-identity contract
that the rest of the engine talks to. It is deliberately small: creation and
destruction, scope binding, an interface-cast, and the flag accessors.

```cpp
// Component.h
struct IComponentBase
{
    virtual ~IComponentBase()                                                              = default;
    virtual IComponentFactory* getComponentFactory() const                                 = 0;
    virtual void               setComponentFactory(IComponentFactory* const pVal, const InterfaceId InstanceInterfaceId) = 0;
    virtual scope::Handle      getScope() const                                             = 0;
    virtual void               setScope(const scope::Handle hScope)                         = 0;
    virtual InterfaceId        getInstanceInterfaceId() const                               = 0;
    virtual Result             getInterface(const InterfaceId InterfaceId, IComponentBase** ppInterface) = 0;
    virtual Result             destroy()                                                    = 0;
    virtual ComponentFlags     getComponentFlags() const                                    = 0;
    virtual void               setComponentFlags(const ComponentFlags Flags)                = 0;
    virtual void               addComponentFlags(const ComponentFlags Flags)                = 0;
    virtual void               removeComponentFlags(const ComponentFlags Flags)             = 0;
};
CE_DECLARE_INTERFACE_ID(ComponentBase)
```

Two derived interfaces add the parts most components actually want. `IComponent`
adds the update lifecycle; `IComponentSingleton` adds reference counting for the
one-instance-per-process case (systems, renderers, device singletons):

```cpp
// Component.h
struct IComponent : IComponentBase
{
    virtual Result init()                                         = 0;
    virtual Result update(const ComponentUpdate& ComponentUpdate) = 0;
};
CE_DECLARE_INTERFACE_ID(Component)

struct IComponentSingleton : IComponent
{
    virtual int getRefCount() const = 0;
    virtual int addRefCount()       = 0;
    virtual int removeRefCount()    = 0;
};
CE_DECLARE_INTERFACE_ID(ComponentSingleton)
```

Notice the `IID_` ids sit right next to the interfaces they name. That placement
is a convention, not an accident: `CE_DECLARE_INTERFACE_ID` goes alongside its
interface in the header, while `CE_DECLARE_COMPONENT_ID` (below) goes in a
module's `Module.h`. Both are just compile-time type hashes, the same
`TypeHash<T>()` idea from [Core](Core.md#hashing-and-type-hashes):

```cpp
// Macros/Component.h
#define CE_DECLARE_COMPONENT_ID(Type) \
    class Type;                       \
    constexpr ceili::core::component::ComponentId CID_##Type = ceili::core::component::MakeComponentId<Type>();

#define CE_DECLARE_INTERFACE_ID(Type) \
    constexpr ceili::core::component::InterfaceId IID_##Type = ceili::core::component::MakeInterfaceId<I##Type>();
```

A `ComponentId` names a *concrete implementation*; an `InterfaceId` names a
*contract*. The whole system is a lookup from the pair `{ComponentId,
InterfaceId}` to a factory. Hold that shape in mind: it is the entire model.

Both ids are derived from a type, and most code never spells one out.
Registration names the interface type and the macro computes its id; creation
names the destination pointer and the template computes the id from that. The
`IID_` constants are for the places that hold an id as a value, such as
`getInterface` and the scope queries. The rule to carry through the rest of this
page: **name the type, let the compiler produce the id.**

---

## Component: the base you inherit

You almost never implement `IComponentBase` by hand. You inherit `Component<I>`
(or `ComponentSingleton<I>`), which supplies the flag storage, the scope binding,
and default no-op lifecycle methods, and leaves you to override only what your
component actually does.

The template parameter is the interface you implement, and the base keeps it
where the rest of the machinery can read it:

```cpp
// Component.h
template <class I>
class Component : public I
{
public:
    // The interface this component ACTUALLY implements, and its id, both known at compile time from I.
    // interface_type is what the registration macros static_assert against; kInterfaceId is what
    // getInterface answers for.
    using interface_type = I;

    static constexpr InterfaceId kInterfaceId = MakeInterfaceId<I>();
```

Those two lines are what the rest of this page hangs on. Once the interface is a
compile-time property of the class, registration and creation can both check
themselves.

```cpp
// Component.h: the defaults Component<I> hands you.
ComponentFlags getComponentFlags() const override { return m_Flags; }
void           setComponentFlags(const ComponentFlags Flags) override { m_Flags = Flags; }
void           addComponentFlags(const ComponentFlags Flags) override { m_Flags |= Flags; }
void           removeComponentFlags(const ComponentFlags Flags) override { m_Flags &= ~Flags; }

virtual Result init() override { return core::results::success::Default; }
Result         destroy() final { return m_pComponentFactory->destroyComponent((IComponentBase*)this); }
virtual Result update([[maybe_unused]] const ComponentUpdate& ComponentUpdate) override { return core::results::success::Default; }

private:
    IComponentFactory* m_pComponentFactory   = nullptr;
    InterfaceId        m_InstanceInterfaceId = kNullInterfaceId;
    scope::Handle      m_hScope;
    ComponentFlags     m_Flags = ComponentFlags::Active;
```

`destroy()` is `final` for a reason: an instance is always freed by the factory
that made it, never by a raw `ceDelete`. That single rule is what lets the same
`destroy()` call work whether the instance is a C++ object, a scope-allocated
struct, or a script wrapper with function-pointer trampolines (more on that
later). `init()` and `update()` are the two hooks you override; both default to
success so a component that needs neither writes neither.

The engine's convention is to define these method bodies **inline in the class
declaration** in the implementation .cpp, not as out-of-line `App::method()`
definitions. Keeping declaration and definition together is easier to read and
keeps the component's behaviour in one place.

### Asking an instance for an interface

`getInterface` hands back a pointer to one of the interfaces an instance
implements. The base answers for `IComponent`, for `IComponentBase`, and for its
own `I`, and refuses anything else:

```cpp
// Component.h
if (InterfaceId == kInterfaceId)
{
    *ppInterface = NarrowCast<I*>(this);
    return core::results::success::Ok;
}

return core::results::failure::UnknownInterface;
```

The comparison is against `kInterfaceId`, which the class derives from its own
`I`, so an instance answers for the interface it actually implements. A component
that implements several interfaces overrides `getInterface`, answers for each one
it adds, and calls the base for the rest. The spatial backends are the worked
example further down this page.

The pointer written back is already adjusted to the requested interface, so a
caller uses it directly with no further cast.

---

## The factory macros

A component registers itself with two macros. `CE_DECLARE_COMPONENT_ID` in the
`Module.h` mints the `CID_`. Then, in the .cpp,
`CE_COMPONENT_FACTORY_DEFINITION` wires the concrete class to the interface it
registers under. **The macro takes the interface type, not its id**, and derives
the id itself:

```cpp
// Macros/Component.h
#define CE_COMPONENT_FACTORY_DEFINITION(ComponentID, Interface, Implementation)                     \
    static int g_##Implementation##RefCount = 0;                                                     \
    Implementation##Factory::Implementation##Factory()                                               \
    {                                                                                                 \
        if (g_##Implementation##RefCount++ == 0)                                                      \
        {                                                                                             \
            typedef ceili::core::component::ComponentFactory<Implementation, Interface> factoryType;  \
            ceili::core::component::IComponentFactory* p_factory = ceNew(factoryType, ComponentID);   \
            ceili::core::component::InterfaceId interface_ids[] =                                     \
                {ceili::core::component::MakeInterfaceId<Interface>()};                               \
            ceili::core::component::RegisterComponentFactory(ComponentID, interface_ids, 1, p_factory); \
            m_pFactory = p_factory;                                                                   \
        }                                                                                             \
    }
```

A registration therefore reads
`CE_COMPONENT_FACTORY_DEFINITION(CID_AgentNull, IAgent, AgentNull)`, and the
factory carries the interface as a template parameter. That is what lets the
factory check the registration against the class:

```cpp
// Component.h
static_assert(types::IsBaseOfV<I, typename C::interface_type>,
              "the interface this factory registers under is not implemented by its component");
```

The check is `IsBaseOf` rather than equality, which lets a family of backends
register under a shared base. The grid is a `Component<ISpatialGrid>` and the
kd-tree a `Component<ISpatialTree>`, and both register as `ISpatialIndex`, so a
consumer can enumerate the family through one id and narrow to a kind with
`getInterface`.

The assert sits inside `createComponent` rather than at class scope, and that
placement is forced. The macro is routinely written before the implementation
class, because `CE_COMPONENT_FACTORY_DECLARATION` forward-declares it, so the
class is incomplete where the factory is instantiated. Only a late-instantiated
member sees the complete type.

Registration runs at DLL load, driven by a nifty-counter struct that
`CE_COMPONENT_FACTORY_DECLARATION` places in the module. The construction of that
struct calls `RegisterComponentFactory`, and its destruction unregisters. There is
no central "register all components" list to maintain: linking the .cpp into the
build is what makes the component exist. `CE_SINGLETON_COMPONENT_FACTORY_DEFINITION`
is the same shape for `IComponentSingleton`, adding ref-count bookkeeping so the
singleton is created once and shared.

The net effect is that a new backend is a new file: declare the `CID`, inherit
`Component<IYourInterface>`, and drop a `CE_COMPONENT_FACTORY_DEFINITION` in the
.cpp. Nothing upstream changes.

---

## Creating a component

You ask for a component by naming the implementation you want and handing over
the pointer to fill:

```cpp
// Agent.cpp
const Result res = component::Create(&g_pActiveAgent, CID_Agent);
```

`g_pActiveAgent` is an `IAgent*`. There is no cast, no interface id, and nothing
for the caller to keep in step. `T` is deduced from the destination pointer and
the interface id is computed from `T`:

```cpp
// Component.h
template <typename T>
Result Create(T** ppInterface, const ComponentId ComponentId, const scope::Handle hScope = scope::Handle())
{
    static_assert(types::IsBaseOfV<IComponentBase, T>, "component::Create: T must be a component interface, derived from IComponentBase");
    static_assert(!types::IsSameV<T, IComponentBase>, "component::Create: name the interface you actually want, not IComponentBase");

    return CreateComponent(ComponentId, MakeInterfaceId<T>(), reinterpret_cast<IComponentBase**>(ppInterface), hScope);
}
```

The scope argument decides which arena the instance lives in: Bootstrap for
engine-lifetime singletons, Scene for per-scene components, and so on, from
[Core's scopes](Core.md#allocators-and-scope). `CreateFromInterface<T>` is the
same shape for the single-implementation-per-interface lookup, and takes no
`ComponentId` at all.

Two things the call cannot get wrong are worth naming, because both are compile
errors rather than runtime surprises. The interface id always matches the pointer
being written, because one is computed from the other. And the two
`static_assert`s reject a `T` that is not a component interface, or a `T` that is
`IComponentBase` itself when you meant a contract.

`Create` is a template, so it sits outside the script-export range and generates
no bindings. The raw `CreateComponent` below stays exported for the
[generated C, C# and Lua surface](ScriptGeneration.md), where there is no
template deduction to lean on.

### The rest of the creation surface

```cpp
// Component.h
CE_API Result CreateComponent(const ComponentId ComponentId, const InterfaceId InterfaceId,
                              IComponentBase** ppInterface, const scope::Handle hScope = scope::Handle());
CE_API Result CreateComponentFromInterfaceId(const InterfaceId InterfaceId,
                              IComponentBase** ppInterface, const scope::Handle hScope = scope::Handle());
CE_API Result CreateAllComponentsFromInterfaceId(const InterfaceId InterfaceId, const scope::Handle hScope = scope::Handle());
CE_API Span<IComponentBase*> GetScopeInstances(const InterfaceId InterfaceId, const scope::Handle hScope);
```

The important detail is inside the create call: the `ComponentId` is passed
through `GetComponentAlias` **before** the factory lookup. That single indirection
is the seam the whole backend-swapping story hangs on:

```cpp
// Component.cpp
Result CreateComponent(const ComponentId CID, const InterfaceId IID, IComponentBase** ppInterface, const scope::Handle hScope)
{
    IComponentFactory* p_factory = nullptr;
    {
        auto                         lock         = LockGuard(g_.Mutex);
        const ComponentId            component_id = GetComponentAlias(CID);
        ComponentFactories::iterator factoryIter  = g_.ComponentFactories.find({component_id, IID});
        if (factoryIter != g_.ComponentFactories.end())
        {
            p_factory = factoryIter->pFactory;
        }
    }
    if (p_factory != nullptr)
    {
        Result ier = p_factory->createComponent(IID, ppInterface, hScope);
        // ...
```

The other three functions cover the "many implementations" case directly.
`CreateAllComponentsFromInterfaceId` instantiates *every* component registered
against an interface, and `GetScopeInstances` returns the live instances of an
interface in a scope. This is exactly how the scheduler builds its system set: it
does not call a `RegisterSystem`; it asks the component system for every
`ISystem` created in the Bootstrap scope and schedules those. As
[Core notes](Core.md#tasks-threading-and-the-fixed-tick-clock), "registration
falls out of the component system." A typed helper makes the iteration clean:

```cpp
// Component.h
template <typename T>
Span<T*> GetScopeInstances(const InterfaceId InterfaceId, const scope::Handle hScope)
{
    Span<IComponentBase*> instances = GetScopeInstances(InterfaceId, hScope);
    return Span<T*>(static_cast<T**>(static_cast<void*>(instances.data())), instances.size());
}
```

---

## The trunk-and-backends pattern

Here is where the interface-drives-the-abstraction rule pays off. A package with
multiple swappable implementations is organized as a **trunk** (the manager
package that owns the interface and shared code) and **backends** (leaf packages,
each a concrete implementation). The graphics device is the canonical example:
`Graphics` is the trunk, `GraphicsD3D12` and `GraphicsVk` are the backends, and a
built-in **Null device** is the no-GPU fallback that unit tests, headless smoke
runs, and dedicated servers render against.

The problem the pattern has to solve is subtle. A caller wants to say "give me the
active graphics device" without naming Direct3D 12 or Vulkan. But which backend is
active depends on what DLLs loaded, and DLL static-init order across the process is
not guaranteed. So the resolution is layered, with four places each covering a
real scenario, and a config file getting the last word.

**Layer 1: the generic `CID` in the trunk's `Module.h`.** The trunk declares a
generic id next to its `Null` fallback id. Callers create through the generic one.

```cpp
// Pkg/Engine/Graphics/Module.h
// CID_Device is the generic "active device" CID; each backend's module
// Initializer aliases it to its own concrete CID (CID_DeviceD3D12 /
// CID_DeviceVk) so GetDevice() resolves to whichever backend is wired in.
// CID_DeviceNull is the trunk's no-GPU fallback for headless tests and servers.
CE_DECLARE_COMPONENT_ID(Device)
CE_DECLARE_COMPONENT_ID(DeviceNull)
```

**Layer 2: a default alias in the trunk's `ModuleFactories.cpp`.** The trunk
points its own generic id at the `Null` device, so the package works standalone
with no backend DLL loaded. This is the seam that makes the whole engine testable
without a GPU: unit tests, the headless smoke runs, and dedicated servers all hold
this default and still exercise every code path above the device line.

```cpp
// Pkg/Engine/Graphics/ModuleFactories.cpp: Initializer::Initializer()
// Default alias: CID_Device -> CID_DeviceNull.  The Null device is a no-op,
// no-GPU implementation, so a process that links no backend (a unit test, a
// headless dedicated server) still resolves a valid device and runs.
component::SetComponentAlias(graphics::CID_Device, graphics::CID_DeviceNull);
```

**Layer 3: each backend overrides the alias to point at itself.** When a backend
DLL loads, its initializer claims the generic id. This is what makes "just link
GraphicsVk" enough for a plain (non-editor) game executable to render on Vulkan:

```cpp
// Pkg/Engine/GraphicsD3D12/ModuleFactories.cpp: Initializer::Initializer()
component::SetComponentAlias(graphics::CID_Device, CID_DeviceD3D12);

// Pkg/Engine/GraphicsVk/ModuleFactories.cpp: Initializer::Initializer()
component::SetComponentAlias(graphics::CID_Device, CID_DeviceVk);
```

**Layer 4: `config.lua` pins the choice deterministically.** With two GPU backends
the need is obvious: link both `GraphicsD3D12` and `GraphicsVk`, and layer 3 alone
would hand the device to whichever DLL happened to initialize last. The runtime
config settles it. This line is the only load-order-independent one:

```lua
-- Resources/Scripts/config.lua (config.apply())
-- Both GPU backends may be linked; whichever DLL loads last would win by
-- static-init order alone, so pin the choice here. The -graphicsBackend arg
-- selects it, defaulting to D3D12. Omit the pin and the default
-- CID_Device -> CID_DeviceNull fallback holds, which is exactly what a
-- headless / unit-test process wants.
if ceili.core.platform.hasCommandLineArgValue("-graphicsBackend", "vk") then
    ceili.core.component.setComponentAlias(ceili.graphics.CID_Device, ceili.graphics.vk.CID_DeviceVk)
else
    ceili.core.component.setComponentAlias(ceili.graphics.CID_Device, ceili.graphics.d3d12.CID_DeviceD3D12)
end
```

The alias itself is a trivial map, resolved on every create with the mutex held:

```cpp
// Component.cpp
ComponentId GetComponentAlias(const ComponentId SrcComponentId)
{
    auto lock = LockGuard(g_.Mutex);
    const auto alias = g_.ComponentAliases.find(SrcComponentId);
    if (alias == g_.ComponentAliases.end()) { return SrcComponentId; }
    return alias->dst;
}

Result SetComponentAlias(const ComponentId SrcComponentId, const ComponentId DstComponentId)
{
    if (DstComponentId == kNullComponentId) { return core::results::failure::UnknownComponent; }
    auto lock = LockGuard(g_.Mutex);
    auto alias = g_.ComponentAliases.find(SrcComponentId);
    if (alias == g_.ComponentAliases.end()) { g_.ComponentAliases.insert({SrcComponentId, DstComponentId}); }
    else { alias->dst = DstComponentId; }
    return core::results::success::Ok;
}
```

Here is how the four layers settle across DLL load, with `config.lua` overwriting
whatever static init left behind:

```mermaid
flowchart TD
    subgraph load["DLL load order (unordered across the process)"]
        T["Graphics trunk loads<br/>Layer 2: CID_Device -> CID_DeviceNull"]
        D["GraphicsD3D12 loads<br/>Layer 3: CID_Device -> CID_DeviceD3D12"]
        V["GraphicsVk loads<br/>Layer 3: CID_Device -> CID_DeviceVk"]
    end
    T --> STATE["alias table:<br/>CID_Device -> whichever<br/>backend initialized last"]
    D --> STATE
    V --> STATE
    STATE --> CFG{"config.apply()<br/>runs setComponentAlias"}
    CFG -->|"Studio / game run"| PIN["CID_Device -> CID_DeviceVk<br/>(deterministic)"]
    CFG -->|"headless / unit tests<br/>(no backend DLL)"| FALL["CID_Device -> CID_DeviceNull<br/>(default holds)"]
    PIN --> CREATE["GetDevice() -> Create(&g_pDevice, CID_Device)<br/>resolves through GetComponentAlias"]
    FALL --> CREATE
```

Why all four? Remove any one and a real scenario breaks. Without layer 2, a
headless unit test or dedicated server with no GPU backend loaded could not create
a device at all; the Null device is what keeps the engine runnable with no
adapter. Without layer 3, linking `GraphicsVk` into a plain (non-editor) game
executable would not make it the active device. Without layer 4, with both GPU
backends linked the choice would depend on DLL load order, which is not stable.
Layer 1 is just the shared name they all agree to alias. Each covers a distinct
case, which is why the redundancy is real and not decoration.

Consumers never see any of this. They create through the generic id (or the
`GetDevice()` facade that wraps it):

```cpp
component::Create(&g_pDevice, CID_Device);
```

`g_pDevice` is an `IDevice*`, and `CID_Device` is the generic id. Which concrete
backend answers is settled by the alias chain inside the call.

Where a leading data-oriented sample would have you compile-time-select a backend
or thread a policy through templates, and the commercial heavyweights lean on a
module registry loaded from config, Ceili's version is one map lookup on a hashed
id, resolved at the create call. The cost is a mutex-guarded `find` per creation,
which is not on any hot path (components are created at load, not per frame), and
the benefit is that swapping a backend is a config line, not a rebuild.

---

## Components in C++, C#, or Lua

Because a component is reached only through its interface and its factory, the
language a component is *implemented* in is invisible to callers. A component can
be written in C++, C#, or Lua, and the create/destroy/update path is identical.

The bridge is a generated **script wrapper**: a C++ class that inherits the same
`Component<I>` base as any native component, but whose method bodies are
function-pointer trampolines that a script fills in. The script generator emits
one per script-facing interface (see [Script Generation](ScriptGeneration.md)):

```cpp
// Core_ScriptingC.cpp: header-marked "// Generated File - DO NOT MODIFY"
class ceili_core_component_IComponentSingletonScriptWrapper
    : public ceili::core::component::Component<ceili::core::component::IComponentSingleton>
{
public:
    ceili::core::Result init() override
    {
        CE_ASSERT(m_initFuncPtr != nullptr)
        if (m_OwnerTid != ceili::core::thread::kInvalidId && ceili::core::thread::GetCurrentId() != m_OwnerTid)
        {
            return ceili::core::results::failure::NotOnOwningThread;
        }
        const ceili::core::WeakResult_t res = m_initFuncPtr();
        return ceili::core::Result{res};
    }
    // ...
    using initFuncPtr = ceili::core::WeakResult_t (*)();
    initFuncPtr m_initFuncPtr;
};
```

The wrapper's factory is registered as a **clone template**: its create function
is null, and each script instance clones the template, wiring its own trampoline
pointers inside the clone's create callback before `init()` runs. A Lua or C#
component registers itself via `registerComponentFactoryFunctions(uniqueCID,
[IID], createCB, destroyCB)`, and from that point the instance is scope-allocated
and tracked in the same instance list as any native component. The engine
discriminates native from script instances by whether `getComponentFactory()`
returns null, and always tears a script instance down through `destroy()` (which
routes back to the factory), never a raw delete.

The consequence is the one that matters for [AI Integration](AiIntegration.md) and
gameplay scripting: a system, an agent backend, or a gameplay behaviour authored
in Lua is a first-class component. It schedules, updates, serializes, and swaps
exactly like the C++ ones, because to the rest of the engine it *is* one.

```mermaid
flowchart LR
    IFACE["IComponentSingleton<br/>(one interface)"]
    IFACE --> CPP["C++ backend<br/>(Component subclass)"]
    IFACE --> CS["C# component<br/>(via script wrapper)"]
    IFACE --> LUA["Lua component<br/>(via script wrapper)"]
    CPP --> SAME["Same CreateComponent /<br/>update / destroy / scope path"]
    CS --> SAME
    LUA --> SAME
```

---

## ComponentFlags and instance state

`ComponentFlags` is the small bitfield every component carries, and today it holds
one meaningful bit, `Active`, defaulted on:

```cpp
// Component.h
CE_BITFIELD enum class ComponentFlags : uint32_t {
    None   = 0,
    Active = 1 << 0,
};
```

Being a `CE_BITFIELD` enum, it composes with the strongly-typed bit helpers from
[Core](Core.md) (`HasBitfield`, `|=`, `&= ~`) rather than raw integer masking. The
scheduler and renderers read `Active` to decide whether to tick or draw an
instance, so toggling it is how a system is paused without being destroyed.

Flags can be set per instance through `IComponentBase`, or per `ComponentId`
through free functions that fan the flag out to every live instance of that id:

```cpp
// Component.h
CE_API ComponentFlags GetComponentFlags(const ComponentId ComponentId);
CE_API void           SetComponentFlags(const ComponentId ComponentId, const ComponentFlags Flags);
CE_API void           AddComponentFlags(const ComponentId ComponentId, const ComponentFlags Flags);
CE_API void           RemoveComponentFlags(const ComponentId ComponentId, const ComponentFlags Flags);
```

```cpp
// Component.cpp
void SetComponentFlags(const ComponentId ComponentId, const ComponentFlags Flags)
{
    auto lock = LockGuard(g_.Mutex);
    // Apply to every scope-tracked instance of this CID.  Systems / renders are singletons
    // (one instance per CID), so this sets exactly one; a multi-instance component would have
    // all instances flagged alike -- the CID is the granularity callers reason about.
    for (const auto& entry : g_.ScopeInstances)
    {
        IComponentFactory* const p_factory = entry.pInstance->getComponentFactory();
        if (p_factory != nullptr && p_factory->getComponentId() == ComponentId)
        {
            entry.pInstance->setComponentFlags(Flags);
        }
    }
}
```

The CID is the granularity callers reason about: because systems and renderers are
singletons, "flag this component id" sets exactly one instance, and the loop reads
naturally as "flag this thing." Note the `getComponentFactory() != nullptr` guard,
the same discriminator that separates native instances from script wrappers.

---

## How it all composes

The component system is deliberately thin, and its leverage comes from how few
ideas it needs:

- An **interface** is a contract (`IID`), a **component** is an implementation
  (`CID`), and creation is a lookup on the `{CID, IID}` pair.
- The lookup runs the `CID` through an **alias** first, so the concrete backend is
  chosen at runtime, not compile time.
- **The ids are derived from types at the point of use.** A registration names the
  interface type and a creation names the destination pointer, so
  `Create(&g_pDevice, CID_Device)` has no cast and no second id to keep in step.
- **Registration falls out of linking**: a nifty-counter struct registers the
  factory at DLL load, so a new backend is a new file and nothing upstream changes.
- **Language is invisible**: a script wrapper is a `Component` like any other, so
  C++, C#, and Lua components share one create/update/destroy path.

That is the concrete realization of "swap any implementation" from
[Philosophy](Philosophy.md). Two places in the engine lean on it hardest and are
the best next reads: the graphics trunk running every graphics backend off one
generic device id ([Rendering](Rendering.md)), and the agent trunk swapping a
hardcoded `Null` stub for a live subprocess backend
([AI Integration](AiIntegration.md)). Both are the same four-layer alias pattern,
pointed at different work.

Next: [Rendering](Rendering.md), or back to the
[documentation index](README.md).
