# Memex Architecture Reference

> Comprehensive technical documentation for contributors and AI coding agents.
> For quick-start instructions, see the root README.md.

## 1. Project Overview

**Memex** (`@touchskyer/memex`, v0.1.26) is a persistent Zettelkasten memory system for AI coding agents. It stores atomic knowledge cards as markdown files in `~/.memex/cards/`, using `[[wikilinks]]` for bidirectional linking. No vector database, no embeddings required.

**Core philosophy**: Recall → Work → Retro. Every session starts by recalling prior knowledge, ends by saving new insights.

**Repository**: https://github.com/iamtouchskyer/memex
**License**: MIT

## 2. Architecture Layers

```
┌─────────────────────────────────────────────┐
│              Client Layer                   │
│       Claude Code │ VS Code                 │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           MCP Server (src/mcp/)             │
│  10 tools: recall, retro, organize,         │
│  search, read, write, links, archive,       │
│  pull, push                                 │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           Command Layer (src/commands/)      │
│  search, read, write, links,               │
│  archive, serve, sync                      │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           Library Layer (src/lib/)           │
│  CardStore, Parser, Formatter, HookRegistry,│
│  GitAdapter, Config                         │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           Storage (~/.memex/)               │
│  cards/  archive/  .sync.json  .memexrc     │
└─────────────────────────────────────────────┘
```

## 3. Source Code Map

```
src/
├── cli.ts                    # CLI entry point (commander)
├── mcp/
│   ├── server.ts             # MCP server factory, client-aware source tagging
│   └── operations.ts         # High-level MCP tools: recall, retro, organize, pull, push
├── commands/
│   ├── search.ts             # Keyword search, manifest pre-filter
│   ├── read.ts               # Read card by slug
│   ├── write.ts              # Write card (validates frontmatter, updates modified date)
│   ├── links.ts              # Link graph stats (single card or global)
│   ├── archive.ts            # Move card to archive/
│   ├── serve.ts              # Web UI server (serve-ui.html)
│   └── sync.ts               # CLI sync orchestrator (init, pull, push, auto toggle)
├── lib/
│   ├── store.ts              # CardStore: scan, resolve, read, write, archive (atomic writes)
│   ├── parser.ts             # Frontmatter parse/stringify, wikilink extraction
│   ├── formatter.ts          # Output formatters (card list, search result, link stats)
│   ├── hooks.ts              # HookRegistry: pre/post lifecycle hooks
│   ├── sync.ts               # GitAdapter, SyncConfig, autoSync/autoFetch
│   └── config.ts             # .memexrc reader (added by this milestone)
skills/                       # Claude Code skills (bundled in plugin)
├── memex-recall/SKILL.md
├── memex-retro/SKILL.md
├── memex-organize/SKILL.md
├── memex-sync/SKILL.md
├── memex-agentic-memory/SKILL.md  # Experimental (requires agenticMemory flag)
hooks/
└── hooks.json                # Claude Code SessionStart hook
.claude-plugin/
├── plugin.json               # Plugin metadata
└── marketplace.json          # Claude Code marketplace registration
vscode-extension/             # VS Code extension (bundles MCP server)
tests/                        # Vitest test suite
```

## 4. Data Model

### Card Format

File: `~/.memex/cards/<slug>.md`

```yaml
---
title: Short Noun Phrase (<=60 chars)
created: 2025-01-15
modified: 2025-01-16
source: claude-code
category: backend
tags: [typescript, gotcha]
status: conflict
---

Atomic insight in own words, with [[wikilinks]] to related cards.

This connects to [[jwt-revocation]] because stateless tokens
need server-side revocation via [[blacklist-pattern]].
```

**Required fields**: `title`, `created`, `source`
**Auto-managed**: `modified` (updated on every write), `source` (injected by MCP server from clientInfo)

### Slug Rules

- **Format**: kebab-case, lowercase English, 3-60 chars
- **Validation** (`store.ts:validateSlug`):
  - No empty/whitespace-only slugs
  - No reserved chars: `: * ? " < > |`
  - No empty path segments, no `..` traversal
  - Path-safe assertion: must resolve within `cardsDir`
- **Special prefixes**: `adr-*`, `gotcha-*`, `pattern-*`, `tool-*`

### Storage Layout

```
~/.memex/
├── cards/              # Active cards (.md)
├── archive/            # Archived cards
├── .sync.json          # Sync config (remote, auto, lastSync)
├── .memexrc            # User config (JSON)
├── .last-organize      # Timestamp of last organize
└── .git/               # Git repo (if sync initialized)
```

## 5. MCP Tools (10 total)

### High-Level (with hooks)

| Tool | Purpose | Hooks |
|------|---------|-------|
| `memex_recall` | Load prior knowledge at task start. Returns index card or card list. | `pre:recall` (autoFetch) |
| `memex_retro` | Save atomic insight at task end. Auto-injects source, date, syncs. | `pre:retro` (autoFetch), `post:retro` (autoSync) |
| `memex_organize` | Analyze network: orphans, hubs, conflicts, contradiction pairs. | `pre:organize` (autoFetch), `post:organize` (autoSync) |
| `memex_pull` | Pull remote changes. | `pre:pull`, `post:pull` |
| `memex_push` | Push local changes. | `pre:push`, `post:push` |

### Low-Level (no hooks)

| Tool | Purpose |
|------|---------|
| `memex_search` | Full-text keyword search (AND logic) or list all cards |
| `memex_read` | Read card by slug |
| `memex_write` | Write/update card with full content |
| `memex_links` | Link stats (per-card or global) |
| `memex_archive` | Move card to archive |

## 6. Hook System

**Registry** (`src/lib/hooks.ts`): `Map<HookKey, HookFn[]>` where `HookKey = "${Phase}:${Operation}"`.

- **Phase**: `pre` | `post`
- **Operation**: `recall` | `retro` | `organize` | `show` | `pull` | `push` | `init`
- **Behavior**: hooks fail silently (infrastructure, not business logic)

**Default hooks** (registered in `server.ts`):

```
pre:recall   → autoFetch (pull latest)
pre:retro    → autoFetch
pre:organize → autoFetch
post:retro   → autoSync (commit + push if auto=true)
post:organize → autoSync
```

## 7. Sync System

**Adapter**: `GitAdapter` (`src/lib/sync.ts`)

- **Init**: Creates/reuses `memex-cards` GitHub repo via `gh` CLI, or accepts custom URL
- **Pull**: `git fetch origin` → `git merge <remoteBranch> --no-edit`
- **Push**: `git add cards archive` → `git commit` → `git push origin HEAD`
- **Remote detection**: `origin/HEAD` → `origin/main` → `origin/master` → fallback `origin/main`
- **Auto-sync**: Enabled with `memex sync on`. Runs after retro/organize.
- **Offline tolerance**: autoFetch/autoSync silently fail when offline

## 8. Search

### Keyword Search

- AND logic: ALL tokens must match
- Case-insensitive, searches title + body (frontmatter excluded)
- Ranked by token frequency

### Manifest Filters

`--category`, `--tag`, `--author/--source`, `--since`, `--before` (applied as pre-filter before search)

## 9. Platform Integrations

### Claude Code Plugin

- **SessionStart hook** (`hooks/hooks.json`): checks CLI install, runs sync, injects recall/retro reminders
- **5 skills** (on main + this milestone): recall, retro, organize, sync, agentic-memory (experimental)
- **Install**: `/plugin install memex@memex`
- **Marketplace**: `.claude-plugin/marketplace.json`

### VS Code Extension

- **Location**: `vscode-extension/`
- Bundles `@touchskyer/memex` as dependency
- Registers MCP server via `vscode.lm.registerMcpServerDefinitionProvider`
- Node discovery: system PATH → common install paths → NVM (sorted by semver)

## 10. Build & Test

### Build

```bash
npm run build      # tsc → dist/
```

**TypeScript**: ES2022, Node16 module resolution, strict mode, declarations, source maps.

### Dependencies

| Dep | Purpose |
|-----|---------|
| `@modelcontextprotocol/sdk` | MCP server framework |
| `commander` | CLI framework |
| `gray-matter` | YAML frontmatter parsing |
| `zod` | Schema validation (MCP tool inputs) |

### Test

```bash
npm test              # vitest run
npm run test:watch    # vitest watch mode
```

**Coverage**: v8 provider, 70% statement threshold, `src/cli.ts` excluded.

## 11. Configuration Reference

### .memexrc (JSON)

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `nestedSlugs` | boolean | false | Path-preserving slugs |
| `searchDirs` | string[] | — | Extra dirs for `--all` |
| `extraLinkDirs` | string[] | — | Extra dirs whose .md files are valid link targets |
| `experimental` | object | — | Experimental feature flags (see below) |

#### Experimental Flags

The `experimental` field is an optional object for gating features that are not yet stable.

| Flag | Type | Default | Notes |
|------|------|---------|-------|
| `agenticMemory` | boolean | `false` | Enables the A-MEM-inspired agentic memory skill workflow. Only `true` activates; `false`, `null`, missing, or non-boolean values are treated as disabled. |

Example `.memexrc` with experimental flags:

```json
{
  "experimental": {
    "agenticMemory": true
  }
}
```

When `agenticMemory` is enabled, agents may use the `memex-agentic-memory` skill for structured knowledge capture (observe → draft → enrich → retrieve → decide → preview → write → verify). When disabled, agents use the standard `memex-retro` workflow. See `skills/memex-agentic-memory/SKILL.md` for the full skill specification.

### Environment Variables

| Var | Purpose |
|-----|---------|
| `MEMEX_HOME` | Override home dir (default `~/.memex`) |

## 12. Key Implementation Details

### Atomic Writes

`CardStore.writeCard()` writes to `<path>.tmp` then `rename()` — prevents corruption on crash.

### Path Safety

- `assertSafePath()`: resolved path must be within `cardsDir` (or `archiveDir`)
- `validateSlug()`: rejects traversal, reserved chars, empty segments
- Windows normalization: `\` → `/` in slugs

### Client Source Tagging

MCP server intercepts `initialize` handshake, captures `clientInfo.name`, normalizes to kebab-case. Auto-injected into `source` frontmatter on writes via `memex_write` and `memex_retro`.

### Frontmatter Stringification

Custom YAML generation (avoids `js-yaml` block scalars `>-`):
- Special chars quoted with single quotes
- Single quotes escaped: `'` → `''`
- Newlines replaced with spaces
