# Learning Records

> AI agents append reusable findings after review or optimization tasks; maintainers periodically review them.
> High-confidence entries may be promoted into formal reference rules after validation.
> These are historical records, not tests reproduced for this standalone repository. Project-specific names remain anonymized. Rules and affected resolutions were qualified on 2026-10-09 after the technical audit; historical confidence labels describe the original records, not a new test certificate. See [VALIDATION.md](../VALIDATION.md) and [AUDIT-RESOLUTION.md](../AUDIT-RESOLUTION.md).

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
- **Resolution:** Reuse independently owned per-task/thread buffers or safely partitioned storage at the correct frame/reset boundary.
- **Rule:** Measure allocation cost in hot ParallelFor work; reuse only exclusively owned/per-task/thread-local or safely partitioned buffers, without concurrent structural mutation.
- **Confidence:** high
- **Scope:** General: high-frequency ParallelFor work
```

## Records

<a id="learning-01"></a>
### 2026-06-08 - architecture

- **Observation:** A plugin's main module added editor-only modules such as UnrealEd, BlueprintGraph, and KismetCompiler to `PublicDependencyModuleNames` inside `if (Target.bBuildEditor)`. Public dependencies propagated editor headers into downstream Runtime modules.
- **Resolution:** The original project moved implementation-only editor dependencies to Private. Generalize only after checking public interfaces and module separation.
- **Rule:** Choose Public or Private dependencies from actual header exposure; move editor interfaces into an appropriate module instead of blindly rewriting every editor-conditional block.
- **Confidence:** high
- **Scope:** General: Build.cs files with editor-conditional dependency blocks.
- **Evidence status (2026-10-09):** Dependency exposure follows Epic Modules guidance; the historical project was not reproduced. See shipping_checklist.md, Module Architecture.

<a id="learning-02"></a>
### 2026-06-08 - blueprint

- **Observation:** A review reported that a `UCurveFloat*` member inside a USTRUCT lacked UPROPERTY and was unsafe for GC. It read only the member declaration, overlooking the `UPROPERTY(...)` macro on the immediately preceding line; the annotation existed and GC tracked the member.
- **Resolution:** Read the preceding line when checking for missing UPROPERTY. Search with one line of preceding context, such as `rg -B 1`, or match both UPROPERTY and the member name.
- **Rule:** Inspect the complete UPROPERTY declaration and reflection/reference chain before reporting an untracked field; one line of preceding context is only a search aid.
- **Confidence:** high
- **Scope:** General: GC and reflection reviews. Rule #1 is already in [safety_patterns.md](safety_patterns.md#gc); this entry records a review-method lesson.
- **Evidence status (2026-10-09):** Review method qualified; generic GC ownership evidence is in safety_patterns.md. The historical field was not independently reproduced.

<a id="learning-03"></a>
### 2026-06-08 - ue-api

- **Observation:** A K2Node's `ValidateNodeDuringCompilation` used `MessageLog.Error`, blocking Blueprint compilation and execution.
- **Resolution:** Use `MessageLog.Warning` for the advisory message so the user can still compile and run.
- **Rule:** Use Warning for a genuinely advisory K2Node message that permits valid execution; retain Error when the node/program is invalid and execution must be blocked.
- **Confidence:** high
- **Scope:** K2Node ValidateNodeDuringCompilation implementations.
- **Evidence status (2026-10-09):** Diagnostic distinction is qualified by validity/contract; no historical K2Node graph was run.

<a id="learning-04"></a>
### 2026-06-08 - architecture / ue-api

- **Observation:** Reviews of K2Node and asset-editor modules repeatedly classified "check is removed in Shipping and causes a crash" or "container/pointer lacks UPROPERTY and leaks through GC" as P0.
- **Resolution:** Confirm the module type before assigning severity. Editor/UncookedOnly code does not enter a Shipping game's runtime. In the original observed Editor/Development configurations, check remained enabled, so the Shipping argument did not apply.
- **Rule:** Assign severity from module, target/configuration and actual effect. Editor-only code is not Shipping-game runtime, but editor crashes/data loss can still be critical; do not downgrade a real invalid-program error to an advisory warning.
- **Confidence:** high
- **Scope:** General: UE plugins containing editor modules.
- **Evidence status (2026-10-09):** Target-aware severity replaces blanket P0/P1 rules; historical editor/cook behavior was not reproduced.

<a id="learning-05"></a>
### 2026-06-08 - ue-api

- **Observation:** `IsValid()` was recommended as a universal replacement for `!= nullptr` in a safe-execution macro that also accepted non-UObject raw pointers.
- **Resolution:** Limit `IsValid()` to UObject pointers. Non-UObject pointers, including custom structs and platform handles, need their appropriate null checks; forcing IsValid can cause compilation or semantic errors.
- **Rule:** Apply UObject IsValid only within a valid object lifetime; non-UObject pointers/handles need their own checks, and IsValid cannot repair a freed raw pointer.
- **Confidence:** high
- **Scope:** General.
- **Evidence status (2026-10-09):** Object validity declaration inspected in UE5.3-5.7; no object-lifetime runtime test.

<a id="learning-06"></a>
### 2026-06-08 - compilation

- **Observation:** A recursive lambda inside a member function, such as a HeapSort Heapify helper, was extracted into a file-level static function to eliminate TFunction heap allocation. C2248 occurred because it accessed private static members such as GetPropertyValuePtr/ComparePropertyValues.
- **Resolution:** Extract it as a **private static member function** of the class: declare it in the header and define it as `UClass::Func` in the source file.
- **Rule:** Preserve legal private access when extracting a helper; a private static member is one suitable approach. Prove allocation cost before claiming TFunction always allocates on the heap.
- **Confidence:** high
- **Scope:** General: extracting lambdas into functions.
- **Evidence status (2026-10-09):** Standard C++ access principle; exact historical C2248 and TFunction allocation were not reproduced.

<a id="learning-07"></a>
### 2026-06-08 - compilation / ue-api

- **Observation:** Replacing `STATGROUP_Game` used by `DECLARE_CYCLE_STAT_EXTERN` with a custom `STATGROUP_Xxx` caused C3646 (unknown TGroup override specifier) and "CompileTimeEnable is not a member" errors in some headers.
- **Resolution:** Make the custom `DECLARE_STATS_GROUP` visible to every header that uses it. STATGROUP_Game is engine-defined and available through CoreMinimal; a custom group should live in a shared foundational header, such as a module Types.h, guarded by `#if STATS`, rather than an infrequently included header.
- **Rule:** Make custom StatGroup declarations visible before every using header/source; match the STATS macro environment and do not assume a group in one private header is visible elsewhere.
- **Confidence:** high
- **Scope:** General: modules sharing StatGroups across multiple headers.
- **Evidence status (2026-10-09):** Stats macro declarations inspected; historical C3646 was not reproduced and no Profiler session was run.

<a id="learning-08"></a>
### 2026-06-08 - compilation / architecture

- **Observation:** An Editor module in a separate DLL switched its logs to its Runtime dependency's category and hit LNK2001 for an unresolved log-category symbol.
- **Resolution:** The Runtime module's DEFINE_LOG_CATEGORY was not exported across modules. Restore the Editor module's own category, for example LogExampleEditor, with a declaration and definition in that module.
- **Rule:** Use a module-local log category or deliberately export a shared category symbol across DLLs; diagnose the actual missing symbol before changing logging ownership.
- **Confidence:** high
- **Scope:** General: organizing logs in multi-module plugins.
- **Evidence status (2026-10-09):** Cross-module symbol/export requirement reviewed; historical LNK2001 was not reproduced in a DLL link test.

<a id="learning-09"></a>
### 2026-06-08 - architecture

- **Observation:** A size-threshold early return intended to prevent O(N^2) stalls changed `ValidatePoissonSampling` from continuing validation to `return false`. Large datasets were reported as invalid; a performance safeguard became a functional failure.
- **Resolution:** Distinguish skipped work from failed validation. The original record proposed true as a conservative fallback; that claim is retired unless the caller contract explicitly defines it safely. Use a documented skipped/unknown result or a sound alternative algorithm.
- **Rule:** A performance guard must preserve the result contract: represent skipped/unknown validation explicitly, or choose a documented safe fallback. Do not report unknown data as validated merely by returning true.
- **Confidence:** high
- **Scope:** General: functions with size-limit safeguards.
- **Evidence status (2026-10-09):** Corrected on 2026-10-09: the original suggestion to return true for skipped validation is not a general conservative result. No original dataset/contract was reproduced.

<a id="learning-10"></a>
### 2026-06-08 - blueprint

- **Observation:** Adding `ClampMin="0.1"` to a numeric UPROPERTY improved the editor UI but prevented its documented sentinel behavior: values at or below zero triggered automatic calculation, and the clamp blocked those values.
- **Resolution:** Remove ClampMin and keep UIMin/UIMax, which suggest a slider range without blocking manually entered sentinel values.
- **Rule:** Check sentinel/business semantics before adding editor ClampMin/ClampMax; use UIMin/UIMax for a suggested range when manual sentinel values must remain possible. Runtime validation is separate.
- **Confidence:** high
- **Scope:** General: UPROPERTY fields with sentinel semantics.
- **Evidence status (2026-10-09):** Editor clamp metadata semantics checked; no sentinel input/serialization runtime test.

<a id="learning-11"></a>
### 2026-08-27 - blueprint

- **Observation:** Moving a config switch from module A's developer settings to module B changed only the reader and new default. Existing explicitly stored ini values were not migrated, and the old field lacked a UI deprecation notice; its ToolTip described only the new default. Projects that had explicitly disabled the feature silently re-enabled it after upgrading.
- **Resolution:** Use one or more of: a one-time read and migration of the old ini value during initialization; deprecation instructions in the old metadata/ToolTip; or retaining the old field as a runtime compatibility mirror.
- **Rule:** When relocating persisted config settings, migrate explicit stored values or provide a documented compatibility path; equal defaults do not preserve customized behavior.
- **Confidence:** high
- **Scope:** General: relocating UPROPERTY(config) switches.
- **Evidence status (2026-10-09):** Migration principle retained; no historical project ini migration test.

<a id="learning-12"></a>
### 2026-08-27 - compilation

- **Observation:** In UE 5.3, passing FString parameters to GConfig->GetString/RemoveKey caused C2664 because Section/Key expected const TCHAR*. Calling nonexistent FlushFile caused C2039; the relevant member was `Flush(bool bRemoveFromCache, const FString& Filename)`. Including "Misc/CoreGlobals.h" failed because GGameIni/GEditorIni were declared in root-level CoreGlobals.h.
- **Resolution:** Use TCHAR literal constants for Section/Key, persist with Flush(false, Filename), and include "CoreGlobals.h".
- **Rule:** Use the exact GConfig API/header for the engine version. For the inspected TCHAR* Section/Key signatures use literals or appropriately dereferenced FString values; use Flush with its actual filename/cache semantics.
- **Confidence:** high
- **Scope:** General: migration/tooling code directly reading and writing GConfig.
- **Evidence status (2026-10-09):** ConfigCacheIni.h/CoreGlobals.h declarations checked across the installed UE5.3-5.7 patches; no migration tool was executed.

<a id="learning-13"></a>
### 2026-08-27 - architecture / shipping

- **Observation:** (a) Assigning AdditionalCompilerArguments in ModuleRules caused CS0103: the property belonged to TargetRules and applied to the entire target through UEBuildTarget. (b) An extracted packaged copy remained under a project's Plugins tree with duplicate plugin/module names and static methods; UBT compiled both rule definitions into the same assembly and reported CS0121 ambiguity. (c) A host Target.cs pinned EngineIncludeOrderVersion.Unreal5_3 and disabled options such as UndefinedIdentifierWarningLevel; newer engines rejected the target because it shared artifacts with UnrealEditor.
- **Resolution:** Address encoding/toolchain settings from actual compiler evidence; a UTF-8 BOM can help a particular configuration but is not a universal fix. Keep packaged duplicates outside scanned plugin trees and validate plugin builds independently of host-target conflicts, with runtime tests separately.
- **Rule:** In the inspected versions AdditionalCompilerArguments belongs to TargetRules, not ModuleRules. Avoid duplicate plugin/module definitions in scanned trees; use an independent BuildPlugin check plus runtime tests rather than assuming a host-target or UTF-8 BOM workaround proves plugin compatibility.
- **Confidence:** high
- **Scope:** General: plugin repository engineering and multi-version packaging practice.
- **Evidence status (2026-10-09):** ModuleRules/TargetRules declarations checked in the installed versions. Historical build conflicts were not reproduced; BOM is not a universal source/build fix.

<a id="learning-14"></a>
### 2026-08-27 - architecture

- **Observation:** A bulk fix replaced an unreliable predicate, such as IsActorInitialized() being false during loading, only on the main path. Two or more secondary branches still used it, leaving the same bug intact elsewhere, even though comments in the new mechanism explained why the predicate was harmful.
- **Resolution:** Search all call sites of the faulty predicate and converge them on a single wrapper. At minimum, list any remaining call sites in the new mechanism's comments.
- **Rule:** When replacing a faulty predicate, inspect all call sites and centralize the corrected decision where appropriate; keep context-specific exceptions explicit and verify the actual lifecycle state.
- **Confidence:** high
- **Scope:** General: bulk replacements of incorrect detection flags or predicates.
- **Evidence status (2026-10-09):** Method retained; no claim that IsActorInitialized is universally wrong or that the original loading scenario was reproduced.

<a id="learning-15"></a>
### 2026-06-08 - architecture

- **Observation:** Bulk fixes or refactors, including function extraction, StatGroup changes, log organization, guards, and LoadingPhase changes, introduced semantic or behavioral errors despite compiling. Examples included interface filtering inconsistent with movement behavior, a later LoadingPhase missing K2Node menu registration, and downgrading Error to Warning for a failure that should block execution.
- **Resolution:** After broad changes, perform a separate reverse review of the git diff: check whether each change solves its problem, introduces an antipattern, or breaks runtime/design behavior. Compilation alone is insufficient.
- **Rule:** After nontrivial changes, review the diff for external behavior, interface, ownership and lifecycle changes; compilation alone does not establish equivalence or rule applicability.
- **Confidence:** high
- **Scope:** General: bulk changes and changes made through multi-agent collaboration.
- **Evidence status (2026-10-09):** Method retained. In particular, late loading or changing Error to Warning can be wrong outside its intended context; no historical project diff/runtime was reproduced.
