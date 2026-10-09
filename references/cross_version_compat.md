# UE Cross-Version Compatibility Guide

## Version Macros

Define plugin-specific version macros to simplify conditional compilation:

```cpp
// Define in MyPluginVersionCompat.h
#define MYPLUGIN_ENGINE_5_3_OR_LATER \
    (ENGINE_MAJOR_VERSION == 5 && ENGINE_MINOR_VERSION >= 3)

#define MYPLUGIN_ENGINE_5_4_OR_LATER \
    (ENGINE_MAJOR_VERSION == 5 && ENGINE_MINOR_VERSION >= 4)

#define MYPLUGIN_ENGINE_5_5_OR_LATER \
    (ENGINE_MAJOR_VERSION == 5 && ENGINE_MINOR_VERSION >= 5)
```

## Common API Changes

### FProperty::ElementSize (Deprecated in UE 5.5+ in the Source Notes)

```cpp
// UE 5.3-5.4: direct field access
const int32 ElementSize = Property->ElementSize;

// UE 5.5+: use the getter
const int32 ElementSize = Property->GetElementSize();

// Cross-version implementation
#if MYPLUGIN_ENGINE_5_5_OR_LATER
    const int32 ElementSize = Property->GetElementSize();
#else
    const int32 ElementSize = Property->ElementSize;
#endif
```

It can also be wrapped in a macro:

```cpp
#if MYPLUGIN_ENGINE_5_5_OR_LATER
    #define MYPLUGIN_GET_ELEMENT_SIZE(Prop) (Prop)->GetElementSize()
#else
    #define MYPLUGIN_GET_ELEMENT_SIZE(Prop) (Prop)->ElementSize
#endif
```

### FCompression::GetMaximumCompressedSize (UE 5.4+)

```cpp
// UE 5.3
int32 MaxSize = FCompression::GetMaximumCompressedSize(
    UncompressedSize, NAME_Zlib, ECompressionFlags::None);

// UE 5.4+: additional argument
int32 MaxSize = FCompression::GetMaximumCompressedSize(
    UncompressedSize, NAME_Zlib, ECompressionFlags::None, nullptr);
```

## Atomic Operations

### TAtomic API Differences

```cpp
// Direct assignment/reads may behave differently across UE versions
// Recommended: wrap atomic operations
template<typename T>
void AtomicStore(TAtomic<T>& Atomic, T Value)
{
    Atomic.Store(Value);
}

template<typename T>
T AtomicLoad(const TAtomic<T>& Atomic)
{
    return Atomic.Load();
}
```

## Compiler Differences

The original reference lists the following recommendations. They are historical source guidance, not a verified toolchain support matrix; check Epic's requirements for your exact engine release before choosing a compiler.

| UE version | Visual Studio recommendation in the source |
|------------|--------------------------------------------|
| UE 5.3 | VS 2019 |
| UE 5.4+ | VS 2022 |

## Compatibility Practices

1. **Centralize version macros:** Put all version checks in one header.
2. **Wrap compatibility layers:** Expose a consistent interface for changing APIs instead of scattering `#if` blocks.
3. **Declare Build.cs dependencies:** Declare compatibility-layer modules in `PublicDependencyModuleNames`.
4. **Use a multi-version CI matrix:** Compile against all targeted UE versions.

```yaml
strategy:
  matrix:
    ue_version: [5.3, 5.4, 5.5]
```

This is an example matrix for a consuming plugin, not a workflow included in this skill repository.

## Debugging

```cpp
// Report the UE version at runtime
UE_LOG(LogMyPlugin, Display, TEXT("UE Version: %d.%d"),
    ENGINE_MAJOR_VERSION, ENGINE_MINOR_VERSION);
```
