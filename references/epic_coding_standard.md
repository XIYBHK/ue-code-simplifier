# Epic C++ Conventions and Plugin Review

Primary sources: [Epic C++ Coding Standard (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine?application_version=5.3), [IWYU](https://dev.epicgames.com/documentation/unreal-engine/include-what-you-use-iwyu-for-unreal-engine-programming), and [Modules (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules?application_version=5.3). Apply the target version and the existing project conventions; a style preference is not automatically a correctness defect.

## Naming

| Prefix | Conventional use |
|--------|------------------|
| A | Actor-derived classes |
| U | UObject-derived classes other than Actor-derived classes |
| F | Ordinary structs/classes under UE conventions |
| E | Enums |
| I | Native interface side |
| T | Templates under the relevant naming conventions |

Use PascalCase for ordinary variable/function names, `b` for booleans, and `G` for engine-style global names. Prefer names communicating the operation or result. A reflected interface commonly has both a U-prefixed reflected interface class and an I-prefixed native interface. Avoid renaming established public/Blueprint symbols solely to match a style preference.

```cpp
int32 ItemCount = 0;
bool bIsActive = false;
void ProcessData(const FString& Data);
```

## Headers and Module Dependencies

```cpp
// MyObject.h: headers needed by this reflected declaration.
#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyObject.generated.h" // Last include in the reflected header.
```

```cpp
// MyObject.cpp: verify this header is self-sufficient.
#include "MyObject.h"
```

Use forward declarations where legal; include definitions for operations requiring complete types. The specific `Engine/Engine.h` declares UEngine/GEngine and may be necessary; the aggregate `Engine.h` is a different header. In .cpp files, include the matching header first. Sort other includes consistently where dependencies permit it. Validate non-Unity builds instead of relying on incidental includes.

Public dependencies are needed when the public interface requires their exposure; private dependencies serve internal source/private headers. Do not mechanically change every Editor-conditional dependency to private, or every compatibility dependency to public.

## Types and Strings

UE containers are often appropriate for reflection, engine APIs, and serialization. Standard-library facilities are not categorically invalid: for example, the inspected Atomic.h recommends std::atomic for new code. Select a type consistent with interface, lifetime, allocation, target/toolchain, and project constraints.

| Type | Typical purpose |
|------|-----------------|
| `FString` | Mutable string data/manipulation |
| `FName` | Names/keys with engine name semantics |
| `FText` | Display text and the localization system |

```cpp
// Source-file scope in a translation unit with localized source text.
#define LOCTEXT_NAMESPACE "MyPlugin"

// Function-body fragment; suitable for a stable, gathered source-text identity.
FText Label = LOCTEXT("StatusReady", "Ready");

// End of the translation unit, after the relevant declarations/definitions:
#undef LOCTEXT_NAMESPACE
```

FText supports localization but converting an arbitrary runtime FString with `FText::FromString` does not automatically provide the same stable namespace/key as gathered LOCTEXT. Inspect gathering configuration, text identity, translations, and packaged resources. Keep locale-independent identifiers separate from display text.

## Blueprint Metadata

```cpp
// Reflected class member declaration; all named AdvancedDisplay parameters exist.
UFUNCTION(BlueprintCallable, Category = "MyPlugin|Utilities",
    meta = (DisplayName = "Process Data", AdvancedDisplay = "Extra"))
static void ProcessData(const FString& Data, int32 Extra = 0);
```

Category hierarchy and display naming should be consistent and useful. Mandatory UHT metadata depends on module/context; optional metadata is not a substitute for correct behavior or completed localization. Preserve Blueprint signatures unless a requested migration permits changing them.

Use the focused references for [ownership/safety](safety_patterns.md), [performance](performance_patterns.md), [API quality](api_patterns.md), and [release checks](shipping_checklist.md).
