---
feature: 'Plugin Directory Submission Fixes'
created: 2026-06-01
status: approved
recommendedScenario: 'Source-driven generator fixes + regenerate all surfaces'
recommendedArchitecture: 'Fix canonical source + 2 generator scripts, selective ${CLAUDE_PLUGIN_ROOT} rewrite'
selectedOption: 'Selective path rewrite; edit 8 source files; keep multi-CLI manifests with clean canonical'
approvedBy: 'Michael'
approvedAt: '2026-06-01'
status_note: approved
---

# Proposal Review: Plugin Directory Submission Fixes

## What We Found

The four CLI surfaces (Claude, Codex, Gemini, Copilot) and all manifests are
**generated** by two scripts: `generate-commands.mjs` and
`package-agent-plugin.mjs`. Every issue in PLUGIN-DIRECTORY-REQUIREMENTS.md traces
to a specific function in those two files (or, for #1, to a pattern difference in
the source command bodies). No fix requires hand-editing generated output.

## Recommended Business Scenario

Apply all fixes at the source/generator layer, then run `gofer:generate` +
`gofer:package-plugin --sync-repo` to propagate to every surface, and verify with
`claude plugin validate`. This is the only approach that keeps the four surfaces
consistent and survives the next regeneration.

## Technology Architecture Recommendation

### Recommended Architecture

Two edit sites, six fixes, one regeneration + validation pass:

- `generate-commands.mjs`
  - `emitClaude()`: synthesize frontmatter from `stage.frontmatter.description`
    when the body has no leading frontmatter (fixes the 8 control commands).
  - `emitClaude()` + skill emitters: selective `${CLAUDE_PLUGIN_ROOT}` rewrite for
    `.specify/scripts/` and `.specify/templates/` only.
  - `buildSkillContent()`: strip leading frontmatter from body (fixes double-frontmatter).
- `package-agent-plugin.mjs`
  - `buildRepoMarketplace()` / sibling marketplace builders: drop invalid top-level `description`.
  - `buildPluginManifest()`: remove `category`/`tags`.
  - `writePluginFolder()`: add `LICENSE` to copied resources.
- Regenerate + `claude plugin validate` (marketplace + plugin manifests).

### Architecture Options

| Option | Why choose it | Why not now |
| ------ | ------------- | ----------- |
| Source + generator fixes, regenerate (RECOMMENDED) | Consistent across 4 surfaces; survives regen | Slightly more upfront understanding |
| Hand-edit generated files | Fast | Overwritten on next generate; diverges surfaces |

## Key Decisions and Why

- **Selective path rewrite** (scripts+templates → plugin root; specs/memory/logs
  stay workspace-relative): a blanket rewrite corrupts 102 workspace-runtime refs.
- **Synthesize control-command frontmatter in `emitClaude()`**: avoids churning 8
  source files and matches the existing Copilot emitter pattern.
- **Keep multi-CLI manifests** but ensure the `.claude-plugin/*` pair validates clean.

## What Can Change Before Specification

- Frontmatter approach (synthesize vs edit source files).
- Path-rewrite scope (selective vs blanket — strongly recommend selective).
- Whether to prune duplicate manifests or keep them for Codex/Copilot discovery.

## Open Questions

- [ ] Confirm selective path rewrite (recommended).
- [ ] Confirm frontmatter synthesis approach (recommended).
- [ ] Keep or prune duplicate/non-standard manifests.

## User Feedback and Overrides

- **Path scope**: Selective rewrite confirmed (scripts + templates → ${CLAUDE_PLUGIN_ROOT}; specs/memory/logs stay workspace-relative). Blanket explicitly rejected after breakage explanation.
- **Frontmatter fix**: Edit the 8 canonical source files (embedded ---description--- block) rather than synthesizing in the generator.
- **Manifests**: Keep multi-CLI manifest copies; ensure canonical .claude-plugin/* validates clean.

## Approval

- Status: approved
- Approved by: Michael, 2026-06-01
- Next action: proceed to `/2_gofer_specify`
