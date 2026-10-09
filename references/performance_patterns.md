# Performance Patterns

The original skill draws on Epic source and production observations. The corrected notes distinguish legal APIs from tuning choices. Installed source evidence and [validation limits](../VALIDATION.md) do not establish a performance improvement in a consuming plugin.

Measure the target workload, preserve behavior, and compare results. Neither a container name nor a profiling macro proves that code is fast or slow.

<a id="stats"></a>
## Stats and Insights

Useful starting points include named scopes around measured hot functions and a module-specific group. Group declarations must be visible to all headers/source files that use them.

```cpp
// Shared module profiling header; include Stats/Stats.h.
DECLARE_STATS_GROUP(TEXT("MyPlugin"), STATGROUP_MyPlugin, STATCAT_Advanced);
DECLARE_STATS_GROUP_VERBOSE(TEXT("MyPluginDetail"), STATGROUP_MyPluginDetail, STATCAT_Advanced);
```

```cpp
// Function-body fragment; the group declaration must already be visible.
DECLARE_SCOPE_CYCLE_COUNTER(TEXT("MyPlugin Update"), STAT_MyPlugin_Update, STATGROUP_MyPlugin);
// Perform the measured work in this scope.
```

`TRACE_CPUPROFILER_EVENT_SCOPE` provides CPU trace scopes, while Stats macros support statistics/groups. The systems can overlap: configurations in the inspected `Stats.h` bridge cycle scopes to CPU trace events. Enable the relevant Stats group (for example `stat MyPlugin`) and trace channels rather than assuming custom counters appear in `stat game`. Instrumentation may compile out or add overhead depending on configuration; do not require a specific macro in every function.

<a id="parallelfor"></a>
## ParallelFor

The simple `ParallelFor(Num, Body, Flags)` overload is legal; named/batched overloads help diagnostics and workload tuning. Missing an explicit name/batch size is not a compilation or correctness error. Select batch size and flags using task size, dependencies, latency, and measurements. `BackgroundPriority` is appropriate only when its scheduling behavior fits.

```cpp
// Function-body fragment. Inputs and Outputs are distinct TArray<float> objects.
// Inputs remains read-only and alive; no other task accesses/mutates Outputs.
const int32 Num = Inputs.Num();
Outputs.SetNumUninitialized(Num);
ParallelFor(TEXT("MyPlugin_SquareValues"), Num, 64,
    [&](int32 Index)
    {
        Outputs[Index] = Inputs[Index] * Inputs[Index];
    }, EParallelForFlags::None);
```

Preallocate output before dispatch and write only independent elements. Do not call `Add`, `Reset`, or `SetNum` concurrently on a shared TArray. Reusable scratch storage must be per task/thread or exclusively partitioned, with no unsafe cross-frame overlap or reentrancy. The inspected header offers `ParallelForWithTaskContext`; use its actual version-specific signature if it fits. Promoting a lambda's scratch array to one shared system member is not a safe optimization by itself.

Follow [UObject thread/lifetime rules](safety_patterns.md#gc); a strong/pinned pointer does not make arbitrary object methods safe to call in ParallelFor. Prefer read-only value snapshots where suitable.

### TInlineAllocator

`TArray<T, TInlineAllocator<N>>` and default `TArray<T>` are different template types. They cannot be passed interchangeably as mutable references. Options include an allocator-templated function, a suitable non-owning view for a non-resizing operation, or default TArray with reserved capacity. Inline capacity is a workload choice, not a universal 64-element cutoff; exceeding it can allocate.

<a id="container"></a>
## Container Selection and Spatial Hashing

| Workload | Candidate | What to verify |
|----------|-----------|----------------|
| Sparse spatial cells | `TMap<Key, Cell>` / `TSet<Cell>` | Occupancy, memory, key/equality/hash distribution |
| Dense bounded grid | Flat `TArray` | Bounds, dimensions/overflow, indexing and updates |
| Small temporary buffer | Inline allocator or reused array | Size distribution, stack cost, function type compatibility |
| Hot-path neighbor output | Reserved/reused array | Ownership, concurrency, lifetime and actual allocation count |

Epic does use TMap in spatial structures: installed `Spatial/SparseGrid3.h`, `SparseGrid2.h`, and Chaos `SpatialHash.h` are examples. Hashing is a key strategy, not a replacement for TMap/TSet. Select the representation from measured requirements.

```cpp
// Illustrative flat, single-level cell key. This is not a full spatial index.
struct FCellKey
{
    FIntVector Coord = FIntVector(0, 0, 0);

    bool operator==(const FCellKey& Other) const { return Coord == Other.Coord; }

    friend uint32 GetTypeHash(const FCellKey& Key)
    {
        return uint32(Key.Coord.X) * 1150168907u
             + uint32(Key.Coord.Y) * 1235029793u
             + uint32(Key.Coord.Z) * 1282581571u;
    }
};
TMap<FCellKey, TArray<int32>> GridCells;
```

This prime scheme can collide and is not proven faster for a particular workload. The inspected UE5.6 `RenderingSpatialHash.h` hashes an additional Level for a multilevel key; UE5.3's implementation differs. Include every field needed by the key's equality/identity and evaluate the actual distribution. Do not change containers just because a generic rule calls TMap a hot-path defect.

<a id="hotpath"></a>
## Hot-Path Memory and Algorithms

### Reuse Buffers with Clear Ownership

```cpp
// Exclusive function-body use of a persistent TArray<int32> Counts.
// N must be nonnegative and within allocation limits.
Counts.Reset();       // Retains suitable capacity; does not promise zero allocation.
Counts.AddZeroed(N);  // int32 has the required zero representation.
```

Reserve for measured capacity needs; reuse at the correct semantic boundary. A loop using independent temporary results is not equivalent to accumulating all results into one array. Check capacity growth/shrink policy and concurrent/reentrant use before claiming allocations were eliminated.

`SetNumZeroed` initializes newly added elements, not every existing element when the size stays the same or shrinks. A generic `SetNumUninitialized + Memzero` is not a safe reset for arbitrary T: object lifetime, zero representation, constructors/destructors, and container invariants matter. Use type-appropriate construction/assignment for nontrivial elements.

### Bulk Copy

```cpp
// Example for existing int32 values, not arbitrary element types.
void CopyValues(const TArray<int32>& Source, TArray<int32>& Destination)
{
    if (&Source == &Destination)
    {
        return;
    }
    Destination.SetNumUninitialized(Source.Num());
    if (Source.Num() > 0)
    {
        FMemory::Memcpy(Destination.GetData(), Source.GetData(),
            SIZE_T(Source.Num()) * sizeof(int32));
    }
}
```

The byte count must fit the platform size type, the objects/storage must be valid, and ranges must not overlap for Memcpy. For generic types, establish trivially-copyable and lifetime requirements; merely having a user-defined default constructor does not rule out trivial copying. FString/TSharedPtr need proper object operations. Use an appropriate overlap-safe operation if required. A compiler may optimize a simple loop already; benchmark rather than promise a speedup.

### Query Filters

Order predicates only when doing so preserves results and the observable callback order/count contract. Side-effecting or stateful callbacks cannot be moved casually. For commutable pure predicates, measure both cost and rejection rate. A distance comparison may be cheaper than a user callback; bit tests and flag checks have no universal one-cycle cost in a full workload.

### Array Removal

```cpp
// Function-body fragments: Index is valid, order may change, no concurrent mutation.
Slots.RemoveAtSwap(Index);

// Removes matching values; there may be more than one match.
RegisteredActors.RemoveSwap(Actor);
```

Swap removal does not preserve order and can invalidate indices/references. Single-index removal without shrinking can perform constant element movement; default shrinking may allocate/move more data. Value-based RemoveSwap searches and removes matching values, so cost also depends on duplicates and element operations. Check the version's shrinking overload before tuning it.

Backward, single-index removal can be valid when the swapped tail element was already handled and the algorithm permits that order change. This is not a blanket guarantee for arbitrary callbacks, counts, or concurrent iteration. Use ordered removal when order is part of the contract.

### Default-Equivalent Factories

```cpp
// Complete ordinary type definition showing the equivalence being claimed.
struct FConfig
{
    float Radius = 60.0f;
};

// Keep the public function if callers depend on it; simplify only its implementation.
static FConfig GetRecommendedConfig() { return FConfig{}; }
```

Replace an implementation with default construction only after comparing all fields, constructors, and side effects. Do not remove an established public factory solely because its implementation is redundant; preserve source/ABI contracts or obtain a deliberate migration request.
