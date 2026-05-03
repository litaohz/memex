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
  -> parses .memexrc including experimental.agenticMemory
  -> default false

optional later:
src/commands/search.ts / src/lib/embeddings.ts
  -> enhanced retrieval behind a flag or general semantic improvement
```

## Codebase Reality Check

The following gaps were identified by reviewing the current codebase. The dev spec must account for them to avoid referencing non-existent features.

### Gap 1: No config module exists

There is no `src/lib/config.ts`. The CLI resolves `MEMEX_HOME` inline in `cli.ts` (line 20) and passes the path directly. `.memexrc` is never parsed for configuration beyond what the MCP server may do. **PR 2 must create `src/lib/config.ts` from scratch.**

### Gap 2: No `--semantic` or `--compact` search flags

`src/commands/search.ts` and the CLI only support `--limit`. There is no semantic search or compact output mode. **The skill (PR 3) must use plain `memex search "<query>"` with keyword matching. PR 4 may add these flags later.**

### Gap 3: `stringifyFrontmatter` flattens arrays

`src/lib/parser.ts` line 22 uses `String(value)` to serialize all frontmatter values. Arrays become `"a,b,c"` strings. **V1 must store metadata (`keywords`, `context`, `tags`) as plain strings, not arrays.** This is the chosen approach for the initial implementation.

### Gap 4: No `.memexrc` location precedence

The CLI only checks `MEMEX_HOME` env var or falls back to `~/.memex`. There is no upward `.memexrc` discovery. **PR 2 should implement `.memexrc` discovery or document that `MEMEX_HOME` must be set.**

## Feature Flag

Add config shape:

```typescript
export interface MemexConfig {
  nestedSlugs: boolean;
  experimental?: {
    agenticMemory?: boolean;
  };
}
```

Parsing rules:

- `experimental` must be an object.
- `agenticMemory` is enabled only when the value is exactly `true`.
- Missing, false, null, string, or number values are treated as disabled.
- Existing config keys remain backward compatible.

Example `.memexrc`:

```json
{
  "experimental": {
    "agenticMemory": true
  }
}
```

### Config module location

Create `src/lib/config.ts` with:

1. A `loadConfig(memexHome: string)` function that reads `<memexHome>/.memexrc`.
2. A `isAgenticMemoryEnabled(config: MemexConfig)` helper that returns `true` only for `config.experimental?.agenticMemory === true`.
3. Graceful fallback: missing `.memexrc` or missing keys return defaults.

## Skill Guard

The skill must start with a guard step:

1. Locate memex home: `MEMEX_HOME` env var, then `~/.memex`.
2. Read `.memexrc`.
3. Continue only if `experimental.agenticMemory === true`.
4. If disabled, fall back to existing `memex-retro` behavior and do not perform agentic update workflow.

A later PR may add a small CLI helper such as `memex config get experimental.agenticMemory`, but the first skill can self-gate through file inspection.

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

- `context`: one sentence explaining domain and purpose (string)
- `keywords`: 3-8 salient terms, comma-separated (string)
- `tags`: broad categories, comma-separated (string)

**V1 decision: store as plain string frontmatter fields.** This avoids the `stringifyFrontmatter` array flattening issue. A future PR may improve YAML serialization and migrate to proper arrays.

### Step 4: Candidate Retrieval

Run retrieval before writing:

```text
memex search "<topic query>"
```

> Note: `--semantic` and `--compact` flags do not exist yet. Use plain keyword search with the default limit of 10.

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
| link   | New card is needed and candidate cards are meaningfully related |
| skip   | Insight is duplicate, too obvious, or not durable |

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

### Step 7: Write

Use existing memex write paths. Preserve frontmatter on update.

For a new card, required fields remain:

```yaml
---
title: <title>
created: <today's date YYYY-MM-DD>
source: <client or agent>
category: <optional>
---
```

Optional experimental fields (v1, stored as strings):

```yaml
context: <one sentence>
keywords: <comma-separated terms>
tags: <comma-separated categories>
```

Links must be embedded in prose with relationship explanations, not appended as an unstructured list.

### Step 8: Verify

After writing:

- read the written card
- confirm required frontmatter exists
- confirm wikilinks are syntactically valid
- optionally run `memex links <slug>` to check graph connectivity

## Frontmatter Serialization Constraint

Current `stringifyFrontmatter` (`src/lib/parser.ts`) stringifies values with `String(value)`. Array values degrade into comma-separated strings on write. Therefore v1 stores all experimental metadata as simple strings.

**Chosen v1 approach: store metadata as simple string fields.**

```yaml
context: One sentence summary.
keywords: retrieval, metadata, agentic memory
tags: memory, workflow, experimental
```

A separate PR may improve `stringifyFrontmatter` to handle arrays properly, after which the skill can be updated to use structured arrays.

## Testing Strategy

### Config Tests

Add tests in `tests/lib/config.test.ts`:

- reads `experimental.agenticMemory: true`
- treats false/missing/invalid values as disabled
- preserves existing config fields
- handles missing `.memexrc` gracefully

### Skill Tests

If the repo has a skill packaging test, add coverage that the new skill exists and contains the feature flag guard. If not, keep skill validation manual in the first PR.

### Behavior Tests

No core behavior should change when the flag is disabled. Existing tests must pass without updating snapshots for default behavior.

## PR Slicing

### PR 1: PRD and Dev Spec (this PR)

Files only:

- `docs/superpowers/specs/2026-05-03-agentic-memory-skill-prd.md`
- `docs/superpowers/specs/2026-05-03-agentic-memory-skill-dev-spec.md`

### PR 2: Feature Flag Parsing

Files:

- `src/lib/config.ts` (new)
- `tests/lib/config.test.ts` (new)
- update to `src/cli.ts` to use new config loader

No behavior change.

### PR 3: Experimental Skill

Files:

- `skills/memex-agentic-memory/SKILL.md` (new)
- optional plugin packaging references if required
- docs update describing opt-in behavior

No core command changes.

### PR 4: Retrieval Substrate (optional)

Improve search command to support `--compact` output. Consider `--semantic` flag if an embedding provider is available. Decide whether this is general or flag-gated.

### PR 5: Helper Workflow (optional)

Add a read-only helper that returns candidate neighbors for agent review. It must not write links or update cards.

## Open Implementation Questions

1. Should the feature flag be checked only by the skill, or should core memex expose a config status command?
2. Should semantic search automatically use enhanced embedding text, or should that require the experimental flag?
3. Should the skill write immediately after preview for new cards, or wait for explicit user confirmation every time?
