---
id: 034-plugin-directory-submission
title: Plugin Directory Submission Fixes
status: draft
created: 2026-06-01
author: Claude
---

# Spec 034 — Plugin Directory Submission Fixes

## Non-application work

This is build-tooling work only. There is no AI-augmented app journey, no
contract-pack, and no EnterpriseAI integration map — those sections are not
applicable to generator/tooling fixes.

---

## Overview

### Problem

The eai-gofer Claude Code plugin fails Anthropic's community plugin marketplace
submission bar (`clau.de/plugin-directory-submission`) for six distinct reasons:

1. Eight control commands ship with no frontmatter/description, making them
   invisible in the `/help` picker.
2. Every generated `SKILL.md` has double-frontmatter (the generator prepends its
   own `---` block onto a body that already starts with one).
3. Both marketplace manifest builders emit an invalid top-level `description`
   key, causing `claude plugin validate` to error with
   `root: Unrecognized key: "description"`.
4. `plugin.json` includes `category` and `tags` fields the validator warns on.
5. No `LICENSE` file ships inside `plugins/eai-gofer/`, despite `plugin.json`
   declaring `"license": "SEE LICENSE IN LICENSE"`.
6. Commands reference `.specify/scripts/` and `.specify/templates/` by bare
   relative path; after install in a stranger's repo those paths don't exist —
   only `${CLAUDE_PLUGIN_ROOT}/.specify/…` resolves correctly.

### Why it matters

All four generated surfaces (Claude, Codex, Gemini, Copilot) are emitted by two
scripts. Every issue traces to a specific function in those scripts, or to the
canonical source command bodies. Fixing at source and regenerating is the only
approach that stays consistent across surfaces and survives the next
`gofer:generate` run.

### Scope

Two edit sites: `.specify/scripts/node/generate-commands.mjs` and
`package-agent-plugin.mjs`, plus the 8 canonical command source files listed in
FR-1.

---

## Constraints / Non-functional Requirements

- **Never hand-edit generated files.** All generated surfaces under
  `.claude/commands/`, `.agents/`, `.system/`, `.gemini/`, `.github/prompts/`,
  and `plugins/eai-gofer/` are overwritten on each `gofer:generate` run.
- **No pipeline logic changes.** Fixes must not alter runtime behavior of the
  gofer pipeline stages.
- **All four surfaces must regenerate cleanly.** After applying fixes, run
  `npm run gofer:generate` then
  `npm run gofer:package-plugin -- --sync-repo`. All four surfaces must be
  consistent.
- **Embedded body-frontmatter in pipeline-stage source files is intentional.**
  The pattern that FR-2 strips from SKILL.md output must NOT be removed from
  pipeline command source bodies — `emitClaude()` relies on it for those commands.
- **Selective path rewrite only.** `.specify/specs/`, `.specify/memory/`, and
  `.specify/logs/` are workspace-runtime references that must remain bare
  (workspace-relative). Only `.specify/scripts/` and `.specify/templates/`
  are plugin-shipped assets that require `${CLAUDE_PLUGIN_ROOT}` prefixing.

---

## Functional Requirements

### FR-1 — Eight control command source files: add embedded frontmatter

**Fix site:** The 8 canonical source files:

```
.specify/commands/gofer_plan.md
.specify/commands/gofer_side.md
.specify/commands/gofer_personality.md
.specify/commands/gofer_vocabulary.md
.specify/commands/gofer_zoom_out.md
.specify/commands/gofer_spec_summary.md
.specify/commands/gofer_tdd.md
.specify/commands/gofer_diagnose.md
```

**What to do:** Add an embedded `---\ndescription: "…"\n---` block at the very
top of each file's body section (immediately after the outer stage frontmatter
closing `---`), matching the pattern that pipeline commands already use.
Descriptions must be taken from the existing `description` field in each file's
outer stage frontmatter (cross-referenced with `canonical-descriptions.mjs`).

**Why source-file edit, not generator synthesis:** Approved in proposal-review
(Michael's override: "Edit the 8 canonical source files rather than synthesizing
in the generator").

**Acceptance criteria:**

- [ ] After running `npm run gofer:generate` and
      `npm run gofer:package-plugin -- --sync-repo`, each of the 8 files
      `.claude/commands/{gofer_plan,gofer_side,gofer_personality,gofer_vocabulary,gofer_zoom_out,gofer_spec_summary,gofer_tdd,gofer_diagnose}.md`
      opens with a YAML frontmatter block containing a non-empty `description`
      field.
- [ ] The corresponding 8 files under
      `plugins/eai-gofer/commands/` also carry that frontmatter block.
- [ ] `claude plugin validate` on the plugin folder reports zero
      "No frontmatter block found" warnings for these 8 commands.
- [ ] The outer stage frontmatter in each source file is unchanged (the embedded
      block is added to the body, not merged into the outer block).

---

### FR-2 — SKILL.md double-frontmatter: strip body frontmatter before wrapping

**Fix site:** `generate-commands.mjs`, function `buildSkillContent()`,
approximately line 421.

**What to do:** Before concatenating `body` into the SKILL.md template,
call the already-existing `splitMarkdownFrontmatter()` utility to strip any
leading frontmatter from `body`. Use only the content portion; discard the
extracted frontmatter block (the generator already supplies `name` and
`description` for SKILL.md from stage metadata).

**Acceptance criteria:**

- [ ] Each generated `SKILL.md` (under `.agents/skills/`, `.system/skills/`, and
      `plugins/eai-gofer/skills/`) contains exactly one frontmatter block (one
      opening `---` and one closing `---` before the body text begins).
- [ ] `splitMarkdownFrontmatter()` is not duplicated; the existing export is
      reused.
- [ ] Pipeline-stage source files are not modified by this fix (the embedded
      body-frontmatter pattern in pipeline command sources is preserved).

---

### FR-3 — marketplace.json: remove invalid top-level `description` key

**Fix site:** `package-agent-plugin.mjs`, functions `buildRepoMarketplace()`
and any sibling marketplace builder (e.g. `buildCodexLocalMarketplace()`).

**What to do:** Delete the top-level `description` property from the returned
manifest object. If a human-readable description is desired, move the value into
`metadata.description` (which is a valid key). The valid top-level keys for a
marketplace manifest are `name`, `owner`, `metadata`, and `plugins`.

**Acceptance criteria:**

- [ ] After regeneration, `.claude-plugin/marketplace.json` has no top-level
      `description` key.
- [ ] `claude plugin validate` on the marketplace manifest returns 0 errors
      (no `root: Unrecognized key: "description"` message).
- [ ] `metadata.description` is present and non-empty (either pre-existing or
      moved from the removed top-level key).

---

### FR-4 — plugin.json: remove `category` and `tags` fields

**Fix site:** `package-agent-plugin.mjs`, function `buildPluginManifest()`.

**What to do:** Remove `category` and `tags` from the object returned by
`buildPluginManifest()`. These fields belong in marketplace metadata, not in
the plugin ID card (`plugin.json`).

**Acceptance criteria:**

- [ ] After regeneration, `.claude-plugin/plugin.json` (and its copy at
      `plugins/eai-gofer/.claude-plugin/plugin.json`) contains neither a
      `category` nor a `tags` key at any level.
- [ ] `claude plugin validate` on `plugin.json` produces no warnings about
      unrecognized `category` or `tags` keys.
- [ ] All required `plugin.json` fields (`name`, `version`, `description`,
      `author`) remain present and unchanged.

---

### FR-5 — LICENSE file: ship inside the plugin folder

**Fix site:** `package-agent-plugin.mjs`, function `writePluginFolder()`,
`copiedResources` array.

**What to do:** Add the string `'LICENSE'` to the `copiedResources` array. The
source is the repo-root `LICENSE` file; the destination is
`plugins/eai-gofer/LICENSE`.

**Acceptance criteria:**

- [ ] After running `npm run gofer:package-plugin -- --sync-repo`, the file
      `plugins/eai-gofer/LICENSE` exists.
- [ ] The file content matches the repo-root `LICENSE` (it is a copy, not a
      symlink).
- [ ] `plugin.json`'s `"license": "SEE LICENSE IN LICENSE"` declaration is
      satisfied (the referenced file is present at the expected relative path).

---

### FR-6 — Selective `${CLAUDE_PLUGIN_ROOT}` path rewrite for shipped assets

**Fix site:** `generate-commands.mjs`, functions `emitClaude()` and the skill
emitters (`emitAgentsSkills()` / `emitSystemSkills()` via `buildSkillContent()`).

**What to do:** In the output body of each generated command and SKILL.md,
rewrite only `.specify/scripts/` and `.specify/templates/` references to their
`${CLAUDE_PLUGIN_ROOT}`-prefixed forms:

| Input (bare) | Output (prefixed) |
|---|---|
| `.specify/scripts/` | `${CLAUDE_PLUGIN_ROOT}/.specify/scripts/` |
| `.specify/templates/` | `${CLAUDE_PLUGIN_ROOT}/.specify/templates/` |

The following subpaths must NOT be rewritten and must remain workspace-relative:

| Subpath | Reason |
|---|---|
| `.specify/specs/` | Artifact output — written to the user's working directory |
| `.specify/memory/` | Per-project constitution — lives in the user's repo |
| `.specify/logs/` | Runtime logs — written to the user's working directory |

**Acceptance criteria:**

- [ ] In every generated `.claude/commands/*.md` and
      `plugins/eai-gofer/commands/*.md`, all occurrences of `.specify/scripts/`
      and `.specify/templates/` are prefixed with `${CLAUDE_PLUGIN_ROOT}/`.
- [ ] In every generated SKILL.md (`.agents/`, `.system/`, `plugins/eai-gofer/skills/`),
      `.specify/scripts/` and `.specify/templates/` refs are prefixed with
      `${CLAUDE_PLUGIN_ROOT}/`.
- [ ] References to `.specify/specs/`, `.specify/memory/`, and `.specify/logs/`
      are bare (no `${CLAUDE_PLUGIN_ROOT}` prefix) in all generated files.
- [ ] Rewritten reference count is approximately 60 (44 scripts + 16 templates);
      untouched reference count is approximately 102 (68 specs + 18 memory + 16 logs).
      Exact counts may vary by ±5 if commands are added/removed; validate directionally.
- [ ] No new utility function is introduced if a simple inline regex substitution
      suffices; if extracted, it is named clearly (e.g. `rewriteShippedAssetPaths`).

---

## Out of Scope

The following items were identified during research but are explicitly excluded
from this feature:

- **Pruning duplicate/non-standard manifests.** The root-level `plugin.json`,
  nested `marketplace.json`, and per-CLI manifest copies emitted by
  `syncRepoManifests()` are kept as-is for Codex/Copilot discovery.
- **Gemini `../../../` include paths.** The `.gemini/` mirror dirs use
  `{{include: ../../../.specify/…}}` patterns. These are inert for Claude and
  are a separate clean-up task.
- **Legacy orphaned generator scripts.** `scripts/codex-skill-generator.js` and
  `scripts/copilot-prompt-enhancer.js` are not wired into `gofer:generate` and
  are out of scope.
- **Any pipeline logic changes.** Stage execution order, LLM calls, prompt
  assembly, and output schemas are unchanged.

---

## Success Criteria

| Metric | Target | How to measure |
|---|---|---|
| `claude plugin validate` errors | 0 | Run `claude plugin validate` on `.claude-plugin/marketplace.json` and `plugins/eai-gofer/.claude-plugin/plugin.json` |
| Commands with frontmatter/description | 8 / 8 | Inspect `.claude/commands/*.md` and `plugins/eai-gofer/commands/*.md` for leading YAML frontmatter |
| Frontmatter blocks per SKILL.md | Exactly 1 | Count `---` delimiters at the start of each `SKILL.md` |
| LICENSE present inside plugin | Yes | File check: `plugins/eai-gofer/LICENSE` exists and is non-empty |
| Rewritten `.specify` refs (scripts + templates) | ~60 | `grep -r 'CLAUDE_PLUGIN_ROOT.*\.specify/scripts' .claude/commands plugins/eai-gofer/commands .agents .system \| wc -l` |
| Preserved workspace-relative `.specify` refs (specs + memory + logs) | ~102 | `grep -r '\.specify/specs\|\.specify/memory\|\.specify/logs' .claude/commands plugins/eai-gofer/commands .agents .system \| grep -v CLAUDE_PLUGIN_ROOT \| wc -l` |
| Surfaces regenerated cleanly | 4 / 4 (Claude, Codex, Gemini, Copilot) | `npm run gofer:generate && npm run gofer:package-plugin -- --sync-repo` exits 0; no diff in unrelated files |

---

## Research Traceability

Maps each FR to the root-cause row in `research.md` (Codebase Analysis table).

| FR | research.md row # | Issue described | Fix location from research |
|---|---|---|---|
| FR-1 | 1 | 8 control commands have no Claude frontmatter / description | `.specify/commands/{8 files}.md`: add embedded `---description---` block to body |
| FR-2 | 2 | Every SKILL.md has double-frontmatter | `generate-commands.mjs ~line 421 buildSkillContent()`: strip leading frontmatter from body via `splitMarkdownFrontmatter()` |
| FR-3 | 3 | `marketplace.json` invalid top-level `description` | `package-agent-plugin.mjs buildRepoMarketplace()` / `buildCodexLocalMarketplace()`: drop top-level `description` |
| FR-4 | 4 | `category`/`tags` in `plugin.json` | `package-agent-plugin.mjs buildPluginManifest()`: remove `category` + `tags` |
| FR-5 | 5 | LICENSE absent from installable plugin | `package-agent-plugin.mjs writePluginFolder()`: add `'LICENSE'` to `copiedResources[]` |
| FR-6 | 6 | `.specify` asset refs not portable after install | `generate-commands.mjs emitClaude()` + skill emitters: selective regex rewrite for `scripts/` and `templates/` only |
