# Shipping and Release Readiness

> Sources cited by the original skill: FAB technical requirements, Epic documentation, and practical experience. Check current submission requirements separately before publishing a plugin.

## FAB/Marketplace Submission Readiness

| Check | Requirement in the source checklist |
|-------|-------------------------------------|
| Compilation | Zero errors in Editor, Game, and Shipping configurations |
| External dependencies | Package all dependencies; do not require additional downloads |
| Platform declaration | Explicit SupportedTargetPlatforms in `.uplugin` |
| Version | Semantic version, such as 1.0.0 |
| Demonstration | At least one test Map |
| Paths | Use `FPaths::` APIs; no hard-coded paths |
| Third-party code | License compliance |

## Cross-Platform Compilation

**Avoid:**

```cpp
#include <windows.h>        // [BAD] Platform-specific header
FString P = TEXT("C:\\.."); // [BAD] Hard-coded path
std::ifstream File(...);    // [BAD] STL file I/O
```

**Use:**

```cpp
FPlatformProcess::GetDllHandle(...)  // DLL loading
FPaths::ProjectConfigDir()           // Paths
IFileManager::Get()                  // File operations
#if PLATFORM_WINDOWS / PLATFORM_LINUX / PLATFORM_MAC  // Conditional compilation
```

## Module Loading Phases

> Sources: Epic's `ELoadingPhase::Type`; imzlp's FModuleManager analysis.

| Phase | Timing | Use |
|-------|--------|-----|
| `PostConfigInit` | After configuration initialization | Core Runtime modules, earliest stage |
| `PreDefault` | Before default loading | Initialization before most modules |
| `Default` | Default | Ordinary Runtime modules |
| `PostDefault` | After default loading | Runtime modules depending on other modules |
| `PostEngineInit` | After full engine initialization | **Required for Editor modules by the original rule** |

- [BAD] Editor modules using `Default`: use `PostEngineInit` under this rule.
- Match `.uplugin` LoadingPhase to the module's purpose.

## Module Architecture

| Directory | Contents |
|-----------|----------|
| `Public/` | Interface headers visible to other modules |
| `Private/` | Implementations and internal headers |

```csharp
PublicDependencyModuleNames  // Modules included by public headers
PrivateDependencyModuleNames // Modules used only in .cpp files
```

**Runtime modules must not depend on Editor modules.** Guard editor-only code with `#if WITH_EDITOR`.

## Hot Reload / Live Coding Traps

> Sources: northstarhana.com; SubsystemBrowserPlugin issue #14.

- **Subsystems are recreated during Hot Reload:** Do not cache Subsystem pointers elsewhere.
- **Engine pointers in static/global variables:** They become dangling after Hot Reload.
- **Class reinstancing can orphan instances:** Listen to `FWorldDelegates::OnWorldCleanup`.
- **IDetailCustomization needs refreshing:** Call `NotifyCustomizationModuleChanged()` after Hot Reload.

<a id="softref"></a>
## Soft References Avoid Loading Chains

> Sources: unreal-engine-cpp-pro skill; Tom Looman's Cast section. These are source attributions, not required skill dependencies.

```cpp
// [BAD] Hard reference: loading this Blueprint loads all referenced assets
UPROPERTY(EditDefaultsOnly)
TSubclassOf<AWeapon> WeaponClass;

// [OK] Soft reference: load on demand
UPROPERTY(EditDefaultsOnly)
TSoftClassPtr<AWeapon> WeaponClass;
```

- [BAD] Hard UPROPERTY references to large assets: use `TSoftObjectPtr<T>`.
- [BAD] TSubclassOf referencing complex Blueprint classes: use `TSoftClassPtr<T>`.

<a id="include"></a>
## Include Order

> Source: Ben UI, 2024 Coding Standards.

```cpp
// [OK] Alphabetical ordering to reduce merge conflicts
#include "MyClass.h"
#include "Components/HealthComponent.h"
#include "GameFramework/Character.h"
#include "MyClass.generated.h" // Always last
```

## Particle System Selection

The source recommends Niagara (`UNiagaraComponent`) over Cascade (`UParticleSystemComponent`) for new UE5 Marketplace plugin submissions.

```cpp
// [BAD] Deprecated Cascade particle system in the source notes
UPROPERTY()
TObjectPtr<UParticleSystemComponent> RainEffect;

// [OK] Niagara, recommended for UE5
UPROPERTY()
TObjectPtr<UNiagaraComponent> RainEffect;
```

- The source notes that Cascade alone does not cause Marketplace rejection, but recommends Niagara as best practice.
- To support older projects, offer both Cascade and Niagara implementations.
- Niagara requires the `"Niagara"` module dependency in Build.cs.

## Serialization and Versioning

```cpp
enum class EMyPluginVersion : int32 {
    Initial = 0,
    AddedNewField = 1,
    LatestPlusOne,
    Latest = LatestPlusOne - 1
};

void Serialize(FArchive& Ar) {
    Ar.UsingCustomVersion(MyPluginGUID);
    const int32 Ver = Ar.CustomVer(MyPluginGUID);
    Ar << BaseData;
    if (Ver >= (int32)EMyPluginVersion::AddedNewField) {
        Ar << NewField;
    }
}
```
