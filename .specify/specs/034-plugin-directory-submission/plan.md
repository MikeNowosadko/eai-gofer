---
feature: 034-plugin-directory-submission
spec: .specify/specs/034-plugin-directory-submission/spec.md
research: .specify/specs/034-plugin-directory-submission/research.md
status: ready
created: 2026-06-01
---

# Plan: Plugin Directory Submission Fixes

## Non-application work

This is build-tooling work only. There are no data entities, no API endpoints,
and no service contracts — those sections are not applicable.

---

## Technical Context

### Tech Stack

Node.js ESM scripts (`*.mjs`), no external runtime dependencies, no bundler.
All edits are to two generator scripts and eight source command files.

### Pipeline Data Flow

```text
.specify/commands/*.md          (canonical source — 24 hand-authored .md files)
         │   parseStageCommand(): strips OUTER stage frontmatter; body = remainder
         ▼
generate-commands.mjs           (npm run gofer:generate)
         ├─ emitClaude()          → .claude/commands/<stem>.md        (body verbatim)
         ├─ emitClaudeMirror()    → extension/resources/claude-commands/
         ├─ emitCopilot()         → via buildCopilotPromptContent()
         ├─ emitGithubPrompts()   → .github/prompts/ (same builder)
         ├─ emitAgentsSkills()    → .agents/skills/<stem>/SKILL.md    (buildSkillContent)
         ├─ emitSystemSkills()    → .system/skills/<stem>/SKILL.md    (buildSkillContent)
         ├─ emitGemini()          → .gemini/commands/gofer/
         └─ emitCodexConfig()     → .specify/outputs/codex-config-fragment.toml
         ▼
package-agent-plugin.mjs --sync-repo   (npm run gofer:package-plugin -- --sync-repo)
         ├─ writePluginFolder()  → dist/<ver>/eai-gofer/  (staged zip)
         └─ syncRepoManifests()  → plugins/eai-gofer/  (repo-local copy)
                                    .claude-plugin/{plugin,marketplace}.json
                                    .github/plugin/*  .codex-plugin/plugin.json
                                    .agents/plugins/marketplace.json
```

**Key constraint:** every generated surface is overwritten on each run. Never
hand-edit generated output; fix only canonical sources and generators.

### Integration Points

| Function | File : line | What changes | FR |
|---|---|---|---|
| `buildSkillContent` | `generate-commands.mjs:421` | Strip leading frontmatter from `body` before wrapping | FR-2 |
| `emitClaude` | `generate-commands.mjs:149` | Apply selective path rewrite to `stage.body` before writing | FR-6 |
| `emitAgentsSkills` | `generate-commands.mjs:443` | Path rewrite happens inside `buildSkillContent` call | FR-6 |
| `emitSystemSkills` | `generate-commands.mjs:483` | Same as above | FR-6 |
| `buildRepoMarketplace` | `package-agent-plugin.mjs:215` | Drop top-level `description` key | FR-3 |
| `buildCodexLocalMarketplace` | `package-agent-plugin.mjs:248` | Drop top-level `description` key (N/A — no top-level `description` present; verify) | FR-3 |
| `buildPluginManifest` | `package-agent-plugin.mjs:145` | Remove `category` (line 159) and `tags` (line 160) fields | FR-4 |
| `writePluginFolder` — `copiedResources` | `package-agent-plugin.mjs:394` | Add `'LICENSE'` to array | FR-5 |
| 8 control command source files | `.specify/commands/gofer_{plan,side,personality,vocabulary,zoom_out,spec_summary,tdd,diagnose}.md` | Add embedded `---description---` block at top of body | FR-1 |

---

## Implementation Phases

### Phase 1 — FR-2: Fix SKILL.md double-frontmatter in `buildSkillContent`

**Goal:** Each generated SKILL.md contains exactly one frontmatter block.

**Tasks:**

- [ ] Open `generate-commands.mjs`. Locate `buildSkillContent` at line 421.
      Current body: `` return `---\nname: ${stageName}\ndescription: "${description}"\n---\n\n${body}`; ``
- [ ] Before returning, strip any leading frontmatter from `body` using the
      already-exported `splitMarkdownFrontmatter()` (line 322). The call is:
      ```js
      const { body: strippedBody } = splitMarkdownFrontmatter(body);
      return `---\nname: ${stageName}\ndescription: "${description}"\n---\n\n${strippedBody}`;
      ```
      Use `strippedBody` in the template; discard the extracted frontmatter object.

**RISK NOTE — do not strip in `emitClaude`:** This stripping applies ONLY inside
`buildSkillContent`, which is called by the skill emitters. `emitClaude` at
line 149 writes `stage.body` verbatim — the embedded `---description---` block
at the top of pipeline-command bodies IS the Claude frontmatter for those
commands. Do not call `splitMarkdownFrontmatter` in `emitClaude`. The 8 control
commands have no embedded block (they have nothing to strip), so FR-2 and FR-1
are orthogonal.

**Verification:**
```bash
npm run gofer:generate --surfaces agents-skills,system-skills
# Then count frontmatter delimiters in any SKILL.md — must be exactly 2 (open + close)
awk '/^---$/{c++} END{print c}' .agents/skills/3_gofer_plan/SKILL.md
# Expected: 2
```

**Rollback:** Revert the one-line change to `buildSkillContent`; re-run generate.

---

### Phase 2 — FR-6: Selective `${CLAUDE_PLUGIN_ROOT}` path rewrite

**Goal:** Shipped asset refs (`.specify/scripts/`, `.specify/templates/`) are
portable after plugin install; workspace-runtime refs (specs, memory, logs)
are left bare.

**Approach:** Introduce a module-level helper `rewriteShippedAssetPaths(text)`
immediately before `emitClaude` (around line 148) in `generate-commands.mjs`:

```js
const SHIPPED_ASSET_RE = /(?<!\$\{CLAUDE_PLUGIN_ROOT\}\/)\.specify\/(scripts|templates)\//g;

function rewriteShippedAssetPaths(text) {
  return text.replace(SHIPPED_ASSET_RE, '${CLAUDE_PLUGIN_ROOT}/.specify/$1/');
}
```

**Idempotency:** The negative lookbehind `(?<!\$\{CLAUDE_PLUGIN_ROOT\}/)` ensures
already-rewritten references are not double-prefixed. Run the function as many
times as needed; output is stable.

**Apply in `emitClaude` (line 163 — the `fs.writeFile` call):**
```js
await fs.writeFile(outPath, rewriteShippedAssetPaths(stage.body), 'utf8');
```

**Apply in `buildSkillContent` (line 422 — after strippedBody from Phase 1):**
```js
return `---\nname: ${stageName}\ndescription: "${description}"\n---\n\n${rewriteShippedAssetPaths(strippedBody)}`;
```

**Gemini and Copilot are out of scope:**
- Gemini (`emitGemini`, line 524) writes `stage.body` to `.md` but uses a TOML
  `{{include:}}` pointer to the canonical source; paths inside are not
  plugin-installed assets — leave as-is.
- Copilot (`buildCopilotPromptContent`, line 283) inlines body and already goes
  through `splitMarkdownFrontmatter` + transform; it is not an installed Claude
  plugin surface — leave as-is.

**Verification:**
```bash
npm run gofer:generate --surfaces claude,agents-skills,system-skills
grep -r 'CLAUDE_PLUGIN_ROOT.*\.specify/scripts' .claude/commands | wc -l   # ~44
grep -r 'CLAUDE_PLUGIN_ROOT.*\.specify/templates' .claude/commands | wc -l  # ~16
grep -r '\.specify/specs' .claude/commands | grep -v CLAUDE_PLUGIN_ROOT | wc -l  # ~68
grep -r '\.specify/memory' .claude/commands | grep -v CLAUDE_PLUGIN_ROOT | wc -l # ~18
grep -r '\.specify/logs' .claude/commands | grep -v CLAUDE_PLUGIN_ROOT | wc -l   # ~16
```

All scripts+templates refs must be prefixed; all specs/memory/logs refs must be bare.

**Rollback:** Remove helper and the two call sites; re-run generate.

---

### Phase 3 — FR-1: Add embedded frontmatter to 8 control command sources

**Goal:** After `gofer:generate`, all 8 control commands in `.claude/commands/`
carry a leading YAML frontmatter block with a non-empty `description`.

**Pattern to replicate** (from `3_gofer_plan.md` body, lines 18–21):
```
---
description: Generate technical implementation plan with architecture and contracts
---
```
Insert this block immediately after the outer stage frontmatter closing `---`
and before the first `#` heading. The outer frontmatter is unchanged.

**Exact embedded block per file** (descriptions from `canonical-descriptions.mjs`
and confirmed against each file's outer frontmatter):

| Source file | `name` in outer FM | Description string |
|---|---|---|
| `gofer_plan.md` | `gofer:plan` | `Toggle plan mode in the active CLI session for the next user prompt; non-pipeline control command.` |
| `gofer_side.md` | `gofer:side` | `Open a side conversation in the active CLI without disturbing the main pipeline state; resumable.` |
| `gofer_personality.md` | `gofer:personality` | `Set the assistant personality for this Gofer session: friendly, pragmatic, or none (default).` |
| `gofer_vocabulary.md` | `gofer:vocabulary` | `Extract domain terminology into a canonical feature glossary.` |
| `gofer_zoom_out.md` | `gofer:zoom-out` | `Show how the current feature connects to broader system boundaries.` |
| `gofer_spec_summary.md` | `gofer:spec-summary` | `Generate a business-friendly summary of feature value and scope.` |
| `gofer_tdd.md` | `gofer:tdd` | `Guide a red-green-refactor loop tied to spec acceptance criteria.` |
| `gofer_diagnose.md` | `gofer:diagnose` | `Run a reproduce-minimize-instrument-fix loop for bugs and failing tests.` |

**Edit instruction per file:** After the closing `---` of the outer stage
frontmatter and before the first `# Heading` line, insert:
```
---
description: "<string from table above>"
---

```
Match the existing convention in `3_gofer_plan.md` exactly (no extra blank line
between the outer close `---` and the embedded open `---`).

**Note:** `plugins/eai-gofer/commands/` is generated output from
`--sync-repo` — do NOT edit it. Edit only the top-level
`.specify/commands/*.md` sources; regeneration propagates the change.

**Verification:**
```bash
npm run gofer:generate --surfaces claude
for cmd in gofer_plan gofer_side gofer_personality gofer_vocabulary gofer_zoom_out gofer_spec_summary gofer_tdd gofer_diagnose; do
  echo -n "$cmd: "
  head -3 .claude/commands/${cmd}.md | grep -c '^---'
done
# Each must print: 1
```

**Rollback:** Remove the 3-line embedded block from each source file; re-run generate.

---

### Phase 4 — FR-3/4/5: Fix `package-agent-plugin.mjs` manifest builders

**Goal:** `claude plugin validate` returns 0 errors and LICENSE ships.

#### FR-3 — Remove top-level `description` from `buildRepoMarketplace`

At `package-agent-plugin.mjs:215`, `buildRepoMarketplace` returns an object
with a top-level `description` key at line 218. The valid schema keys are
`name`, `owner`, `metadata`, `plugins`.

- [ ] Delete lines 218–219 (`description: 'Public EAI Gofer...',`) from the
      returned object in `buildRepoMarketplace`.
- [ ] Confirm `metadata.description` at line 224 is already present —
      it is (`'Install the EAI Gofer agent plugin...'`); no move needed.
- [ ] Audit `buildCodexLocalMarketplace` (line 248): the returned object has no
      top-level `description` key — no change required.

#### FR-4 — Remove `category` and `tags` from `buildPluginManifest`

At `package-agent-plugin.mjs:145`, `buildPluginManifest` returns an object with:
- `category: 'Coding'` at line 159
- `tags: ['eai-gofer', 'gofer', 'agentic-coding']` at line 160

- [ ] Delete both lines. Required fields (`name`, `version`, `description`,
      `author`) remain untouched at lines 146–164.
- [ ] `buildClaudeManifest` (line 167) also has `category` (line 182) and
      `tags` (line 183). The spec and validator target `plugin.json`; confirm
      whether Claude-specific manifest is validated by `claude plugin validate`.
      If so, remove from `buildClaudeManifest` as well.

#### FR-5 — Add `'LICENSE'` to `copiedResources`

At `package-agent-plugin.mjs:394`, the `copiedResources` array lists paths
copied from repo-root into `pluginRoot`. Add `'LICENSE'`:
```js
const copiedResources = [
  'LICENSE',                        // ← add this line
  '.specify/commands',
  '.specify/templates',
  ...
];
```
Source: `<root>/LICENSE`. Destination: `<pluginRoot>/LICENSE`. `copyIfExists`
handles `ENOENT` gracefully if the file is absent.

**Verification:**
```bash
npm run gofer:package-plugin -- --sync-repo
# FR-3
node -e "const m=require('./plugins/eai-gofer/.claude-plugin/marketplace.json'); console.log('description' in m)"
# Expected: false

# FR-4
node -e "const p=require('./plugins/eai-gofer/.claude-plugin/plugin.json'); console.log('category' in p, 'tags' in p)"
# Expected: false false

# FR-5
test -f plugins/eai-gofer/LICENSE && echo "LICENSE present" || echo "MISSING"
```

**Rollback:** Re-add the removed fields; remove `'LICENSE'` from `copiedResources`; re-run package-plugin.

---

### Phase 5 — Regenerate + full validate

**Goal:** All four surfaces regenerate cleanly and `claude plugin validate` passes.

**Tasks:**

- [ ] Run full generate:
      ```bash
      npm run gofer:generate
      ```
- [ ] Run package + sync:
      ```bash
      npm run gofer:package-plugin -- --sync-repo
      ```
- [ ] Validate Claude plugin manifests:
      ```bash
      claude plugin validate .claude-plugin/marketplace.json
      claude plugin validate plugins/eai-gofer/.claude-plugin/plugin.json
      # Both must exit 0 with 0 errors
      ```
- [ ] Verify 8 control commands have frontmatter in both output locations:
      ```bash
      for cmd in gofer_plan gofer_side gofer_personality gofer_vocabulary \
                 gofer_zoom_out gofer_spec_summary gofer_tdd gofer_diagnose; do
        echo -n ".claude/commands/${cmd}.md: "
        head -1 .claude/commands/${cmd}.md
        echo -n "plugins/eai-gofer/commands/${cmd}.md: "
        head -1 plugins/eai-gofer/commands/${cmd}.md
      done
      # Each must print: ---
      ```
- [ ] Verify SKILL.md single-frontmatter (pick a pipeline stage with embedded body FM):
      ```bash
      awk '/^---$/{c++} END{print "blocks="c}' .agents/skills/3_gofer_plan/SKILL.md
      # Expected: blocks=2
      ```
- [ ] Verify shipped-asset path rewrites (directional counts):
      ```bash
      grep -r 'CLAUDE_PLUGIN_ROOT.*\.specify/scripts' .claude/commands .agents/skills .system/skills | wc -l   # ~44
      grep -r 'CLAUDE_PLUGIN_ROOT.*\.specify/templates' .claude/commands .agents/skills .system/skills | wc -l # ~16
      grep -r '\.specify/specs' .claude/commands | grep -v CLAUDE_PLUGIN_ROOT | wc -l  # ~68
      ```
- [ ] Confirm no regressions in unrelated files:
      ```bash
      git diff --name-only | grep -v -E '(\.claude/commands|\.agents/skills|\.system/skills|plugins/eai-gofer|\.claude-plugin|plugin\.json|marketplace\.json|\.specify/commands)'
      # Expected: empty (no off-target diffs)
      ```

---

## File Structure

### Files modified (source — edit these):

| File | Changes |
|---|---|
| `.specify/scripts/node/generate-commands.mjs` | Phase 1 (`buildSkillContent` line 421) + Phase 2 (`rewriteShippedAssetPaths` helper + 2 call sites in `emitClaude` and `buildSkillContent`) |
| `.specify/scripts/node/package-agent-plugin.mjs` | Phase 4: `buildRepoMarketplace` line 218, `buildPluginManifest` lines 159–160, `writePluginFolder` copiedResources line 394 |
| `.specify/commands/gofer_plan.md` | Phase 3: add embedded frontmatter block |
| `.specify/commands/gofer_side.md` | Phase 3 |
| `.specify/commands/gofer_personality.md` | Phase 3 |
| `.specify/commands/gofer_vocabulary.md` | Phase 3 |
| `.specify/commands/gofer_zoom_out.md` | Phase 3 |
| `.specify/commands/gofer_spec_summary.md` | Phase 3 |
| `.specify/commands/gofer_tdd.md` | Phase 3 |
| `.specify/commands/gofer_diagnose.md` | Phase 3 |

**Total edited files: 10** (2 generators + 8 command sources)

### Files regenerated (generated output — never edit directly):

- `.claude/commands/*.md` (24 files)
- `.agents/skills/*/SKILL.md`
- `.system/skills/*/SKILL.md`
- `.gemini/commands/gofer/*.{md,toml}`
- `.github/prompts/*.prompt.md`
- `extension/resources/claude-commands/*.md`
- `extension/resources/copilot-prompts/*.prompt.md`
- `.specify/outputs/codex-config-fragment.toml`
- `plugins/eai-gofer/**` (entire tree, via `--sync-repo`)
- `.claude-plugin/{plugin,marketplace}.json`
- `.github/plugin/{plugin,marketplace}.json`
- `.codex-plugin/plugin.json`
- `.agents/plugins/marketplace.json`
- `plugin.json` (repo root)

---

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| **FR-2 stripping bleeds into `emitClaude`** — `splitMarkdownFrontmatter` called in wrong emitter, destroying Claude pipeline-command frontmatter | HIGH — pipeline commands lose descriptions from Claude's perspective | Medium if not carefully scoped | Call `splitMarkdownFrontmatter` ONLY inside `buildSkillContent`; never inside `emitClaude`. The call in `buildCopilotPromptContent` (line 285) already does this correctly — use it as the reference. |
| **Path-rewrite double-prefix** — regex runs twice on already-rewritten output | Medium — garbled paths like `${CLAUDE_PLUGIN_ROOT}/${CLAUDE_PLUGIN_ROOT}/` | Low with negative lookbehind | The `(?<!\$\{CLAUDE_PLUGIN_ROOT\}/)` lookbehind in `SHIPPED_ASSET_RE` makes the substitution idempotent. Verify with a unit test or manual spot-check after running Phase 5 verification. |
| **Regeneration overwrites uncommitted manual edits** in generated surfaces | Medium — work in progress lost if generated surfaces were hand-edited | Low (policy prohibits it) | Confirm `git diff` is clean on generated directories before running `npm run gofer:generate`. |
| **`plugins/eai-gofer/commands/` edited instead of top-level sources** — changes are lost on next `--sync-repo` | HIGH | Low if this plan is followed | `plugins/eai-gofer/` is the output of `syncRepoManifests`'s `fs.cp` call (line 453). Only edit `.specify/commands/*.md`; never edit `plugins/eai-gofer/`. |
| **`buildClaudeManifest` also has `category`/`tags`** — validator may warn if it validates Claude-specific manifests | Low–Medium | Medium — `buildClaudeManifest` at lines 167–185 has same fields | Check whether `claude plugin validate` reads `claudeManifest` output; if so, remove from `buildClaudeManifest` as well in Phase 4. |
| **LICENSE absent at repo root** — `copyIfExists` silently skips ENOENT | Low — plugin ships without LICENSE | Low (repo has a LICENSE) | Verify `ls <root>/LICENSE` before Phase 4; the helper swallows ENOENT so a missing file would silently produce no copy. |

---

## Spec Traceability

| FR | Phase | Location | Covered |
|---|---|---|---|
| FR-1 — 8 control commands: add embedded frontmatter | Phase 3 | `.specify/commands/gofer_{plan,side,personality,vocabulary,zoom_out,spec_summary,tdd,diagnose}.md` | Yes |
| FR-2 — SKILL.md double-frontmatter strip | Phase 1 | `generate-commands.mjs:421 buildSkillContent` | Yes |
| FR-3 — marketplace.json invalid top-level `description` | Phase 4 | `package-agent-plugin.mjs:218 buildRepoMarketplace` | Yes |
| FR-4 — plugin.json `category`/`tags` removal | Phase 4 | `package-agent-plugin.mjs:159–160 buildPluginManifest` | Yes |
| FR-5 — LICENSE file in plugin | Phase 4 | `package-agent-plugin.mjs:394 writePluginFolder copiedResources` | Yes |
| FR-6 — selective `${CLAUDE_PLUGIN_ROOT}` path rewrite | Phase 2 | `generate-commands.mjs emitClaude` + `buildSkillContent` | Yes |

**Coverage: 6/6 FRs covered across 5 phases.**
