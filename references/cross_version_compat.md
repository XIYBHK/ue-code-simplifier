# UE Cross-Version Compatibility Guide

These API notes were checked against installed UE **5.3.2, 5.4.4, 5.5.4, 5.6.1, and 5.7.4** declarations. This is source inspection, not a multi-version plugin build. See [VALIDATION.md](../VALIDATION.md) for changelists and evidence paths. Forks and other patches require their own checks.

## Version Macros

Include the engine's version definitions before using them. Scope a compatibility branch to versions it was actually designed to support; do not assume an unknown major version is compatible.

```cpp
#include "Runtime/Launch/Resources/Version.h"

#if ENGINE_MAJOR_VERSION != 5 || ENGINE_MINOR_VERSION < 3 || ENGINE_MINOR_VERSION > 7
#error This example compatibility header supports only UE 5.3 through 5.7.
#endif

// This plugin branch is specifically for supported UE 5.5-5.7 installations.
#define MYPLUGIN_ENGINE_5_5_TO_5_7 \
    (ENGINE_MAJOR_VERSION == 5 && ENGINE_MINOR_VERSION >= 5 && ENGINE_MINOR_VERSION <= 7)
```

For a production compatibility header, reject unsupported versions or implement/test a separate branch. Centralize version conditions, but do not use a threshold macro as evidence that all future APIs behave the same way.

## FProperty::ElementSize

`ElementSize` is directly accessible in the inspected 5.3/5.4 headers. In 5.5 it is marked deprecated and `GetElementSize()` is available; use the getter in the supported 5.5-5.7 branch.

```cpp
// Function-body fragment: Property must be a valid FProperty*.
// The compatibility header must reject versions outside the supported range.
#if MYPLUGIN_ENGINE_5_5_TO_5_7
const int32 ElementSize = Property->GetElementSize();
#else // Supported UE 5.3-5.4 only.
const int32 ElementSize = Property->ElementSize;
#endif
```

Evidence: `Source/Runtime/CoreUObject/Public/UObject/UnrealType.h`. This particular 5.5 boundary is supported by the installed headers.

## FCompression::GetMaximumCompressedSize

The 5.3/5.4 declaration takes the format name first, then the 32-bit uncompressed size, compression flags, and an integer compression parameter. There is no new `nullptr` argument at 5.4 and no `ECompressionFlags::None` enumerator in those headers.

```cpp
// UE 5.3-5.4 function-body fragment. Validate UncompressedSize first.
const int32 MaxSize = FCompression::GetMaximumCompressedSize(
    NAME_Zlib, UncompressedSize, COMPRESS_NoFlags, 0);
// Check the format's failure/result contract before using MaxSize.
```

Starting in the inspected 5.5 source, the 64-bit overload returns success as `bool` and writes the maximum size to an `int64&`; the old 32-bit overload is marked deprecated in 5.5. The same declarations are present in 5.6/5.7.

```cpp
// UE 5.5-5.7 function-body fragment; InUncompressedSizeBytes is validated int64.
int64 MaxSize = 0;
if (!FCompression::GetMaximumCompressedSize(
        NAME_Zlib, MaxSize, InUncompressedSizeBytes, 0))
{
    return; // Or propagate the compression-format error.
}
// Validate conversions and allocation limits before using MaxSize.
```

`GetMaximumCompressedSize` and `CompressMemoryBound` have different contracts: the source directs encoding-buffer allocation to `CompressMemoryBound`. Select the correct query for encoding versus checking a possible compressed input, use the corresponding version's signature, and handle failures and size overflow. Do not assume one return type or an `int32` allocation size for every version.

Evidence: `Source/Runtime/Core/Public/Misc/Compression.h` and `CompressionFlags.h`; see the precise lines in [VALIDATION.md](../VALIDATION.md#api-and-behavior-evidence).

## Atomic Operations

`TAtomic::Load` and `Store` are present in all five inspected versions. A wrapper can centralize a project's compatibility or memory-order policy, but no specific API incompatibility is established merely by adding a wrapper. The engine's `Atomic.h` already recommends `std::atomic` for new code in the inspected 5.3 source.

Preserve existing ownership/concurrency contracts when changing atomic types. Choose memory ordering from the synchronization requirement, not solely for speed. UE-container conventions do not prohibit every standard-library facility.

## Toolchain Selection

Use the official requirements for the exact engine version, patch, target, and distribution method. [Epic's 5.3 Visual Studio setup page](https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-visual-studio-development-environment-for-cplusplus-projects-in-unreal-engine?application_version=5.3) lists VS2019 16.11.5+ as supported and VS2022 17.4+, with 17.6 recommended/default. Its VS2019 advice for a particular precompiled Marketplace compatibility case is for UE5.2; it is not a general UE5.3 default.

Do not reduce later releases' requirements to just "VS2022": check their supported MSVC toolset and SDK. A local header review does not certify a compiler matrix.

## Compatibility Practices

1. Centralize actual version differences and reject unsupported branches.
2. Keep public/private module dependencies consistent with header exposure, not with a blanket rule for compatibility modules.
3. Compile a separate consuming plugin against each claimed engine version and supported platform. `BuildPlugin` is a useful packaging/build check; it does not replace runtime or Blueprint tests.
4. Test migrated serialization/config values and behavior, not just successful compilation.

```yaml
# Illustrative consuming-plugin matrix, not a workflow in this repository.
strategy:
  matrix:
    ue_version: [5.3, 5.4, 5.5, 5.6, 5.7]
```

```cpp
// Function-body fragment; define LogMyPlugin and include version definitions.
UE_LOG(LogMyPlugin, Display, TEXT("UE Version: %d.%d"),
    ENGINE_MAJOR_VERSION, ENGINE_MINOR_VERSION);
```
