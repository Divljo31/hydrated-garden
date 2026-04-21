---
{"dg-publish":true,"permalink":"/claude/","title":"div-wiki — Schema","dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"title":"div-wiki — Schema"}}
---


# div-wiki — Schema

## Purpose

This vault is a **persistent context cache for Claude Code** working on the [[wiki/hydration\|hydration]] protocol. Another Claude agent (typically Claude Code in VS Code) will read these pages to bootstrap context when working on Hydration codebases. Pages are optimized for fast retrieval, direct source pointers, and scannable structure — not narrative readability.

The vault owner is **div**, a core contributor to Hydration. Raw sources (cloned repos, uploaded docs) live in `raw/`. Curated pages live in `wiki/`. This file (`CLAUDE.md`) defines how the wiki is structured and how to extend it.

## Entry points for Claude Code

When another Claude reads this vault for a task, it should start here:

1. `wiki/overview.md` — 30-second briefing on what's indexed
2. `wiki/routing.md` — task → pages cheat sheet ("I need to X → read Y, Z")
3. `wiki/index.md` — full routing table organized by repo and domain

From there, follow wikilinks to land on specific pallet / package / concept pages, each of which points to exact file paths in `raw/`.

## Directory layout

```
div-wiki/
├── raw/                           # Source documents — IMMUTABLE
│   ├── hydration-general-context.md
│   ├── omnipool-deep-context.md
│   ├── sdk/                       # Cloned github.com/galacticcouncil/sdk
│   ├── hydration-node/            # Cloned github.com/galacticcouncil/hydration-node
│   └── assets/
├── wiki/                          # Curated pages — Claude owns this layer
│   ├── index.md                   # Routing table
│   ├── overview.md                # 30-second briefing
│   ├── routing.md                 # Task → pages cheat sheet
│   └── *.md                       # Pallet / package / concept / source pages
├── log.md                         # Append-only activity log
└── CLAUDE.md                      # This file
```

## Page types

Each page has a `type:` in its frontmatter. Allowed values:

| type | Purpose | Filename convention |
|---|---|---|
| `source` | Summary of a raw source (repo clone, doc, article) | `source-<slug>.md` |
| `pallet` | Substrate pallet reference (types, storage, extrinsics, hooks) | `pallet-<name>.md` |
| `package` | TypeScript / Rust package reference | `<package-name>.md` |
| `protocol` | Protocol-level entity (Omnipool, HOLLAR, HDX, etc.) | `<name>.md` |
| `concept` | Shared concept (IL, XCM, dynamic fees) | `<concept>.md` |
| `auditor` | Audit firm / bug bounty | `<name>.md` |
| `runbook` | Operational playbook | `runbook-<slug>.md` |
| `note` | Tribal knowledge / gotcha | `note-<slug>.md` |
| `index` | The index file | `index.md` |
| `overview` | The overview file | `overview.md` |
| `routing` | The routing table | `routing.md` |
| `log` | The activity log | `log.md` |

Filenames: lowercase kebab-case, no spaces.

## Frontmatter

Every page must have YAML frontmatter. Fields are additive — include the ones that apply. Arrays use flow or block style consistently.

### Common fields (all page types)

```yaml
---
type: <page-type>
title: "<Display Title>"
tags: [tag1, tag2]
last_updated: YYYY-MM-DD
---
```

### Pallet page

```yaml
---
type: pallet
title: "pallet-omnipool"
repo: hydration-node
paths:
  - pallets/omnipool/src/lib.rs
  - pallets/omnipool/src/types.rs
symbols: [Pallet, Config, AssetState, sell, buy, add_liquidity, remove_liquidity]
traits_impl: [OmnipoolHooks, AssetRegistryInspect]
depends_on: [pallet-ema-oracle, pallet-circuit-breaker, pallet-asset-registry]
tags: [amm, omnipool, runtime, rust, substrate]
last_updated: 2026-04-13
---
```

### Package page (TypeScript / Rust lib)

```yaml
---
type: package
title: "sdk-next"
repo: sdk
paths:
  - packages/sdk-next/src/index.ts
  - packages/sdk-next/src/pool
key_exports: [createSdkContext, TradeRouter, TradeScheduler]
tags: [sdk, trading, typescript]
last_updated: 2026-04-13
---
```

### Source page

```yaml
---
type: source
title: "hydration-node codebase"
source_kind: repo_clone          # or: document | article | transcript
raw_path: raw/hydration-node/    # path relative to vault root
upstream: https://github.com/galacticcouncil/hydration-node
cloned_at: 2026-04-13
produces_pages: [pallet-omnipool, pallet-stableswap, ...]
tags: [hydration, runtime, substrate]
last_updated: 2026-04-13
---
```

### Protocol / concept / auditor / runbook / note

Use common fields plus `paths:` and `symbols:` wherever useful. For `concept`, include a `code_locations:` section in the body with file paths across repos.

## Writing style

- **TL;DR first.** Every page opens with a 1–3 line TL;DR summarizing purpose and scope.
- **Facts over prose.** Prefer tables and bullet lists. Avoid "according to source X" attribution — the Sources section handles it.
- **Code pointers.** Reference source like `` `pallets/omnipool/src/lib.rs` `` (path relative to repo root). When relevant, name the symbol: `` `pallet-omnipool → Pallet::sell()` ``.
- **Code excerpts.** Short excerpts (≤30 lines) are permitted and encouraged when they pin down a `Config` trait, a key function signature, a storage item definition, or a complex type. Prefix every excerpt with a comment indicating the source path.
- **Wikilinks.** Always wikilink entities / concepts / pallets / packages that have their own page: `[[pallet-omnipool]]`. Unresolved links are fine — they flag pages that should be created.
- **Sources section.** Every page ends with a `## Sources` section listing the source pages that back it (as wikilinks).

## Workflows

### Ingest a new codebase (repo clone)

1. Clone into `raw/<repo-name>/` (use `--depth=1` unless history is needed).
2. Read root `Cargo.toml` / `package.json`, `README.md`, top-level `CLAUDE.md` (if present), and any architecture docs.
3. Map the structure: list crates / packages / modules.
4. Create `wiki/source-<repo-name>-codebase.md` (type: `source`) with:
   - TL;DR
   - File tree overview (top 2–3 levels)
   - Inventory of pallets / packages with one-line descriptions
   - Links to follow-on pages to be created
5. For each logical unit (pallet, package, major module), create a dedicated page with:
   - Frontmatter (with `paths:` and `symbols:`)
   - TL;DR
   - Role in the protocol
   - Key types / Config trait / storage items (with short excerpts)
   - Extrinsics / public API (name + one-line summary)
   - Hooks / lifecycle
   - Integration points (who calls it, what it calls)
   - Cross-references to concept pages
   - `## Sources` section
6. Update `wiki/index.md` — add new pages under the appropriate repo / domain section.
7. Update `wiki/overview.md` — add/extend the relevant section.
8. Update `wiki/routing.md` — add task→page routes that involve the new content.
9. Append to `log.md`.

### Ingest a document / article

1. Save the file under `raw/` (or capture pasted text as markdown).
2. Create `wiki/source-<slug>.md` (type: `source`).
3. For each entity / concept referenced: create or update the page, add to `paths:` if applicable, add to Sources.
4. Update `index.md`, `overview.md`, `log.md`.

### Query

1. Read `wiki/routing.md` and `wiki/index.md` to find relevant pages.
2. Read the identified pages — they point to `raw/` file paths and symbols.
3. If needed, grep / read the actual source under `raw/` to answer precise code-level questions.
4. Answer the user with citations (wikilinks + file paths).
5. If the answer is reusable, offer to capture it as a `note-<slug>.md` or `runbook-<slug>.md`.

### Lint

1. **Wikilink integrity.** For every `[[link]]` in `wiki/`, verify the target file exists.
2. **Frontmatter path integrity.** For every page with `paths:`, verify the paths exist under `raw/`.
3. **Symbol integrity (spot check).** For a sample of pages, verify listed `symbols:` appear in the referenced files.
4. **Orphan check.** Flag pages with no inbound wikilinks.
5. **Staleness.** Flag pages whose `last_updated` is older than a threshold and whose source repos have moved.
6. Report findings, fix with user approval, append to `log.md`.

## Special files

### `wiki/index.md`
The full routing table. Organized by:
- **Start here** — pointers to `overview.md`, `routing.md`, and top-level pages
- **By repo** — `hydration-node`, `sdk`, (future: `hydration-ui`)
- **By domain** — AMM, cross-chain, stablecoin, governance, risk controls, auditors
- **Source pages**

Each entry: `- [[page-name]] — one-line description`. Keep one-liners keyword-dense.

### `wiki/overview.md`
30-second briefing. Reads linearly top to bottom:
- What the vault indexes and where
- The three protocol pillars (Trading, Lending, Stable Value)
- What's in each repo
- Pointers forward to `routing.md` and `index.md`

### `wiki/routing.md`
Task → pages cheat sheet. Format:
```
## <Task category>
- "<Task description>" → [[page-1]], [[page-2]], [[page-3]]
```
Tasks should be phrased the way a developer would describe them out loud.

### `log.md`
Append-only. Format: `## [YYYY-MM-DD] <action> | <title>` then a short paragraph. Actions: `init`, `ingest`, `query`, `lint`, `update`, `schema-change`.

## Rules

- **Never modify `raw/`.** Sources are immutable.
- **Own the `wiki/` layer.** Create, rewrite, split, merge pages as the vault evolves.
- **Always update the index, routing, overview, and log** when creating or substantially editing pages.
- **Prefer file paths + symbol names over prose description.** The downstream consumer is another Claude agent that will grep.
- **When in doubt, create a page.** Easier to merge later than to reconstruct scattered mentions.
- **This schema evolves.** Update this file when conventions change; log it as a `schema-change` entry.
