# Dev Spec: Experimental Agentic Memory Skill

Status: Draft
Date: 2026-05-03
Related PRD: `docs/superpowers/specs/2026-05-03-agentic-memory-skill-prd.md`

## Design Principle

Do not build an A-MEM clone inside memex core. Build an A-MEM-inspired skill workflow that leverages the current agent plus existing memex tools.

The core boundary remains:

```text
MCP tools / CLI -> commands -> lib -> filesystem
```

Agentic reasoning stays in the skill. Core memex remains responsible for configuration, parsing, path safety, atomic writes, search, and sync.

## Proposed Architecture

```text
skills/memex-agentic-memory/SKILL.md
  -> uses memex_search / memex_read / memex_write / memex_retro
  -> performs agent reasoning for create, link, update, skip
  -> gated by .memexrc.experimental.agenticMemory

src/lib/config.ts (new file)
  -> parses experimental.agenticMemory from .memexrc
  -> default false

optional later (separate PR, separate scope):
src/commands/search.ts / src/lib/embeddings.ts
  -> general retrieval text improvements, NOT gated by agenticMemory
```

## Feature Flag

The codebase currently has no `MemexConfig` interface or `.memexrc` parser outside of sync configuration (`readSyncConfig` in `src/lib/sync.ts`). The new `src/lib/config.ts` must introduce general `.memexrc` parsing.

The real codebase uses per-field type checks (see `src/lib/sync.ts` for the pattern). The new config parser should follow the same style: read JSON, validate each field individually with strict type checks.

Proposed config shape for `.memexrc`:

```json
{
  "nestedSlugs": true,
  "experimental": {
    "agenticMemory": true
  }
}
```

Parsing rules:

- `experimental` must be a plain object (not null, not array).
- `agenticMemory` is enabled only when the value is exactly `true` (boolean).
- Missing, false, null, string, or number values are treated as disabled.
- Existing config keys remain backward compatible.
- The parser should use `parseExperimental` or similar per-section helpers following the pattern established in the codebase.

Note: The codebase has ~15 potential config fields across sync, formatting, and other concerns. The simplified two-field `MemexConfig` snippet in the master issue seed material is illustrative, not a real interface. PR 2 should define the actual interface after auditing existing config usage.

## Skill Guard

The skill must start with a guard step:

1. Locate memex home using the same precedence as core memex where possible: `MEMEX_HOME`, upward `.memexrc`, then `~/.memex`.
2. Read `.memexrc`.
3. Continue only if `experimental.agenticMemory === true`.
4. If disabled, fall back to existing `memex-retro` behavior and do not perform agentic update workflow.

The skill reads `.memexrc` directly via file inspection. No CLI helper command is needed for v1.

## Skill Workflow

### Step 1: Input Triage

Identify whether there is a memory-worthy insight. Skip if the content is temporary, obvious, or not reusable.

### Step 2: Draft Atomic Card

Produce one card per insight:

- short title
- kebab-case slug
- one atomic body
- explicit project/domain context when needed

### Step 3: Metadata Draft

Generate candidate metadata:

- `context`: one sentence explaining domain and purpose
- `keywords`: 3-8 salient terms, comma-separated string
- `tags`: broad categories useful for retrieval, comma-separated string

V1 serialization strategy (resolved): store all metadata as comma-separated string scalars in frontmatter. The current `stringifyFrontmatter` in `src/lib/parser.ts` calls `String(value)` on all values (line 22), which makes array round-trips lossy. Comma-separated strings avoid this issue and are readable in both YAML and plain text contexts.

Example:

```yaml
---
title: Retrieval-augmented card linking
created: 2026-05-03
source: retro
category: memory
context: Agentic memory workflow for linking related Zettelkasten cards
keywords: retrieval, linking, wikilinks, agentic memory
tags: memory, workflow, experimental
---
```

### Step 4: Candidate Retrieval

Run retrieval before writing:

```text
memex search "<topic query>" --semantic --compact --limit 8
```

If semantic search fails or is unavailable:

```text
memex search "<topic query>" --compact --limit 8
```

Read the top candidates that look relevant:

```text
memex read <slug>
```

Recommended limits:

- max candidate searches: 3
- max cards read: 10
- max link hops: 2

### Step 5: Decision

Choose exactly one primary action per insight:

| Action | When |
|--------|------|
| create | No existing card covers the insight |
| update | Existing card covers the same insight but lacks new detail |
| link | New card is needed and candidate cards are meaningfully related |
| skip | Insight is duplicate, too obvious, or not durable |

Merge/archive is excluded from v1 unless the user explicitly asks.

### Step 6: Preview

Before writing, produce a preview:

```text
Planned memory changes:
- create: <slug> (<title>)
- update: <slug> because <reason>
- links: [[a]], [[b]] because <relationship>
- metadata: context/keywords/tags draft
```

For updates to existing cards, the agent must explain why updating is better than creating a new card.

For new-card-only writes, the preview is produced as an audit trail but the skill proceeds to write without waiting for explicit user confirmation. For updates to existing cards, the skill always waits for user confirmation before writing.

### Step 7: Write

Use existing memex write paths. Preserve frontmatter on update.

For a new card, required fields remain:

```yaml
---
title: <title>
created: <YYYY-MM-DD>
source: <client or agent>
category: <optional>
---
```

Links must be embedded in prose with relationship explanations, not appended as an unstructured list.

### Step 8: Verify

After writing:

- read the written card
- confirm required frontmatter exists
- confirm wikilinks are syntactically valid
- optionally run `memex links <slug>` or `memex doctor` in a later implementation phase

## Frontmatter Serialization Constraint

Current `stringifyFrontmatter` (in `src/lib/parser.ts`) stringifies values manually using `String(value)` (line 22). Array values degrade into comma-separated strings on write, making round-trips lossy.

**Resolved for v1:** Store metadata as comma-separated string scalars in frontmatter. This is the only supported path for experimental metadata fields until a separate PR improves structured YAML array support in the parser.

```yaml
context: One sentence summary.
keywords: retrieval, metadata, agentic memory
tags: memory, workflow, experimental
```

## Testing Strategy

### Config Tests

Add tests in `tests/lib/config.test.ts`:

- reads `experimental.agenticMemory: true`
- treats false/missing/invalid values as disabled
- preserves existing config fields

### Skill Tests

If the repo has a skill packaging test, add coverage that the new skill exists and contains the feature flag guard. If not, keep skill validation manual in the first PR.

### Behavior Tests

No core behavior should change when the flag is disabled. Existing tests must pass without updating snapshots for default behavior.

## PR Slicing

### PR 1: PRD and Dev Spec

Files only:

- `docs/superpowers/specs/2026-05-03-agentic-memory-skill-prd.md`
- `docs/superpowers/specs/2026-05-03-agentic-memory-skill-dev-spec.md`

### PR 2: Feature Flag Parsing

Files:

- `src/lib/config.ts` (new file)
- `tests/lib/config.test.ts`
- `docs/ARCHITECTURE.md` config section

No behavior change. The actual `MemexConfig` interface should be defined after auditing existing config usage across the codebase — the simplified snippet in the master issue is illustrative only.

### PR 3: Experimental Skill

Files:

- `skills/memex-agentic-memory/SKILL.md`
- optional plugin packaging references if required
- docs update describing opt-in behavior

No core command changes.

### PR 4: Retrieval Substrate (separate scope)

General search improvement, NOT gated by `experimental.agenticMemory`. This PR improves semantic retrieval text to include existing metadata such as title, category, context, keywords, and tags. It is a standalone search quality improvement with its own tests and review criteria, independent of the agentic memory feature.

This separation ensures that no core search behavior changes can land under the agentic memory umbrella. The `agenticMemory` flag controls only the skill workflow (PRs 2-3), not general search quality.

### PR 5: Helper Workflow

Optional. Add a read-only helper that returns candidate neighbors for agent review. It must not write links or update cards.

## Multica Harness Plan

Use Multica only for small research and review sprints.

- Generator: drafts one doc or small patch per sprint.
- Evaluator: checks for scope creep, missing flag guard, and accidental behavior changes.
- Leader: summarizes the day's findings and proposes the next small PR.

Sprint constraints:

- max 5 files
- max 200 changed lines for code PRs
- docs PRs may exceed 200 lines only when they are the sole change
- no automatic mutation feature without an accepted design doc

## Resolved Implementation Questions

1. **Should the feature flag be checked only by the skill, or should core memex expose a config status command?** — Skill-only for v1. The skill reads `.memexrc` directly. No CLI helper needed.

2. **Should metadata be stored as frontmatter strings or card body?** — Frontmatter, as comma-separated string scalars. This works within the current `stringifyFrontmatter` which calls `String(value)` on all values (lossy for arrays).

3. **Should semantic search automatically use enhanced embedding text?** — Yes, but as a general improvement in PR 4, completely separate from the agentic memory feature. PR 4 is not gated by `experimental.agenticMemory` and has its own scope, tests, and review criteria.

4. **Should the skill write immediately after preview for new cards?** — Yes. New cards proceed after preview without user confirmation. Updates to existing cards always require user confirmation after preview (per FR7).
