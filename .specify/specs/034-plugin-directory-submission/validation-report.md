---
feature: Plugin Directory Submission Fixes
validated: 2026-06-01
validator: Claude
status: PASS
score: 6
score_max: 6
scoring_basis: FR acceptance criteria (proportionate; full 110-pt council intentionally not run for a verified build-tooling generator fix)
has_ui: false
deploy_in_scope: false
blast_radius_verdict: CONTAINED
classification: non-application (build tooling)
---

# Validation Report: Plugin Directory Submission Fixes

## Scope & Method

Non-application build-tooling change to two generator scripts + eight canonical
command sources, propagated to all four CLI surfaces by deterministic
regeneration. Per the proportionate-validation directive, this report scores the
**6 functional requirements against real executed-command evidence** rather than
running the full 11-agent / 110-point council (which targets feature code with
test suites and is disproportionate here). All evidence below is from commands
executed in this session.

## FR Acceptance — Evidence Table

| FR | Requirement | Result | Evidence (executed) |
|----|-------------|--------|---------------------|
| FR-1 | 8 control commands emit frontmatter+description | ✅ PASS | All 8 `plugins/eai-gofer/commands/gofer_*.md` start with `---` + `description:`; `claude plugin validate` reports **0** "No frontmatter" warnings (was 8) |
| FR-2 | SKILL.md single frontmatter block | ✅ PASS | awk double-frontmatter detector across all `plugins/eai-gofer/skills/*/SKILL.md` → **0** files with a second leading block; top-of-file inspection shows one `---name/description---` block then body |
| FR-3 | Marketplace top-level `description` removed | ✅ PASS | `claude plugin validate` on root + nested `.claude-plugin/marketplace.json` → **0 errors** (was `Unrecognized key: "description"`); `metadata.description` retained/added |
| FR-4 | `category`/`tags` removed from plugin.json | ✅ PASS | `claude plugin validate` on root + nested `.claude-plugin/plugin.json` → **0 warnings** (was 2: category, tags) |
| FR-5 | LICENSE shipped inside installable plugin | ✅ PASS | `test -f plugins/eai-gofer/LICENSE` → present (49 lines); packager `assertNoPersonalPaths` passed |
| FR-6 | Selective `${CLAUDE_PLUGIN_ROOT}` rewrite | ✅ PASS | `plugins/eai-gofer/commands` + `.claude/commands`: **44** `.specify/scripts` + **16** `.specify/templates` refs rewritten to `${CLAUDE_PLUGIN_ROOT}`; **65** specs + **18** memory + **16** logs refs left workspace-relative |

**Final manifest validation (all four, after regeneration):**

```
.claude-plugin/marketplace.json                   errors=0 warnings=0  ✔ passed
.claude-plugin/plugin.json                        errors=0 warnings=0  ✔ passed
plugins/eai-gofer/.claude-plugin/marketplace.json errors=0 warnings=0  ✔ passed
plugins/eai-gofer/.claude-plugin/plugin.json      errors=0 warnings=0  ✔ passed
```

Before this work: 1 marketplace **error** + 9 warnings. After: **0 errors, 0 warnings.**

## Automated Checks

| Check | Command | Result |
|-------|---------|--------|
| Generator syntax | `node --check` on both `.mjs` | PASS (both OK) |
| Generate surfaces | `npm run gofer:generate` | PASS (24 commands × all surfaces emitted) |
| Package + sync repo | `npm run gofer:package-plugin -- --sync-repo` | PASS (staged, zipped, synced; no personal paths) |
| Plugin validation | `claude plugin validate` ×4 manifests | PASS (0/0) |

## Blast Radius (Phase B) — CONTAINED

| Dimension | Finding | Verdict |
|-----------|---------|---------|
| Change graph / ripple | Hand-edits confined to 2 generators + 8 canonical command sources; everything else is deterministic regenerated output | CONTAINED |
| Interface contracts | No public API/signature changes; manifest schema now *more* compliant | CONTAINED |
| Out-of-scope surfaces | Gemini (`{{include}}`) and Copilot (inlined) verified **0** `${CLAUDE_PLUGIN_ROOT}` occurrences — rewrite correctly scoped to Claude surface | CONTAINED |
| Workspace-runtime paths | specs/memory/logs refs (99 total) verified unchanged — no risk of writing into read-only install dir | CONTAINED |
| Dependency / CVE | No dependency changes | CONTAINED |
| Rollback | `git checkout` of 10 source files + re-run `gofer:generate`/`package-plugin` deterministically restores prior state | CONTAINED |

## Fix Sites (final)

- `generate-commands.mjs`: `rewriteShippedAssetPaths()` helper; `emitClaude()` rewrite; `buildSkillContent()` strip-leading-frontmatter + rewrite.
- `package-agent-plugin.mjs`: `buildStageSkill()` strip + rewrite; `buildRepoMarketplace()` + `buildLocalMarketplace()` dropped top-level `description` (added `metadata.description` to local); `buildPluginManifest()` + `buildClaudeManifest()` dropped `category`/`tags`; `writePluginFolder()` `copiedResources` gained `LICENSE`.
- 8 × `.specify/commands/gofer_*.md`: embedded `---description---` block.

**Corrections discovered during implementation (beyond original plan):**
`buildClaudeManifest` (not just `buildPluginManifest`) carried category/tags;
`buildLocalMarketplace` (not `buildCodexLocalMarketplace`) was the second
top-level-description offender and the source of the nested
`plugins/eai-gofer/.claude-plugin/marketplace.json`; `buildStageSkill` (not just
`buildSkillContent`) builds the installable skills and needed the FR-2/FR-6 fix.

## Verdict

**PASS — 6/6 FRs satisfied with executed evidence.** Plugin validates clean
(0 errors / 0 warnings) across all four manifests; portable shipped-asset paths;
LICENSE shipped; all four CLI surfaces regenerated consistently.

**Submission readiness:** The three mandatory (🔴) items from
PLUGIN-DIRECTORY-REQUIREMENTS.md are resolved, plus the 🟡 frontmatter/category
polish. Ready for community-marketplace submission
(`clau.de/plugin-directory-submission`). Remaining optional polish (🟢, not
blocking): prune duplicate non-`.claude-plugin` manifests; strip non-Claude
mirror dirs for a Claude-only directory listing.
