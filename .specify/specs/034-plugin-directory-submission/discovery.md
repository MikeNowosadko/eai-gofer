---
feature: 'Plugin Directory Submission Fixes'
created: '2026-06-01'
discoveredBy: Claude + Michael
status: complete
---

# Business Discovery: Plugin Directory Submission Fixes

## Problem Statement

**Pain Point**: The eai-gofer plugin (Claude/Codex/Gemini/Copilot) does not pass
Anthropic's community plugin marketplace submission bar (clau.de/plugin-directory-submission).
`claude plugin validate` fails on the marketplace manifest, the installed plugin
omits a LICENSE, several commands lack frontmatter, and command scripts reference
`.specify/...` by bare relative path so they break once installed in another repo.

**Current State**: Plugin works for self-hosted/local install because the gofer
repo itself contains the `.specify` scaffold. It is not portable to a fresh user.

**Impact**: Blocks stage-two submission to the Anthropic community marketplace.

## Application Classification

| Field | Decision |
| ----- | -------- |
| Classification | Non-application work (build-tooling / code hygiene) |
| Reason | Fixing plugin packaging/manifests/generators; no durable end-user app workflow is being built |
| Four-step AI journey required | No |

## Target Users

- **Persona**: Developers installing eai-gofer from the marketplace + Anthropic's
  automated submission review.
- **Technical Level**: Technical.
- **Key Needs**: Plugin validates clean, installs portably, license present, docs accurate.

## Value Proposition

**Primary Value**: Quality/compliance — plugin passes automated marketplace review.
**Quantified Goal**: `claude plugin validate` returns 0 errors; portable install works in any repo.

## Success Metrics

| Metric | Target | Measurement |
| ------ | ------ | ----------- |
| Validation errors | 0 | `claude plugin validate` on marketplace + plugin manifests |
| Portable internal refs | 100% | All shipped-file refs use ${CLAUDE_PLUGIN_ROOT} |
| Surfaces regenerated | 4/4 | claude, codex, gemini, copilot built from canonical source |
| LICENSE shipped | yes | LICENSE present in plugins/eai-gofer |

## Locked Decisions

| Decision | Choice | Rationale |
| -------- | ------ | --------- |
| Process depth | Full Gofer Pro pipeline | User directive |
| .specify path fix | Option A: ${CLAUDE_PLUGIN_ROOT}/.specify/... | User directive; no user-side bootstrap needed |
| Fix propagation | Canonical source + generators, then regenerate | Surfaces are generated, not hand-maintained |

## Authoritative Fix List

See ops/tech-docs/mod-tools/gofer/PLUGIN-DIRECTORY-REQUIREMENTS.md
