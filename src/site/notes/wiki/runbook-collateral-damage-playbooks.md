---
{"dg-publish":true,"permalink":"/wiki/runbook-collateral-damage-playbooks/","title":"Runbook — Collateral-damage playbooks (pre-decided pauses per dependency)","tags":["runbook","security","incident-response","playbook","bridges","oracles"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"runbook","title":"Runbook — Collateral-damage playbooks (pre-decided pauses per dependency)","repo":"hydration-node","paths":["pallets/omnipool/src/lib.rs","pallets/stableswap/src/lib.rs","pallets/circuit-breaker/src/lib.rs","pallets/transaction-pause/src/lib.rs","pallets/dispatcher/src/lib.rs","pallets/evm-accounts/src/lib.rs","ai_skills/circuit-breaker-incident/SKILL.md"],"symbols":["set_asset_tradable_state","lockdown_asset","set_global_withdraw_lockdown","set_trade_volume_limit","pause_transaction","dispatch_as_emergency_admin","dispatch_as_aave_manager","clear_ntt_minter","remove_collateral_asset","remove_currency"],"depends_on":["note-emergency-levers","note-dependency-registry","note-dependency-aware-monitoring","tradability-flags","pallet-circuit-breaker","hydration-borrow","pallet-hsm"],"tags":["runbook","security","incident-response","playbook","bridges","oracles"],"last_updated":"2026-11-02"}}
---


# Runbook — Collateral-damage playbooks

**TL;DR:** "Decide now what you pause if one gets hit." One playbook per **failure class** (§A–§E), each a fixed sequence of levers from [[wiki/note-emergency-levers\|note-emergency-levers]], applied to every asset that [[wiki/note-dependency-registry\|note-dependency-registry]] puts behind the failed dependency. Status: **draft** — sequences are proposals until the team signs them off and each step has encoded call data that has been dry-run on a Chopsticks fork.

## Principles

1. **Stop inflow of bad value first, keep exits open for honest users.** Disable `SELL` + `ADD_LIQUIDITY` on the hit asset; leave `BUY` + `REMOVE_LIQUIDITY` unless the asset itself is being drained out.
2. **Narrowest lever first.** Asset-scoped before chain-scoped. `pause_transaction` and the global withdraw lockdown are last resorts.
3. **Collateral before AMM.** Bridged assets concentrate in the money market. An unbacked asset used as MM collateral to borrow good assets is the largest loss path — freeze the reserve in the same motion as the AMM flags.
4. **Every step is pre-encoded.** Call data + origin + who proposes + who votes is written down per asset *before* the incident. During the incident the job is "propose and approve", never "compose".
5. **Reversible only, unless a human decides otherwise.** Anything an auto-responder fires must be undoable by the same origin.

## Severity ladder

| Level | Trigger | Action |
|---|---|---|
| S0 watch | Upstream anomaly, no confirmed loss (e.g. bridge pause announced, oracle stale < heartbeat) | Alert channel; pre-stage TC motion; no on-chain change |
| S1 contain | Confirmed upstream incident affecting a dependency we hold, or our own fuse tripped | Asset-scoped levers for every asset in the group |
| S2 isolate | Unbacked value confirmed entering Hydration, or MM collateral impacted | S1 + MM freeze + HSM removal + egress limits |
| S3 halt | Loss spreading across groups / unknown root cause | `Parameters.suspend_xcm` (one TC member — **not live at spec 455**; until then `PolkadotXcm.force_suspension` by TC 2/3 + tx-pause of outbound XCM), global withdraw lockdown, `ICE.set_solver_mode(Disabled)`, `pause_transaction` on the affected call |

## §A — Bridge compromise / unbacked mint

Examples: Hyperbridge 2026-04 (1B bridged DOT minted on Ethereum), Snowbridge light-client bug, Wormhole Guardian / NTT manager compromise.

For **each asset in the group** (Snowbridge: ~19 assets; Wormhole: ~11):

| # | Step | Lever | Origin |
|---|---|---|---|
| 0 | Bridge/XCM-wide and the blast radius is unclear | `Parameters.suspend_xcm()` — any single TC member, Operational; inbound messages are held, not dropped. Resume needs TC 2/3. ⚠ **master only — not on the live chain (spec 455) as of 2026-11-02**, see [[wiki/note-live-chain-check\|note-live-chain-check]]; until it ships use `PolkadotXcm.force_suspension` (TC 2/3) + tx-pause outbound XCM (TC majority) | one TC member (after upgrade) |
| 1 | Stop new mint landing on Hydration | XCM-routed: `CircuitBreaker.lockdown_asset(asset, until)` (`until` in 2 s blocks). NTT-routed: `EVMAccounts.clear_ntt_minter(asset)` + NTT `pause()` via `dispatch_as_emergency_admin` | TC / `NttEmergencyOrigin` |
| 2 | Stop it being dumped into the pool | `Omnipool.set_asset_tradable_state(asset, BUY \| REMOVE_LIQUIDITY)`; same per stablepool via `Stableswap.set_asset_tradable_state`; and stop ICE routing through it: `ICE.update_routing(OmnipoolAsset(asset) / StableswapPool(id), Some(Excluded))` | TC |
| 3 | Stop it being used as collateral | MM: freeze reserve + LTV 0 (via whichever dispatch holds the Aave role — see open gap in [[wiki/note-emergency-levers\|note-emergency-levers]]) | TC if EmergencyAdmin holds the role, else referendum |
| 4 | Stop HOLLAR minting against it | `HSM.remove_collateral_asset` or `pause_transaction(HSM, sell)` | referendum / TC |
| 5 | Contain outflow of good assets | Tighten `set_global_withdraw_limit_params`; S3 → `set_global_withdraw_lockdown(until)` | TC |
| 6 | Derived assets | Repeat 2–4 for every derived asset in the registry (aTokens, gTokens, stablepool shares containing the asset) | TC |
| 7 | Triage | If our deposit fuse tripped: upstream `ai_skills/circuit-breaker-incident` | — |

## §B — Oracle failure (stale / wrong / signer compromise)

| # | Step | Lever |
|---|---|---|
| 1 | Detect: DIA feed age > heartbeat, or DIA vs EMA deviation > threshold, or updates from an unexpected signer | watcher |
| 2 | Freeze affected MM reserves (no new borrows against a wrong price) | MM freeze |
| 3 | If the price is wrong **low**: liquidations are the damage → pause liquidations path (`pause_transaction` on the liquidation call; stop [[wiki/pepl-worker\|pepl-worker]] fast path) | TC |
| 4 | If the price is wrong **high**: borrowing is the damage → freeze + LTV 0 | MM |
| 5 | EMA-priced reserves: check Omnipool spot manipulation; tighten `set_trade_volume_limit` for the asset | TC |

## §C — Stablecoin depeg / issuer action (USDT/USDC on Asset Hub, NTT/Snowbridge stables)

| # | Step | Lever |
|---|---|---|
| 1 | Classify: market depeg vs issuer freeze/blacklist of a bridge escrow (the latter makes *all* wrapped units unbacked → treat as §A) | — |
| 2 | Protect HOLLAR: HSM collateral params / removal for the depegged stable; watch stablepools pairing it with HOLLAR | referendum / TC pause |
| 3 | Stablepools: disable `SELL` of the depegged leg into pools holding good stables | TC |
| 4 | MM: LTV 0 on the reserve | MM |

## §D — LST / wrapped-asset depeg (vDOT, iBTC, wstETH, jitoSOL, gTokens)

| # | Step | Lever |
|---|---|---|
| 1 | Distinguish market discount vs redemption failure at the issuer (Bifrost, Interlay, Lido, Jito) | — |
| 2 | Redemption failure → treat like §A for that asset (it is now unbacked) | §A |
| 3 | Market discount → MM LTV down / freeze if the oracle lags the discount; widen nothing on the AMM side | MM |
| 4 | Cascade: gDOT ← vDOT, gETH ← wstETH, gSOL ← jitoSOL — apply to the derived token too | — |

## §E — Origin chain halt / sunset

Not an emergency if announced — it is an unwind: `REMOVE_ONLY` → exit farms → remove liquidity → `FROZEN` → `remove_token`; MM deleverage → freeze → LTV 0; `remove_currency`; cancel DCA/OTC; xc-cfg route removal (inbound before outbound). An **unannounced** halt with a later exploit risk → §A step 1–3.

## Per-asset playbook card (template)

Fill one per asset from the registry; store next to the registry YAML.

```yaml
asset: { id: …, key: … }
dependency: …                 # registry group
levers:
  - step: stop-mint
    call: "0x…"               # SCALE-encoded, e.g. TC motion wrapping CircuitBreaker.lockdown_asset
    origin: TechCommitteeMajority
    dry_run: { fork_block: …, result: ok, date: … }
  - step: amm-flags
    call: "0x…"
  - step: mm-freeze
    call: "0x…"
  - step: hsm
    call: "0x…"
undo: ["0x…"]                 # reverse calls, also dry-run
blast_radius_usd: …           # from generator
decided_by: …                 # who signed this card off
decided_on: YYYY-MM-DD
```

## Drills

- Dry-run every card on Chopsticks (fork `hdx.tarn`, not lark; `dev_setBlockBuildMode("Instant")`; verify by events — `evm.ExecutedFailed` means failure even if the extrinsic is Ok).
- Re-run cards after every runtime upgrade (call indices / origins drift).
- Tabletop one §A scenario per quarter with the TC: time from alert → motion closed.

## Open questions

- Which Aave roles does `EmergencyAdminAccount` hold? Decides whether §A step 3 is TC-speed or referendum-speed.
- ~~Does the deposit fuse see NTT mints?~~ **Yes**: `INttToken.mint` on the multicurrency precompile deposits through `pallet_currencies`, so over-limit mints take the same circuit-breaker path as XCM (excess reserved + `AssetLockdown`) — `runtime/hydradx/src/evm/precompiles/multicurrency.rs`. `clear_ntt_minter` remains the way to stop the minter entirely.
- Should `set_asset_tradable_state` become `DispatchClass::Operational`? (Still `Normal` at spec 458; `suspend_xcm` / `resume_xcm` are the only Operational emergency calls.)
- What may an automated responder do without a human? Proposal: nothing on-chain except via a narrowly scoped proxy limited to §A step 2 for listed assets — needs a governance decision and probably a new origin.
- TC availability: how many members can be reached at 3am, and what is the motion threshold?

## Sources

- [[wiki/note-emergency-levers\|note-emergency-levers]], [[wiki/note-dependency-registry\|note-dependency-registry]], [[wiki/note-dependency-aware-monitoring\|note-dependency-aware-monitoring]]
- [[wiki/note-ai-skills\|note-ai-skills]] (`circuit-breaker-incident`)
- [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]
- [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]]
