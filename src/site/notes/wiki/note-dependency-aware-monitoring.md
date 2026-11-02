---
{"dg-publish":true,"permalink":"/wiki/note-dependency-aware-monitoring/","title":"Dependency-aware monitoring & response — design (draft)","tags":["security","monitoring","incident-response","design","draft"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Dependency-aware monitoring & response — design (draft)","repo":"hydration-node","paths":["ai_skills/circuit-breaker-incident/SKILL.md","pallets/circuit-breaker/src/lib.rs","pallets/parameters/src/lib.rs","pallets/ice/src/lib.rs"],"depends_on":["note-dependency-registry","note-emergency-levers","runbook-collateral-damage-playbooks","source-onchain-monitoring-landscape","sdk-next","papi","pallet-circuit-breaker"],"tags":["security","monitoring","incident-response","design","draft"],"last_updated":"2026-11-02"}}
---


# Dependency-aware monitoring & response — design (draft)

**TL;DR:** A Hydration-native version of what Blockaid / Hypernative sell (see [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]]): watch the things we depend on, map any incident to **our** exposure through a dependency registry, and hand the responder a **pre-decided, pre-encoded** playbook. Most of the response levers already exist in the runtime ([[wiki/note-emergency-levers\|note-emergency-levers]]); what is missing is the registry, the external watchers, and the decisions written down. Status: **design draft**. The phased build plan, gap list and external patterns live in [[wiki/note-security-hardening-plan\|note-security-hardening-plan]]. The concrete list of signals now lives in [[wiki/note-monitoring-plan\|note-monitoring-plan]].

## Why

- Most of what we list is bridged or issued elsewhere ([[wiki/note-dependency-registry\|note-dependency-registry]]: Snowbridge ~19 assets, Wormhole ~11, Bifrost LSTs, Asset Hub stables). An incident upstream hits a *group* of assets at once, plus everything derived from them (aTokens, gTokens, stablepool shares, HOLLAR).
- Today: snakewatch alerts on our own fuses (`AssetLockdown`) and upstream `ai_skills/circuit-breaker-incident` triages them. Nothing watches upstream (bridges, issuers, oracles) and nothing says "this incident → these assets → these calls".

## Components

| # | Component | Does | Built from |
|---|---|---|---|
| 1 | **Dependency registry** | asset → upstream dependency → oracle → venues → $ exposure, transitively | YAML in a repo + generator over chain state ([[wiki/note-dependency-registry\|note-dependency-registry]]) |
| 2 | **Internal watchers** | on-chain signals on Hydration | [[wiki/papi\|papi]] subscriptions / [[wiki/sdk-next\|sdk-next]] |
| 3 | **External watchers** | upstream signals (bridge contracts, origin chains, issuers) + incident feeds | EVM log subscriptions, parachain RPCs, optional vendor feed |
| 4 | **Correlator** | signal → dependency → affected assets (incl. derived) → blast radius → playbook | registry lookup |
| 5 | **Playbook store** | per-asset cards with encoded calls, undo calls, dry-run results | [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] |
| 6 | **Response** | alert war room with the motion ready; optional narrow auto-actions | TC motion tooling; Chopsticks for dry runs |

## Watcher catalogue (first cut)

**Internal (Hydration):**
- `CircuitBreaker.AssetLockdown` / `WithdrawLockdownTriggered` (already covered by snakewatch — consume, don't duplicate)
- `Parameters.XcmSuspended` / `XcmSendSuspended` / `XcmExecutionSuspended` and `ICE.SolverModeSet` / `RoutingUpdated` — someone pulled a chain-level lever
- Issuance jump per asset vs registry baseline (catches mints the deposit fuse may not see, e.g. NTT)
- `EVMAccounts.NttMinters` changes; NTT manager admin events on Hydration
- DIA feed age vs 24 h heartbeat; DIA vs EMA deviation per MM reserve
- Unexpected DIA update signer (not `0x33a5e905…` / `0xff0c6240…`)
- Stable depeg: Omnipool / stablepool spot vs $1 for HOLLAR, USDT, USDC variants
- MM: utilisation spikes / large borrows against a single collateral class
- Large Omnipool `SELL` of one asset across consecutive blocks (circuit breaker only limits per block)
- Governance: unexpected referenda / TC motions touching levers

**External (upstream):**
- Snowbridge Gateway (Ethereum): admin / upgrade events, escrow balance vs our supply of each Snowbridge asset
- NTT managers (Ethereum): `owner`/`pauser` changes, pause events, rate-limit changes, escrow balance vs Hydration-side supply (4-of-7 Safe `0xd557…81e7`, see [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]])
- Bifrost: vDOT redemption rate vs market price
- Origin-chain liveness for each parachain reserve (block production, sunset announcements)
- Issuer actions: USDT/USDC blacklist events on bridge escrows
- Ecosystem incident feeds (vendor or public) → keyword/address match against the registry

## Response policy (proposal)

- **Default: human-in-the-loop.** Correlator posts to the war room: what happened, which assets, $ exposure, which playbook, and the **ready-to-submit TC motion** (encoded + dry-run result).
- **Auto-actions: none at launch.** If added later, only reversible, asset-scoped §A step 2 (tradability flags) for registry-listed assets, through a dedicated narrow origin — needs a governance decision. Copy BlockSec's pattern: the automated key can only submit those exact calls.
- **Latency budget:** the levers are TC-speed at best. Measure alert → motion closed in drills; that number decides whether automation is worth the governance cost.

## Build order (MVP → v1)

1. **Registry generator** over chain state → per-dependency matrices (asset × venue × $ exposure) for all groups. *Outcome: the "write it all down" list, regenerated on demand.*
2. **Playbook cards** for the top-N assets by blast radius, with encoded calls dry-run on Chopsticks.
3. **Internal watchers** (oracle age/deviation, issuance jumps, depegs) → alert routing, reusing snakewatch's channel.
4. **External watchers** for Snowbridge + NTT escrows (highest $ concentration).
5. **Correlator** joining 3–4 to the registry; war-room message with the motion attached.
6. Quarterly drills; consider a vendor feed or a vendor for the EVM side (Hypernative / Hexagate ecosystem programs) — see [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]].

## Decisions needed

- Owner and on-call for the system; where alerts land.
- Aave role of `EmergencyAdminAccount` (decides MM response speed) — [[wiki/note-emergency-levers\|note-emergency-levers]].
- Whether any auto-action is acceptable, and under which origin.
- Build vs buy for the EVM-side watchers.
- Where the registry lives (this vault for docs; a repo for YAML + generator).

## Sources

- [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]]
- [[wiki/note-dependency-registry\|note-dependency-registry]], [[wiki/note-emergency-levers\|note-emergency-levers]], [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]]
- [[wiki/note-ai-skills\|note-ai-skills]], [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]], [[wiki/pepl-worker\|pepl-worker]]
