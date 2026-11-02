---
{"dg-publish":true,"permalink":"/wiki/note-security-hardening-plan/","title":"Security hardening plan — monitoring, pausing, NTT custody (draft)","tags":["security","monitoring","incident-response","ntt","xcm","plan","draft"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Security hardening plan — monitoring, pausing, NTT custody (draft)","tags":["security","monitoring","incident-response","ntt","xcm","plan","draft"],"depends_on":["note-dependency-aware-monitoring","note-dependency-registry","note-emergency-levers","runbook-collateral-damage-playbooks","source-onchain-monitoring-landscape","wormhole","pallet-circuit-breaker","pallet-parameters","whm-oracle-relay"],"last_updated":"2026-11-02"}}
---


# Security hardening plan — monitoring, pausing, NTT custody (draft)

**TL;DR:** Hydration already has strong *on-chain* brakes (circuit breaker deposit fuse that also covers NTT mints, global egress limits, `suspend_xcm` by a single TC member, tradability flags, tx-pause) and a lot of *designed-but-not-running* ops work (NTT fast-pause in `hydration-ntt/ops/PAUSE.md`, custody script `_status.ts`). The biggest risks today are: (1) **no fast pause on the NTT hubs** — pauser = 4-of-7 owner Safe, Solana legs have no pauser, no drill ever run; (2) **single verifier** — every NTT leg is Wormhole-only, threshold 1; (3) **the mint fuse delays, it doesn't stop** — lockdowns auto-expire and `funds-unlocker` auto-releases; (4) **nothing watches custody, admin keys or governance continuously**. Plan: 4 phases — verify (days) → wire the fast paths + paging (weeks) → build the watchtower (weeks) → runtime + bridge-architecture upgrades (1–2 releases). Patterns borrowed from Wormhole Governor/Accountant, CCIP RMN, Snowbridge RFC-0166/0167, Lido GateSeal, Aave Guardian, Chainlink Secure Mint.

## 1. What we already have (inventory, 2026-11-02)

| Layer | Control | Where | State |
|---|---|---|---|
| Limit | Per-asset **deposit fuse** (issuance increase > `xcm_rate_limit` → `AssetLockdown`) — covers XCM **and** NTT mints | [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] | Live; coverage per asset **unverified** (an asset with no limit has no fuse) |
| Limit | Global withdraw limit + lockdown, egress accounts, XCM per-message netting | [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] | Live |
| Limit | Per-block trade / add / remove liquidity ratios | [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] | Live |
| Limit | NTT rate limits — **hub legs** real (e.g. USDC/USDT/DAI/sUSDS 100k/24h, WETH 10k, WBTC 10, HYPE 1.2k), **Hydration legs ∞ by design** | `hydration-ntt/ops/tokens/*/deployment.json`, `ops/LIMITS.md` | Live |
| Respond | `suspend_xcm` — any single TC member, Operational; resume TC 2/3 | [[wiki/pallet-parameters\|pallet-parameters]] | **On master (spec 458), not live at spec 455** ([[wiki/note-live-chain-check\|note-live-chain-check]]) — the model for other fast paths |
| Respond | `lockdown_asset`, tradability flags, `pause_transaction`, `clear_ntt_minter`, ICE `SolverMode` | [[wiki/note-emergency-levers\|note-emergency-levers]] | Live; TC-majority speed, tradability is `Normal` class |
| Respond | NTT `pause()` per leg | `ops/PAUSE.md` | **Designed, not wired**: pauser == owner Safe (4/7) on every EVM leg; Solana no pauser; Sui pre-signed blob not set up; drill table empty |
| Detect | snakewatch: `AssetLockdown` → Slack/Discord ping; HF, rate, price-delta, EVM deployment intel; event feeds | `galacticcouncil/snakewatch` | Live |
| Detect | breakers dashboard (deposit limits, global egress, NTT limits) | `sdk/apps/src/breakers` | Live, pull-based (no alerting) |
| Detect | governator: posts referenda + status to Discord/Telegram | `galacticcouncil/governator` | Live; does **not** classify what a referendum touches |
| Detect | forkwatch: relay/parachain fork attribution | `galacticcouncil/forkwatch` | Live |
| Detect | NTT custody vs supply, all legs | `hydration-ntt/ops/scripts/_status.ts` | On-demand script only |
| Triage | `circuit-breaker-incident` agent skill | `hydration-node/ai_skills/` ([[wiki/note-ai-skills\|note-ai-skills]]) | Live |
| Recover | `funds-unlocker` — auto `releaseDeposit` after lockdown expiry | `galacticcouncil/funds-unlocker` | Live — **dangerous during a real incident** (see G3) |
| Respond tooling | TC voting + dispatch apps | `sdk/apps/src/{tc-voting,dispatch}` | Live |

## 2. Gaps, ranked

| # | Gap | Why it matters | Evidence |
|---|---|---|---|
| **G1** | **NTT hubs can't be paused fast** | Hub custody is where real money leaves. 4-of-7 at 3am is minutes-to-hours; Solana legs have no pauser at all | `deployment.json`: `pauser == owner` everywhere; `PAUSE.md` steps not done; empty drill log |
| **G2** | **Single verifier per NTT leg** (Wormhole transceiver only, threshold 1) | A guardian-quorum compromise or forged VAA mints on Hydration bounded only by the deposit fuse (Hydration side is ∞) and drains hubs up to their outbound limit. Same shape as Kelp rsETH (1-of-1 DVN, Apr 2026) | `thr=1` on all 34 legs |
| **G3** | **Fuse = delay, not stop** | Lockdown auto-expires; `funds-unlocker` then releases reserved excess automatically. With 2 s blocks the window is short in wall-clock time. An exploit that trips the fuse gets paid out unless a human extends it | `release-deposits.js` releases once `until < current`; `ASSET_LOCKDOWN_PERIOD_BLOCKS` |
| **G4** | **No continuous custody invariant** | Unbacked supply (Hyperbridge, Kelp) is detectable as `Σ supply(burning legs) > Σ custody(locking legs)` before it is sold | `_status.ts` exists, not scheduled, no alerting |
| **G5** | **No admin / governance event watch** | Ownership, upgrade, pauser, peer, limit, transceiver changes; Safe module/guard changes; Solana upgrades; Hydration referenda or TC motions touching levers — a key compromise shows up here first | `PAUSE.md` watchdog spec P1 list — not implemented |
| **G6** | **Slow levers where collateral sits** | MM response is `AaveManagerOrigin` (referendum) unless `EmergencyAdminAccount` holds Aave emergency/risk roles; HSM has no TC path; WHM oracle relays are renounced; external EMA pushers de-authorisable only by Root / EconomicParameters | [[wiki/note-emergency-levers\|note-emergency-levers]] |
| **G7** | **Fast path is inconsistent** | Only `suspend_xcm` is single-member + Operational. Asset lockdown, tradability, NTT-minter revoke need TC majority and are `Normal` class | runtime config |
| **G8** | **Upstream signals not ingested** | Snowbridge halts / velocity-cap trips (RFC-0166/0167), Asset Hub safe-mode, Wormhole Governor holds, Bifrost/Jito/Lido rate anomalies, vendor incident feeds | — |
| **G9** | **No paging / on-call / drills** | Alerts go to Slack/Discord channels; nothing escalates; no measured time-to-pause | snakewatch config |

## 3. What other projects do (patterns we can copy)

| Pattern | Who | Mechanism | Apply to Hydration |
|---|---|---|---|
| **Notional rate limit at the verifier** | Wormhole **Governor** | Guardians delay token-bridge VAAs when a chain's 24 h notional outflow exceeds its limit; large single transfers delayed 24 h | Check chain 73's Governor config (Wormholescan `/api/v1/governor/*`); watch its holds as a signal |
| **Cross-chain supply invariant enforced by verifiers** | Wormhole **Global Accountant** (available to NTT on request) | Guardians refuse to sign a transfer that would let burned/sent exceed minted | Ask Wormhole to enable for our NTT tokens — closes most of G2 at the attestation layer |
| **Per-lane token buckets + global kill** | Chainlink **CCIP** (token-pool rate limits + RMN "curse") | Independent inbound/outbound buckets per chain; a curse halts lanes or everything | Our NTT limits already = buckets; the "curse" = our emergency brake (Phase 3) |
| **Check reserves before minting** | Chainlink **Secure Mint** / PoR | Mint reverts if new supply would exceed attested reserves | Runtime "mint gate": NTT/XCM mint allowed only while `supply + amount ≤ attested hub custody` (research item; oracle in the critical path is the trade-off) |
| **Permissionless, deposit-gated halt** | **Snowbridge RFC-0166** | Anyone posts ~100k DOT to halt the bridge end-to-end; Fellowship resolves genuine (refund, stays halted) vs malicious (slash, auto-resume); 7-day backstop | Optional Phase 3: bonded "pull the brake" for asset groups; deters griefing via slash |
| **Per-asset 24 h net-flow velocity caps** | **Snowbridge RFC-0167** — explicitly lifted from Hydration's circuit-breaker net-volume tracker | Token-unit caps (no oracle), trip into per-asset lockdown, auto-lift 24 h | Upstream will throttle Snowbridge assets for us; ingest trips as signals |
| **Chain-wide call filters** | FRAME **safe-mode** / **tx-pause** (proposed for system chains, fellows runtimes #1089/#1164) | Safe-mode = deposit-triggered chain-wide filter; tx-pause = per-call pause | We have tx-pause; safe-mode too blunt for us (would stop exits). Watch Asset Hub adopting it — a dependency signal |
| **Pause-only committee that can't unpause** | Lido **GateSeal** → CircuitBreaker; NTT `pause` = owner-or-pauser, `unpause` = owner only | Small fast committee can only stop; restart needs full governance; one-time / expiring seals | Exactly `PAUSE.md`'s 1/N Pauser Safe design — ship it |
| **Standing emergency multisig + readiness drills** | **Aave Protocol Guardian** (4/7; freeze/pause; quarterly signer verifications) | Froze rsETH/wrsETH and set LTV 0 within hours of Kelp (Apr 2026) | Grant `EmergencyAdminAccount` Aave emergency/risk roles; quarterly drills |
| **Invariant-based bridge anomaly detection** | **XChainWatcher** (Ronin/Nomad replay) | Datalog rules over decoded events on both chains; a flow-balance invariant flags every known attack | Model for the custody watcher + unmatched-mint tracer |
| **Freeze before contagion leaves** | **Acala aUSD** (2022) | Maintenance mode, transfers + XCM-out disabled within hours; 99% of error mints stayed on-chain | Our `suspend_xcm` + global egress lockdown are the equivalent; make them one click |

## 4. Target design — four layers

1. **Limit (automatic, on-chain):** NTT hub buckets sized to TVL; deposit fuse on **every** bridged asset; global egress limit; tradability; *new:* large-excess lockdowns don't auto-expire.
2. **Detect (watchtower):** custody invariant, unmatched-mint tracer, admin/governance events, oracle sanity, XCM inflow anomalies, upstream feeds, dead-man switch — correlated through [[wiki/note-dependency-registry\|note-dependency-registry]] into blast radius.
3. **Respond (fast, scoped, reversible):** 1/N pause on every hub; single-TC-member Operational **emergency brake** on Hydration; MM emergency role; pre-encoded calls in every P1 alert ([[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]]).
4. **Recover (slow, deliberate):** unpause / lift only by full threshold; incident flag freezes `funds-unlocker`; drills; postmortems.

## 5. Roadmap

### Phase 0 — verify (days, no code)

- [x] Read every bridged asset's deposit limit (`xcm_rate_limit` / registry) — all bridged collateral has a fuse; none on HOLLAR, uBIL, stHDX, SIGIL and pool shares 143 / 146 / 90001 ([[wiki/note-live-chain-check\|note-live-chain-check]] §4).
- [x] Read Aave `ACLManager.isEmergencyAdmin / isRiskAdmin(EmergencyAdminAccount)` — **no role on main / BIL, EMERGENCY_ADMIN only on GIGAHDX** ([[wiki/note-live-chain-check\|note-live-chain-check]]).
- [x] Read owners of the WHM-fed oracle contracts — CheckedOracles owned by aave-manager; SPY has **no check oracle**; apyUSD **frozen ~47 days** ([[wiki/note-live-chain-check\|note-live-chain-check]] §3).
- [ ] Ship spec ≥ 458 (adds `suspend_xcm`); until then the XCM halt is TC 2/3 `force_suspension`.
- [ ] Wormhole: Governor limits for chain 73; is Global Accountant enabled for our NTT tokens?
- [ ] Confirm `clear_ntt_minter` origin + real latency (`NttEmergencyOrigin`).
- [ ] TC reachability: who answers at 3am; motion threshold; time to close a motion.
- [ ] Name an owner + on-call rotation.

### Phase 1 — wire the fast paths (1–3 weeks, config + ops)

- [ ] Execute `PAUSE.md`: 1/N **Pauser Safe** (all TC members) on Ethereum, Base, HyperEVM, Robinhood; `transferPauserCapability` on every manager **and** transceiver.
- [ ] Solana: pre-approved Squads `set_paused(true)` per program; every TC member has Executor permission.
- [ ] Sui: dedicated gas coin, pre-signed pause blob, daily dry-run in the watchdog.
- [ ] `funds-unlocker`: refuse to release while an **incident flag** is set (on-chain parameter or signed config) and for excess above a threshold.
- [ ] Paging: P1 → phone (PagerDuty / Opsgenie / Telegram with escalation), P2 → channel. Every P1 carries the action link.
- [ ] **First drill**: pause + unpause one low-traffic leg end to end; log tx hashes in `PAUSE.md`.

### Phase 2 — build the watchtower (3–6 weeks)

Extend snakewatch (already subscribed to Hydration, Slack/Discord plumbing) or start a sibling service; feed the registry for blast radius.

| Module | Signal | Source / basis |
|---|---|---|
| Custody invariant | per token: Σ burning-leg supply vs Σ locking-leg custody; drift beyond in-flight tolerance; custody drop without burn | port `_status.ts` to a loop |
| Unmatched mint | every Hydration NTT/XCM mint traces to a VAA from the expected emitter + a hub lock | Wormholescan API + hub events; XChainWatcher model |
| Admin events | `OwnershipTransferred`, `PauserTransferred`, `Upgraded`, `PeerUpdated`, transceiver/threshold/limit changes; Safe owners/threshold/module/guard; Solana upgrades / config owner / `SetPaused` / IDL writes; Sui `UpgradeCap`/`AdminCap`; **any unpause** | `PAUSE.md` P1 list |
| Governance decoder | referenda / TC motions / whitelisted-caller calls that touch levers, NTT minters, asset registry, oracles, dispatcher | extend governator: decode preimage, match against a lever list |
| Rate-limit pressure | NTT utilisation > 50/80%, queued transfers, large single transfers | NTT events |
| Hydration fuses | `AssetLockdown`, `WithdrawLockdownTriggered`, near-limit deposits | snakewatch (exists) + breakers data |
| Oracle sanity | DIA staleness vs heartbeat, DIA vs EMA deviation, WHM relay age vs 300 s, unexpected signer | [[wiki/note-dependency-registry\|note-dependency-registry]] Oracles |
| Upstream feeds | Snowbridge operating mode / RFC-0167 trips, Asset Hub safe-mode, Wormhole Governor holds, issuer blacklists on escrows, vendor incident feed | public RPCs/APIs |
| Dead-man switch | per-chain heartbeat + head lag; silence pages | — |

### Phase 3 — runtime upgrades (1–2 releases)

- [ ] **Emergency brake origin**: single TC member, `Operational`, scoped and reversible — `lockdown_asset` for registry-listed assets, `clear_ntt_minter`, tradability `SELL`/`ADD_LIQUIDITY` off, global withdraw lockdown; resume/lift = TC 2/3 (mirror `suspend_xcm`). Optionally one call that applies a whole dependency group's card.
- [ ] Make `set_asset_tradable_state` `Operational`.
- [ ] Fuse: excess above X× limit stays locked until explicit TC lift (no auto-expiry).
- [ ] Optional: bonded permissionless trigger (RFC-0166 style) — deposit, TC resolves genuine (refund) vs malicious (slash).
- [ ] MM: grant `EmergencyAdminAccount` Aave `EMERGENCY_ADMIN` + `RISK_ADMIN` (freeze, LTV 0, pause reserve at TC speed); HSM pause path.
- [ ] Oracles: TC-speed de-authorisation of external EMA pushers; decide on WHM oracle target contracts' owner.

### Phase 4 — bridge architecture (quarter)

- [ ] **Second verifier** on high-value NTT legs (NTT supports multiple transceivers; threshold 2) — the Kelp lesson.
- [ ] Global Accountant on all our NTT tokens.
- [ ] Re-size hub limits to blast-radius budget (value we accept losing in one window), per token.
- [ ] Research a Secure-Mint-style runtime check (mint ≤ attested custody).
- [ ] Vendor for the EVM side (Hypernative / Hexagate ecosystem program) vs build — see [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]].

### Continuous

Quarterly pause drill per chain family; tabletop of [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] §A; registry regeneration; re-encode playbook cards after each runtime upgrade.

## 6. Success metrics

- **Time-to-detect** (exploit tx → P1 page) — target < 2 min.
- **Time-to-pause** per leg (page → pause landed) — target < 10 min any hour; measured in drills.
- **Coverage**: % bridged assets with a deposit fuse; % legs with a 1/N pauser; % P1 alerts carrying an action link.
- **Blast radius** per dependency (from the registry) vs the limit budget.

## 7. Decisions needed

1. Owner + on-call for the watchtower and the pausers.
2. Keep Hydration-side NTT legs ∞ (fuse-only) or add real inbound limits on Hydration too?
3. Appetite for a second verifier (cost, latency) and Global Accountant.
4. Emergency brake: single TC member like `suspend_xcm`, or 2-of-N? Permissionless bonded trigger yes/no?
5. Build the watchtower in snakewatch or separately; buy for the EVM side?

## Sources

- Internal: `galacticcouncil/hydration-ntt` (`ops/LIMITS.md`, `ops/PAUSE.md`, `ops/CUSTODY.md`, `ops/scripts/_status.ts`, `ops/tokens/*/deployment.json`), `galacticcouncil/snakewatch` (`ALERTS.md`, `src/handlers/circuitbreaker.js`), `galacticcouncil/funds-unlocker`, `galacticcouncil/governator`, `galacticcouncil/forkwatch`, `galacticcouncil/hydration-security`, `sdk/apps/src/breakers` — all read 2026-11-02
- [[wiki/note-emergency-levers\|note-emergency-levers]], [[wiki/note-dependency-registry\|note-dependency-registry]], [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]], [[wiki/note-dependency-aware-monitoring\|note-dependency-aware-monitoring]], [[wiki/wormhole\|wormhole]], [[wiki/whm-oracle-relay\|whm-oracle-relay]], [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]], [[wiki/pallet-parameters\|pallet-parameters]]
- [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]] (external patterns + links)
