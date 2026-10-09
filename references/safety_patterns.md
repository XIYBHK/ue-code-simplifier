# Safety Patterns

Sources: [Epic Asserts (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/asserts-in-unreal-engine?application_version=5.3), [UObject Handling (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-object-handling-in-unreal-engine?application_version=5.3), and [Object Pointers (5.6)](https://dev.epicgames.com/documentation/en-us/unreal-engine/object-pointers-in-unreal-engine?application_version=5.6). The original skill also credits Tom Looman and MrRobinOfficial's GC guidance. See [validation scope](../VALIDATION.md) for the local engine versions inspected.

Examples are fragments for the stated context, not independently compiled plugins. Supply actual class declarations, includes, module dependencies, and project error-handling contracts.

<a id="assert"></a>
## Assertion Macro Selection

| Macro | Default Shipping game behavior | Purpose |
|-------|--------------------------------|---------|
| `check` / `checkf` | Diagnostic disabled; do not rely on expression execution or put required side effects here | Development invariants |
| `ensure` / `ensureMsgf` | Expression evaluated and boolean result returned; diagnostic reporting disabled by default | Nonfatal development diagnostics alongside explicit error handling |
| `verify` / `verifyf` | Expression evaluated; diagnostic disabled by default | An operation that must execute even without checks |

These defaults are not unconditional promises: `USE_CHECKS_IN_SHIPPING`, `USE_ENSURES_IN_SHIPPING`, target type, and build configuration can change them. Shipping Editor is not the same target as a Shipping game. In configurations with reporting enabled, ordinary `ensure` variants generally report once per call site; `ensureAlways` variants request repeated reporting, subject to platform/reporting configuration.

An assertion does not provide recovery. External input must have real validation and a defined failure path. Required operations must not be hidden inside `check`, `#if !UE_BUILD_SHIPPING`, or another optional diagnostic path. If production telemetry is required, configure and verify it separately; do not assume an `ensure` is a Shipping log.

```cpp
// Function-body fragment: preserve the failure path in every configuration.
if (!IsValid(Target))
{
    ensureMsgf(false, TEXT("Expected a valid Target"));
    return; // Or return an error / execute the documented fallback.
}
// Perform the intended operation on Target here.
```

Use `check` for an invariant established by program logic, without side effects. Use explicit handling for user data, files, network data, or legitimately absent objects. `verify(Init())` executes `Init()` when checks are disabled, but does not handle a failed return value for you.

<a id="gc"></a>
## GC Safety and Ownership

### UObject Creation

Common application APIs are `CreateDefaultSubobject` for default subobjects in construction, `NewObject` for suitable runtime UObjects, and `UWorld::SpawnActor` for actors. Do not use ordinary `new` / `delete` for UObject lifetime. These three APIs are common choices, not an exhaustive list of engine-internal allocation or loading mechanisms.

```cpp
// Separate fragments; the surrounding class and ownership are project-specific.
// UObject/Actor constructor: retain the component in an appropriate reflected field.
MyComp = CreateDefaultSubobject<UMyComponent>(TEXT("MyComp"));

// Runtime UObject creation: Outer alone is not a universal GC ownership guarantee.
UMyObject* Object = NewObject<UMyObject>(this);
// Establish the required strong reference before it outlives this operation.

// Actor method: SpawnClass must be compatible with AMyActor.
if (UWorld* World = GetWorld())
{
    AMyActor* Actor = World->SpawnActor<AMyActor>(SpawnClass, SpawnTransform);
    // Handle a null result; spawning can fail.
}
```

### Reflected Strong References

In a reachable reflected UObject or a reflected struct reached through its owner, `UPROPERTY` object fields and supported containers participate in reference collection. Prefer `TObjectPtr` for strong reflected UE5 fields where supported. Unreflected raw fields do not acquire this protection automatically.

```cpp
// Member declarations inside a reflected class; distinct names are intentional.
UPROPERTY()
TArray<TObjectPtr<UObject>> TrackedObjects;

UPROPERTY()
TMap<int32, TObjectPtr<UObject>> ObjectsById;

UPROPERTY()
TSet<TObjectPtr<UObject>> UniqueObjects;
```

The owner and the entire reflection/reference chain must be reachable. Adding a macro to an arbitrary ordinary C++ struct does not create that chain. Non-reflected owners can deliberately collect references through `FGCObject` / `AddReferencedObjects`, or use an appropriate `TStrongObjectPtr`; weak observation uses `TWeakObjectPtr`. Select ownership first, then the mechanism.

The modern source evidence for container contents is `FMapProperty::EmitReferenceInfo` and `FSetProperty::EmitReferenceInfo` in `GarbageCollection.cpp`. Legacy `UMapProperty/USetProperty::AddReferencedObjects` collects property descriptors; it is not evidence of traversal of a container's object values.

### IsValid and Object Lifetime

`IsValid(UObject*)` tests null and object validity flags. It cannot safely inspect memory that was already freed, keep an object alive, or make an API thread-safe. Establish a valid reference/lifetime strategy before checking it. For weak references, resolve or pin them with the appropriate API for the engine version and use the result within the required lifetime and thread constraints.

`IsPendingKill()` already carries a UE 5.0 deprecation marker in the inspected UE 5.3 source; this is not a change that begins in 5.4. Use current object-validity APIs appropriate to your version. Do not replace null checks on non-UObject pointers, structs, or platform handles with UObject `IsValid`.

### Ownership Quick Reference

| Need | Typical mechanism | Constraint |
|------|-------------------|------------|
| Strong reflected UObject field | `UPROPERTY() TObjectPtr<T>` | Reachable reflected owner; supported field type |
| Non-owning UObject observation | `TWeakObjectPtr<T>` | Resolve safely; not a strong GC owner |
| Asset/class path loaded on demand | `TSoftObjectPtr<T>` / `TSoftClassPtr<T>` | Explicit loading, loaded-object retention, and cook policy |
| Strong UObject reference outside reflection | Reference collector or suitable `TStrongObjectPtr<T>` | Follow version-specific lifetime/thread rules |
| Shared non-UObject ownership | `TSharedPtr<T>` / `MakeShared<T>()` | Break unwanted reference-counting cycles with weak edges |
| Exclusive non-UObject ownership | `TUniquePtr<T>` / `MakeUnique<T>()` | Complete type where deletion/construction requires it |

UObject GC uses reachability, not `TSharedPtr` reference counting. An unreachable UObject cycle can be collected; a cycle alone does not prove a leak. Trace roots and external referencers before changing strong references to weak ones. A weak edge must match the intended ownership, not merely make a diagram acyclic.

An owning `TUniquePtr<T>` can be declared with a forward-declared type when the operations requiring completeness, particularly destruction with the default deleter, are defined after the complete type is available. Out-of-line owner constructors/destructors are a common PImpl arrangement. Custom deleters may have different requirements. A declaration is not proof that every inline operation is safe.

<a id="tick"></a>
## Tick Management

### Tick Capability versus Initial State

The inspected base `AActor` constructor sets `bCanEverTick=false` and `bStartWithTickEnabled=true`. Derived classes, components, and Blueprint defaults may differ; inspect the actual type. `SetActorTickEnabled` requires `bCanEverTick` to be true and the actor not to be a template object.

```cpp
// Constructor of an actor that will never tick.
PrimaryActorTick.bCanEverTick = false;
```

```cpp
// Alternative constructor setup for an actor that ticks on demand.
// Do not combine this with the "never tick" choice above.
PrimaryActorTick.bCanEverTick = true;
PrimaryActorTick.bStartWithTickEnabled = false;
```

```cpp
// Actor method after an appropriate tick function has been registered.
SetActorTickEnabled(true);
// ... later, when work is no longer needed ...
SetActorTickEnabled(false);
```

Reducing frequency with `TickInterval` changes latency, timing, and potentially simulation behavior. Do so only if the function's contract permits it, and handle elapsed time correctly. Caching `Cast`/component results can help a measured hot path, but cached references need invalidation or safe re-resolution when components, worlds, or instances change. A cast inside Tick is not automatically a defect.

### Tick Outside Actors and Components

```cpp
// Ordinary C++ class fragment. Define Tick in a source file and arrange ownership.
class FMySystem : public FTickableGameObject
{
public:
    void Tick(float DeltaTime) override;
    TStatId GetStatId() const override
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMySystem, STATGROUP_Tickables);
    }
};
```

Choose `FTickableGameObject` only when its ownership and tick behavior fit. Review editor/paused-world behavior, `IsTickable`, teardown, and the relevant includes; it is not a universal replacement for Actor/Component Tick.

<a id="defensive"></a>
## Defensive Programming

Distinguish expected absence from violated invariants. A player controller can legitimately be unavailable during startup/teardown or in a dedicated-server context. An unexpected absence should be diagnosed, but still handled in Shipping.

```cpp
// Function-body fragment inside an Actor, on the GameThread.
UWorld* World = GetWorld();
if (!World)
{
    return; // Choose the documented missing-world behavior.
}

APlayerController* PC = World->GetFirstPlayerController();
if (!IsValid(PC))
{
    // Emit an optional diagnostic only if absence violates this operation's contract.
    ensureMsgf(false, TEXT("No player controller available for this operation"));
    return;
}
// Perform the intended operation using the real controller/custom interface here.
```

For a player index use `UGameplayStatics::GetPlayerController(WorldContextObject, Index)` with its header/module dependencies. `UWorld::GetPlayerController(int)` is not the inspected API. `APlayerController` has no built-in `AddToInventory`; an inventory example must name its actual custom type/interface.

```cpp
// Function-body fragment; no concurrent structural mutation of SomeArray.
if (SomeArray.IsValidIndex(Index))
{
    SomeArray[Index].DoSomething();
}
else
{
    // Return an error or apply the documented out-of-range behavior.
}
```

An index proven valid by a loop or another invariant need not be checked redundantly. Index validity does not validate an element's object lifetime or concurrent access.

## Thread Safety and Shipping Review

Default to GameThread access for UObject APIs unless the specific API and version explicitly support the intended worker-thread operation. Lifetime retention/pinning and data synchronization are separate requirements; neither grants permission to call arbitrary UObject methods concurrently. Prefer value snapshots prepared on the GameThread for parallel computation, then apply results on the correct thread.

Check the consuming target's diagnostic configuration and required error paths. Test the relevant Shipping behavior, GC ownership, Tick registration, dedicated-server/PIE cases, and delegate teardown before reporting runtime safety. Non-reflected system classes are legitimate when reflection is not needed; do not add reflection or remove it merely as an unmeasured optimization.
