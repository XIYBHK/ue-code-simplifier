---
name: ue-code-simplifier
description: Review and optimize existing Unreal Engine C++ plugin code. Use for UE/UE5/Unreal C++ code review, checking, optimization, refactoring, simplification, performance, quality, memory, GC, Tick, container selection, or FAB/Marketplace submission readiness. Also use for UE C++ performance problems (TMap, spatial queries, hot paths, frame rate), memory/GC/safety concerns (UPROPERTY, check/ensure, IsValid), compilation warnings (C5038, TInlineAllocator), and preparing .h/.cpp files for FAB/Marketplace submission. Do not use for writing new UE code from scratch, Blueprint-only tasks, HLSL/shaders, packaging/deployment troubleshooting, editor crash debugging, or project migration between UE versions.
---

# UE C++ Plugin Review and Optimization Standard

Based on UE5 engine-source observations, Epic's coding standards, FAB/Marketplace submission guidance, and the original author's production experience. See [README.md](README.md#version-scope-and-validation-limits) for the limits of this translated edition.

## Core Principles

- **Preserve functionality:** Change the implementation, not the logical result.
- **Keep Blueprint interfaces stable:** Do not change UFUNCTION/UPROPERTY signatures unless explicitly requested.
- **Focus on recent changes:** By default, review or optimize only code changed during the current session, unless the user explicitly requests a full review.
- **Verify critically:** Check that every recommendation or proposed solution is effective and reasonable.
- **Fix bugs with minimal changes:** Never add incidental refactoring to a bug fix.

## Avoid Excessive Simplification

Exercise restraint when optimizing code. Do not make it harder to understand or extend:

- **Prefer readability to brevity:** Explicit, clear code is better than clever, compact code.
- **Do not introduce nested template metaprogramming** unless it provides a clearly measured performance benefit.
- **Do not combine unrelated logic:** Each function should do one thing; do not merge functions merely to reduce their number.
- **Keep meaningful abstractions:** If an intermediate class or function clarifies the structure, do not remove it just to eliminate a layer.
- **Do not minimize line count:** Ten clear lines are better than three lines that require comments to explain them.
- **Keep valuable comments:** Remove comments that state the obvious; retain comments explaining why and the reasoning behind domain decisions.

## Task Routing

Follow the user's intent and focus on the corresponding steps. **Read only the reference files listed for that task.** Unrelated references consume context without adding value.

Read [references/learnings.md](references/learnings.md) at the beginning of each task, as required by the learning protocol below. Then load the task-specific references progressively.

| User intent | Focus | Task-specific references |
|-------------|-------|--------------------------|
| "Review this code" / "Code review" | Steps 1 through 5 | Load references sequentially as each step requires; do not read them all at once |
| "Optimize performance" / "Too slow" | Step 3 | [performance_patterns.md](references/performance_patterns.md) |
| "Prepare for release" / "Submit to Marketplace" | Steps 2 and 5 | [safety_patterns.md](references/safety_patterns.md), [shipping_checklist.md](references/shipping_checklist.md) |
| "Simplify this code" / "Refactor" | Steps 1 and 4 | [api_patterns.md](references/api_patterns.md), [epic_coding_standard.md](references/epic_coding_standard.md) |
| "Fix a bug" / "Compilation error" | Targeted fix | [common_issues.md](references/common_issues.md) |
| "Blueprint API" / "UPROPERTY" | Step 4 | [api_patterns.md](references/api_patterns.md) |
| "GC leak" / "Memory problem" | Step 2 | [safety_patterns.md](references/safety_patterns.md) |
| "Cross-version compatibility" | Focused compatibility review | [cross_version_compat.md](references/cross_version_compat.md) |

For a full review, read each reference when its step begins, complete that step, and only then load the next reference.

## Review Workflow

### Step 1: Assess Structure

1. Module structure: Runtime/Editor separation and Public/Private separation.
2. `.uplugin` and `Build.cs` configuration: LoadingPhase and declared dependencies.
3. IWYU headers, alphabetical include order, and `.generated.h` last.
4. Naming conventions: A/U/F/E/I/T prefixes.

See [references/epic_coding_standard.md](references/epic_coding_standard.md).

### Step 2: Review Safety

1. Assertion macros: `check` for development, `ensure` for runtime, and `verify` for expressions with side effects.
2. GC safety: UPROPERTY annotations for containers containing UObject pointers, and UObject creation methods.
3. Pointer safety: `IsValid()` instead of `!= nullptr`, and defensive programming.
4. Tick management: Disable by default, cache references, and use FTickableGameObject where appropriate.
5. Thread safety: Access UObjects only on the GameThread; do not access them inside ParallelFor.

See [references/safety_patterns.md](references/safety_patterns.md).

### Step 3: Review Performance

1. Stats profiling: Wrap each subsystem's hot functions with `SCOPE_CYCLE_COUNTER`.
2. ParallelFor: Include `TEXT("Name")` and MinBatchSize.
3. Container selection: Use spatial-prime hashing for spatial grids; avoid TMap on hot paths.
4. Hot paths: Reuse members instead of local allocations; use Memcpy instead of element-by-element copying where suitable.
5. TInlineAllocator: Its TArray specialization is a different type and cannot be passed interchangeably by reference with default TArray.

See [references/performance_patterns.md](references/performance_patterns.md).

### Step 4: Review API Quality

1. UPROPERTY metadata: UIMin/UIMax, ForceUnits, and EditCondition.
2. Delegate selection: Use Dynamic only when Blueprint exposure is required.
3. Subsystem selection: WorldSubsystem versus GameInstanceSubsystem.
4. API exports: MinimalAPI, selective exports, and full exports as progressively broader levels.
5. BlueprintPure: Add `BlueprintPure = false` to expensive const functions.
6. Custom log categories and console variables.

See [references/api_patterns.md](references/api_patterns.md).

### Step 5: Validate Release Readiness

1. Compile Editor, Game, and Shipping configurations.
2. Perform the seven FAB/Marketplace readiness checks.
3. Cross-platform support: No hard-coded paths or platform-specific headers.
4. Module loading phases: Use PostEngineInit for Editor modules under the original rule set.
5. Hot Reload safety: Do not cache Subsystem pointers or retain engine objects in static variables.
6. Soft references: Use TSoftObjectPtr/TSoftClassPtr for large assets.

See [references/shipping_checklist.md](references/shipping_checklist.md).

## Required Report Format

Organize review results consistently:

```text
# [Filename] Review Report

> Review scope / User request / Review basis

## Issue Overview

| # | Severity | Issue | Location | Rule |
|---|----------|-------|----------|------|
(List every issue, one per row, in P0 -> P1 -> P2 order.)

## Detailed Analysis

### [P0-1] Issue Title
**Location:** file:line
**Problem:** Specific description
**Fix:** Code example
**Rule #N:** Corresponding rule name

(Group by severity: all P0 issues first, then P1, then P2.)

## Fix Priorities
(Numbered list in the recommended order of implementation.)
```

The overview table is mandatory so the user can see the full picture immediately. Each issue in the detailed analysis must cite a rule number from the 25-rule table below so recommendations can be traced to their basis.

## 25-Rule Quick Reference

Rules are ordered by severity for quick issue identification. Detailed explanations and examples are in the linked references.

### P0: Must Fix (Crashes, Leaks, Data Loss)

| # | Rule | Summary | Reference |
|---|------|---------|-----------|
| 1 | Required UPROPERTY | Annotate UObject pointer members and containers with UPROPERTY() so GC tracks them, including TMap/TSet | [GC](references/safety_patterns.md#gc) |
| 2 | UObject creation | Use only NewObject/SpawnActor/CreateDefaultSubobject; never `new` | [GC](references/safety_patterns.md#gc) |
| 3 | Assertion selection | checkf is removed in Shipping; use ensureMsgf on runtime paths | [Assertions](references/safety_patterns.md#assert) |
| 4 | Side effects in check | Expressions inside check do not execute in Shipping | [Assertions](references/safety_patterns.md#assert) |
| 5 | IsValid() | Use IsValid() instead of a nullptr check for UObjects; the source notes PendingKill deprecation in UE 5.4+ | [GC](references/safety_patterns.md#gc) |

### P1: Important (Performance, Maintainability)

| # | Rule | Summary | Reference |
|---|------|---------|-----------|
| 6 | Stats profiling | Subsystem hot functions require SCOPE_CYCLE_COUNTER | [Stats](references/performance_patterns.md#stats) |
| 7 | ParallelFor signature | Include TEXT("Name") and MinBatchSize | [ParallelFor](references/performance_patterns.md#parallelfor) |
| 8 | Disable Tick by default | Set bCanEverTick = false; cache references in BeginPlay | [Tick](references/safety_patterns.md#tick) |
| 9 | Hot-path heap allocation | Reuse members instead of allocating TArrays every frame | [Hot paths](references/performance_patterns.md#hotpath) |
| 10 | TInlineAllocator | It creates a different TArray type; references are not interchangeable | [ParallelFor](references/performance_patterns.md#parallelfor) |
| 11 | Delegate selection | Avoid Dynamic when Blueprint exposure is unnecessary | [Delegates](references/api_patterns.md#delegate) |
| 12 | Custom log categories | Do not use LogTemp for every log | [Logging](references/api_patterns.md#log) |
| 13 | Defensive programming | Expose unexpected null pointers with check; do not silently skip them | [Defensive programming](references/safety_patterns.md#defensive) |
| 14 | Array safety | Call IsValidIndex() before indexing | [Defensive programming](references/safety_patterns.md#defensive) |
| 15 | Container selection | Use spatial-prime hashing rather than TMap for spatial grids under the original rule set | [Containers](references/performance_patterns.md#container) |
| 16 | API exports | Default to no exports; broaden as needed through MinimalAPI, selective, and full exports | [Exports](references/api_patterns.md#export) |
| 17 | BlueprintPure | Add BlueprintPure = false to expensive const functions | [BlueprintPure](references/api_patterns.md#blueprintpure) |

### P2: Improvements (Editor Experience, Code Quality)

| # | Rule | Summary | Reference |
|---|------|---------|-----------|
| 18 | UPROPERTY UIMin/UIMax | Provide slider ranges for numeric properties | [UPROPERTY](references/api_patterns.md#uproperty) |
| 19 | ForceUnits | Show units for distance, speed, and time properties | [UPROPERTY](references/api_patterns.md#uproperty) |
| 20 | EditCondition | Conditionally display child properties controlled by a bool | [UPROPERTY](references/api_patterns.md#uproperty) |
| 21 | Subsystem selection | Use WorldSubsystem instead of an Actor singleton | [Subsystems](references/api_patterns.md#subsystem) |
| 22 | CVar debugging parameters | Use TAutoConsoleVariable for runtime-tunable parameters | [CVars](references/api_patterns.md#cvar) |
| 23 | Include order | Alphabetical order; .generated.h last | [Includes](references/shipping_checklist.md#include) |
| 24 | Soft references | Use TSoftObjectPtr for large assets to avoid loading chains | [Soft references](references/shipping_checklist.md#softref) |
| 25 | Redundant factories | Return {} when the result is identical to default construction | [Hot paths](references/performance_patterns.md#hotpath) |

## Learning Protocol

This skill supports continued learning. After each review or optimization task, record new findings in [references/learnings.md](references/learnings.md) so knowledge can accumulate across sessions.

### Before a Task

1. Read `references/learnings.md` and apply relevant high-confidence entries.
2. Check for historical entries in categories relevant to the current task.

### Capture Compilation Errors

After a UE C++ compilation failure has been fixed:

1. Determine whether the error is worth recording. Any one of these conditions is sufficient:
   - UE-specific errors: UHT, reflection, module linking, or platform differences.
   - The fix required more than one attempt.
   - Implicit UE API behavior or version differences were involved.
   - A type-system trap was involved, such as incompatible TInlineAllocator types or untracked GC references caused by missing UPROPERTY.
2. If worth recording, show the distilled rule and ask the user:

   ```text
   Reusable compilation lesson found:
   - Observation: [Anonymized error description]
   - Rule: [One reusable sentence]
   - Category: [compilation / ue-api / ...]
   Add this to the skill's learning record?
   ```

3. After confirmation, anonymize the entry and append it to `references/learnings.md`.
4. Skip low-value errors such as simple typos, missing semicolons, and ordinary missing includes.

### After a Task

1. Review the problems and discoveries from the task.
2. Select new knowledge that is **reusable and independent of the project**, excluding private information such as project names and paths.
3. Append entries using the required format.
4. Skip any rule already covered by an existing reference; do not duplicate it.

### Promotion

Promote a learning entry into a formal rule when it has `high` confidence and has been validated in at least two distinct scenarios. Add it to the corresponding reference and mark the original entry `[Promoted]`.

### Maintenance

- When there are more than 50 entries, review and remove outdated or already-promoted entries.
- Keep entries specific and actionable; reject vague descriptions.

See [references/learnings.md](references/learnings.md) for categories and formatting.

## Reference Navigation

| File | Contents | When to read |
|------|----------|--------------|
| [performance_patterns.md](references/performance_patterns.md) | Stats, ParallelFor, containers, spatial hashing, hot paths | Performance review (Step 3) |
| [safety_patterns.md](references/safety_patterns.md) | Assertions, GC, Tick, defensive programming, IsValid, Shipping | Safety review (Step 2) |
| [api_patterns.md](references/api_patterns.md) | UPROPERTY metadata, delegates, subsystems, exports, CVars, logs, BlueprintPure | API review (Step 4) |
| [shipping_checklist.md](references/shipping_checklist.md) | FAB, cross-platform support, loading phases, Hot Reload, soft references, serialization, includes | Release validation (Step 5) |
| [epic_coding_standard.md](references/epic_coding_standard.md) | Naming, types, headers, Blueprint exposure | Structural assessment (Step 1) |
| [common_issues.md](references/common_issues.md) | Compilation warnings, TInlineAllocator, spatial data structures, reflection, thread safety | Bug fixes |
| [cross_version_compat.md](references/cross_version_compat.md) | UE 5.3-5.7 compatibility notes | Cross-version work |
| [learnings.md](references/learnings.md) | Accumulated learning records and promotion protocol | At the beginning of every task |
