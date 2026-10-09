# API Patterns

> Sources cited by the original skill: UE source `CharacterMovementComponent.h`, Tom Looman, and Ben UI.

<a id="uproperty"></a>
## UPROPERTY Metadata

> Source: UE source `CharacterMovementComponent.h`.

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Module|Subclass",
    meta = (
        ClampMin = "0", ClampMax = "100",  // Hard limits (enforced during serialization)
        UIMin = "0", UIMax = "100",        // Slider range (may extend beyond Clamp)
        ForceUnits = "cm",                 // Display units: cm, cm/s, s, degrees
        EditCondition = "bEnabled",        // Conditional display
        EditConditionHides                 // Hide rather than disable when false
    ))
float MyProperty = 50.0f;
```

| Check | Priority |
|-------|----------|
| Numeric property lacks `UIMin/UIMax` | P1: medium |
| Distance/speed/time property lacks `ForceUnits` | P1: medium |
| Bool-controlled child property lacks `EditCondition` | P1: medium |
| Advanced parameter lacks `AdvancedDisplay` | P2: low |

<a id="delegate"></a>
## Delegate Selection

> Sources: Tom Looman; Ben UI, Advanced Delegates.

| Type | Blueprint-visible | Performance | Use |
|------|-------------------|-------------|-----|
| `DECLARE_DELEGATE` | No | Fastest | C++ callback |
| `DECLARE_MULTICAST_DELEGATE` | No | Fast | C++ event with multiple bindings |
| `DECLARE_DYNAMIC_DELEGATE` | Yes | Slow | Blueprint single-cast delegate |
| `DECLARE_DYNAMIC_MULTICAST_DELEGATE` | Yes | Slowest | Blueprint event dispatcher |

**Key rules:**

- [BAD] Making every delegate Dynamic: use Dynamic only when Blueprint exposure is needed.
- [BAD] Binding delegates in constructors: bindings can be serialized into Blueprints; use `PostInitializeComponents()` or `BeginPlay()`.
- [BAD] Marking ordinary delegate callbacks `UFUNCTION()`: non-Dynamic bindings do not need it.

<a id="subsystem"></a>
## Subsystem Selection

> Sources: Epic documentation; uhiyama-lab.com.

| Subsystem | Lifetime | Use |
|-----------|----------|-----|
| `UEngineSubsystem` | Engine startup to shutdown | Global services, rarely needed |
| `UEditorSubsystem` | Editor startup to shutdown | Editor tools |
| `UGameInstanceSubsystem` | Game startup to exit | Persistent data across levels |
| `UWorldSubsystem` | Level load to unload | Level-scoped systems |
| `ULocalPlayerSubsystem` | Player joins to leaves | Split-screen/multiplayer UI state |

- Prefer `UWorldSubsystem` for plugin singleton systems.
- Use `UGameInstanceSubsystem` for state shared across levels.
- Do not use an `AActor` singleton in place of a Subsystem.

<a id="export"></a>
## API Export Strategy

> Source: Ben UI, "Don't Export Everything".

```cpp
// Export levels, from smallest to broadest
// 1. No export by default
UCLASS()
class AMyInternalActor : public AActor { ... };

// 2. MinimalAPI: permit Cast<T> only under the source's description
UCLASS(MinimalAPI)
class AMyActor : public AActor { ... };

// 3. Export specific functions
UCLASS()
class AMyActor : public AActor {
    MYPLUGIN_API void PublicFunction();
    void InternalFunction();
};

// 4. Export everything only when the whole class needs exposure
UCLASS()
class MYPLUGIN_API AMyPublicActor : public AActor { ... };
```

<a id="log"></a>
## Custom Log Categories

> Sources: Ben UI, "Log Until It Gets Annoying"; Epic documentation.

```cpp
// Header
DECLARE_LOG_CATEGORY_EXTERN(LogMyPlugin, Log, All);

// Source file
DEFINE_LOG_CATEGORY(LogMyPlugin);

// Usage
UE_LOG(LogMyPlugin, Display, TEXT("Initialized: %d instances"), N);
UE_LOG(LogMyPlugin, Warning, TEXT("Config out of range: %f"), V);
```

- [BAD] Using `LogTemp` for all logging: declare a custom category.
- [BAD] No logging on important decision paths: add `UE_LOG`.
- [OK] Prefer `UE_LOGFMT` in UE 5.2+.

<a id="cvar"></a>
## Console Variables

> Source: Epic's C++ Console Variables documentation.

```cpp
static TAutoConsoleVariable<int32> CVarMyFeature(
    TEXT("my.plugin.FeatureEnabled"), 1,
    TEXT("Enable my plugin feature (0=off, 1=on)"),
    ECVF_Default);

int32 Value = CVarMyFeature.GetValueOnGameThread();
```

- Use CVars instead of UPROPERTY for debugging parameters.
- Name CVars as `pluginname.system.parameter`.

<a id="blueprintpure"></a>
## The BlueprintPure Trap

> Source: MrRobinOfficial.

UE automatically marks const UFUNCTIONs as Pure. The original reference warns that pure functions are **re-evaluated for each connected wire** in a Blueprint:

```cpp
// [BAD] Expensive const function becomes Pure automatically
UFUNCTION(BlueprintCallable)
TArray<AActor*> FindNearbyEnemies() const;

// [OK] Disable purity explicitly
UFUNCTION(BlueprintCallable, BlueprintPure = false)
TArray<AActor*> FindNearbyEnemies() const;
```
