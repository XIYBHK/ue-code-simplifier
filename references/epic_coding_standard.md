# Epic Games Coding Standard

## Naming Conventions

### Class Prefixes

| Prefix | Use | Example |
|--------|-----|---------|
| A | Derived from AActor | `AActorPool`, `AMyCharacter` |
| U | Derived from UObject | `UMyLibrary`, `UMyComponent` |
| F | Structs and ordinary classes | `FVector`, `FMyConfig` |
| E | Enums | `EInstanceState`, `EMyEnum` |
| I | Interfaces | `IPoolInterface`, `IMyInterface` |
| T | Templates | `TArray`, `TSharedPtr<FMyType>` |

### Variable Names

```cpp
// Members: camel case with an initial capital
int32 MyVariable;
float CurrentHealth;

// Booleans: b prefix
bool bIsActive;
bool bHasInitialized;

// Function parameters: camel case with an initial capital
void ProcessData(const FString& Data, int32 Count);

// Globals: G prefix
extern int32 GGlobalCounter;
```

### Function Names

```cpp
// Member functions: start with a verb
void CalculateValue();
int32 GetItemCount() const;
bool IsValid() const;
```

## Header Management

### IWYU (Include What You Use)

```cpp
// Include only required headers
#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyModuleTypes.h"
#include "MyInterface.h"
#include "MyObject.generated.h"      // Must be the last include

// Prefer forward declarations in headers
class AMyActor;
struct FMyData;

// Avoid overly broad headers
// #include "Engine/Engine.h"        // Too broad
```

## Type Selection

### Containers

| UE type | STL equivalent | Description |
|---------|----------------|-------------|
| TArray | std::vector | Dynamic array |
| TMap | std::unordered_map | Key-value map |
| TSet | std::unordered_set | Unique-value set |

Always use UE containers instead of STL containers under this skill's conventions.

### Strings

| Type | Use | Example |
|------|-----|---------|
| FString | String manipulation and modification | `FString Path = FPaths::Combine(...);` |
| FName | Keys, indexing, fast comparisons | `FName Tag = FName("MyTag");` |
| FText | UI display and localization | `FText::FromString("Hello");` |

```cpp
// FString: when modification is needed
FString Result = SourceStr.Replace(*SearchStr, *ReplaceStr);

// FName: keys or fast comparisons
FName MyTag = FName("Player");
if (Actor->Tags.Contains(MyTag)) { }

// FText: UI display with localization support
FText DisplayText = LOCTEXT("MyKey", "Hello World");
```

## Blueprint Exposure

### UFUNCTION Metadata

```cpp
UFUNCTION(BlueprintCallable, Category = "MyPlugin|MyModule|Subcategory",
    meta = (
        DisplayName = "My Function",
        Keywords = "keyword,search",
        ToolTip = "Detailed description.",
        AdvancedDisplay = "Param2,Param3"
    ))
static void MyFunction(
    UPARAM(DisplayName="Parameter") const FString& Param,
    int32& OutResult);
```

For pointer safety, memory management, performance, and thread safety, use the corresponding references:

- [safety_patterns.md](safety_patterns.md): GC, assertions, defensive programming, Tick.
- [performance_patterns.md](performance_patterns.md): Stats, ParallelFor, containers, hot paths.
- [api_patterns.md](api_patterns.md): UPROPERTY metadata, delegates, subsystems, exports, CVars.
