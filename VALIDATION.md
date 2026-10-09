# Validation Scope and Evidence

Reviewed on **2026-10-09**. This repository is an instruction/reference skill for existing UE C++ plugins, not a buildable Unreal plugin. Its examples are contextual fragments or explicitly labeled diagnostic alternatives; unspecified project types/functions are placeholders to be supplied by the consuming project.

## Installed Source Versions

| Version | Build.version changelist |
|---------|-------------------------|
| 5.3.2 | 29314046 |
| 5.4.4 | 35576357 |
| 5.5.4 | 40574608 |
| 5.6.1 | 44394996 |
| 5.7.4 | 51494982 |

Paths below are relative to an installed engine's `Engine/` directory. Line numbers identify the inspected patch and can drift in other releases/forks. The audit kept local path/hash/excerpt evidence without uploading engine files, binaries, assets, private absolute paths, or credentials.

## API and Behavior Evidence

| Topic | Representative installed source location | Conclusion / boundary |
|-------|------------------------------------------|-----------------------|
| Shipping defaults | UE5.6 `Source/Runtime/Core/Public/Misc/Build.h:206`; UE5.6 `Source/Runtime/Core/Public/Misc/Build.h:211`; UE5.6 `Source/Runtime/Core/Public/Misc/AssertionMacros.h:464` | All five versions retain the relevant configuration/macros; use the disabled branch as well as the enabled declaration. |
| Compression 32-bit | UE5.3 `Source/Runtime/Core/Public/Misc/Compression.h:69`; UE5.4 `Source/Runtime/Core/Public/Misc/Compression.h:69` | FName first; flags and integer compression data, no new nullptr parameter at 5.4. |
| Compression 64-bit | UE5.5 `Source/Runtime/Core/Public/Misc/Compression.h:86` | Same new signature at 5.6/5.7; old 32-bit declaration marked deprecated in 5.5. |
| FProperty | UE5.5 `Source/Runtime/CoreUObject/Public/UObject/UnrealType.h:194` | 5.3/5.4 direct field; 5.5-5.7 getter and 5.5 deprecation marker. |
| Actor Tick | UE5.6 `Source/Runtime/Engine/Private/Actor.cpp:273`; UE5.6 `Source/Runtime/Engine/Private/Actor.cpp:1691` | Base defaults and enable condition inspected across all five versions; derived/Blueprint behavior not run. |
| Delegate binding | UE5.6 `Source/Runtime/Core/Public/Delegates/DelegateSignatureImpl.inl:806`; UE5.6 `Source/Runtime/Core/Public/Delegates/DelegateSignatureImpl.inl:973` | All five headers distinguish functors from member binding; dynamic reflection requirement inspected. |
| UObject validity | UE5.6 `Source/Runtime/CoreUObject/Public/UObject/Object.h:1856`; UE5.3 `Source/Runtime/CoreUObject/Public/UObject/UObjectBaseUtility.h:210` | Flags/null check is not lifetime retention; existing 5.0 deprecation marker disproves 5.4 boundary. |
| GC container contents | UE5.6 `Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp:6643`; UE5.6 `Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp:6662` | Modern GC schema traversal is distinct from legacy property-descriptor reference collection. |
| World/controller | UE5.6 `Source/Runtime/Engine/Classes/Engine/World.h:2625`; UE5.6 `Source/Runtime/Engine/Classes/Kismet/GameplayStatics.h:192` | Real accessor declarations inspected; no UWorld indexed GetPlayerController declaration in five inspected headers. |
| Soft class/object type | UE5.3 `Source/Runtime/CoreUObject/Public/UObject/SoftObjectPtr.h:146,297,722`; UE5.6 same path `:239` | Older wide object construction does not turn UClass into Actor; incompatible conversion deprecated in newer inspected headers. |
| Metadata clamp | UE5.6 `Source/Editor/PropertyEditor/Private/PropertyHandleImpl.cpp:992` | PropertyEditor input handling, not general runtime serialization validation. |
| Pure evaluation | UE5.6 `Source/Programs/Shared/EpicGames.UHT/Types/UhtFunction.cs:591`; UE5.6 `Source/Editor/KismetCompiler/Private/KismetCompiler.cpp:2848` | UHT auto-pure conditions and compiler set-based gathering inspected; no Blueprint runtime measurement. |
| Spatial TMap | UE5.6 `Source/Runtime/GeometryCore/Public/Spatial/SparseGrid3.h:30`; UE5.6 `Source/Runtime/Experimental/Chaos/Public/Chaos/SpatialHash.h:71` | Counterexamples to a universal TMap ban; no performance benchmark. |
| Spatial hash versions | UE5.3 `Source/Runtime/Engine/Public/Rendering/RenderingSpatialHash.h:33`; UE5.6 `Source/Runtime/Engine/Public/Rendering/RenderingSpatialHash.h:42` | Implementations differ; multilevel identity includes Level in 5.6. |
| Stats bridge | UE5.6 `Source/Runtime/Core/Public/Stats/Stats.h:576` | Stats/CPU trace can overlap by configuration; not every target enables this. |
| Editor loading | UE5.6 `Plugins/AudioInsights/AudioInsights.uplugin:35-37`; UE5.6 `Source/Runtime/Projects/Public/ModuleDescriptor.h:29` | Legal Editor Default counterexample; PostConfigInit is not the first enum phase. |
| Reinstancing/cleanup | UE5.6 `Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h:3232`; UE5.6 `Source/Runtime/Engine/Classes/Engine/World.h:4398` | Distinct event contracts; no experiment proving all subsystems rebuild. |
| Custom versions | UE5.6 `Source/Runtime/Core/Public/Serialization/Archive.h:1945`; UE5.6 `Source/Runtime/Core/Public/Serialization/Archive.h:231` | Registration, absent tag and containing-archive version metadata required; no save/load test. |
| Atomic policy | UE5.3 `Source/Runtime/Core/Public/Templates/Atomic.h:13,236,257,636` | Load/Store available; source recommends std::atomic for new code, without a claimed unspecified compatibility repair. |
| GC reachability | UE5.3 `Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp:3582,3638,3777`; UE5.6 same path `:4313,4324,4339,4466` | Root/reachability algorithm supports collecting unreachable cycles; no dedicated GC cycle runtime test. |

These are targeted declaration/implementation checks, not an exhaustive proof of every engine configuration. Published official links in each reference support the documented mechanisms and conventions. Current/unversioned documentation is not used to invent historical API boundaries; the installed headers establish the listed version differences.

## Checks Performed

- All nine knowledge files and their fenced examples were reviewed after correction; 25 stable rule IDs and 15 historical entries were reconciled.
- All 40 audit finding IDs have an explicit correction, qualification, or retirement in [AUDIT-RESOLUTION.md](AUDIT-RESOLUTION.md).
- Skill frontmatter/identifier, required local references and anchors, text encoding, repository diff whitespace, and publication content scope were checked.
- Installed engine declarations/implementations above were inspected across the applicable listed versions. This does not mean every illustrative fragment was compiled.
- Local GCC 15.2 C++17 syntax-only probes verified standard C++ initialization-order diagnostics, a user-defined default constructor that is still trivially copyable, and incomplete std::unique_ptr ownership with out-of-line definitions. The revised positive/negative initializer examples were also extracted and checked: the positive example was warning-free and the deliberate negative produced the expected reorder warning. These were not Unreal/TUniquePtr compiler tests.
- Clang preprocessing of the revised version-compatibility header against each of the five actual installed engine version headers passed, selecting the expected 5.3/5.4 versus 5.5-5.7 branches. This checked include/macro preprocessing only, not a UE translation unit, API instantiation or plugin build.
- The original collection's source hashes and pre-existing modified/untracked state were preserved. The standalone MIT license and original attribution were retained.

Local audit/probe artifacts are outside this repository. They are not a build/CI package or a runtime dependency of the skill.

## Not Performed / Still Required

No Unreal Engine compilation, UHT, BuildPlugin, cross-DLL link, Editor/Game/Shipping execution, Blueprint/GC/parallel/Tick test, asset cook/load, serialization migration round trip, Hot Reload/Live Coding experiment, performance benchmark, cross-platform build, or Fab submission was performed. Historical projects and their success/confidence claims were not reproduced.

For a consuming plugin, build supported versions/platforms and applicable targets/configurations in an isolated test project. Verify declaration fragments with real reflection/module context; test lifetime/error paths, old/new serialized/config data, Blueprint interfaces, loading/cooking and scheduling contracts. Measure any claimed performance gain using equivalent behavior. A compile pass alone is insufficient.

## Policy and Time-Sensitive Limits

The [Fab format/structure page](https://dev.epicgames.com/documentation/en-us/fab/asset-file-format-and-structure-requirements-in-fab) was readable. The [technical-requirements page](https://www.fab.com/o/technical-requirements) was accessible but its full policy body was not returned by the audit's text reader; no complete/current compliance claim is made. Product-specific requirements must be checked before submission.

The [official UE5.3 Visual Studio page](https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-visual-studio-development-environment-for-cplusplus-projects-in-unreal-engine?application_version=5.3) supports the corrected historical toolchain distinction. Verify each later release/patch's compiler and SDK rather than assuming all VS2022 versions work.

No claim of error-free knowledge or all-version runtime certification is made. Follow the actual consuming engine, module, target and user contract, and update evidence when those change.
