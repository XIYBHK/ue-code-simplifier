# Common Issues and Solutions

These are contextual review examples, not a collection of independent compile-ready plugins. Paired bad/good fragments are alternatives. A deliberately bad example is a diagnostic demonstration, not recommended code. Actual UHT, module, linker, target, and runtime checks are still required; see [VALIDATION.md](../VALIDATION.md).

## Contents

- [Compilation warnings](#compilation-warnings)
- [Header issues](#header-issues)
- [Reflection and delegates](#reflection-and-delegates)
- [Performance](#performance)
- [Thread safety](#thread-safety)
- [Ownership and leaks](#ownership-and-leaks)
- [Blueprint metadata](#blueprint-metadata)
- [Array types and initialization](#array-types-and-initialization)
- [Spatial data structures](#spatial-data-structures)
- [Debugging and tracing](#debugging-and-tracing)

## Compilation Warnings

### Initialization Order

Members initialize in declaration order, not initializer-list order. The original negative example had matching orders and was wrong. A genuine mismatch is illustrated below; warning number/enablement depends on the compiler.

```cpp
// Intentionally mismatched order: A is declared before B, but listed after it.
struct FBadInitializer
{
    int32 A;
    int32 B;
    FBadInitializer() : B(1), A(2) {}
};
```

```cpp
// Correct list for the same declaration order.
struct FOrderedInitializer
{
    int32 A;
    int32 B;
    FOrderedInitializer() : A(2), B(1) {}
};
```

Match the initializer list to the required declaration order; do not reorder members just to silence a warning without checking initialization dependencies and ABI/layout.

### Class versus Object Soft References

```cpp
// Function-body fragment: ActualActorClass is a valid, compatible UClass*.
TSubclassOf<AActor> ActorClass = ActualActorClass;
TSoftClassPtr<AActor> SoftClass(ActorClass.Get());
```

An Actor class (`UClass*`) is not an Actor instance. `TSoftObjectPtr<AActor>` must not be used as the class-reference destination. Some older soft-object constructors accept incompatible types and may compile while yielding unusable results; inspected newer headers deprecate such conversions. Use the correct type instead of assuming `.Get()` repairs all conversions. Validate unset/invalid classes and loading behavior separately. See [source evidence](../VALIDATION.md#api-and-behavior-evidence).

## Header Issues

### TUniquePtr and Incomplete Types

```cpp
// MyOwner.h: ordinary class declarations, with the appropriate UniquePtr header.
class FPrivateData;
class FMyOwner
{
public:
    FMyOwner();
    ~FMyOwner();
private:
    TUniquePtr<FPrivateData> Data;
};
```

```cpp
// MyOwner.cpp: complete type available before the relevant definitions.
#include "MyOwner.h"
#include "PrivateData.h"
FMyOwner::FMyOwner() = default;
FMyOwner::~FMyOwner() = default;
```

The default deletion operation requires completeness where instantiated; the member declaration alone does not always require the full header. Other inline operations or custom deleters may change the requirement. `TSharedPtr` can also hold a forward-declared type, but creation and actual member access still need the relevant definition.

### Generated Includes

```cpp
// A reflected MyActor.h: do not include itself.
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h" // Last include.
```

In MyActor.cpp include MyActor.h first, without a separate generated include. Other includes must be self-sufficient for non-Unity builds. `Engine/Engine.h` can be the correct specific header for UEngine/GEngine; avoid the aggregate Engine.h when it is unnecessary.

## Reflection and Delegates

### Native Member Binding

```cpp
// UObject method-body fragment; callback signature matches the native delegate.
FDelegateHandle Handle = MyDelegate.AddUObject(this, &UMyClass::MyCallback);
// Retain/remove Handle according to the actual lifetime.
```

`AddLambda(this, &Class::Method)` is not a member-binding API. Supply a callable lambda to AddLambda or use AddUObject/another appropriate binding. Native binding does not need UFUNCTION by itself; preserve any other Blueprint/RPC/reflection use of the function. Dynamic bindings use the matching dynamic declaration plus UFUNCTION and can be needed for serialization/reflection as well as Blueprint. See [delegate patterns](api_patterns.md#delegate).

### GENERATED_BODY

```cpp
// Reflected header class fragment; its generated header must already be included.
UCLASS()
class AMyActor : public AActor
{
    // Comments before the macro do not change its syntax or position.
    GENERATED_BODY()
public:
    void PerformWork();
};
```

Place the generated-body macro at the expected beginning of a reflected declaration and use the appropriate UCLASS/USTRUCT/interface form. A comment before it is not an invalid statement. Do not invent a UHT failure from a pair of examples that differ only in comments.

## Performance

Measure before changing scheduling, data structures, or ownership. Reducing update frequency can change latency, integration, and call counts. Cache only with a clear invalidation/lifetime policy. Removing independent temporary arrays and accumulating their contents into one array is not necessarily equivalent.

For scratch reuse, choose the correct reset boundary and preserve required outputs. Shared buffers need exclusive access or safe partitioning; see [hot paths](performance_patterns.md#hotpath) and [ParallelFor](performance_patterns.md#parallelfor).

## Thread Safety

```cpp
// Source-file illustration. Include HAL/CriticalSection.h and Misc/ScopeRWLock.h.
FRWLock Lock1;
FRWLock Lock2;

void FirstOperation()
{
    FWriteScopeLock First(Lock1);
    FWriteScopeLock Second(Lock2);
    // Work while holding both locks.
}

void SecondOperation()
{
    FWriteScopeLock First(Lock1);  // Same global order.
    FWriteScopeLock Second(Lock2);
    // Work while holding both locks.
}
```

Consistent acquisition order prevents the illustrated opposite-order ABBA problem; scoped locks also release on normal scope exits. It is not proof of freedom from every deadlock or race. Check callbacks, reentrancy, lock hierarchy, and GameThread dependencies; do not hold an inappropriate lock while invoking external callbacks.

## Ownership and Leaks

UObject references use tracing GC. An unreachable strong-reference cycle can be collected. Diagnose roots/external referencers before changing ownership, and check the actual Actor/World destruction lifecycle. TSharedPtr reference-counting cycles have a different mechanism and may need TWeakPtr edges. See [GC safety](safety_patterns.md#gc).

```cpp
// Ordinary non-UObject allocation with explicit RAII ownership.
TUniquePtr<FMyData> Data = MakeUnique<FMyData>();
```

A naked new is not automatically proof of a leak if ownership is properly transferred and deletion occurs elsewhere. Prefer clear RAII ownership, and verify the caller contract. Do not use these non-UObject ownership tools to ordinary-delete UObjects.

## Blueprint Metadata

```cpp
// Declaration in an appropriate reflected class; define the function separately.
UFUNCTION(BlueprintCallable, Category = "MyPlugin|Utilities",
    meta = (DisplayName = "Process Data", ToolTip = "Process the supplied data.",
            Keywords = "process,data", AdvancedDisplay = "Extra"))
static void ProcessData(UPARAM(DisplayName = "Data") const FString& Data, int32 Extra = 0);
```

Category depth and naming are team conventions, not a mandatory three-segment grammar. Some Engine-module Blueprint APIs require an explicit category; apply UHT's actual context. Display strings and comments are not proof of completed localization. Optional metadata must reference real parameters and preserve existing Blueprint interfaces. See [metadata](api_patterns.md#uproperty).

## Array Types and Initialization

```cpp
// Alternative declarations, not two overloads with interchangeable reference types.
template<typename AllocatorType>
void QuerySphere(TArray<int32, AllocatorType>& OutResult);
```

TArray allocator specializations are distinct types; a default mutable TArray reference cannot bind to a TInlineAllocator specialization. Alternatively use a suitable view for a non-resizing operation or keep the default allocator and reserve capacity.

`SetNumZeroed` only zero-initializes newly added elements. Use a type-safe reset for the entire reused range; byte-clearing arbitrary nontrivial elements is unsafe. Capacity/shrinking, bounds, and thread ownership still matter. Avoid accidental local variables shadowing persistent members, but interpret warning severity in the actual compiler/configuration.

## Spatial Data Structures

```cpp
// Valid candidate for a sparse grid; benchmark its key distribution and access pattern.
TMap<FIntVector, TArray<int32>> GridCells;
```

Epic uses TMap in spatial data structures. A custom hash can still be used with TMap; it is not a different container. Include multilevel identity in keys where required and measure collisions/memory/query costs before replacing a valid structure. See [container selection](performance_patterns.md#container).

## Debugging and Tracing

```cpp
// Function-body fragment: optional diagnostic only.
#if UE_BUILD_DEBUG
UE_LOG(LogMyPlugin, Verbose, TEXT("Diagnostic value: %d"), Value);
#endif
```

`!UE_BUILD_SHIPPING` includes multiple non-Shipping configurations, not only Development. A required operation, even one inside verify, disappears if its entire preprocessor branch is excluded.

```cpp
// Function-body fragment: separate scopes give separate measured sections.
{
    TRACE_CPUPROFILER_EVENT_SCOPE(Section1);
    // First work section.
}
{
    TRACE_CPUPROFILER_EVENT_SCOPE(Section2);
    // Second work section.
}
```

Scope instrumentation belongs inside the function/lexical scope being measured. Consecutive scope declarations in the same block remain active until block exit and are nested, not sequential section timers. Stats and CPU traces can be bridged by configuration; see [profiling](performance_patterns.md#stats).
