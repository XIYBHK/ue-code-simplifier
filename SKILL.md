---
name: ue-code-simplifier
description: Review and optimize existing Unreal Engine C++ plugin code. Use for UE/UE5/Unreal C++ code review, checking, optimization, refactoring, simplification, performance, quality, memory, GC, Tick, container selection, or FAB/Marketplace submission readiness. Also use for UE C++ performance problems (TMap, spatial queries, hot paths, frame rate), memory/GC/safety concerns (UPROPERTY, check/ensure, IsValid), compilation warnings (C5038, TInlineAllocator), and preparing .h/.cpp files for FAB/Marketplace submission. Do not use for writing new UE code from scratch, Blueprint-only tasks, HLSL/shaders, packaging/deployment troubleshooting, editor crash debugging, or project migration between UE versions.
---

# UE C++ Plugin Review and Optimization

Review existing plugin code for structure, correctness, safety, measured performance, API quality, and release readiness. Use the actual project, engine version, target, module, and user request to establish whether a rule applies. The source-check scope and unperformed tests are documented in [VALIDATION.md](VALIDATION.md); [AUDIT-RESOLUTION.md](AUDIT-RESOLUTION.md) tracks the technical corrections to the translated edition.

## Core Principles

- Preserve functionality, observable ordering, ownership, failure behavior, and timing contracts. An implementation change is not equivalent merely because it compiles.
- Keep Blueprint/public interfaces stable unless the user requests a change and the required migration is handled.
- Focus on code changed during the current session by default; expand scope when explicitly requested.
- Confirm evidence against actual declarations/configuration and current official policy. Historical confidence and a generic example are not proof for this project.
- Fix bugs with minimal changes; avoid incidental refactoring and unmeasured performance promises.

## Avoid Excessive Simplification

Prefer clear explicit code over brevity. Keep meaningful abstractions and comments explaining decisions. Do not merge unrelated logic or introduce complex template machinery just to reduce line count. Preserve extensibility and compatibility unless a measured/user-requested change justifies the tradeoff.

## Task Routing

Read [references/learnings.md](references/learnings.md) first, including its evidence-status limitations. Load the focused references progressively; do not load every reference for an unrelated narrow task. For version claims, check [VALIDATION.md](VALIDATION.md) and the consuming engine's declarations/build definitions.

| Intent | Focus | References |
|--------|-------|------------|
| Code review | Steps 1-5 as applicable | Read references when their step begins |
| Performance / too slow | Step 3 | [performance](references/performance_patterns.md) |
| Release / Marketplace | Steps 2 and 5 | [safety](references/safety_patterns.md), [shipping](references/shipping_checklist.md) |
| Simplify / refactor | Steps 1 and 4 | [API](references/api_patterns.md), [conventions](references/epic_coding_standard.md) |
| Bug / compiler error | Targeted fix | [common issues](references/common_issues.md) |
| Blueprint API / UPROPERTY | Step 4 | [API](references/api_patterns.md) |
| GC / ownership | Step 2 | [safety](references/safety_patterns.md) |
| Version compatibility | Focused review | [compatibility](references/cross_version_compat.md) |

## Review Workflow

### Step 1: Establish Context and Structure

Determine the supported engine/patch, platform, target/configuration, module types, and public contracts before assigning severity. Review Runtime/Editor separation, Public/Private exposure, `.uplugin`/Build.cs dependencies and loading phases. Check specific includes, .cpp matching-header-first, reflected .h generated-include-last, and naming conventions. See [conventions](references/epic_coding_standard.md).

### Step 2: Review Safety and Ownership

Check explicit runtime validation/fallback paths separately from optional assertions. Establish strong/weak ownership, reflected/reference-collector reachability, creation and teardown. `IsValid` is an object-flags check, not protection for freed memory. Distinguish Tick capability from initial enabled state, preserve timing, and invalidate cached references appropriately. Default UObject API work to the GameThread unless a specific supported API plus lifetime/synchronization evidence permits an exception. See [safety](references/safety_patterns.md).

### Step 3: Review Measured Performance

Use profiling suited to the configuration/workload. Legal ParallelFor overloads do not require explicit tuning parameters; choose batching/flags from evidence. Prevent shared scratch-buffer races, including reentrancy across frames. Choose containers and key hashes separately; TMap is a valid spatial candidate. Check capacity, element lifetime/copy/zero traits, ordering, callback side effects, and removal costs before optimizing. See [performance](references/performance_patterns.md).

### Step 4: Review API Quality

Review editor metadata without treating it as runtime validation or completed localization. Choose delegates for reflection/serialization/Blueprint and lifetime needs; preserve other UFUNCTION uses. Select subsystem ownership from the actual World/GameInstance/local-player lifecycle. Match export symbols and C++ access to public contracts. Analyze actual Blueprint pure evaluation and any requested pin migration. Check log-category linkage and CVar thread/configuration needs. See [API](references/api_patterns.md).

### Step 5: Review Release Evidence

Build the targets/configurations/platforms applicable to the module and declared support, and report what was actually tested. Check the current product-specific Fab policy; the reference is a non-exhaustive preflight. Select loading phases from dependencies/events, distinguish reload/reinstancing/World cleanup, and verify soft loading/cook/retention and serialization/config migrations. See [shipping](references/shipping_checklist.md) and [compatibility](references/cross_version_compat.md).

## Report Format and Severity

Report real problems separately from team-style suggestions, tuning hypotheses, and unverified assumptions. Assign P0 only for an established critical crash/data-loss/security/lifetime risk in its actual reachable context; Editor crashes can also be critical. P1 covers important defects or substantiated risks; P2 covers lower-impact improvements. A rule violation without its applicability/trigger is not enough to assign severity.

```text
# [Scope] Review Report
> Request, engine/patch, module/target, evidence and verification limits

## Issue Overview
| ID | Severity | Finding | Location | Applicable rule / other evidence |
|----|----------|---------|----------|----------------------------------|

## Details
- Observed problem and trigger
- Evidence, version/configuration, and uncertainty
- Minimal proposed fix preserving the relevant contract
- Applicable rule number, or a separate specific basis if not in the table

## Validation and Priorities
- Checks actually run and results
- Checks not run, remaining risks, and ordered fixes
```

Use an overview table and trace findings to applicable rules/evidence. Do not force an unrelated rule number onto an issue outside the 25-item table. A compile pass does not certify runtime, Blueprint, performance, or submission behavior.

## 25-Rule Quick Reference

Numbers remain stable for existing reports. Priority depends on a real trigger and context, not solely on placement in this table.

| # | Review rule | Applicability / limit | Reference |
|---|-------------|-----------------------|-----------|
| 1 | GC ownership | Reflected strong fields need a reachable reference chain; valid collector/strong/weak alternatives exist | [GC](references/safety_patterns.md#gc) |
| 2 | UObject creation | Use the appropriate construction/spawn/loading API; no ordinary new/delete UObject lifetime | [GC](references/safety_patterns.md#gc) |
| 3 | Runtime error handling | Explicit validation/fallback in every required target; ensure is optional diagnostic, not a Shipping logging guarantee | [Assertions](references/safety_patterns.md#assert) |
| 4 | Required side effects | Keep required work out of check/optional preprocessor paths; verify executes its expression but does not recover | [Assertions](references/safety_patterns.md#assert) |
| 5 | UObject validity | IsValid checks flags/null within a valid lifetime; not for generic pointers or already-freed memory | [GC](references/safety_patterns.md#gc) |
| 6 | Profiling | Measure meaningful scopes with appropriate Stats/trace configuration; no mandatory macro everywhere | [Stats](references/performance_patterns.md#stats) |
| 7 | ParallelFor | Legal simple/named overloads; tune batch/flags only when workload and latency justify them | [ParallelFor](references/performance_patterns.md#parallelfor) |
| 8 | Tick | Separate can-ever-tick from starts-enabled; changes to frequency/caches must preserve contracts | [Tick](references/safety_patterns.md#tick) |
| 9 | Scratch allocation | Reuse only at the right reset boundary with exclusive/per-task/partitioned ownership | [Hot paths](references/performance_patterns.md#hotpath) |
| 10 | TInlineAllocator | Allocator specializations are distinct types; mutable references are not interchangeable | [ParallelFor](references/performance_patterns.md#parallelfor) |
| 11 | Delegates | Select for reflection/serialization/Blueprint and lifetime; use actual binding APIs | [Delegates](references/api_patterns.md#delegate) |
| 12 | Logging | Useful module categories with deliberate cross-DLL exports and target logging configuration | [Logging](references/api_patterns.md#log) |
| 13 | Null/error paths | Expected absence versus invariant violation; diagnose and retain required failure control flow | [Defensive](references/safety_patterns.md#defensive) |
| 14 | Array access | Prove/check indices and element lifetime; guard does not solve concurrent structural mutation | [Defensive](references/safety_patterns.md#defensive) |
| 15 | Containers/hashes | Choose from actual density/access/memory needs; TMap and custom hashing can coexist | [Containers](references/performance_patterns.md#container) |
| 16 | Exports | Match needed symbols, visibility and ABI; MinimalAPI is not merely Cast and does not change access control | [Exports](references/api_patterns.md#export) |
| 17 | BlueprintPure | Const auto-pure has conditions; inspect graph execution and preserve/migrate existing pins deliberately | [Pure](references/api_patterns.md#blueprintpure) |
| 18 | Numeric UI ranges | Optional editor ranges matched to business/sentinel semantics; not runtime serialization enforcement | [Metadata](references/api_patterns.md#uproperty) |
| 19 | Units | Useful display metadata; actual calculation/conversion still needs validation | [Metadata](references/api_patterns.md#uproperty) |
| 20 | Edit conditions | EditCondition normally enables/disables; Hides requests hiding | [Metadata](references/api_patterns.md#uproperty) |
| 21 | Subsystems | Select owner/world types and lifetime; UWorld is not each streamed Level or a guaranteed process singleton | [Subsystems](references/api_patterns.md#subsystem) |
| 22 | CVars | Suitable debug/tuning configuration with correct thread access; not universal replacement for UPROPERTY settings | [CVars](references/api_patterns.md#cvar) |
| 23 | Includes | .cpp matching header first; generated include last in its reflected .h; specific IWYU dependencies | [Includes](references/shipping_checklist.md#include) |
| 24 | Asset references | Soft versus hard by contract; class/object types, load/retain/cook/failure paths must be correct | [Soft refs](references/shipping_checklist.md#softref) |
| 25 | Default-equivalent factories | Prove all defaults/side effects equivalent; retain public API unless migration is requested | [Hot paths](references/performance_patterns.md#hotpath) |

## Learning Protocol

Read [learnings](references/learnings.md) at task start and apply only relevant, qualified rules. Historical confidence labels are inherited, not new test certificates.

After a reusable UE-specific compiler/UHT/linker/platform or type-system problem is fixed, present the anonymized observation, evidence/version, reusable conditional rule and category, and ask whether to add it. After confirmation, append it without private project names/paths. Ordinary typos and missing semicolons usually do not merit a record.

After other review/optimization work, record reusable findings with actual evidence, conditions and test limits when authorized within the task. Do not duplicate existing rules or promote an assumption into a fact. Do not treat read-only/audit-only work as authorization to edit learning records.

Promote a high-confidence entry only after validation in at least two distinct relevant scenarios, recording those scenarios and resolving conflicting rules. When records grow beyond 50 entries, review outdated/promoted material. Keep specific actionable conditions and distinguish source inspection from compilation/runtime/performance results.

## Reference Navigation

| File | Contents |
|------|----------|
| [performance](references/performance_patterns.md) | Profiling, parallel ownership, containers, copy/zero/removal semantics |
| [safety](references/safety_patterns.md) | Assertions, GC, lifetime, Tick, error paths and threads |
| [API](references/api_patterns.md) | Metadata, delegates, subsystem owners, exports, logs, CVars, pure evaluation |
| [shipping](references/shipping_checklist.md) | Product preflight, platform guards, loading, reload, assets, serialization |
| [conventions](references/epic_coding_standard.md) | Naming, headers, type selection, localization and interface conventions |
| [common issues](references/common_issues.md) | Diagnostic examples with explicit contexts and valid fixes |
| [compatibility](references/cross_version_compat.md) | Qualified UE5.3-5.7 declarations, versions and toolchains |
| [learnings](references/learnings.md) | Historical records with corrected rules/evidence status |
