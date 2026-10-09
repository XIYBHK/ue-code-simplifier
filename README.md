# ue-code-simplifier

An English, standalone Claude Code skill for reviewing, simplifying, and optimizing **existing Unreal Engine C++ plugin code**. Extracted from [XIYBHK/claude-skills](https://github.com/XIYBHK/claude-skills), with all eight supporting references included.

The skill guides an AI coding agent through structure, safety, performance, API quality, and release readiness. It is a set of instructions and reference notes; it does not install an Unreal plugin or run a compiler by itself.

## Scope

- Review recent `.h` / `.cpp`, `.uplugin`, and `Build.cs` changes, or a wider scope when explicitly requested.
- Preserve runtime behavior and Blueprint interfaces while simplifying implementation.
- Check UObject/GC safety, assertions, Tick, thread safety, containers, hot paths, delegates, metadata, subsystems, and exports.
- Produce a severity-ranked report with 25 numbered rules and traceable references.
- Consult historical UE 5.3-5.7 compatibility notes and FAB/Marketplace readiness guidance.
- Maintain anonymized learning records, including the original confirmation step for recording compilation lessons.

The original trigger boundaries exclude creating new UE code from scratch, Blueprint-only tasks, shaders/HLSL, packaging/deployment troubleshooting, editor crash debugging, and whole-project engine-version migration. Compatibility review of existing plugin code remains within scope.

## Install in Claude Code

Requirements: Claude Code with skill support, Git if cloning, and a UE C++ plugin project for review. There are no additional scripts, packages, MCP servers, or sibling skills required by this skill.

Clone the repository as a single skill directory. **Do not clone it over an existing skill or an existing skills collection.** If a skill with this name is already installed, review Claude Code's name-precedence rules before selecting a location. These commands stop if the destination already exists.

### Personal installation: Windows PowerShell

```powershell
$skillPath = Join-Path $env:USERPROFILE '.claude\skills\ue-code-simplifier'
if (Test-Path -LiteralPath $skillPath) {
    throw 'The skill directory already exists. Choose another location or review the existing installation first.'
}
New-Item -ItemType Directory -Path (Split-Path -Parent $skillPath) -Force | Out-Null
git clone https://github.com/XIYBHK/ue-code-simplifier.git $skillPath
```

### Personal installation: macOS or Linux

```bash
skill_path="$HOME/.claude/skills/ue-code-simplifier"
if [ -e "$skill_path" ]; then
  printf '%s\n' 'The skill directory already exists; review it before installing.' >&2
else
  mkdir -p "$HOME/.claude/skills"
  git clone https://github.com/XIYBHK/ue-code-simplifier.git "$skill_path"
fi
```

For a project installation, place this repository at `<project>/.claude/skills/ue-code-simplifier/` instead. For manual installation, copy the repository's files (excluding `.git`) into that skill directory, including `SKILL.md`, the entire `references/` directory, `VALIDATION.md`, `AUDIT-RESOLUTION.md`, and the license/attribution/provenance files. Keep all relative paths intact.

Claude Code discovers skills in its personal and project skill directories and supports direct `/skill-name` invocation. See the [official Claude Code skills documentation](https://code.claude.com/docs/en/skills) for discovery, same-name precedence, and troubleshooting.

## Use

From your plugin's project in Claude Code:

```text
/ue-code-simplifier Review the C++ plugin files changed in this session.
Preserve behavior and Blueprint signatures. Report findings before editing.
```

Other examples:

```text
/ue-code-simplifier Review the hot path in Source/MyPlugin/Private/MySystem.cpp.
Focus on allocations, ParallelFor, containers, and profiling.
```

```text
/ue-code-simplifier Check UObject ownership and GC safety in this plugin.
```

```text
/ue-code-simplifier Review this plugin's FAB submission readiness.
Report which build and runtime checks still need to be performed.
```

Claude may also select the skill automatically when the request matches its description. Actual edits and tool execution depend on the coding agent's permissions and your request. The default scope is code changed in the current session.

The report contains an issue overview, detailed findings citing rule numbers, and an ordered fix list. References load progressively by task. `references/learnings.md` is read at task start and may be updated during later reviews; inspect those changes before pulling updates or publishing a fork.

## Files

```text
ue-code-simplifier/
|-- SKILL.md
|-- README.md
|-- VALIDATION.md
|-- AUDIT-RESOLUTION.md
|-- LICENSE
|-- NOTICE.md
|-- PROVENANCE.md
|-- .gitattributes
|-- .gitignore
`-- references/
    |-- api_patterns.md
    |-- common_issues.md
    |-- cross_version_compat.md
    |-- epic_coding_standard.md
    |-- learnings.md
    |-- performance_patterns.md
    |-- safety_patterns.md
    `-- shipping_checklist.md
```

## Version Scope and Validation Limits

The initial English extraction preserved the source's historical examples. A technical correction on 2026-10-09 addressed 40 audit findings and reconciled all nine knowledge files, 25 rule IDs and 15 historical learning entries. See [AUDIT-RESOLUTION.md](AUDIT-RESOLUTION.md) for each disposition.

Selected declarations and implementations were inspected in installed UE 5.3.2, 5.4.4, 5.5.4, 5.6.1 and 5.7.4. Version-specific boundaries, official sources and source locations are documented in [VALIDATION.md](VALIDATION.md). These checks do not certify a consuming plugin against every configuration or fork.

**No Unreal compilation/UHT, editor or game execution, performance benchmark, asset cooking, Hot Reload/Live Coding test, serialization runtime test, cross-platform build, or Fab submission was performed.** GCC probes checked standard C++ semantics and Clang separately preprocessed the compatibility macros against actual engine version headers; neither was a UE plugin build. This skill repository contains no Unreal build CI.

Examples require their stated class/function/reflection and module context; they are not independently compiled plugins. Historical confidence labels describe the original records and are now accompanied by evidence-status notes. Before applying changes, verify the consuming engine and contract, build/test the supported targets, and check the full current product-specific Fab policy. Neither source inspection nor successful compilation establishes every runtime/performance/submission claim.

## Source and License

Original skill author and maintainer: **XIYBHK**. The source collection's README declares **MIT License**. This extraction retains that declaration in [LICENSE](LICENSE); it does not adopt a new license for the source.

See [PROVENANCE.md](PROVENANCE.md) for the source commit, extraction details, and hashes. See [NOTICE.md](NOTICE.md) for the source's Epic and community attributions and the scope of the MIT declaration. The original collection is preserved separately.
