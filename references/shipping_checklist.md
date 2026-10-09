# Shipping and Release Readiness

Primary sources: [Epic Modules (5.3)](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules?application_version=5.3), [IWYU](https://dev.epicgames.com/documentation/unreal-engine/include-what-you-use-iwyu-for-unreal-engine-programming), [Fab file/structure requirements](https://dev.epicgames.com/documentation/en-us/fab/asset-file-format-and-structure-requirements-in-fab), and [Fab technical requirements](https://www.fab.com/o/technical-requirements). The original skill also credits Ben UI, imzlp, northstarhana.com, SubsystemBrowserPlugin, Tom Looman, and unreal-engine-cpp-pro. Those are attributions, not installed dependencies.

## FAB/Marketplace Engineering Preflight

This is a **non-exhaustive engineering checklist**, not a certificate of current Fab acceptance. The publicly readable format/structure page has product-type-specific rules. On 2026-10-09, the technical-requirements page's complete body was not available to the audit's text reader. Verify the full current policy and the applicable code-plugin category before submission; do not infer that the seven checks below are all requirements or universally mandatory in exactly this form.

| Area | Review action |
|------|---------------|
| Builds | Build the supported targets/configurations/platforms; Runtime and Editor-only modules have different applicable checks |
| Dependencies | Identify packaged code/libraries, required SDKs/services/setup, distribution permissions, and current policy constraints |
| Platforms | Align descriptor/module allowlists and claimed support with actual tested platforms; a declaration alone proves nothing |
| Version | Use valid descriptor fields and clear user-facing version/release notes; do not assert universal semantic-version policy without current evidence |
| Demonstration | Supply the documentation/examples/maps required for this product type and enough material to demonstrate the advertised functionality |
| Paths/files | Use portable engine facilities where appropriate; validate paths, packaging, external resource availability, and cleanup |
| Licensing | Preserve attribution and comply with applicable third-party/engine distribution terms |

For code plugins, also check the current required module/source/config/content layout and any additional official requirements. Do not require a test Map for every possible code-plugin category based solely on this reference. No Fab submission or approval was performed for this skill.

## Cross-Platform Compilation

Use appropriate abstractions such as `FPaths`, `IFileManager`, and `FPlatformProcess`, but verify their contracts and resource lifetimes. Standard file I/O or platform headers are not inherently forbidden; isolate platform-specific code and follow Unreal's platform-header/type-wrapper guidance for the exact version.

```cpp
// Preprocessor structure example. Platform-specific implementation goes in each branch.
#if PLATFORM_WINDOWS
// Windows implementation with appropriate Unreal platform-header wrappers.
#elif PLATFORM_LINUX || PLATFORM_MAC
// Supported Unix-like implementation.
#else
// Explicit unsupported-platform behavior, if applicable.
#endif
```

Avoid machine-specific paths. Check DLL load failures, appropriate `FreeDllHandle` cleanup, plugin/packaged paths, and non-supported platform behavior. A successful Windows build is not cross-platform validation.

## Module Loading Phases

Choose the phase from real dependencies and event timing. `PostEngineInit` is **not required for all Editor modules**: the inspected UE5.6 `AudioInsightsEditor` is Type Editor with LoadingPhase Default. Other engine Editor modules also use earlier phases.

| Phase | Useful boundary |
|-------|-----------------|
| `EarliestPossible` | Very early facilities needed as plugin files become discoverable |
| `PostConfigInit` | Immediately after configuration initialization, for relevant low-level hooks |
| `PreDefault` | Work that must precede default module loading |
| `Default` | Ordinary default loading; legal for suitable Runtime and Editor modules |
| `PostDefault` | Work after default-phase loading |
| `PostEngineInit` | Work that specifically needs completed engine initialization |

The table is not the full enum. Later loading can miss registration events, including relevant node/menu setup. Delay only the work requiring a later state, or use the appropriate callback; do not mechanically delay an entire Editor module. See `Source/Runtime/Projects/Public/ModuleDescriptor.h` and [validation evidence](../VALIDATION.md).

## Module Architecture

Public headers expose an interface to downstream modules. Private headers and source files implement internal details. [Epic's module dependency guidance](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules?application_version=5.3) selects public/private dependencies according to exposure.

```csharp
// Build.cs fragments, with actual module names chosen from the code's needs.
PublicDependencyModuleNames.Add("Core"); // If the public interface requires it.
PrivateDependencyModuleNames.Add("Projects"); // If used only by private implementation.
```

A compatibility layer is not a reason to make all its dependencies public. An `if (Target.bBuildEditor)` block is not a reason to make all contained dependencies private: a public Editor interface can require public dependencies. Avoid unconditionally exposing Editor dependencies from shipping Runtime modules; put editor interfaces in an appropriate Editor/UncookedOnly module where possible. `#if WITH_EDITOR` guards alone do not configure dependency visibility, module type, or target eligibility.

## Hot Reload, Live Coding, and Reinstancing

These are distinct mechanisms. Do not assume every reload recreates every subsystem, or that every static pointer always becomes dangling. Structural class changes, object replacement, module unload, World cleanup, and shutdown have different lifetime effects.

- Match caches and delegates to the actual owner. Use safe references/re-resolution and invalidate them on the relevant event.
- `FWorldDelegates::OnWorldCleanup` concerns World cleanup; it does not cover every object reinstancing case. The inspected engine also exposes `FCoreUObjectDelegates::OnObjectsReinstanced` with a replacement map.
- Clear module-owned static state and unregister bindings when their owners unload or shut down.
- `NotifyCustomizationModuleChanged()` is available for refreshing relevant PropertyEditor customization state; arrange proper registration/unregistration.

These API distinctions were inspected in source; no Hot Reload/Live Coding experiment was run. Reproduce the intended reload scenario before claiming a cache is safe or broken.

<a id="softref"></a>
## Soft References and Asset Availability

```cpp
// Reflected member: an Actor class path, not an Actor instance.
UPROPERTY(EditDefaultsOnly, Category = "MyPlugin|Assets")
TSoftClassPtr<AWeapon> WeaponClass;
```

Use `TSoftClassPtr` for classes and `TSoftObjectPtr` for object assets/instances of the appropriate type. Soft references can avoid a direct hard loading dependency but do not provide strong GC ownership of a loaded object. Establish when/how to load, failure handling, retention/streamable-handle lifetime, and unload behavior. Confirm the cooking/asset-discovery strategy in packaged builds.

A hard reference may be correct when immediate availability is part of the contract. Inspect the real reference graph and budget before changing it: other hard references and load operations can preserve the chain. Do not universally replace complex Blueprint or large-asset references.

<a id="include"></a>
## Include Roles and Order

```cpp
// MyActor.h: include declarations actually used by this reflected header.
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h" // Last include in the reflected header.
```

```cpp
// MyActor.cpp: its matching header first. No generated include here.
#include "MyActor.h"
#include "Components/HealthComponent.h"
```

Order other includes consistently if their semantics permit it. Do not confuse the aggregate `Engine.h` with the specific `Engine/Engine.h` that declares UEngine/GEngine. Keep code valid in non-Unity builds rather than relying on another translation unit's includes.

## Particle System Integration

Niagara is generally a suitable choice for new UE5 VFX integrations; include its actual component header and declare the Niagara module/plugin dependency as required. Cascade integration is not automatically an invalid API or a proven Fab rejection. Preserve legacy product compatibility where needed and verify the target engine and current product policy before replacing a working implementation.

## Serialization and Versioning

Register a stable format GUID/current version, request its use in the archive, and handle absent/old versions explicitly. `CustomVer` returns -1 when a tag is absent. `UsingCustomVersion` requires prior registration and does not add a version during loading.

```cpp
// Source-file illustration. Include Serialization/Archive.h,
// Serialization/CustomVersion.h, and Misc/Guid.h.
// Generate a distinct GUID once for the real format; never change it for each release.
namespace MyPluginVersion
{
    const FGuid GUID(0x71A07B36, 0xB3464B19, 0xAE224013, 0x98712DAD);
    enum Type : int32 { Initial = 0, AddedNewField = 1, Latest = AddedNewField };
    static FCustomVersionRegistration Registration(GUID, Latest, TEXT("MyPluginData"));
}

// Ordinary struct, serialized within a format/archive that persists custom versions.
// This is not a complete standalone raw-memory file format.
struct FMyData
{
    int32 BaseData = 0;
    int32 NewField = 0;

    void Serialize(FArchive& Ar)
    {
        Ar.UsingCustomVersion(MyPluginVersion::GUID);
        const int32 Version = Ar.CustomVer(MyPluginVersion::GUID);
        if (Version > MyPluginVersion::Latest)
        {
            Ar.SetError(); // This example does not understand future layouts.
            return;
        }
        Ar << BaseData;
        if (Version >= MyPluginVersion::AddedNewField)
        {
            Ar << NewField;
        }
        else if (Ar.IsLoading())
        {
            NewField = 0; // Defined migration/default for this example's old layout.
        }
    }
};
```

If overriding UObject `Serialize`, use the actual override signature and call the appropriate `Super::Serialize`; that instruction does not apply to this ordinary struct. Ensure the surrounding file/archive persists and restores its custom-version container; raw byte archives do not automatically form a versioned file. Test current round trips, old/absent versions, future-version rejection, initialization, error paths, and data migration. This repository has not run those UE serialization tests.
