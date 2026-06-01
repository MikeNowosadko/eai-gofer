# EAI Gofer — Plugin Directory Submission Requirements

Status of the eai-gofer plugin against Anthropic's plugin-marketplace bar.
**All mandatory items are now resolved** (see the Gofer pipeline run in
`.specify/specs/034-plugin-directory-submission/`). This document is updated to
reflect the post-fix state.

## Where you submit (corrected — two tiers)

There are **two** directories, and the self-serve form goes to the community one:

| | **Official directory** (`claude-plugins-official`) | **Community marketplace** (`claude-plugins-community`) |
|---|---|---|
| How you get in | Anthropic curates at its discretion — **no application** | You apply via the form |
| Submission form | ❌ form does **not** add you here | ✅ `clau.de/plugin-directory-submission` (or Console: `platform.claude.com/plugins/submit`) |
| Review | Hand-picked by Anthropic | Automated review + optional "Anthropic Verified" badge |

`clau.de` is legitimate — it's Anthropic's URL-shortener domain, referenced
directly by the official `anthropics/claude-plugins-community` GitHub repo. Your
stage-two target is the **community marketplace**; the automated review there
checks exactly the items below.

## Requirements — now all passing

| # | Requirement | Status | Footnote |
|---|-------------|--------|----------|
| 1 | `marketplace.json` valid + passes `claude plugin validate` | ✅ Pass | [^1] |
| 2 | `plugin.json` with name / version / description / author | ✅ Pass | [^2] |
| 3 | Components well-formed (commands, agents, skills) | ✅ Pass | [^3] |
| 4 | README explaining what it does, install, and usage | ✅ Pass | [^4] |
| 5 | LICENSE file present *inside the installed plugin* | ✅ Pass | [^5] |
| 6 | No hardcoded paths (usernames, home dirs, machine paths) | ✅ Pass | [^6] |
| 7 | `${CLAUDE_PLUGIN_ROOT}` used for all internal file references | ✅ Pass | [^7] |
| 8 | No references outside the plugin directory (`../`) | ⚠️ Optional polish | [^8] |
| 9 | Works for someone else after install | ✅ Pass | [^9] |

`claude plugin validate` across all four manifests: **0 errors, 0 warnings**
(was 1 error + 9 warnings before this work).

## Fix checklist — completed

| Priority | Fix | Status |
|----------|-----|--------|
| 🔴 1 | Remove invalid top-level `"description"` from marketplace manifests | ✅ Done (`buildRepoMarketplace` + `buildLocalMarketplace`; `metadata.description` kept/added) |
| 🔴 2 | Make `.specify` portable via `${CLAUDE_PLUGIN_ROOT}` | ✅ Done (selective: 60 shipped-asset refs rewritten, 99 workspace refs left relative) |
| 🔴 3 | Ship LICENSE inside `plugins/eai-gofer/` | ✅ Done (added to `writePluginFolder` copiedResources) |
| 🟡 4 | Frontmatter on the 8 control commands | ✅ Done (embedded `---description---` in 8 canonical sources) |
| 🟡 5 | Remove `category`/`tags` from `plugin.json` | ✅ Done (`buildPluginManifest` + `buildClaudeManifest`) |
| 🟢 6 | Fix double-frontmatter in `SKILL.md` | ✅ Done (`buildSkillContent` + `buildStageSkill` strip leading frontmatter) |
| 🟢 7 | Prune duplicate non-`.claude-plugin` manifests | ◻️ Not done (optional; kept for Codex/Copilot discovery) |
| 🟢 8 | Strip non-Claude `.gemini/`/`.github/` mirror dirs | ◻️ Not done (optional; only matters for a Claude-only listing) |

All fixes were applied at the **source/generator layer** (`generate-commands.mjs`,
`package-agent-plugin.mjs`, and the 8 canonical command sources) and propagated
to all four CLI surfaces (Claude, Codex, Gemini, Copilot) by regeneration —
never hand-edited into generated output.

### Fix sites that differed from the original plan

Implementation surfaced three extra sites the first review didn't catch:

- `buildClaudeManifest` also carried `category`/`tags` (not just `buildPluginManifest`).
- `buildLocalMarketplace` was the real source of the nested marketplace's bad
  top-level `description` (not `buildCodexLocalMarketplace`, which never had one).
- `buildStageSkill` builds the *installable* skills, so the double-frontmatter
  fix had to go there too — not only in `generate-commands.mjs`.

## How the pieces fit together

```text
marketplace.json  →  points to  →  plugin source dir (./plugins/eai-gofer)
                                        ├── .claude-plugin/plugin.json   (ID card) ✅
                                        ├── commands/        (descriptions) ✅
                                        ├── agents/          ✅
                                        ├── skills/*/SKILL.md (single frontmatter) ✅
                                        ├── LICENSE          (shipped) ✅
                                        ├── README.md        ✅
                                        └── shipped files → ${CLAUDE_PLUGIN_ROOT}/.specify/{scripts,templates} ✅
```

The directory's bar reduces to four things — **validate clean, no path
assumptions, license present, docs accurate** — all now satisfied.

## To regenerate / re-verify after any future source edit

```bash
npm run gofer:generate
npm run gofer:package-plugin -- --sync-repo
claude plugin validate .claude-plugin/marketplace.json
claude plugin validate .claude-plugin/plugin.json
```

---

## Footnotes

[^1]: **`marketplace.json` — the storefront.** At `.claude-plugin/marketplace.json`,
    read first by `/plugin marketplace add`. Allowed top-level keys: `name`,
    `owner`, `metadata`, `plugins` — there is **no** top-level `description`.
    The plugin previously set one (→ `Unrecognized key: "description"` error);
    it is now removed in `buildRepoMarketplace()` and `buildLocalMarketplace()`,
    with the description kept under `metadata.description`. Both the root and the
    nested `plugins/eai-gofer/.claude-plugin/marketplace.json` now validate clean.

[^2]: **`plugin.json` — the plugin's ID card.** At
    `<plugin-source>/.claude-plugin/plugin.json` (the only path Claude reads).
    Required: `name`, `version` (3.4.0), `description`, `author` — all present
    and correct.

[^3]: **Components.** 24 commands, 42 agents, 25 skills. The 8 control commands
    (`gofer_plan`, `gofer_side`, `gofer_personality`, `gofer_vocabulary`,
    `gofer_zoom_out`, `gofer_spec_summary`, `gofer_tdd`, `gofer_diagnose`) now
    emit YAML frontmatter with a `description` (added as an embedded block in
    each canonical source). Every `SKILL.md` now has exactly one frontmatter
    block (the previous double-frontmatter leak is fixed in `buildSkillContent`
    and `buildStageSkill`). `claude plugin validate` reports 0 "No frontmatter"
    warnings.

[^4]: **README.md.** Covers what the plugin does, install, and per-CLI usage
    across Claude/Codex/Copilot. Unchanged by this work.

[^5]: **LICENSE.** Now ships inside the installable plugin — `'LICENSE'` was
    added to the `copiedResources` array in `writePluginFolder()`, so
    `plugins/eai-gofer/LICENSE` is present (49 lines). The packager's
    `assertNoPersonalPaths` guard passes.

[^6]: **No hardcoded paths.** The plugin source contains no usernames or home
    directories. The packager actively enforces this with
    `assertNoPersonalPaths`, which scans the staged bundle and throws on any
    `/Users/...` or `/home/...` path.

[^7]: **`${CLAUDE_PLUGIN_ROOT}` for internal references.** Applied **selectively**:
    only `.specify/scripts/` (44 refs) and `.specify/templates/` (16 refs) — the
    plugin-shipped, read-only assets — are rewritten to
    `${CLAUDE_PLUGIN_ROOT}/.specify/...`. Workspace-runtime paths
    (`.specify/specs/` 65, `.specify/memory/` 18, `.specify/logs/` 16) are
    intentionally left relative so the pipeline writes artifacts into the user's
    project, not the read-only install dir. The rewrite runs in `emitClaude()`
    and the skill builders and is idempotent. Gemini (`{{include}}`) and Copilot
    (inlined) surfaces are out of scope and verified to contain zero
    `${CLAUDE_PLUGIN_ROOT}` occurrences.

[^8]: **No references outside the plugin directory.** The `.gemini/` and
    `.github/` mirrors use `{{include: ../../../.specify/...}}`. These stay
    inside the plugin tree and are inert for Claude, so they do not fail
    validation — but stripping the non-Claude mirror dirs is recommended polish
    for a Claude-only directory listing. Tracked as optional fix #8, not done.

[^9]: **Works for someone else.** The portability gap is closed: shipped scripts
    and templates now resolve from the installed plugin via
    `${CLAUDE_PLUGIN_ROOT}`, while feature artifacts/checkpoints/logs still write
    to the user's working directory. A stranger can `/plugin marketplace add` the
    repo, install, open an unrelated project, and run the commands without the
    "script not found" failure the bare relative paths previously caused.
