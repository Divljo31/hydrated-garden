---
{"dg-publish":true,"permalink":"/wiki/note-mrl-moonbeam-sunset-dependency-graph/","title":"MRL Moonbeam Sunset — Asset Dependency Graph","tags":["mrl","moonbeam","wormhole","governance","runbook"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"MRL Moonbeam Sunset — Asset Dependency Graph","tags":["mrl","moonbeam","wormhole","governance"],"source_count":3,"last_updated":"2026-07-07"}}
---

# MRL Moonbeam sunset — asset dependency graph

Moonbeam is shutting down: **parachain sunset on 31 July 2026** (announced publicly; GLMR migrates 1:1 to Base, remaining on-chain assets "may become inaccessible" after the deadline). All 12 MRL assets on Hydration are registered at XCM location **parachain 2004** and enter/exit via Wormhole VAA → BasejumpProxy on Moonbeam → XCM. When Moonbeam sunsets, that path dies; everything downstream must be unwound or migrated first. **From this note's snapshot date that is a 24-day window** for ≈$13.8M of bridged supply.

**On-chain snapshot: block #13,037,544 (2026-07-07, runtime hydradx/428, `wss://rpc.hydradx.cloud`).** Interactive graph: [/graphs/mrl-dependency-graph.html](/graphs/mrl-dependency-graph.html) (passthrough-copied at build; source `src/site/graphs/`). Companion: [[glmr-omnipool-delist-calldata]] (the unwind template), [[pallet-hsm]], [[tradability-flags]], [[note-hollar-only-bridged-stables]].

## Live bridged supply (block 13,037,544; ≈USD at 3-Jul prices)

PRIME 9,879,279 (≈$10.34M) · WBTC 13.3462 (≈$827k) · SOL 6,909.3 (≈$562k) · sUSDS 481,031 (≈$531k) · EURC 350,693 (≈$402k) · jitoSOL 2,623.65 (≈$274k) · USDC 228,922 (≈$229k) · USDT 200,892 (≈$201k) · GLMR 22,263,487 (≈$194k) · SUI 201,099 (≈$150k) · WETH 68.392 (≈$119k) · DAI 3,665.6 (≈$3.5k). **Total ≈$13.8M** must exit over the bridge (or be migrated to other representations) before Moonbeam sunsets. Note supply drift since the 3-Jul sheet snapshot: EURC +50k, USDC −25k, USDT +20k, SOL −98, jitoSOL −155 — flows are live.

## Live dependency matrix (chain-verified)

Sorted by USD exposure. "Fee" = accounts with the asset set as tx-fee currency.

| Asset (id) | Supply (≈USD) | Treasury | Omnipool | Stablepools | Money market | HSM | Fee | DCA/OTC | Farms | Key blocker |
|---|---|---|---|---|---|---|---|---|---|---|
| **PRIME** (43) | 9,879,279 (≈$10.34M) | 686 | — | 143 vs HOLLAR (Propeller) | LTV 85 · **aPRIME 9.38M = 95% of supply** | — | 4 | 2 DCA | — | Propeller + 95% locked in MM |
| **WBTC** (19) | 13.3462 (≈$827k) | **5.57 (42%)** | — | 101 vs iBTC | **frozen, LTV 0** · aWBTC 6.85 = 51% of supply | — | 23 | — | — | residual MM positions + treasury swap |
| **SOL** (1000752) | 6,909.3 (≈$562k) | 1.3 | bits 15 · 104 pos | 90001 leg via aSOL(1009) | LTV 70 · aSOL 6,169 = 89% of supply | — | 77 | — | 2 stopped | sequence with GSOL chain |
| **sUSDS** (1000745) | 481,031 (≈$531k) | — | — | 112 (HUSDS) vs HOLLAR | HUSDS shares 134k · LTV 70 | **LIVE collateral** (inv 0, cap 2M) | 2 | — | — | HOLLAR peg-defense capacity |
| **EURC** (44) | 350,693 (≈$402k) | 955 | — | HEURC 10044 via aEURC(1044) | LTV 75 · aEURC 436.5k | — | 10 | 3 DCA | — | deleverage before HEURC drain |
| **jitoSOL** (40) | 2,623.65 (≈$274k) | 5.0 | via GSOL(9001) · 148 pos | 90001 vs aSOL | GSOL 9,230 · LTV 60 | — | 1 | — | **131 ACTIVE** | longest chain: farm → GSOL → 90001 |
| **USDC** (21) | 228,922 (≈$229k) | 2.6 | — | 100 + 105 (HOLLAR) | — | — | 228 | 2 OTC | — | HSM-X migration decision |
| **USDT** (23) | 200,892 (≈$201k) | 12 | — | 100 + 105 (HOLLAR) | — | — | 343 | 1 OTC | — | HSM-X migration decision |
| **GLMR** (16) | 22,263,487 (≈$194k) | 4,718 | bits 11 · 295 pos | — | — | — | 821 | 1 DCA | 3 stopped (475 entries) | **REF #360 LIVE** — delist in flight |
| **SUI** (1000753) | 201,099 (≈$150k) | 1.2 | bits 15 · 43 pos | — | — | — | 23 | 1 OTC | 1 stopped | none — easiest template run |
| **WETH** (20) | 68.392 (≈$119k) | 0.2 | — | 104 vs aETH | — | — | **909** | — | — | **EVM GAS CURRENCY** — runtime change |
| **DAI** (18) | 3,665.6 (≈$3.5k) | 294 | — | 100 (4-Pool) | — | — | 18 | — | — | none — trivial |

Aave Pool `0x1b02e051683b5cfac5929c25e84adb26ecf87b38`; underlyings confirmed via `UNDERLYING_ASSET_ADDRESS()`: aWBTC→19, aPRIME→43, aEURC→44, aSOL→1000752, aETH→34 (snowbridge, NOT MRL), GSOL(9001)→pool-90001 shares, HUSDS(1112)→pool-112 shares.

## Full-chain sweep findings (block 13,037,544)

1. **Referendum #360 is live** (whitelisted-caller track, preimage `0xec9ce0b4…`) — this IS the GLMR omnipool delist from [[glmr-omnipool-delist-calldata]], confirmed ongoing on-chain.
2. **Fee payments: 2,459 accounts** have an MRL asset set as their tx-fee currency (`accountCurrencyMap`): WETH 909, GLMR 821, USDT 343, USDC 228, SOL 77, WBTC 23, SUI 23, DAI 18, EURC 10, PRIME 4, sUSDS 2, jitoSOL 1. Each asset needs `remove_currency` and those users need `reset_payment_currency` (or they lose fee-payment ability).
3. **WETH(20) is the runtime EVM gas currency** (pallet_evm `WethCurrency`) — the single most system-critical dependency in the set; also MetaMask/unbound-account default and faucet asset.
4. **GSOL farm 131 is still Active** — must be stopped before any jitoSOL/GSOL unwind.
5. **EMA oracle: 23 live pair entries** involve MRL assets (LRNA/asset pairs for omnipool assets, pool-share pairs 18/100, 21/105, 40/90001, 20/104, 43/143, 112/1000745 etc., plus DOT/USDC(5/21), USDC/USDT(21/23), GLMR vs HDX/DOT/asset-31). Fee-payment pricing, DCA stability checks and HSM pricing consume these; they go stale as soon as trading stops — sequence `remove_currency` before halting trade.
6. **XYK exposure is dust**: USDC/DOT ≈ $7, GLMR pool 211 GLMR, SUI pool 4.25 SUI. The 3 Active XYK farms on chain are on non-MRL pools. Ignorable.
7. **Bonds: none** reference MRL underlyings.
8. **HSM sUSDS inventory is currently 0** (cap 2M) — `remove_collateral_asset` is cheap right now, no holdings to drain.
9. Live order flow: DCA (GLMR 1, PRIME 2, EURC 3), OTC (USDC 2, USDT 1, SUI 1).
10. **Money market concentration**: the Aave market holds the majority of three assets — aPRIME 9.38M (**95% of all bridged PRIME**), aSOL 6,169 (89% of SOL), aWBTC 6.85 (51% of WBTC). MM deleveraging IS the unwind for these, the AMM legs are secondary. **Outstanding borrows** (variableDebt totalSupply): EURC 99,513 (**28% of bridged EURC is lent out** — the real deleverage job), SOL 1,897, PRIME 21.9, WBTC 0.13; HUSDS/GSOL collateral classes have zero borrows (supply-only, LTV-collateral use).
11. **Treasury holds MRL inventory** needing swaps before sunset: 5.57 WBTC (~$345k, 42% of supply), 4,718 GLMR (swept further by ref #360), 955 EURC, 686 PRIME + dust in the rest.
12. **Router: no stored routes** reference MRL assets (`routeExecutor.routes` clean) — nothing to clear there. Off-chain: xc-cfg / UI route configs and Basejump tags still list these assets and need a frontend PR.
13. **Holder counts** (orca indexer, balance > 0): GLMR 3,172 · WETH 1,099 · USDT 756 · USDC 706 · SOL 442 · WBTC 221 · SUI 155 · PRIME 102 · DAI 79 · EURC 45 · sUSDS 24 · jitoSOL 15 — ≈6,800 accounts hold raw MRL assets (aToken/GSOL holders extra). This is the comms audience.
14. **Distinct borrowers ever, per MRL reserve** (mmBorrows, deduped): WBTC 70 · EURC 33 · SOL 24 · PRIME 23 — the deleverage outreach list is small; net current debtors are a subset (netting via `accountMmPositionHistoricalData` or omniwatch).

## Frontend / SDK surface (xcm-cfg v10.26.0, scanned from npm)

`@galacticcouncil/xcm-cfg` (the route-config layer the UI and wallet consume) carries **20 outbound MRL routes from Hydration** — glmr→astar/bifrost/centrifuge/moonbeam/zeitgeist, {dai,weth,wbtc,usdt,usdc}\_mwh→ethereum+moonbeam (tagged `Mrl,Wormhole`), usdc\_mwh→zeitgeist, sol→moonbeam/solana, prime→solana, sui→sui — plus **17 inbound routes to Hydration** from 9 chains (ethereum, base, astar, bifrost, centrifuge, moonbeam, zeitgeist, solana, sui). Moonbeam is a full chain entry in `chainsMap`. Removal = xcm-cfg PR deleting the Mrl/Wormhole-tagged routes + the moonbeam chain def, then version bumps in hydration-ui / apps. Inbound routes should be pulled **before** outbound (stop refills while users exit). Note the GLMR routes to astar/bifrost/centrifuge/zeitgeist also die — other parachains lose their GLMR path through us.

## Moonbeam sunset timeline (public, as of 2026-07-07)

Hard deadline **31 July 2026**: parachain operations end, GLMR swaps 1:1 to native ERC-20 on Base; project pivots to an AI-agent payment protocol on Base. Public guidance: withdraw DeFi positions and bridge out before the deadline — assets remaining on the parachain after sunset may be inaccessible. Implication for the order below: with a 24-day window, the MM deleveraging (EURC 28% borrowed, PRIME/SOL/WBTC supply-heavy) and the 6,800-holder comms should start immediately; governance latency (referendum enactment) eats several days per asset unless batched.

## Unwind template (from [[glmr-omnipool-delist-calldata]])

Per omnipool asset: `set_asset_tradable_state(REMOVE_ONLY)` → per-entry `exit_farms` → per-position `remove_all_liquidity` → `FROZEN` → `remove_token(asset, Treasury)` → sweep. Per stablepool asset: stop incentives → migrate/withdraw → delist pool (we control the registry). Per MM reserve: deleverage borrowers → freeze → LTV 0 → disable borrow → withdraw suppliers. Plus per asset: `multiTransactionPayment.remove_currency`, cancel DCA/OTC.

## Suggested order (dependency-driven)

1. **GLMR** — referendum ready, chopsticks-verified. Ships the pipeline.
2. **SUI** — smallest omnipool job (43 positions), exercises the template.
3. **DAI** — trivial ($3.5k), clears 4-Pool of one MRL leg.
4. **WBTC** — MM already frozen; migrate residual aWBTC, drain pool 101.
5. **SOL + jitoSOL together** — the GSOL chain (stop farm 131 → GSOL omnipool removal → unwrap → pool 90001 → MM deleverage) spans both.
6. **EURC** — deleverage LTV-75 reserve, drain HEURC, cancel 3 DCAs.
7. **USDC + USDT** — decide HSM-X 1:1 migration vs plain drain ([[note-hollar-only-bridged-stables]]); pool 105 holds 289k HOLLAR.
8. **sUSDS** — `hsm.remove_collateral_asset` reduces HOLLAR peg-defense capacity; sequence when peg posture allows.
9. **PRIME** — product decision (Propeller) gates the technical unwind; LTV-85 borrowers + bespoke oracle.
10. **WETH last (or first as runtime change)** — swapping `WethAssetId` to snowbridge ETH is a runtime upgrade touching every EVM interaction; everything else EVM-related should be stable when it lands.

Chopsticks dry-runs per [[chopsticks-testing-hydration]]: fork `hdx.tarn` (NOT lark — reset relay height breaks farm-exit `InvalidPeriod`), `dev_setBlockBuildMode("Instant")`, verify by events (`evm.ExecutedFailed` = fail even when extrinsic is Ok).
