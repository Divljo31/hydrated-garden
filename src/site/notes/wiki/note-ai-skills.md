---
{"dg-publish":true,"permalink":"/wiki/note-ai-skills/","title":"note: ai_skills/ — agent skills shipped inside hydration-node","tags":["ai-skills","agents","security-audit","incident-response","tooling","meta"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"note: ai_skills/ — agent skills shipped inside hydration-node","repo":"hydration-node","paths":["ai_skills/hydration_cl0wdit/SKILL.md","ai_skills/hydration_cl0wdit/VERSION","ai_skills/hydration_cl0wdit/references/","ai_skills/circuit-breaker-incident/SKILL.md","ai_skills/circuit-breaker-incident/scripts/","AGENTS.md","CLAUDE.md"],"symbols":["hydration_cl0wdit","circuit-breaker-incident"],"tags":["ai-skills","agents","security-audit","incident-response","tooling","meta"],"last_updated":"2026-08-15"}}
---


# note: ai_skills/ — agent skills shipped inside hydration-node

**TL;DR:** As of the 2026-08-15 sync, hydration-node ships its own agent skills in `ai_skills/` — a parallelised Substrate security-audit workflow (`hydration_cl0wdit`) and a circuit-breaker incident runbook (`circuit-breaker-incident`). They are agent-agnostic: `.claude/skills/` and `.codex/skills/` are now symlinks into `ai_skills/`. **Read this before writing an audit or lockdown-triage page here — upstream already owns that workflow.**

## Why this matters to this vault

This vault is a context cache for a Claude agent working on Hydration. Upstream now carries *executable* agent workflows in-repo. Division of labour:

| Concern | Owner |
|---|---|
| "What is X, where does it live, what calls it" | this vault (`wiki/`) |
| "Run a security audit over this diff/PR" | upstream `ai_skills/hydration_cl0wdit` |
| "An asset just got locked down — what happened" | upstream `ai_skills/circuit-breaker-incident` |

Do **not** re-derive audit heuristics or lockdown triage steps as vault pages. Point at `raw/hydration-node/ai_skills/…` instead — those files move with the code and the vault would go stale against them.

## Layout

```
raw/hydration-node/
├── ai_skills/
│   ├── hydration_cl0wdit/        # security audit (VERSION 0.2.0)
│   │   ├── SKILL.md
│   │   ├── VERSION
│   │   └── references/
│   │       ├── judging.md                    # 4 sequential validation gates
│   │       ├── known-false-positives.md      # FP-001…, drop-silently list
│   │       ├── report-formatting.md
│   │       ├── attack-vectors/               # hydration + substrate-1/-2 vector catalogues
│   │       └── hacking-agents/               # 10 auditor personas + shared-rules
│   └── circuit-breaker-incident/
│       ├── SKILL.md
│       ├── references/price-from-omnipool.md
│       └── scripts/{query-lockdown.cjs, get-trigger-events.cjs, scan-deposits.js,
│                    generate-tc-unlock.js}
├── .claude/skills/hydration_cl0wdit -> ../../ai_skills/hydration_cl0wdit   # symlink
├── .codex/skills/hydration_cl0wdit  -> ../../ai_skills/hydration_cl0wdit   # symlink
├── CLAUDE.md   # "Shared AI skills" section points at ai_skills/
└── AGENTS.md   # Codex entrypoint, defers to CLAUDE.md, maps Claude tool names → Codex
```

`hydration_cl0wdit` moved out of `.claude/skills/` in this window; the directory was not created from scratch. `circuit-breaker-incident` is genuinely new.

## `hydration_cl0wdit` — security audit

Orchestrator skill: builds file bundles under a `mktemp -d /tmp/audit-XXXXXX`, then runs **11 parallel audit agents** over the same production source, each with a different lens.

| Bundle | Lens |
|---|---|
| 1–4 | vector scan against `hydration-attack-vectors.md`, `substrate-attack-vectors{,-1,-2}.md` |
| 5 | math / precision |
| 6 | access control |
| 7 | economic security |
| 8 | execution trace |
| 9 | invariants |
| 10 | first principles |
| 11 | test / bench / mock code + a production dispatchable summary |

Then a single dedup + gate pass: group by `Pallet | function | bug-class`, detect composite chains, run every finding through the four gates in `judging.md` (**Refutation → Reachability → Trigger → Impact**; `UNCERTAIN` counts as `ALLOWS`), drop anything matching `known-false-positives.md`, and emit per `report-formatting.md`.

Invocation: default scans all `.rs` excluding `tests/`, `benchmarking/`, `mock/` and `*test*.rs` / `*mock*.rs` / `*bench*.rs`; `--pr <ref>` audits a PR via the GitHub API through `WebFetch` (explicitly **not** `gh`); `--file-output` is off by default and never writes a report unless passed.

Self-versioning: `VERSION` (currently `0.2.0`) is compared against `https://raw.githubusercontent.com/galacticcouncil/hydration-node/main/ai_skills/hydration_cl0wdit/VERSION` at run time and warns when local is behind.

Useful even outside a full audit run: `references/known-false-positives.md` documents patterns Hydration maintainers consider settled — e.g. FP-001 (missing `#[transactional]` on a `#[pallet::call]` dispatchable is *not* a bug; FRAME v2 wraps dispatchables automatically — only flag it in hooks / offchain workers), FP-002 (`saturating_*` with documented intent).

## `circuit-breaker-incident` — lockdown triage

Runbook for snakewatch `AssetLockdown` alerts, covering the XCM deposit fuse (issuance increase), trade-volume limits and liquidity limits — see [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]].

Expected event sequence at the trigger block: `messageQueue.Processed` → `tokens.Deposited` → `circuitBreaker.AssetLockdown`, where `tokens.Reserved` equals the over-limit excess and `messageQueue.Processed` carries the XCM origin (`{"sibling":2004}` = Moonbeam, `{"sibling":1000}` = Asset Hub Polkadot).

Scripts must be run **from `hydration-node/scripts/mint-limit/`** — that is where `@polkadot/api` and `@galacticcouncil/sdk` are installed. `.cjs` = CommonJS, `.js` = ESM (`scripts/mint-limit/` has `"type": "module"`). `generate-tc-unlock.js` emits the TC proposal hex that lifts a lockdown and raises the limit.

## Related upstream docs

`CLAUDE.md` also indexes operator runbooks the vault should defer to rather than duplicate: `scripts/mint-limit/README.md`, `scripts/dca-monitor/README.md`, `scripts/onchain-routes/README.md`, `integration-tests/README.md`.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
