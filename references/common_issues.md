# Common Issues and Solutions

These examples are translated from the source skill, including its original illustrative API calls. They are not independently compiled examples. See [the validation limits](../README.md#version-scope-and-validation-limits) before copying code into a plugin.

## Contents

- [Compilation warnings](#compilation-warnings): C5038 initialization order, C2440 conversions.
- [Header issues](#header-issues): complete types for TUniquePtr, .generated.h order.
- [Reflection](#reflection): UFUNCTION callbacks, GENERATED_BODY placement.
- [Performance](#performance): expensive Tick work, unnecessary allocations.
- [Thread safety](#thread-safety): nested locking.
- [Memory leaks](#memory-leaks): circular UObject references, non-UObject memory.
- [Blueprint issues](#blueprint-issues): incomplete metadata, Category format.
- [TArray type incompatibility](#tarray-type-incompatibility): TInlineAllocator, SetNumZeroed, member shadowing.
- [Spatial data structures](#spatial-data-structures): TMap and spatial grids.
- [Debugging](#debugging): conditional compilation, tracing.

## Compilation Warnings

### C5038: Initializer List Order

```cpp
// Incorrect: initialization order differs from declaration order
class FMyStruct
{
    int32 B;
    int32 A;

    FMyStruct()
        : B(1)  // C5038 warning in the original example
        , A(2)
    {}
};

// Correct: initialize in declaration order
class FMyStruct
{
    int32 A;  // Declared first
    int32 B;  // Declared second

    FMyStruct()
        : A(2)  // Initialized first
        , B(1)  // Initialized second
    {}
};
```

### C2440: Type Conversion

```cpp
// Incorrect: direct TSubclassOf conversion
TSubclassOf<AActor> ActorClass;
TSoftObjectPtr<AActor> SoftPtr = ActorClass;  // C2440

// Correct according to the source example: obtain UClass* with Get()
TSubclassOf<AActor> ActorClass;
TSoftObjectPtr<AActor> SoftPtr = ActorClass.Get();
```

## Header Issues

### TUniquePtr Requires a Complete Definition

```cpp
// Incorrect: forward declaration only
class FMyData;
class FMyClass
{
    TUniquePtr<FMyData> Data;  // Error: destructor needs the complete definition
};

// Correct: include the full header
#include "MyData.h"
class FMyClass
{
    TUniquePtr<FMyData> Data;  // Correct
};

// TSharedPtr permits a forward declaration
class FMyOtherData;
class FMyOtherClass
{
    TSharedPtr<FMyOtherData> Data;  // Correct: forward declaration is sufficient
};
```

### .generated.h Include Order

```cpp
// Incorrect: .generated.h is not last
#include "MyActor.h"
#include "MyActor.generated.h"
#include "OtherHeader.h"  // Incorrect

// Correct: .generated.h must be last
#include "MyActor.h"
#include "OtherHeader.h"
#include "MyActor.generated.h"  // Correct
```

## Reflection

### UFUNCTION Is Unnecessary for Non-Dynamic Delegate Callbacks

```cpp
// Incorrect: callback bound with AddLambda is marked UFUNCTION
UFUNCTION()
void MyCallback();

MyDelegate.AddLambda(this, &UMyClass::MyCallback);

// Correct according to the source example: remove UFUNCTION
void MyCallback();

MyDelegate.AddLambda(this, &UMyClass::MyCallback);

// Or use a dynamic delegate, which requires UFUNCTION
DECLARE_DYNAMIC_DELEGATE_OneParam(FMyDelegate, int32, Value);

UFUNCTION()
void MyDynamicCallback(int32 Value);

MyDynamicDelegate.BindDynamic(this, &UMyClass::MyDynamicCallback);
```

### GENERATED_BODY() Placement

```cpp
// Incorrect: GENERATED_BODY() is not the first line
UCLASS()
class AMyActor : public AActor
{
    // Other code
    GENERATED_BODY()  // Incorrect
};

// Correct: GENERATED_BODY() must be first
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()  // Correct

    // Other code
};
```

## Performance

### Expensive Work in Tick

```cpp
// Incorrect: expensive work every frame
void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    for (int32 i = 0; i < 10000; ++i)
    {
        ExpensiveCalculation();  // Every frame!
    }
}

// Correct: interval-based updates
void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (GetWorld()->GetTimeSeconds() - LastUpdateTime < 0.1f)
    {
        return;  // Skip
    }

    LastUpdateTime = GetWorld()->GetTimeSeconds();
    for (int32 i = 0; i < 10000; ++i)
    {
        ExpensiveCalculation();
    }
}
```

### Unnecessary Allocations

```cpp
// Incorrect: repeated allocation inside the loop
for (int32 i = 0; i < 1000; ++i)
{
    TArray<FString> TempArray;  // Allocates each iteration
    TempArray.Add(Item);
}

// Correct: allocate in advance
TArray<FString> TempArray;
TempArray.Reserve(1000);  // Preallocate
for (int32 i = 0; i < 1000; ++i)
{
    TempArray.Add(Item);
}
```

## Thread Safety

### Nested Locking

```cpp
// Incorrect: nested locks can deadlock
FRWLock Lock1;
FRWLock Lock2;

void Thread1()
{
    Lock1.WriteLock();
    Lock2.WriteLock();  // May deadlock
    // ...
    Lock2.WriteUnlock();
    Lock1.WriteUnlock();
}

void Thread2()
{
    Lock2.WriteLock();
    Lock1.WriteLock();  // May deadlock
    // ...
    Lock1.WriteUnlock();
    Lock2.WriteUnlock();
}

// Correct: consistent lock ordering
void Thread1()
{
    Lock1.WriteLock();
    Lock2.WriteLock();
    // ...
    Lock2.WriteUnlock();
    Lock1.WriteUnlock();
}

void Thread2()
{
    Lock1.WriteLock();  // Same order
    Lock2.WriteLock();
    // ...
    Lock2.WriteUnlock();
    Lock1.WriteUnlock();
}
```

## Memory Leaks

### Circular UObject References

```cpp
// Incorrect under the source's rule: circular raw-pointer UObject references
UCLASS()
class AActorA : public AActor
{
    UPROPERTY()
    AActorB* RefB;  // Strong reference
};

UCLASS()
class AActorB : public AActor
{
    UPROPERTY()
    AActorA* RefA;  // Strong reference: cycle
};

// Correct under the source's rule: one weak reference
UCLASS()
class AActorB : public AActor
{
    UPROPERTY()
    TWeakObjectPtr<AActorA> RefA;  // Weak reference breaks the cycle
};
```

### Non-UObject Memory Management

```cpp
// Incorrect: new without a matching delete
class FMyData
{
    static FMyData* Create()
    {
        return new FMyData();  // No matching delete
    }
};

// Correct: smart pointer ownership
class FMyData
{
    static TSharedPtr<FMyData> Create()
    {
        return MakeShared<FMyData>();  // Automatic management
    }
};
```

## Blueprint Issues

### Incomplete Metadata

The source uses Chinese display metadata. The demonstration labels below are translated into English while retaining the metadata fields and structure.

```cpp
// Incorrect: missing localized metadata
UFUNCTION(BlueprintCallable)
static void MyFunction(FString Data);

// Correct: complete display metadata
UFUNCTION(BlueprintCallable, Category = "MyPlugin|MyModule",
    meta = (
        DisplayName = "My Function",
        Keywords = "keyword",
        ToolTip = "Detailed function description\nParameters:\nData - Data description"
    ))
static void MyFunction(
    UPARAM(DisplayName="Data") const FString& Data);
```

### Inconsistent Category Format

```cpp
// Incorrect: inconsistent formats
UFUNCTION(BlueprintCallable, Category = "MyFunction")  // Missing module prefix
UFUNCTION(BlueprintCallable, Category = "MyPlugin|MyFunction")  // Missing subcategory

// Correct: consistent format
UFUNCTION(BlueprintCallable, Category = "MyPlugin|MyModule|Subcategory")
```

## TArray Type Incompatibility

### TInlineAllocator and the Default Allocator Are Not Interchangeable

```cpp
// [BAD] Compilation error: incompatible types
TArray<int32, TInlineAllocator<64>> LocalBuffer;
void QuerySphere(TArray<int32>& OutResult);  // Default allocator
QuerySphere(LocalBuffer);  // C2664: cannot convert

// [OK] Option 1: default TArray with Reserve
TArray<int32> LocalBuffer;
LocalBuffer.Reserve(64);
QuerySphere(LocalBuffer);

// [OK] Option 2: template the downstream function
template<typename AllocType>
void QuerySphere(TArray<int32, AllocType>& OutResult);
```

### SetNumZeroed Versus SetNumUninitialized + Memzero

```cpp
// SetNumZeroed zeros only newly added elements, not existing elements when shrinking
// For a reused array that must be fully zeroed every frame:
Array.SetNumUninitialized(N);
FMemory::Memzero(Array.GetData(), N * sizeof(T));

// Or Reset + SetNumZeroed when the previous size is uncertain
Array.Reset();
Array.SetNumZeroed(N);
```

### Shadowing a Member Array

```cpp
// [BAD] Local variable shadows a member: C4458
class FSystem {
    TArray<float> CorrectionSums;  // Member

    void SolveObstacles() {
        TArray<float> CorrectionSums;  // [BAD] Shadows the member!
    }
};

// [OK] Use a different local name
void SolveObstacles() {
    TArray<float> ObstacleCorrectionSums;  // [OK] No shadowing
}
```

## Spatial Data Structures

### TMap and Spatial Grids

The source discourages a plain FIntVector TMap spatial grid and demonstrates a custom spatial hash key below. This example still uses TMap; preserve that distinction when applying the rule.

```cpp
// [BAD] The original comment says Epic never uses TMap for spatial grids
TMap<FIntVector, TArray<int32>> GridCells;

// [OK] Custom spatial hash key
struct FCellKey {
    FIntVector Coord;
    explicit FCellKey(const FIntVector& C) : Coord(C) {}
    bool operator==(const FCellKey& O) const { return Coord == O.Coord; }
    friend uint32 GetTypeHash(const FCellKey& K) {
        return uint32(K.Coord.X) * 1150168907u
             + uint32(K.Coord.Y) * 1235029793u
             + uint32(K.Coord.Z) * 1282581571u;
    }
};
TMap<FCellKey, TArray<int32>> GridCells;
```

## Debugging

The Stats system (`stat game`) differs from Insights tracing (`TRACE_CPUPROFILER_EVENT_SCOPE`). See [performance_patterns.md](performance_patterns.md#stats).

### Conditional Debug Code

```cpp
// Debug builds only
#if UE_BUILD_DEBUG
    UE_LOG(LogTemp, Warning, TEXT("Debug: Value = %d"), MyValue);
#endif

// Development builds only
#if !UE_BUILD_SHIPPING
    verify(MyCondition);  // Compiled out in Shipping because of this guard
#endif
```

### Performance Tracing

```cpp
// CPU Profiler tracing
TRACE_CPUPROFILER_EVENT_SCOPE(MyFunction);

void MyFunction()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(Section1);
    // Code 1

    TRACE_CPUPROFILER_EVENT_SCOPE(Section2);
    // Code 2
}
```
