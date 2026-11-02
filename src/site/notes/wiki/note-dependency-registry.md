---
{"dg-publish":true,"permalink":"/wiki/note-dependency-registry/","title":"Dependency registry — every asset and oracle we lean on","tags":["security","incident-response","dependencies","assets","oracles","bridges"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Dependency registry — every asset and oracle we lean on","repo":"sdk","paths":["packages/xc-cfg/src/chains/polkadot/hydration.ts","packages/xc-cfg/src/assets.ts"],"symbols":["hydration","assetsData","xcmLocation","GlobalConsensus","AccountKey20","Parachain"],"depends_on":["pallet-asset-registry","pallet-omnipool","pallet-stableswap","hydration-borrow","pallet-hsm","pallet-ema-oracle","snowbridge","wormhole","xcm","xc-cfg"],"tags":["security","incident-response","dependencies","assets","oracles","bridges"],"last_updated":"2026-11-02"}}
---


# Dependency registry — every asset and oracle we lean on

**TL;DR:** The "write it all down" half of [[wiki/note-dependency-aware-monitoring\|note-dependency-aware-monitoring]]. For each asset: **what it trusts upstream** (bridge, issuer, origin chain), **which oracle prices it**, and **where it sits on Hydration** (Omnipool, stablepools, MM, HSM, fee currency). Grouped by upstream dependency, because an incident hits a *dependency*, and every asset behind it fails together. The seed below is derived from `xc-cfg` at sdk d16ff85 (2026-11-02 sync) — it is a **starting list, not a chain snapshot**. The "venues" columns must be filled by a generator run against chain state (method at the bottom).

## Dependency groups (seed — from `xc-cfg/src/chains/polkadot/hydration.ts` @ sdk d16ff85, 55 assets)

Classification from each asset's `xcmLocation` + the chain's `wormhole.ntt` registry: `parents:2 + GlobalConsensus(Ethereum)` → Snowbridge; `parents:1 + Parachain(n)` → that parachain is the reserve; `parents:1, Here` → relay; `parents:0` + an NTT entry → **bridge-minted on Hydration** (burning manager), except the two hub legs (`hdx`, `hollar`) which are native and *lock*.

| Upstream dependency | Assets (id) | Failure mode that hurts us | Playbook class |
|---|---|---|---|
| **Snowbridge** (Ethereum ↔ BridgeHub ↔ Asset Hub) — 19 | eth (34), weth (1000189), wbtc (1000190), tbtc (1000765), lbtc (1000851), wsteth (1000809), usdc_eth (1000766), usdt_eth (1000767), susde (1000625), susds (1000626), aave (1000624), ena (38), ldo (1000796), link (1000794), sky (1000795), paxg (39), trac (35), cfg_new (41), apyusd (46) | Bridge mint bug / light-client compromise → unbacked supply arrives via XCM | Bridge compromise ([[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] §A) |
| **Wormhole NTT — Ethereum / Base hubs** (locking; Ethereum managers owned by a 4-of-7 Safe) — 7 | usdc_wh (21), usdt_wh (23), weth_wh (20), wbtc_wh (19), dai_wh (18), susds_wh (1000745), eurc_wh (44, Base) | Guardian / NTT manager / Safe compromise → unbacked mint **on Hydration via EVM**, possibly bypassing the XCM deposit fuse | §A + `clear_ntt_minter` |
| **Wormhole NTT — Solana / Sui** — 4 | sol (1000752), jito_sol (40), prime (43), sui (1000753) | same | §A |
| **Wormhole NTT — HyperEVM** (our own locking manager, not Wormhole's canonical HYPE group) — 1 | hype (1001355) | same; HyperEVM node software is closed-source | §A |
| **Wormhole NTT — Robinhood** (locking on Robinhood) — 1 | spy (1001356) | same + tokenised-equity issuer risk | §A / §C |
| **Wormhole NTT — NEAR** (WHM managers `ntt-near` / `ntt-zec.hydration-whm.near`) — 2 | near (1001358, locks `wrap.near`), zec (1001357, locks Omni's `zec.omft.near`) | same; **zec also depends on Omni / NEAR MPC** for the Zcash leg | §A |
| **Hydration as NTT hub** (we lock, others mint) | hdx (0), hollar (222) → Robinhood | Spoke-side compromise mints unbacked hdx/hollar on Robinhood and redeems against **our** escrow | §A (reverse direction: pause the Hydration manager) |
| **Asset Hub** (reserve) — 3 | usdt (10), usdc (22), wud (1000085) | Issuer freeze/blacklist, AH halt | Stable depeg / issuer action (§C) |
| **Relay chain** | dot (5) | Relay halt | Out of scope — chain-level |
| **Bifrost** (LST issuer) — 3 | vdot (15), vastr (33), bnc (14) | vDOT redemption-rate bug / depeg | LST depeg (§D) |
| **Interlay** — 2 | ibtc (11), intr (17) | iBTC backing failure | Wrapped-asset depeg (§D) |
| **Other parachains with routes** | astr (9) Astar, myth (30) Mythos, neuro (36) NeuroWeb, pen (1000081) Pendulum, unq (25) Unique, ewt (252525) EnergyWebX | Origin chain exploit / sunset | Origin chain halt (§E) |
| **Reserve chain not in xc-cfg any more** (no routes; stranded representation) | cru (27, Crust), ajun (32, Ajuna); Interlay chain def also gone | Already dead/unreachable upstream — holders cannot exit | §E unwind |
| **Kusama bridge** | ksm (1000771) | P↔K bridge failure | §A |
| **Native** | hdx (0), HOLLAR (222), GIGAHDX/stHDX, gDOT/gETH/gSOL, aTokens, stablepool shares | Our own code | Upstream `hydration_cl0wdit` / [[wiki/note-ai-skills\|note-ai-skills]] |

**Derived assets inherit their underlyings' dependencies.** gDOT ← vDOT (Bifrost) + aDOT; gETH ← wstETH (Snowbridge) + aETH; gSOL ← jitoSOL + aSOL (Wormhole); HOLLAR ← every MM collateral + HSM collateral (sUSDS etc.); every stablepool share ← all its legs. The registry must expand these transitively — chains can be long, e.g. farm → GSOL → stablepool → MM.

## Oracles

| Oracle | What it prices | Consumers | Failure mode |
|---|---|---|---|
| **DIA** (EVM, signer-pushed) — feeds: WBTC, WETH, DOT, USDT, USDC, tBTC, AAVE, vDOT; 0.5% deviation / 120 s refresh / 24 h heartbeat; contracts `0xc756bd33…6917`, `0x5d8320f3…2bf52`; update signers `0x33a5e905…`, `0xff0c6240…` (see [[wiki/pepl-worker\|pepl-worker]]) | MM reserves | [[wiki/hydration-borrow\|hydration-borrow]] liquidations (PEPL fast path watches these txs) | Stale (signer down), wrong price, signer key compromise |
| **EMA oracle** ([[wiki/pallet-ema-oracle\|pallet-ema-oracle]]) via `ChainlinkOraclePrecompile` (`0x000001…` addresses) | MM reserves without a DIA feed; GIGAHDX | MM, HSM (`OraclePriceProvider`), fee-currency pricing, DCA stability, dynamic fees, price barrier | Manipulation over the EMA window; goes stale when trading stops on an asset |
| **WHM oracle relay** (Wormhole VAA → renounced `OracleReceiver` → `setPrice` on per-asset Hydration oracle contracts) — sources: Kamino Scope (PRIME, SOL), Jito stake-pool rate (jitoSOL), wstETH `stEthPerToken`, apyUSD ERC-4626 `convertToAssets`, Chainlink SPY on Robinhood | PRIME, SOL, jitoSOL, wstETH, apyUSD, SPY | MM reserves | Source manipulation (apyUSD share price), stale after `maxPriceAge` 300 s if broadcaster/relayer down, guardian compromise. Receivers are renounced — **no pause lever**; only kill switch is the Hydration oracle contract's authorization or an MM freeze. Owners of the six target oracle contracts not in any repo — verify on chain. See [[wiki/whm-oracle-relay\|whm-oracle-relay]] |

**TODO (generator):** for every MM reserve, read `AaveOracle.getSourceOfAsset(reserve)` and classify DIA vs EMA-adapter vs other. Which Hydration representation each DIA feed prices (e.g. WETH → id 20 vs 34 vs 1000189) is not recorded anywhere in the vault yet.

## Registry schema (proposed)

One YAML file per dependency group, PR-reviewed; machine-generated fields marked `gen:`.

```yaml
# registry/snowbridge.yaml
dependency: snowbridge
kind: bridge                 # bridge | issuer | lst | origin-chain | oracle
upstream_contracts:          # what external watchers subscribe to
  - chain: ethereum
    address: "0x…"           # Gateway
watchers: [bridge-supply-vs-escrow, gateway-admin-events]
assets:
  - id: 34
    key: eth
    gen:                     # filled by the generator, never by hand
      supply: "…"
      omnipool: { tradable: "…", positions: "…" }
      stablepools: ["…"]
      money_market: { reserve: "…", ltv: "…", supplied: "…", borrowed: "…" }
      hsm_collateral: "…"
      fee_currency_accounts: "…"
      ema_pairs: ["…"]
    oracle: { mm: "dia | ema-adapter | other", feed: "…" }   # example values are placeholders
playbook: runbook-collateral-damage-playbooks §A
owner: "@…"                  # who gets paged
```

## Generator — how to fill the venues (chain snapshot)

Queries, run for every asset (papi / [[wiki/sdk-next\|sdk-next]]):

| Venue | Query |
|---|---|
| Supply | `Tokens.TotalIssuance(asset)`; stableswap share tokens: `Stableswap.ShareIssuance(pool)` |
| Omnipool | `Omnipool.Assets(asset)` → `tradable` bits, `hub_reserve`; positions via `Omnipool.Positions` |
| Stablepools | `Stableswap.Pools` → legs |
| Money market | Pool `getReservesList()` → `getReserveData` / aToken `totalSupply` / variableDebt `totalSupply`; `AaveOracle.getSourceOfAsset` |
| HSM | `HSM.Collaterals` |
| Fee currency | `MultiTransactionPayment.AcceptedCurrencies`, `AccountCurrencyMap` counts |
| EMA | `EmaOracle.Oracles` keys containing the asset |
| Farms / DCA / OTC | LM global farms, `DCA.Schedules`, `OTC.Orders` |
| Circuit breaker | `CircuitBreaker.AssetLockdownState`, `GlobalAssetOverrides`, asset registry `xcm_rate_limit` |
| Treasury | treasury account balances |

Output: one matrix per dependency group with $ exposure, plus a single "blast radius" number per dependency. Re-run on a schedule; diff the output so new listings / new collateral show up as registry PRs.

## Gotchas

- `xc-cfg` lists **routable** assets, not every registered asset. Assets that are only on-chain (pool shares, aTokens, illiquid listings) must come from `AssetRegistry.Assets`.
- `parents:0` does **not** mean "native" — every NTT-registered asset except `hdx` / `hollar` is bridge-minted on Hydration. Use the asset registry + NTT minter registry (`EVMAccounts.NttMinters`) to tell native from bridge-minted.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]] (`xc-cfg/src/chains/polkadot/hydration.ts`, sdk d16ff85)
- [[wiki/pepl-worker\|pepl-worker]], [[wiki/hydration-precompiles\|hydration-precompiles]], [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]
- DIA Hydration feeds — https://nexus.diadata.org/how-to-guides/fetch-price-data/chain-specific-guide/hydration
- [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]]
