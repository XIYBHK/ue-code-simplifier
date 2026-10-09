# Safety Patterns

> Sources cited by the original skill: UE source `AssertionMacros.h`, MrRobinOfficial's GC chapter, and Tom Looman's defensive programming guidance.

<a id="assert"></a>
## Assertion Macro Selection

> Source: UE source `AssertionMacros.h`, lines 351-374 in the source edition consulted by the author.

| Macro | Shipping behavior in the source notes | Use | Example |
|-------|--------------------------------------|-----|---------|
| `check(expr)` | **Removed; not evaluated** | Programming errors that should never occur | `check(Index >= 0)` |
| `checkf(expr,fmt)` | **Removed; not evaluated** | Same, with a message | `checkf(Ptr, TEXT("Null"))` |
| `ensure(expr)` | Evaluated; failure logs only | Recoverable errors and telemetry | `if (ensure(Obj)) { ... }` |
| `ensureMsgf(expr,fmt)` | Evaluated; failure includes a message | Same, with context | `ensureMsgf(SOA, TEXT("SOA null"))` |
| `verify(expr)` | **Expression always evaluated** | Assertions with side effects | `verify(Init())` |

**Key rules:**

- [BAD] `checkf` for runtime input validation: Shipping may crash or skip the check.
- [BAD] Expressions with side effects inside `check()`: they do not execute in Shipping.
- [BAD] `check` for external inputs such as user data, files, or network data: use `ensure` plus error handling.

<a id="gc"></a>
## GC Safety

> Sources: MrRobinOfficial/Guide-UnrealEngine, GC chapter; Epic documentation.

### UObject Creation: Three Methods

```cpp
// [OK] Inside the constructor
MyComp = CreateDefaultSubobject<UMyComponent>(TEXT("MyComp"));

// [OK] Create a UObject at runtime
UMyObject* Obj = NewObject<UMyObject>(this);

// [OK] Spawn an Actor at runtime
AMyActor* Actor = GetWorld()->SpawnActor<AMyActor>(SpawnClass, SpawnTransform);

// [BAD] Never do this
UMyObject* Obj = new UMyObject();  // Not tracked by GC; leaks under the source rule
```

### UPROPERTY as the GC Tracking Entry Point

> Original engine-source observation: `UnrealTypePrivate.h`; UMapProperty and USetProperty implement `AddReferencedObjects`.

**With UPROPERTY, GC tracks references in all these containers:**

```cpp
UPROPERTY()
TArray<UObject*> Tracked;           // [OK] Tracked by GC

UPROPERTY()
TMap<int32, UObject*> AlsoTracked;  // [OK] UE5 UMapProperty supports tracking

UPROPERTY()
TSet<UObject*> AlsoTracked;         // [OK] UE5 USetProperty supports tracking
```

**Without UPROPERTY, none of these containers is tracked under this rule:**

```cpp
TArray<UObject*> NotTracked;        // [BAD] Missing UPROPERTY: not tracked by GC
TMap<int32, UObject*> NotTracked;   // [BAD] Same
```

The key issue is **missing UPROPERTY**, not the container type.

### IsValid() Versus nullptr

> Original engine-source observation: `ObjectMacros.h:591`; the source notes PendingKill deprecation in UE 5.4 and replacement with garbage elimination.

```cpp
// [BAD] Checks only null; the object may already be marked as garbage
if (MyActor != nullptr) { ... }

// [OK] Checks both null and garbage state
if (IsValid(MyActor)) { ... }
```

The original guidance replaces deprecated `IsPendingKill()` with `IsValid()` in UE 5.4+.

### Memory Management Quick Reference

| Scenario | Use |
|----------|-----|
| UObject member reference | `UPROPERTY() TObjectPtr<T>` in UE5 |
| Weak UObject reference | `TWeakObjectPtr<T>` |
| Lazily loaded asset | `TSoftObjectPtr<T>` |
| Shared non-UObject ownership | `TSharedPtr<T>` / `MakeShared<T>()` |
| Exclusive non-UObject ownership | `TUniquePtr<T>` / `MakeUnique<T>()` |
| Circular references | Use `TWeakObjectPtr` / `TWeakPtr` on one side |

**Key rules:**

- Annotate UObject pointer members with `UPROPERTY()` so GC tracks them.
- `TUniquePtr<T>` members require a complete type definition through a header include under the source's rule.
- `TSharedPtr<T>` members may use a forward declaration.

<a id="tick"></a>
## Tick Management

> Sources: MrRobinOfficial best_practices; unreal-engine-cpp-pro.

### Disable Tick by Default

```cpp
// Actor constructor
PrimaryActorTick.bCanEverTick = false;
PrimaryActorTick.bStartWithTickEnabled = false;

// Component constructor
PrimaryComponentTick.bCanEverTick = false;
PrimaryComponentTick.bStartWithTickEnabled = false;
```

### Optimize Necessary Tick Work

```cpp
PrimaryActorTick.TickInterval = 0.2f;  // 5 Hz instead of 60 Hz
SetActorTickEnabled(true);   // Enable on demand
SetActorTickEnabled(false);  // Disable on demand
```

### Tick Outside Actors and Components

```cpp
class FMySystem : public FTickableGameObject {
    void Tick(float DeltaTime) override;
    TStatId GetStatId() const override {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMySystem, STATGROUP_Tickables);
    }
};
```

**Review rules:**

- [BAD] Leaving bCanEverTick at its default true value.
- [BAD] `Cast<T>()` inside Tick: cache references in BeginPlay.
- [BAD] `FindComponentByClass<T>()` inside Tick.

<a id="defensive"></a>
## Defensive Programming

> Source: Tom Looman, "Defensive Programming".

### When to Check nullptr

- [OK] Check a pointer **expected to be nullable**, such as `GetFocusedActor()` when no target exists.
- [BAD] Silently skip a pointer **that should not be null**, such as `GetPlayerController()` when runtime logic requires it to exist.

```cpp
// [BAD] Silently swallow an unexpected null pointer
APlayerController* PC = GetWorld()->GetPlayerController(0);
if (PC) {
    PC->AddToInventory(NewItem);  // Skipping loses the item without notice
}

// [OK] Expose unexpected null pointers with check/ensure
APlayerController* PC = GetWorld()->GetPlayerController(0);
check(PC);
PC->AddToInventory(NewItem);
```

### Safe Array Access

> Source: Ben UI, "IsValidIndex before accessing TArray".

```cpp
// [BAD] Direct indexing; out-of-bounds access crashes
SomeArray[Index].DoSomething();

// [OK] Validate the index first
if (SomeArray.IsValidIndex(Index)) {
    SomeArray[Index].DoSomething();
}
```

## Shipping Build Safety

- Replace `check/checkf` on runtime input paths with `ensure` plus fallback behavior.
- Verify that debugging code guarded by `#if !UE_BUILD_SHIPPING` does not contain required side effects.
- Side effects inside `verify()` execute in Shipping without the assertion.

**Non-UCLASS system classes:** The original reference points to Epic's non-reflected system classes in `RootMotionSource.h`. Performance-critical code need not incur USTRUCT/UCLASS reflection overhead.
