---
{"dg-publish":true,"permalink":"/wiki/note-emergency-levers/","title":"Emergency levers — what we can pause, at what scope, and who can pull it","tags":["security","incident-response","governance","origins","runbook","circuit-breaker"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Emergency levers — what we can pause, at what scope, and who can pull it","repo":"hydration-node","paths":["pallets/circuit-breaker/src/lib.rs","pallets/circuit-breaker/src/types.rs","pallets/omnipool/src/lib.rs","pallets/stableswap/src/lib.rs","pallets/transaction-pause/src/lib.rs","pallets/dispatcher/src/lib.rs","pallets/parameters/src/lib.rs","pallets/ice/src/lib.rs","pallets/ema-oracle/src/lib.rs","runtime/hydradx/src/assets.rs","runtime/hydradx/src/system.rs","runtime/hydradx/src/governance/mod.rs","ai_skills/circuit-breaker-incident/SKILL.md"],"symbols":["set_asset_tradable_state","set_trade_volume_limit","set_add_liquidity_limit","set_remove_liquidity_limit","lockdown_asset","force_lift_lockdown","release_deposit","set_global_withdraw_limit_params","set_global_withdraw_lockdown","reset_withdraw_lockdown","add_egress_accounts","remove_egress_accounts","set_asset_category","GlobalAssetCategory","AssetLockdownState","WithdrawLockdownUntil","pause_transaction","unpause_transaction","dispatch_as_emergency_admin","dispatch_as_aave_manager","EmergencyAdminAccount","AaveManagerAccount","remove_collateral_asset","remove_currency","clear_ntt_minter","suspend_xcm","resume_xcm","suspend_xcm_send","suspend_xcm_execution","XcmSendSuspended","XcmExecutionSuspended","set_solver_mode","SolverMode","update_routing","RoutingTarget","remove_authorized_account","set_migration_enabled"],"depends_on":["pallet-circuit-breaker","pallet-omnipool","pallet-stableswap","pallet-transaction-pause","pallet-dispatcher","pallet-hsm","pallet-evm-accounts","pallet-transaction-multi-payment","hydration-borrow"],"tags":["security","incident-response","governance","origins","runbook","circuit-breaker"],"last_updated":"2026-11-02"}}
---


# Emergency levers — what we can pause, at what scope, and who can pull it

**TL;DR:** The menu every playbook in [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] picks from. One row per lever: the call, the scope it touches, whether it is reversible, and the **fastest origin** that can dispatch it. Origins read from `runtime/hydradx/src/{assets,system,governance/mod}.rs` at the 2026-11-02 `raw/hydration-node` sync (`a35840f`, spec 458) — re-verify after runtime upgrades. Fastest single lever: `Parameters.suspend_xcm` (any one TC member, Operational). Rule of thumb: prefer the **narrowest** lever that stops the bleeding; chain-wide levers (`suspend_xcm`, global withdraw lockdown, ICE `Disabled`, `pause_transaction`) are last resorts.

## Origin cheat sheet

| Origin | What it is | Speed |
|---|---|---|
| Single TC member (`EnsureSignedBy<TechCommAccounts>`) | One signed tx from any TC member | **Fastest** — no motion. Only for `Parameters.suspend_xcm` (master only, not live at spec 455) and conviction-vote removal |
| `TechCommitteeMajority` | TC motion, majority | Fast on-chain human path (no referendum) |
| `TechCommitteeSuperMajority` | TC motion, 2/3 | Needed to *resume* XCM |
| `OmnipoolAdmin` | OpenGov track | Referendum latency |
| `GeneralAdmin` | OpenGov track | Referendum latency |
| `EconomicParameters` | OpenGov track | Referendum latency |
| `EnsureRoot` | Root track / whitelisted-caller | Slowest unless whitelisted |
| `EmergencyAdminOrigin` (dispatcher) | `EitherOf<Root, TechCommitteeMajority>` → dispatches **as** `EmergencyAdminAccount` (`0xaa7e…aa7e1`) | TC speed; the account is a signed EVM/Substrate identity, so it can own EVM roles (e.g. NTT manager owner on Hydration) |
| `AaveManagerOrigin` (dispatcher) | `EitherOf<Root, EconomicParameters>` → dispatches as `AaveManagerAccount` (`0xaa7e…aa7e0`) | Referendum latency — **not** a TC path |

## Levers

### Omnipool / Stableswap (trading surface)

| Lever | Call | Scope | Reversible | Fastest origin |
|---|---|---|---|---|
| Disable SELL / BUY / ADD / REMOVE for one asset | `Omnipool.set_asset_tradable_state(asset, flags)` | 1 asset in Omnipool | Yes | TC majority (`UpdateTradabilityOrigin = Root \| TC \| OmnipoolAdmin`) |
| Same, for a stablepool | `Stableswap.set_asset_tradable_state(pool, asset, flags)` (`UpdateTradabilityOrigin = Root \| TC`) | 1 asset in 1 pool | Yes | TC majority |
| Tighten per-block trade volume ratio | `CircuitBreaker.set_trade_volume_limit(asset, (n,d))` | 1 asset, per block | Yes | TC majority |
| Tighten per-block add / remove liquidity ratio | `set_add_liquidity_limit` / `set_remove_liquidity_limit` | 1 asset, per block | Yes | TC majority |

Infinite-mint heuristic: the attacker wants to **sell the bad asset into the pool** and **add it as liquidity**. Turn off `SELL` + `ADD_LIQUIDITY` for that asset only; leave `BUY` / `REMOVE_LIQUIDITY` so honest LPs and holders can exit. `SELL`+`BUY` both off = `safe_withdrawal` (withdrawal fee waived) — see [[wiki/tradability-flags\|tradability-flags]]. Per-block volume limits only **slow** a multi-block dump; they are not a stop.

### Circuit breaker — mint/deposit fuse and egress

| Lever | Call | Scope | Reversible | Fastest origin |
|---|---|---|---|---|
| Lock an asset for minting until block N | `CircuitBreaker.lockdown_asset(asset, until)` | Deposits/mints of 1 asset | Yes (`force_lift_lockdown`) | TC majority |
| Lift lockdown | `force_lift_lockdown(asset)` | 1 asset | — | TC majority |
| Release a deposit reserved during lockdown | `release_deposit(...)` | 1 deposit | — | see code |
| Global withdraw (egress) limit params | `set_global_withdraw_limit_params(limit, window)` — limit in reference-currency units | All egress that participates | Yes | TC majority |
| Halt all participating withdrawals until T | `set_global_withdraw_lockdown(until)` | Chain-wide egress | Yes (`reset_withdraw_lockdown`, auto-lift at T) | TC majority |
| Mark an asset External / Local | `set_asset_category(asset, Some(External\|Local))` | Which egress accounting applies | Yes | TC majority |
| Egress account set | `add_egress_accounts` / `remove_egress_accounts` | Which accounts count as egress | Yes | TC majority |

The **XCM deposit fuse** trips automatically: an issuance increase beyond the asset's `xcm_rate_limit` over the period puts the asset into `AssetLockdownState = Locked(until)` and emits `AssetLockdown`. snakewatch alerts on it; triage = upstream `ai_skills/circuit-breaker-incident` (`query-lockdown.cjs`, `get-trigger-events.cjs`, `scan-deposits.js`, `generate-tc-unlock.js`). Do not duplicate that runbook here — see [[wiki/note-ai-skills\|note-ai-skills]].

### XCM kill switch ([[wiki/pallet-parameters\|pallet-parameters]])

| Lever | Call | Scope | Reversible | Fastest origin |
|---|---|---|---|---|
| Stop **all** inbound + outbound XCM | `Parameters.suspend_xcm()` — `DispatchClass::Operational` | Chain-wide XCM | Yes — `resume_xcm` needs TC **super-majority** (2/3) / GeneralAdmin / Root | **Any single TC member** (signed, `EmergencyOrigin`) — ⚠ **master only — not on the live chain (spec 455) as of 2026-11-02**, see [[wiki/note-live-chain-check\|note-live-chain-check]]. Live today: `PolkadotXcm.force_suspension` (TC 2/3 on master config) or tx-pause of outbound XCM calls (TC majority) |
| Stop one direction | `suspend_xcm_send` / `suspend_xcm_execution` (+ `resume_*`) | Outbound or inbound | Yes | TC super-majority / GeneralAdmin (master only, not live at spec 455) |

Outbound: every send fails (`PausableRouter`). Inbound: relay + sibling message queues pause (`XcmExecutionPausedQuery`) — messages are **held, not dropped**, and execute after resume. This is the fastest chain-level lever for a bridge/XCM incident (one signer, Operational class); pair it with asset-level controls before resuming.

### ICE (intents) ([[wiki/pallet-ice\|pallet-ice]])

| Lever | Call | Scope | Fastest origin |
|---|---|---|---|
| Kill switch — accept no solutions, refuse new intents | `ICE.set_solver_mode(Disabled)` | All intent settlement | TC majority |
| Emergency drain — each intent executed alone via router at its stored limit | `ICE.set_solver_mode(Passthrough)` | All intent settlement | TC majority |
| Stop the solver routing through a venue | `ICE.update_routing(target, Some(Excluded))` — `OmnipoolAsset`, `StableswapPool`, `XykPool`, `AaveWrap`, `UniswapV3Pool` | One venue | TC majority |
| Stop legacy DCA schedules executing (they migrate/cancel instead) | `DCA.set_migration_enabled(true)` / `force_cancel_schedules(ids)` | Legacy DCA | TC majority (`TerminateOrigin`) |

### Oracles

| Lever | Call | Fastest origin |
|---|---|---|
| De-authorize an external price pusher | `EmaOracle.remove_authorized_account(source, pair, account)` / `remove_external_source(source)` | `AuthorityOrigin` = Root \| **EconomicParameters** (referendum — no TC path) |

Fallback for a compromised pusher at TC speed: `TransactionPause.pause_transaction("EmaOracle", "set_external_oracle")` (+ `set_external_oracle_by_ids`).

### Call-level kill switch

| Lever | Call | Scope | Reversible | Fastest origin |
|---|---|---|---|---|
| Block a specific extrinsic | `TransactionPause.pause_transaction(pallet, fn)` | Every caller of that call | Yes | TC majority (`Root \| TC \| GeneralAdmin`) |

Blunt — affects every asset. Use for "the bug is in this extrinsic", not for "this asset is bad". Names are matched as strings (see [[wiki/pallet-transaction-pause\|pallet-transaction-pause]] gotchas).

### Money market (Aave fork, EVM)

| Lever | Call | Scope | Fastest origin |
|---|---|---|---|
| Freeze reserve / LTV 0 / disable borrow / pause reserve | `PoolConfigurator.*` via `dispatch_as_aave_manager` | 1 reserve | `AaveManagerOrigin` = Root \| **EconomicParameters** (referendum) |

**Answered on chain (2026-11-02, [[wiki/note-live-chain-check\|note-live-chain-check]]):** `EmergencyAdminAccount` (`0xaa7e…aa7e1`) holds **no Aave role on the main or BIL market** and only `EMERGENCY_ADMIN` on GIGAHDX. So the main market's fastest response is a **referendum** via aave-manager, which holds DEFAULT_ADMIN, POOL_ADMIN and EMERGENCY_ADMIN (+ RISK_ADMIN on GIGAHDX/BIL). Fix: grant `EMERGENCY_ADMIN` (pause) and `RISK_ADMIN` (freeze / LTV 0) to `0xaa7e…aa7e1` on main and BIL by referendum.

### HOLLAR / HSM

| Lever | Call | Scope | Fastest origin |
|---|---|---|---|
| Remove an HSM collateral | `HSM.remove_collateral_asset(asset)` | Stops HOLLAR mint/burn vs that collateral | Root \| EconomicParameters \| GeneralAdmin (referendum) |
| Update collateral params (caps, fees) | `HSM.update_collateral_asset(...)` | 1 collateral | same |

No TC fast path found — candidate for `pause_transaction(HSM, sell)` as the emergency equivalent.

### Bridges / mint authority

| Lever | Call | Scope | Fastest origin |
|---|---|---|---|
| Revoke an NTT minter | `EVMAccounts.clear_ntt_minter(asset)` | NTT mints of 1 asset on Hydration | `NttEmergencyOrigin` (faster than `set_ntt_minter`'s ControllerOrigin) — see [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] |
| Pause NTT manager on Hydration | NTT `pause()` as owner `EmergencyAdminAccount` | 1 NTT asset | TC via `dispatch_as_emergency_admin` |
| Ethereum-side NTT managers (usdc/usdt/weth/wbtc) | owner = pauser = **4-of-7 Gnosis Safe** `0xd557…81e7` | Escrow side | Off-chain signers |
| Pause / drain Basejump fast path | `BasejumpLanding.setAuthorizedBridge(receiver, false)` (pause) / `withdraw(asset, amount, to)` (drain) on `0x70e9…f976` | USDC (21) landing pool | TC via `dispatch_as_emergency_admin` — see [[wiki/whm-basejump\|whm-basejump]] |
| HyperEVM HYPE NTT manager (ours, locking) `0xdcDd08BF…676f` | owner / pauser not yet read on chain | HYPE (1001355) escrow | **unknown** — see [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] |
| hdx / hollar escrow (Hydration is the locking hub → Robinhood) | NTT `pause()` on our hub manager | Our escrow of HDX (0) / HOLLAR (222) | TC via `dispatch_as_emergency_admin` (verify owner) — see [[wiki/wormhole\|wormhole]] |
| WHM oracle relays (PRIME, SOL, jitoSOL, wstETH, apyUSD, SPY) | **none** — `OracleReceiver`s are renounced | Price push into MM oracles | Only de-authorise the pusher on the Hydration oracle contract, or freeze the MM reserve — see [[wiki/whm-oracle-relay\|whm-oracle-relay]] |

**NTT mints are covered by the deposit fuse:** `INttToken.mint` (multicurrency precompile) deposits via `pallet_currencies`, so an over-limit mint reserves the excess and locks the asset down exactly like an XCM deposit.

### Collateral side-effects to remember

- `MultiTransactionPayment.remove_currency(asset)` — accounts that chose the asset as fee currency silently fall back to HDX (WETH for EVM accounts); they need HDX/WETH to transact. No per-account reset required. Origin: Root \| TC \| GeneralAdmin.
- `DCA` terminate (`TerminateOrigin = Root \| TC \| GeneralAdmin`) — schedules touching the asset.
- EMA oracle entries go stale once trading stops → downstream consumers (fee pricing, DCA stability checks, HSM, Chainlink-adapter precompile → MM reserves priced off EMA) degrade. Sequence accordingly.

## Gotchas

- **Blocks are 2 s.** Every lever parameter given in blocks (`lockdown_asset(…, until)`, circuit-breaker per-block limits, `DeferDuration`) is 3× shorter in wall-clock than under 6 s blocks.

- Almost every **fast** lever is TC-majority. The slow ones (MM via AaveManager, HSM) are exactly the venues where bridged collateral concentrates. That asymmetry is the main design gap.
- `set_asset_tradable_state` (omnipool call_index 7, stableswap call_index 9) carries only a `#[pallet::weight]` — **no `DispatchClass::Operational`** — so it is `Normal` class and competes for block space during a spam/attack. [[wiki/tradability-flags\|tradability-flags]] calls it an Operational *candidate*; it is not one today.
- Encode every lever as call data **ahead of time** and dry-run it on Chopsticks; origins and call indices drift with runtime upgrades.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]], [[wiki/tradability-flags\|tradability-flags]], [[wiki/pallet-transaction-pause\|pallet-transaction-pause]], [[wiki/pallet-dispatcher\|pallet-dispatcher]], [[wiki/pallet-hsm\|pallet-hsm]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]
- [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] (NTT ownership, `EmergencyAdminAccount`)
- [[wiki/note-ai-skills\|note-ai-skills]] (`circuit-breaker-incident`)
- [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]]
