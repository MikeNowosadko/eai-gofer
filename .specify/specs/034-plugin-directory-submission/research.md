---
date: 2026-06-01
researcher: Claude
feature: 'Plugin Directory Submission Fixes'
status: complete
classification: non-application (build tooling / code hygiene)
---

# Research: Plugin Directory Submission Fixes

## Feature Summary

Make the eai-gofer plugin pass Anthropic's community plugin marketplace
submission bar (clau.de/plugin-directory-submission) across all four generated
CLI surfaces (Claude, Codex, Gemini, Copilot). All fixes must be applied to the
**canonical source + generator scripts** so regeneration propagates them to every
surface — never hand-edited into generated output.

## Structured Discovery Output

### Problem Statement

- **Problem**: Plugin fails `claude plugin validate` (marketplace schema error),
  ships no LICENSE inside the installable plugin, has 8 commands with no
  frontmatter/description, leaks double-frontmatter into every SKILL.md, and
  references `.specify/...` assets by bare relative path so they break once
  installed in a stranger's repo.
- **Current State Friction**: Works only because the gofer repo itself contains a
  `.specify` scaffold; not portable to a fresh install.
- **Desired Outcome**: `claude plugin validate` returns 0 errors; the four
  surfaces regenerate cleanly; LICENSE present; shipped asset paths portable.

### Target Persona

- **Primary Persona**: Plugin maintainer (Michael) + Anthropic automated review.
- **Skill Level**: Advanced.
- **Top Needs**: Source-driven fixes, no regressions across surfaces, clean validation.

### Value Proposition

- **Primary Value**: Compliance — clears stage-two community marketplace submission.
- **Measurable Goal**: 0 validation errors; 60/60 shipped-asset refs portable; 4/4 surfaces regenerated; LICENSE shipped.

## Generation Pipeline (data flow)

```text
.specify/commands/*.md   (canonical source — hand-authored, 24 stage files)
        │   parseStageCommand(): strips OUTER stage frontmatter, body = remainder
        ▼
generate-commands.mjs   (npm run gofer:generate, step 1)
        ├─ emitClaude()        → .claude/commands/<stem>.md        (body verbatim)
        ├─ emitGithubPrompts() → .github/prompts/<stem>.prompt.md  (buildCopilotPromptContent)
        ├─ emitGemini()        → .gemini/commands/gofer/<stem>.{md,toml}
        ├─ emitAgentsSkills()  → .agents/skills/<stem>/SKILL.md    (buildSkillContent)
        ├─ emitSystemSkills()  → .system/skills/<stem>/SKILL.md    (buildSkillContent)
        └─ emitCodexConfig()   → .specify/outputs/codex-config-fragment.toml
        ▼
sync-extension-resources.mjs   (npm run gofer:generate, step 2)
        └─ copies surfaces into extension/resources/*, codex-config.toml
        ▼
package-agent-plugin.mjs --sync-repo   (npm run gofer:package-plugin)
        ├─ writePluginFolder()   → plugins/eai-gofer/{commands,skills,.specify,...}
        └─ syncRepoManifests()   → root plugin.json, .claude-plugin/{plugin,marketplace}.json,
                                    .github/plugin/*, .codex-plugin/plugin.json,
                                    .agents/plugins/marketplace.json
```

**Key fact:** every generated surface and every manifest is emitted by these
scripts. The descriptions live in each source file's frontmatter (the
`canonical-descriptions.mjs` map is validated for budget but the per-command
`description` is read from the file's own frontmatter).

## Codebase Analysis — Root Causes & Fix Locations

| # | Issue | Root cause (evidence) | Fix location |
| - | ----- | --------------------- | ------------ |
| 1 | 8 control commands have no Claude frontmatter → no description | `emitClaude()` writes `stage.body` verbatim. Pipeline commands embed a second `---description---` block at the top of their body (becomes Claude frontmatter); the 8 control commands' bodies have none. `generate-commands.mjs` `emitClaude()` | `generate-commands.mjs` `emitClaude()`: synthesize `---\ndescription: "..."\n---` from `stage.frontmatter.description` when body has no leading frontmatter (mirror `buildCopilotPromptContent`) |
| 2 | Every SKILL.md has double-frontmatter | `buildSkillContent()` prepends `---name/description---` then appends a body that already starts with an embedded `---description---` block | `generate-commands.mjs:~421` `buildSkillContent()`: strip leading frontmatter from `body` via existing `splitMarkdownFrontmatter()` before concatenating |
| 3 | marketplace.json invalid top-level `description` | Hardcoded string in `buildRepoMarketplace()` (and sibling marketplace builders) | `package-agent-plugin.mjs` `buildRepoMarketplace()` / `buildCodexLocalMarketplace()`: drop the top-level `description` (move to `metadata.description` if desired) |
| 4 | `category`/`tags` in plugin.json (validator warns) | Hardcoded in `buildPluginManifest()` | `package-agent-plugin.mjs` `buildPluginManifest()`: remove `category` + `tags` |
| 5 | LICENSE absent from installable plugin | `writePluginFolder()` `copiedResources[]` omits LICENSE | `package-agent-plugin.mjs` `writePluginFolder()`: add `'LICENSE'` to copied resources (source: repo-root `LICENSE`) |
| 6 | `.specify` asset refs not portable after install | No generator rewrites paths | `emitClaude()` + skill emitters: selectively rewrite shipped-asset refs to `${CLAUDE_PLUGIN_ROOT}` (see below) |
| 7 | Duplicate/non-standard manifests (root `plugin.json`, nested marketplace.json) | All emitted by `syncRepoManifests()` for multi-CLI; the non-`.claude-plugin` copies are for Codex/Copilot discovery | Decide in plan: keep multi-CLI copies but ensure the canonical `.claude-plugin/*` pair is correct and schema-clean |

## Critical Refinement to Locked Decision (Option A)

Locked decision was: repoint `.specify/...` references to `${CLAUDE_PLUGIN_ROOT}`.
Research shows this must be **selective**, not blanket. Reference counts in the
canonical command sources:

| `.specify` subpath | Refs | Nature | Action |
| ------------------ | ---- | ------ | ------ |
| `.specify/scripts/` | 44 | Plugin-shipped read-only asset | **Rewrite → `${CLAUDE_PLUGIN_ROOT}/.specify/scripts/`** |
| `.specify/templates/` | 16 | Plugin-shipped read-only asset | **Rewrite → `${CLAUDE_PLUGIN_ROOT}/.specify/templates/`** |
| `.specify/specs/` | 68 | Artifact output in user's workspace | Keep relative to CWD |
| `.specify/memory/` | 18 | Per-project config (constitution) | Keep relative to CWD |
| `.specify/logs/` | 16 | Runtime logs in user's workspace | Keep relative to CWD |

A blanket rewrite would corrupt 102 workspace-runtime references (artifacts would
try to write into the read-only installed plugin). Selective rewrite touches only
the 60 shipped-asset references. This realizes the *intent* of Option A
(portability of shipped assets) without breaking artifact writes.

## Reuse-Before-Create Scan

| Candidate | Existing Evidence | Decision | Rationale |
| --------- | ----------------- | -------- | --------- |
| Frontmatter synthesis | `buildCopilotPromptContent()` already synthesizes frontmatter | Reuse pattern | Apply same approach in `emitClaude()` |
| Frontmatter stripping | `splitMarkdownFrontmatter()` already exists | Reuse | Use in `buildSkillContent()` |
| Path rewriting | None exists | Create new | Minimal selective regex in emitters |
| LICENSE | repo-root `LICENSE` exists | Reuse | Just add to copied resources |
| Manifest builders | `buildRepoMarketplace`/`buildPluginManifest` exist | Extend | Edit returned objects |

No new dependencies. No new platform concepts.

## Brownfield Analysis

- **Constraint**: Surfaces are generated; never hand-edit generated files — they
  are overwritten on next `gofer:generate`/`package-plugin`.
- **Caution**: The embedded body-frontmatter pattern is *intentional* for Claude
  pipeline commands. Fix #2 (skills) strips it for SKILL.md output only; do not
  remove it from the source bodies of pipeline commands (emitClaude relies on it).
- **Legacy/orphaned**: `scripts/codex-skill-generator.js`,
  `scripts/copilot-prompt-enhancer.js` are NOT wired into `gofer:generate`; ignore.
- **Downstream**: After source/generator edits, must run `gofer:generate` then
  `gofer:package-plugin --sync-repo` to refresh all surfaces + `plugins/eai-gofer/`.

## Constraints & Considerations

- Validate after regeneration with `claude plugin validate` on both the
  marketplace manifest and the plugin manifest.
- Keep changes scoped to the fix list; no behavioral changes to pipeline logic.

## Open Questions

- [ ] Frontmatter for the 8 control commands: synthesize in `emitClaude()`
      (recommended, DRY) vs add embedded blocks to 8 source files? → recommend synthesize.
- [ ] Path rewrite scope: selective (scripts+templates) — recommended — confirm.
- [ ] Duplicate manifests: keep multi-CLI copies (recommended) or prune to
      `.claude-plugin/*` only for the submission?

## Recommendations

1. Apply all 6 fixes in `generate-commands.mjs` + `package-agent-plugin.mjs`.
2. Synthesize control-command frontmatter in `emitClaude()` (no source-file churn).
3. Selective `${CLAUDE_PLUGIN_ROOT}` rewrite (scripts + templates only).
4. Regenerate all surfaces, then `claude plugin validate` to confirm 0 errors.
