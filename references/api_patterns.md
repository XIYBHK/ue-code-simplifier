# API Patterns

Primary sources: [Epic Metadata Specifiers (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/metadata-specifiers-in-unreal-engine?application_version=5.3), [Class Specifiers (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/class-specifiers?application_version=5.3), and [Subsystems (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/programming-subsystems-in-unreal-engine?application_version=5.3). The original skill credits Tom Looman, Ben UI, MrRobinOfficial, and uhiyama-lab.com. See [VALIDATION.md](../VALIDATION.md) for source/version evidence and limits.

Examples are fragments. Reflection declarations require a real reflected class, generated header, includes, module dependencies, and compatible definitions.

<a id="uproperty"></a>
## UPROPERTY Metadata

```cpp
// Fields in a reflected class. Range is illustrative, not a product requirement.
UPROPERTY(EditAnywhere, Category = "MyPlugin|Settings")
bool bEnabled = true;

UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "MyPlugin|Settings",
    meta = (ClampMin = "0", ClampMax = "100",
            UIMin = "0", UIMax = "100", ForceUnits = "cm",
            EditCondition = "bEnabled", EditConditionHides))
float Radius = 50.0f;
```

`ClampMin/ClampMax` constrain applicable editor input; they are not runtime or generic serialization validation. `UIMin/UIMax` describe a slider range. Check sentinel values such as zero or -1 before adding clamps. Runtime assignments, ini data, deserialization, and network input need validation appropriate to their contracts.

`EditCondition` alone normally enables/disables editing; `EditConditionHides` requests hiding. `ForceUnits` controls display units and does not prove the algorithm uses or converts those units correctly. `AdvancedDisplay`, categories, and tooltips can improve discoverability, but missing optional display metadata is not automatically an engine correctness defect.

English `DisplayName` or `ToolTip` text is not proof of completed localization. Check gathering configuration and translation resources when localization is required.

<a id="delegate"></a>
## Delegate Selection and Lifetime

| Type | Typical need | Tradeoff |
|------|--------------|----------|
| Native single-cast | One C++ callback | No dynamic reflection/serialization requirement |
| Native multicast | Multiple C++ listeners | Cost depends on listener count and work |
| Dynamic single-cast | Reflected/name-based binding or serialization, including Blueprint use | Dynamic lookup/reflection overhead |
| Dynamic multicast | Reflected event dispatcher, including Blueprint bindings | Same needs plus multiple listeners |

Dynamic delegates are not only for Blueprint: reflection and serialization can require them. See [Epic Dynamic Delegates](https://dev.epicgames.com/documentation/en-us/unreal-engine/dynamic-delegates-in-unreal-engine); target-version mechanics are also visible in `DelegateSignatureImpl.inl`.

```cpp
// Method-body fragments for a UObject listener and a native multicast delegate.
// The delegate signature and MyCallback must match.
FDelegateHandle Handle = MyDelegate.AddUObject(this, &UMyClass::MyCallback);
// Store Handle if explicit removal is required by the listener's lifetime.

// During the appropriate cleanup phase:
MyDelegate.Remove(Handle);
```

`AddLambda` takes a callable lambda/functor, not `(this, &Class::Method)`. A plain lambda capturing `this` does not itself provide weak UObject lifetime tracking; use an appropriate weak binding or carefully manage removal. Dynamic `BindDynamic/AddDynamic` targets require a matching reflected `UFUNCTION`.

A native binding does not require `UFUNCTION`, but that function may still serve Blueprint, RPC, or another reflected interface. Do not remove the annotation without checking those uses. Choose binding/cleanup phases from the actual event and owner lifetime; account for CDO/default subobjects, serialization, duplicate bindings, and early events. Moving every constructor binding to BeginPlay is not universally correct.

<a id="subsystem"></a>
## Subsystem Selection

| Owner | Typical subsystem | Lifetime boundary |
|-------|-------------------|-------------------|
| Engine | `UEngineSubsystem` | Engine-owned service lifecycle; module behavior matters |
| Editor | `UEditorSubsystem` | Editor-owned service lifecycle |
| Game instance | `UGameInstanceSubsystem` | That game instance; can retain state across world travel |
| World | `UWorldSubsystem` | That UWorld, not each streamed Level |
| Local player | `ULocalPlayerSubsystem` | That local player |

Select the owner matching the service's needs. Check supported world types, dedicated-server/client behavior, initialization dependencies, and multiple PIE instances. These are not necessarily process-wide singletons. An Actor is still appropriate for a replicated or scene-bound entity; do not automatically replace it with a subsystem.

<a id="export"></a>
## API Export Strategy

Export what consumers need. No export, `MinimalAPI`, selected function exports, and a class-wide module API macro have different symbol/ABI effects. `MinimalAPI` exports type information; it does not mean "Cast only", and visible inline functions can be callable without an out-of-line export.

```cpp
// Header fragment for a selectively exported Actor.
// Include CoreMinimal.h and GameFramework/Actor.h, then MyActor.generated.h last.
// MYPLUGIN_API must be the real module's generated API macro.
UCLASS(MinimalAPI)
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    MYPLUGIN_API void PublicFunction();

private:
    void InternalFunction();
};
```

Export visibility does not change C++ access control: a class member is private by default unless declared otherwise. Review external construction, inheritance, virtual methods, templates, and cross-DLL linkage before reducing exports. Do not break existing public interfaces solely to minimize exported symbols.

<a id="log"></a>
## Custom Log Categories

```cpp
// Header in this module; add the module API export if another DLL needs the symbol.
DECLARE_LOG_CATEGORY_EXTERN(LogMyPlugin, Log, All);

// Source file in the same module:
DEFINE_LOG_CATEGORY(LogMyPlugin);

// Function-body usage, with matching argument types:
UE_LOG(LogMyPlugin, Display, TEXT("Initialized: %d instances"), Count);
```

Custom categories improve filtering and ownership. A shared category requires deliberate cross-module symbol export; otherwise use each module's own category. Configure and test Shipping logging if it is a product requirement. `UE_LOGFMT` is an option in supporting versions, not an automatic replacement for correctly typed `UE_LOG` calls.

<a id="cvar"></a>
## Console Variables

```cpp
// Source-file scope; include HAL/IConsoleManager.h.
static TAutoConsoleVariable<int32> CVarMyFeature(
    TEXT("my.plugin.FeatureEnabled"), 1,
    TEXT("Enable my plugin feature (0=off, 1=on)"), ECVF_Default);
```

```cpp
// GameThread function-body fragment:
const int32 Value = CVarMyFeature.GetValueOnGameThread();
```

CVars suit runtime debugging/tuning when that access model fits. They do not universally replace reflected designer settings or persisted configuration. Choose the getter and synchronization model for the calling thread, and handle any callback lifetime/reload behavior.

<a id="blueprintpure"></a>
## BlueprintPure and Expensive Computation

In the inspected UHT sources, a const BlueprintCallable function with an output is automatically made pure unless explicitly forced impure. Not every const UFUNCTION meets these conditions.

```cpp
// Reflected class member declaration; only change existing Blueprint pin semantics
// when authorized and with a migration plan for affected graphs.
UFUNCTION(BlueprintCallable, BlueprintPure = false, Category = "MyPlugin|Queries")
TArray<AActor*> FindNearbyEnemies() const;
```

Pure evaluation is not a fixed "one call per connected wire" formula. The compiler can deduplicate a pure node within the data gathering needed for one consuming node; distinct execution consumers or iterations can still repeat the work. Inspect the actual graph and target engine, then choose explicit impure execution or suitable caching if needed. Preserve existing Blueprint interfaces unless the user requests a change.
