# Performance Patterns

> Basis stated by the original skill: UE 5.6 engine-source observations and practical validation in plugin projects. These observations were not re-tested during translation.

<a id="stats"></a>
## Stats Profiling

> Sources: UE source `BodyInstance.cpp`, `NaniteStreamingManager.cpp`, `TickTaskManager.cpp`.

Cover every subsystem with performance statistics so it is visible in UE Profiler.

**Declare a StatGroup once per module:**

```cpp
// In a Private header
DECLARE_STATS_GROUP(TEXT("MyPlugin"), STATGROUP_MyPlugin, STATCAT_Advanced);

// Optional detailed group, disabled by default
DECLARE_STATS_GROUP_VERBOSE(TEXT("MyPluginDetail"), STATGROUP_MyPluginDetail, STATCAT_Advanced);
```

**Instrument hot functions:**

```cpp
// Method 1: reusable macro; recommended for a consistent prefix
#define MYPLUGIN_SCOPE_CYCLE_COUNTER(Name) DECLARE_SCOPE_CYCLE_COUNTER(TEXT(#Name), STAT_MyPlugin_##Name, STATGROUP_MyPlugin)

void FMySystem::Update() {
    MYPLUGIN_SCOPE_CYCLE_COUNTER(Update);
    // ...
}

// Method 2: quick temporary measurement
QUICK_SCOPE_CYCLE_COUNTER(STAT_MyPlugin_OneOffTest);

// Method 3: nested parent/child scopes
void FMySystem::Tick() {
    MYPLUGIN_SCOPE_CYCLE_COUNTER(Tick_Total);
    {
        MYPLUGIN_SCOPE_CYCLE_COUNTER(Tick_Phase1);
        // ...
    }
}
```

`TRACE_CPUPROFILER_EVENT_SCOPE` is for Insights tracing; `SCOPE_CYCLE_COUNTER` is for the Stats system and the `stat game` command. They have different purposes.

<a id="parallelfor"></a>
## Standard ParallelFor Usage

> Sources: UE source `ParallelFor.h`, `PhysScene_Chaos.cpp`, `NaniteStreamingManager.cpp`.

**Signature recommended by the source:**

```cpp
ParallelFor(TEXT("DebugName"), Num, MinBatchSize, [&](int32 Index) { ... }, Flags);
```

| Check | Incorrect | Correct |
|-------|-----------|---------|
| DebugName | `ParallelFor(N, ...)` | `ParallelFor(TEXT("PBD_Solve"), N, 1, ...)` |
| MinBatchSize | Missing; defaults to 1 | Set explicitly for the workload |
| Flags | Missing | Set `EParallelForFlags::BackgroundPriority` when appropriate |
| Allocation inside lambda | `TArray<T> Temp; Temp.Reserve(N);` | Preallocate or reuse members |

### The TInlineAllocator Trap

`TArray<T, TInlineAllocator<N>>` and `TArray<T>` are **different types**, and their references are not interchangeable. A function accepting `TArray<T>&` cannot receive `TArray<T, TInlineAllocator<N>>`.

Options:

1. Template the downstream function: `template<typename Alloc> void Func(TArray<T, Alloc>&)`.
2. Revert to `TArray<T>` plus `Reserve()`.

<a id="container"></a>
## Container Selection

> Sources: UE source `HierarchicalHashGrid2D.h`, `SimpleCellGrid.h`, `RenderingSpatialHash.h`.

| Scenario | Epic pattern cited by the source | Discouraged |
|----------|----------------------------------|-------------|
| Sparse spatial grid | `TSet<FCell>` with spatial-prime hashing | `TMap<FIntVector, ...>` |
| Dense fixed grid | `TArray<T>` with `Y*Width+X` indexing | `TMap` |
| Hot-path neighbor list | `TArray<T>` with `Reserve()` | `new TArray` every frame |
| Small fixed buffer, at most 64 elements | `TArray<T, TInlineAllocator<N>>` | Heap allocation |

**Spatial hash using the prime scheme cited from Epic:**

```cpp
// Source attribution retained: RenderingSpatialHash.h, line 42
uint32 Hash = uint32(X) * 1150168907u + uint32(Y) * 1235029793u + uint32(Z) * 1282581571u;
```

<a id="hotpath"></a>
## Hot-Path Optimization

### Reuse Members Instead of Local Allocations

```cpp
// [BAD] Allocation every frame/iteration
void SolveIteration(int32 N) {
    TArray<int32> Counts;
    Counts.SetNumZeroed(N);  // Allocates each iteration
}

// [OK] Reuse a member
class FSystem {
    TArray<int32> Counts;

    void SolveIteration(int32 N) {
        Counts.SetNumUninitialized(N);
        FMemory::Memzero(Counts.GetData(), N * sizeof(int32));
    }
};
```

### Memcpy Instead of Element-by-Element Copying

```cpp
// [BAD] Element-by-element copy for POD types
for (int32 i = 0; i < N; ++i) Dst[i] = Src[i];

// [OK] Bulk copy only for trivially copyable types: int32, float, FVector3f, etc.
FMemory::Memcpy(Dst.GetData(), Src.GetData(), N * sizeof(T));
```

**Do not use Memcpy** for non-POD types with constructors/destructors, such as FString or TSharedPtr. Use `TArray::Append` or element-by-element assignment.

### Query Filter Order: Cheap to Expensive

1. Bitmask operations, approximately one cycle.
2. Boolean flag checks, approximately one cycle.
3. User callbacks/lambdas, variable cost.
4. Distance calculations: subtraction, multiplication, comparison.

### TArray Removal Operations

```cpp
// [BAD] RemoveAt preserves order and shifts subsequent elements: O(N)
Slots.RemoveAt(Index);

// [OK] RemoveAtSwap does not preserve order: O(1); swaps in the last element
Slots.RemoveAtSwap(Index);

// [BAD] Remove searches by value and preserves order: O(N) search + O(N) shift
RegisteredActors.Remove(Actor);

// [OK] RemoveSwap searches by value without preserving order: O(N) search + O(1) swap
RegisteredActors.RemoveSwap(Actor);
```

**Selection rules:**

- If array order does not matter, as in many runtime scenarios, use `RemoveAtSwap` / `RemoveSwap`.
- If order must be preserved, as in UI lists or ordered data, use `RemoveAt` / `Remove`.
- When removing elements while iterating backward, `RemoveAtSwap` is safe: the swapped element comes from a later index and does not affect the unvisited portion.

### Redundant GetRecommendedConfig

If `GetRecommendedConfig()` returns exactly the default-constructed configuration, it creates a maintenance risk:

```cpp
// [BAD] Redundant
static FConfig GetRecommendedConfig() { FConfig C; C.Radius = 60.0f; return C; }

// [OK] Eliminate duplication
static FConfig GetRecommendedConfig() { return FConfig{}; }
```
