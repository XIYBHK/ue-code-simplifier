# Provenance

## Source Snapshot

- Source repository: [XIYBHK/claude-skills](https://github.com/XIYBHK/claude-skills).
- Source directory: [`ue-code-simplifier/`](https://github.com/XIYBHK/claude-skills/tree/57fafa04ebea018b31ceecd84c0ba46508441688/ue-code-simplifier).
- Source repository HEAD: `57fafa04ebea018b31ceecd84c0ba46508441688`.
- Most recent committed change to the skill: `04a374ea74e3134f8a7c55fb56358297020b1e97`, dated 2026-07-29.
- Original author: XIYBHK, as declared in the source README and recorded in the skill's Git history.
- Original license declaration: [MIT License in the source README](https://github.com/XIYBHK/claude-skills/blob/57fafa04ebea018b31ceecd84c0ba46508441688/README.md#%E8%AE%B8%E5%8F%AF%E8%AF%81).

The local source snapshot additionally contained four uncommitted learning entries dated 2026-08-27: config migration, GConfig APIs, plugin build/packaging practices, and replacing faulty predicates. They are translated in `references/learnings.md`. This edition therefore represents the local working snapshot, not only the publicly committed base. The original collection and its local changes were left intact.

## Extraction Changes

- Translated SKILL.md, all eight reference files, comments, example UI labels, and report/learning templates into English.
- Normalized the frontmatter name from the source's bilingual display name to the valid identifier `ue-code-simplifier`, matching the directory and invocation name.
- Preserved the review workflow, trigger boundaries, 25 numbered rules, learning protocol, historical dates, confidence labels, and C++ example implementation.
- Converted local reference mentions into navigable links and added stable anchors where the source's fragment names had no corresponding anchors.
- Anonymized concrete project/module names in historical learning descriptions while preserving their technical lessons.
- Added standalone installation/use documentation, provenance, attribution, the standard text for the existing MIT declaration, and explicit validation limitations.
- Added contextual notes for source inconsistencies without silently changing C++ example bodies.

## Technical Correction (2026-10-09)

The extraction bullets above describe the initial standalone commit. After the user authorized correction and publication, the standalone project revised its technical rules and examples rather than retaining known source errors. All 40 audit IDs are tracked in [AUDIT-RESOLUTION.md](AUDIT-RESOLUTION.md); source/version evidence and test limits are in [VALIDATION.md](VALIDATION.md).

The review purpose, trigger boundaries, eight supporting references, 25 stable rule IDs, original historical dates, MIT declaration and attribution remain. Historical rules/confidence are distinguished from new evidence; affected rules/resolutions have qualification notes. This is a corrected derivative of the source, not a claim that the original files or input hashes changed. No edits were made to the source collection.

## Original File Hashes

SHA-256 hashes below identify the original local input bytes before translation. Paths are relative to the source skill directory. The learning-record hash includes the four local additions described above.

| Source file | SHA-256 |
|-------------|---------|
| `SKILL.md` | `2f5dbb1dab098e1cda028bb3c9d75174ad5e023f7578407e2d9e6bc51ed2a25f` |
| `references/api_patterns.md` | `dc59c1e546902cd3200763ebd60ba725bef945933a958e6e30e545a3e54be288` |
| `references/common_issues.md` | `b90ee81a0e15506b2c86fff86efb4e2957f65f1dd11e460cf48c46f42cf904fd` |
| `references/cross_version_compat.md` | `ccb53f90295998ed9c99f95702755d1df0d13a29b3e429ab96749e424e66855d` |
| `references/epic_coding_standard.md` | `8d569feac8bb14fe35b7167e64ec4e5ea282860d9ecdf924a28a17af816dd129` |
| `references/learnings.md` | `1c666ba144a27c7ac915f094507433c0c0eba05a9e02bc7c07ce70ad8f94133e` |
| `references/performance_patterns.md` | `627ec6acb7dcce8cedf46bf70e3895b90cf5185e56b87e2624d0f1503d301c66` |
| `references/safety_patterns.md` | `4f9201e5a4e0a968c8e08deca0e1097bf6de9fda035efb8c845f991ae36c22bd` |
| `references/shipping_checklist.md` | `67d4bf78b02bb21044f322bfdf536d8946d7ef44aee2bb33bf0c1b40dd3cbfd7` |
