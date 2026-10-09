# Learning Records

> AI agents append reusable findings after review or optimization tasks; maintainers periodically review them.
> High-confidence entries may be promoted into formal reference rules after validation.
> These are translated historical records, not tests performed for this standalone repository. Project-specific names have been anonymized.

## Entry Format

```text
### [Date] - [Category]
- **Observation:** Specific problem or discovery
- **Resolution:** Action taken and result
- **Rule:** One reusable sentence
- **Confidence:** high / medium / low
- **Scope:** General / description of the specific scenario
```

## Categories

| Category | Meaning |
|----------|---------|
| `compilation` | Compiler errors, linking, UHT |
| `performance` | Bottlenecks, hot paths, Stats |
| `gc-memory` | GC leaks, memory management, pointer safety |
| `blueprint` | Blueprint exposure, UPROPERTY/UFUNCTION, metadata |
| `threading` | Thread safety, ParallelFor, GameThread |
| `shipping` | Shipping configuration, cross-platform support, FAB submissions |
| `architecture` | Module structure, dependencies, subsystems, export strategy |
| `ue-api` | UE APIs, version differences, implicit behavior |

## Promotion Criteria

Promote an entry when all these conditions hold:

1. Confidence is `high`.
2. It has been validated in at least two distinct projects or scenarios.
3. It does not conflict with existing rules.

Add the rule to the corresponding reference and mark the entry `[Promoted -> xxx_patterns.md]` here.

## Examples

These demonstrate the format; the actual rules already appear in the formal references.

**Compilation error:**

```text
### 2026-04-04 - compilation
- **Observation:** Replacing TArray<int32> with TArray<int32, TInlineAllocator<64>> caused C2664 when passing it to a function accepting TArray<int32>&.
- **Resolution:** Revert to TArray<int32> plus Reserve(), or template the downstream function.
- **Rule:** TArray<T, AllocA> and TArray<T, AllocB> are different UE types whose references are not interchangeable.
- **Confidence:** high
- **Scope:** General: any use of TInlineAllocator
```

**Review finding:**

```text
### 2026-04-04 - performance
- **Observation:** Every instance in a ParallelFor lambda called Reserve() on a new TArray; tens of thousands of instances produced tens of thousands of heap allocations.
- **Resolution:** Promote temporary arrays into system members and reuse them across frames.
- **Rule:** Avoid heap allocation in hot ParallelFor paths; reuse members or thread-local buffers.
- **Confidence:** high
- **Scope:** General: high-frequency ParallelFor work
```

## Records

### 2026-06-08 - architecture

- **Observation:** A plugin's main module added editor-only modules such as UnrealEd, BlueprintGraph, and KismetCompiler to `PublicDependencyModuleNames` inside `if (Target.bBuildEditor)`. Public dependencies propagated editor headers into downstream Runtime modules.
- **Resolution:** Move all modules in that editor-conditional block to `PrivateDependencyModuleNames`.
- **Rule:** Dependencies inside `if (Target.bBuildEditor)` should use Private under this convention to prevent editor headers from contaminating downstream Runtime modules.
- **Confidence:** high
- **Scope:** General: Build.cs files with editor-conditional dependency blocks.

### 2026-06-08 - blueprint

- **Observation:** A review reported that a `UCurveFloat*` member inside a USTRUCT lacked UPROPERTY and was unsafe for GC. It read only the member declaration, overlooking the `UPROPERTY(...)` macro on the immediately preceding line; the annotation existed and GC tracked the member.
- **Resolution:** Read the preceding line when checking for missing UPROPERTY. Search with one line of preceding context, such as `rg -B 1`, or match both UPROPERTY and the member name.
- **Rule:** UE commonly places `UPROPERTY(...)` on a separate line above its field; inspect that line before reporting a missing annotation.
- **Confidence:** high
- **Scope:** General: GC and reflection reviews. Rule #1 is already in [safety_patterns.md](safety_patterns.md#gc); this entry records a review-method lesson.

### 2026-06-08 - ue-api

- **Observation:** A K2Node's `ValidateNodeDuringCompilation` used `MessageLog.Error`, blocking Blueprint compilation and execution.
- **Resolution:** Use `MessageLog.Warning` for the advisory message so the user can still compile and run.
- **Rule:** Advisory messages in K2Node `ValidateNodeDuringCompilation`, such as a missing initialization-node suggestion, should use Warning; Error blocks compilation.
- **Confidence:** high
- **Scope:** K2Node ValidateNodeDuringCompilation implementations.

### 2026-06-08 - architecture / ue-api

- **Observation:** Reviews of K2Node and asset-editor modules repeatedly classified "check is removed in Shipping and causes a crash" or "container/pointer lacks UPROPERTY and leaks through GC" as P0.
- **Resolution:** Confirm the module type before assigning severity. Editor/UncookedOnly code does not enter a Shipping game's runtime. In the original observed Editor/Development configurations, check remained enabled, so the Shipping argument did not apply.
- **Rule:** Evaluate editor-module check/GC issues by their effect on editor crashes, Blueprint compilation, and cooking, usually P1, rather than automatically applying Shipping-runtime P0 reasoning; K2Node advisory failures should degrade gracefully to MessageLog.Warning.
- **Confidence:** high
- **Scope:** General: UE plugins containing editor modules.

### 2026-06-08 - ue-api

- **Observation:** `IsValid()` was recommended as a universal replacement for `!= nullptr` in a safe-execution macro that also accepted non-UObject raw pointers.
- **Resolution:** Limit `IsValid()` to UObject pointers. Non-UObject pointers, including custom structs and platform handles, need their appropriate null checks; forcing IsValid can cause compilation or semantic errors.
- **Rule:** The IsValid-versus-nullptr rule applies to UObject pointers, not generic macros serving non-UObject types.
- **Confidence:** high
- **Scope:** General.

### 2026-06-08 - compilation

- **Observation:** A recursive lambda inside a member function, such as a HeapSort Heapify helper, was extracted into a file-level static function to eliminate TFunction heap allocation. C2248 occurred because it accessed private static members such as GetPropertyValuePtr/ComparePropertyValues.
- **Resolution:** Extract it as a **private static member function** of the class: declare it in the header and define it as `UClass::Func` in the source file.
- **Rule:** A lambda accessing private class members must be extracted as a class member function, possibly static; a file-level helper lacks private access.
- **Confidence:** high
- **Scope:** General: extracting lambdas into functions.

### 2026-06-08 - compilation / ue-api

- **Observation:** Replacing `STATGROUP_Game` used by `DECLARE_CYCLE_STAT_EXTERN` with a custom `STATGROUP_Xxx` caused C3646 (unknown TGroup override specifier) and "CompileTimeEnable is not a member" errors in some headers.
- **Resolution:** Make the custom `DECLARE_STATS_GROUP` visible to every header that uses it. STATGROUP_Game is engine-defined and available through CoreMinimal; a custom group should live in a shared foundational header, such as a module Types.h, guarded by `#if STATS`, rather than an infrequently included header.
- **Rule:** Put custom StatGroup declarations in a shared foundational header inside #if STATS so they are visible before every DECLARE_CYCLE_STAT_EXTERN that uses them.
- **Confidence:** high
- **Scope:** General: modules sharing StatGroups across multiple headers.

### 2026-06-08 - compilation / architecture

- **Observation:** An Editor module in a separate DLL switched its logs to its Runtime dependency's category and hit LNK2001 for an unresolved log-category symbol.
- **Resolution:** The Runtime module's DEFINE_LOG_CATEGORY was not exported across modules. Restore the Editor module's own category, for example LogExampleEditor, with a declaration and definition in that module.
- **Rule:** Use each module's own log category unless a deliberate cross-module export exists; default log categories referenced across separate DLLs can produce LNK2001.
- **Confidence:** high
- **Scope:** General: organizing logs in multi-module plugins.

### 2026-06-08 - architecture

- **Observation:** A size-threshold early return intended to prevent O(N^2) stalls changed `ValidatePoissonSampling` from continuing validation to `return false`. Large datasets were reported as invalid; a performance safeguard became a functional failure.
- **Resolution:** Return a semantically neutral or conservative result when skipping expensive work: the source example conservatively passes validation with true, or returns a cheap partial statistic such as an O(N) centroid. Log that work was skipped rather than failed.
- **Rule:** A performance early return must preserve the function's semantic contract: skipped or uncomputed work is not equivalent to failure; retain valid results available cheaply.
- **Confidence:** high
- **Scope:** General: functions with size-limit safeguards.

### 2026-06-08 - blueprint

- **Observation:** Adding `ClampMin="0.1"` to a numeric UPROPERTY improved the editor UI but prevented its documented sentinel behavior: values at or below zero triggered automatic calculation, and the clamp blocked those values.
- **Resolution:** Remove ClampMin and keep UIMin/UIMax, which suggest a slider range without blocking manually entered sentinel values.
- **Rule:** Before adding ClampMin/ClampMax, check for sentinel values such as <=0, -1, or 0; use only UIMin/UIMax when those values carry special semantics.
- **Confidence:** high
- **Scope:** General: UPROPERTY fields with sentinel semantics.

### 2026-08-27 - blueprint

- **Observation:** Moving a config switch from module A's developer settings to module B changed only the reader and new default. Existing explicitly stored ini values were not migrated, and the old field lacked a UI deprecation notice; its ToolTip described only the new default. Projects that had explicitly disabled the feature silently re-enabled it after upgrading.
- **Resolution:** Use one or more of: a one-time read and migration of the old ini value during initialization; deprecation instructions in the old metadata/ToolTip; or retaining the old field as a runtime compatibility mirror.
- **Rule:** When moving config settings across classes or modules, an unchanged default does not guarantee unchanged behavior; handle stored explicit values to preserve customized projects' behavior.
- **Confidence:** high
- **Scope:** General: relocating UPROPERTY(config) switches.

### 2026-08-27 - compilation

- **Observation:** In UE 5.3, passing FString parameters to GConfig->GetString/RemoveKey caused C2664 because Section/Key expected const TCHAR*. Calling nonexistent FlushFile caused C2039; the relevant member was `Flush(bool bRemoveFromCache, const FString& Filename)`. Including "Misc/CoreGlobals.h" failed because GGameIni/GEditorIni were declared in root-level CoreGlobals.h.
- **Resolution:** Use TCHAR literal constants for Section/Key, persist with Flush(false, Filename), and include "CoreGlobals.h".
- **Rule:** For the GConfig file APIs described in these UE 5.x notes, use TCHAR* keys and Flush for persistence; include CoreGlobals.h for global ini variables rather than the misleading Misc/ path.
- **Confidence:** high
- **Scope:** General: migration/tooling code directly reading and writing GConfig.

### 2026-08-27 - architecture / shipping

- **Observation:** (a) Assigning AdditionalCompilerArguments in ModuleRules caused CS0103: the property belonged to TargetRules and applied to the entire target through UEBuildTarget. (b) An extracted packaged copy remained under a project's Plugins tree with duplicate plugin/module names and static methods; UBT compiled both rule definitions into the same assembly and reported CS0121 ambiguity. (c) A host Target.cs pinned EngineIncludeOrderVersion.Unreal5_3 and disabled options such as UndefinedIdentifierWarningLevel; newer engines rejected the target because it shared artifacts with UnrealEditor.
- **Resolution:** (a) Fix file character encoding with a UTF-8 BOM for MSVC recognition instead of module compiler arguments. (b) Move packaged copies outside the Plugins scan tree. (c) Validate multiple versions through an independent BuildPlugin -Plugin= pipeline and handle host-target upgrades separately.
- **Rule:** The original records recommend target-level compiler options only, no duplicate plugin copies inside Plugins, and an independent BuildPlugin pipeline as the reliable route for validating plugin versions when host targets share editor artifacts.
- **Confidence:** high
- **Scope:** General: plugin repository engineering and multi-version packaging practice.

### 2026-08-27 - architecture

- **Observation:** A bulk fix replaced an unreliable predicate, such as IsActorInitialized() being false during loading, only on the main path. Two or more secondary branches still used it, leaving the same bug intact elsewhere, even though comments in the new mechanism explained why the predicate was harmful.
- **Resolution:** Search all call sites of the faulty predicate and converge them on a single wrapper. At minimum, list any remaining call sites in the new mechanism's comments.
- **Rule:** Replacing a faulty predicate requires updating all call sites together, not just the triggering path; centralize the decision in one function to avoid preserving equivalent bugs elsewhere.
- **Confidence:** high
- **Scope:** General: bulk replacements of incorrect detection flags or predicates.

### 2026-06-08 - architecture

- **Observation:** Bulk fixes or refactors, including function extraction, StatGroup changes, log organization, guards, and LoadingPhase changes, introduced semantic or behavioral errors despite compiling. Examples included interface filtering inconsistent with movement behavior, a later LoadingPhase missing K2Node menu registration, and downgrading Error to Warning for a failure that should block execution.
- **Resolution:** After broad changes, perform a separate reverse review of the git diff: check whether each change solves its problem, introduces an antipattern, or breaks runtime/design behavior. Compilation alone is insufficient.
- **Rule:** After nontrivial changes compile, add a meta-review focused on altered external semantics and rules applied outside their valid context.
- **Confidence:** high
- **Scope:** General: bulk changes and changes made through multi-agent collaboration.
