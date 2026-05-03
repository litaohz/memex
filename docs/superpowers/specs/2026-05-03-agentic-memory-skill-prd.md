# PRD: A-MEM-Inspired Agentic Memory Skill

Status: Draft
Date: 2026-05-03
Owner: memex contributors

## Summary

Memex already has the core primitives needed for agent memory: create cards, search cards, read cards, write cards, and connect cards with wikilinks. A-MEM is useful because it packages those primitives into a repeatable agentic workflow: create a note, enrich it with semantic metadata, retrieve related memories, decide links, and optionally evolve existing memories.

This product direction should not turn memex into a heavyweight LLM memory engine. Instead, it should add an experimental skill that uses the agent's own reasoning with existing memex tools. The feature should be gated behind a user config flag and default to the current memex behavior.

## Problem

Today, memory quality depends heavily on each agent remembering the right process. A strong agent may create atomic cards, search related cards, add meaningful wikilinks, and update stale cards. A weaker or rushed agent may save isolated notes, miss duplicates, or add no links.

A-MEM shows a useful structure for this process, but its reference implementation assumes a more integrated memory engine with LLM calls inside the memory system. Memex should preserve its simpler architecture: markdown files are the source of truth, and agents perform reasoning outside the core storage layer.

## Goals

- Provide a standard agent workflow for create, metadata enrichment, retrieval, link decisions, and controlled updates.
- Keep all agentic memory behavior experimental and default-off.
- Reuse existing memex storage, search, read, write, retro, and wikilink primitives.
- Avoid adding a mandatory LLM provider or vector database to core memex.
- Make review easy by shipping small PRs with clear product and technical boundaries.

## Non-Goals

- Do not replace `memex_retro` or existing skills in the first version.
- Do not automatically mutate existing cards without an explicit preview step.
- Do not add heuristic auto-linking based only on token overlap or embedding score.
- Do not add a core LLM orchestration engine to `src/commands` in the first version.
- Do not change default card behavior for users who do not enable the feature flag.

## Users

Primary users are AI coding agents using memex through CLI, MCP, or bundled skills. The human user benefits from better memory quality without manually curating every card.

Secondary users are maintainers reviewing memex PRs. The feature must be easy to reason about and must not make the core product feel like a different system.

## User Experience

A user opts in through `.memexrc`:

```json
{
  "experimental": {
    "agenticMemory": true
  }
}
```

When enabled, an agent may use the experimental agentic memory skill after meaningful work or when explicitly asked to save knowledge. The skill guides the agent through this flow:

```text
observe content
-> draft atomic card
-> enrich semantic metadata
-> retrieve candidate neighbors
-> decide create/update/link/skip
-> preview proposed memory changes
-> write through existing memex tools
```

When disabled, agents continue using the existing recall/retro workflow.

## Functional Requirements

### FR1: Feature Flag

The experimental workflow must be gated by `.memexrc.experimental.agenticMemory === true`. Default behavior is off.

### FR2: Skill-Based Reasoning

The first version should be implemented as a skill workflow, not as automatic core logic. The skill can use the agent's reasoning to decide card shape, metadata, links, and updates.

### FR3: Atomic Note Creation

The agent must convert raw observations into one or more atomic Zettelkasten cards. Each card should contain one reusable insight, not a transcript or mechanical summary.

### FR4: Metadata Enrichment

The agent should generate metadata candidates: `keywords`, `context`, and `tags`. For v1, these are stored as comma-separated string scalars in frontmatter (e.g. `keywords: retrieval, metadata, agentic memory`). This avoids YAML array round-trip issues with the current `stringifyFrontmatter` implementation, which calls `String(value)` on all values.

### FR5: Candidate Retrieval

The skill should retrieve related cards before writing. It may use semantic search when configured, and must fall back to keyword search when semantic search is unavailable.

### FR6: Link Decisions

Links must be chosen by the agent after reading candidate cards. Embedding similarity or keyword match is only a candidate signal, not an automatic link decision.

### FR7: Controlled Updates

Updating existing cards is allowed only after a preview and explicit user confirmation. The agent must preserve existing frontmatter and explain why the update is preferable to creating a new card.

For new-card-only writes, the skill still produces a preview showing the proposed card content, but proceeds without requiring explicit user confirmation. The preview serves as an audit trail rather than a gate.

### FR8: No Silent Merge

Merge/archive operations are out of scope for the first version unless the user explicitly approves them.

## Acceptance Criteria

- With the flag disabled, existing recall/retro/search behavior is unchanged.
- With the flag enabled, the skill can guide an agent through create, retrieve, link, and previewed update decisions using existing memex tools.
- The first implementation PR is reviewable as either docs-only or a small config/skill change.
- Tests cover feature flag parsing before any runtime behavior depends on it.
- Documentation clearly explains that A-MEM is inspiration for the workflow, not a replacement for memex architecture.

Note: PR 4 (retrieval substrate improvements) is a separate, non-agentic general search improvement. It is not gated by the `agenticMemory` flag and is not part of the agentic memory acceptance criteria. It has its own scope and tests independent of this feature.

## Risks

- Scope creep: reviewers may reject a large PR that mixes config, skill, retrieval changes, and write behavior.
- Over-linking: agents may add weak links if the skill does not require relationship explanations.
- Metadata drift: new fields may become inconsistent if serialization and update rules are unclear.
- Hidden mutation: updating existing cards can damage memory quality if not previewed.

## Review Strategy

Ship in small PRs:

1. PRD and dev spec only.
2. Feature flag config parsing and tests only.
3. Experimental skill with default-off guard only.
4. Optional general retrieval substrate improvements (separate scope, not gated by `agenticMemory` flag, own tests).
5. Optional helper tool or MCP workflow after the skill proves useful.

## Resolved Questions

1. **Should metadata fields live in frontmatter or card body?** — Frontmatter, as comma-separated string scalars for v1. This works within the current `stringifyFrontmatter` implementation which calls `String(value)` on all values, making array round-trips lossy.

2. **Should feature flag status be exposed through a CLI command?** — No CLI helper in v1. The skill reads `.memexrc` directly using file inspection.

3. **Should the skill always preview changes?** — New-card-only writes produce a preview for audit but proceed without requiring user confirmation. Updates to existing cards always require explicit user confirmation after preview (per FR7).

4. **Should enhanced retrieval be behind the same flag or a general improvement?** — General improvement. PR 4 is a separate, non-agentic search improvement with its own scope and tests. It is not gated by `experimental.agenticMemory` and is not part of the agentic memory acceptance criteria. This keeps the agentic memory scope clean: no core search behavior changes can land under the agentic memory umbrella while the flag is false.
