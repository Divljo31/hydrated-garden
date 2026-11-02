---
{"dg-publish":true,"permalink":"/wiki/note-oracle-inventory/","title":"Oracle inventory & failure modes","tags":["oracles","security","monitoring","risk","inventory"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Oracle inventory & failure modes","repo":"hydration-node","paths":["pallets/ema-oracle/src/lib.rs","runtime/hydradx/src/assets.rs","runtime/hydradx/src/evm/precompiles/chainlink_adapter.rs","runtime/hydradx/src/ice_oracle_routes.rs","pallets/stableswap/src/types.rs","primitives/src/constants.rs"],"symbols":["OMNIPOOL_SOURCE","STABLESWAP_SOURCE","XYK_SOURCE","UNISWAPV3_SOURCE","GIGAHDX_SOURCE","BIFROST_SOURCE","InternalOracleSources","OracleWhitelist","ChainlinkOraclePrecompile","EMPTY_SOURCE","PegSource","DerivedRouteShortPrice","set_external_oracle","set_external_oracle_by_ids","update_bifrost_oracle","add_authorized_account"],"depends_on":["pallet-ema-oracle","hydration-precompiles","whm-oracle-relay","hydration-borrow","pepl-worker","pallet-hsm","pallet-stableswap","note-dependency-registry","note-security-hardening-plan","note-collateral-map","note-risk-register"],"tags":["oracles","security","monitoring","risk","inventory"],"last_updated":"2026-11-02"}}
---


# Oracle inventory & failure modes

**TL;DR:** Every price Hydration consumes comes from one of **seven oracle families**: (1) the on-chain **EMA oracle** fed by our own AMMs, (2) the **Chainlink-adapter precompile** that exposes EMA prices to EVM contracts, (3) **DIA** push feeds for the money market, (4) the **WHM Wormhole relay** (Kamino Scope, Jito, Lido, apyUSD vault, Chainlink SPY), (5) **external EMA pushers** (Bifrost and any account authorized via `set_external_oracle`), (6) **Aave's own `AaveOracle`** that maps each reserve to one of the above, and (7) **stableswap peg sources** (fixed value, EMA, or an MM oracle contract). On top of these the money market wraps prices in its own contracts — **ManagedOracle** (manual `setPrice`), **CheckedOracle** (±bps guard vs a pool Day-EMA), USD adapters and aggregators — and the per-reserve map is in [[wiki/note-collateral-map\|note-collateral-map]]. Below: what each one prices, who consumes it, and everything that can go wrong. Monitoring plan comes next — see [[wiki/note-security-hardening-plan\|note-security-hardening-plan]]; protocol-wide context in [[wiki/note-risk-register\|note-risk-register]].

## 1. Inventory

| # | Oracle | What it prices | Trust root | Update path | Consumers | Who controls it |
|---|---|---|---|---|---|---|
| 1 | **EMA oracle** ([[wiki/pallet-ema-oracle\|pallet-ema-oracle]]) — internal sources `omnipool`, `stablesw`, `hydraxyk`, `uniswpv3` | Every whitelisted pair traded on our AMMs; `SupportedPeriods` = `LastBlock`, `Short`, `TenMinutes`, `Day` (2 s blocks) | Our own pool state | AMM hooks every trade / liquidity change, rolled in `on_finalize` | Omnipool price barrier + dynamic withdrawal fee, dynamic fees (`Short`), route validation (`TenMinutes`), DCA, fee-currency pricing, liquidity mining + referrals (`TenMinutes`), HSM, ICE (`DerivedRouteShortPrice` = `Short` over derived routes), stableswap pegs, precompile #2 | Whitelist: `Root \| EconomicParameters` |
| 2 | **Chainlink-adapter precompile** (`0x000001…` addresses) | Any pair: empty source → on-chain **route** price via `OraclePriceProvider`; `omnipool` → asset/LRNA; other sources (e.g. `gigahdxs` stHDX) → that EMA entry. 8 decimals | Same as #1 | Read-only view over #1 | Money market reserves without an external feed (e.g. GIGAHDX market, HOLLAR-related, long-tail), outbound HDX price to Robinhood | Address encodes period + source — fixed per reserve config |
| 3 | **DIA** push feeds | WBTC, WETH, DOT, USDT, USDC, tBTC, AAVE, vDOT (DIA guide) + SOL, HYPE and EUR/USD used by main-market reserves (EURC is priced at **EUR/USD**, not EURC/USD) — 0.5% deviation / 120 s refresh / 24 h heartbeat; contracts `0xc756bd33…6917` (WBTC/WETH/DOT/USDT/USDC), `0x5d8320f3…2bf52` (tBTC/WETH/vDOT) | DIA off-chain aggregation of 100+ CEX/DEX venues | Signed txs from DIA update signers (`0x33a5e905…`, `0xff0c6240…`) | Main money market; **PEPL liquidation fast path** watches the update txs in the mempool and liquidates in the same block | DIA (contract owner + signer keys) |
| 4 | **WHM oracle relay** ([[wiki/whm-oracle-relay\|whm-oracle-relay]]) | PRIME + SOL (Kamino Scope idx 383 / 1), jitoSOL (Jito stake-pool rate), wstETH (`stEthPerToken`), apyUSD (`convertToAssets`), SPY (Chainlink SPY/USD on Robinhood) → per-asset target oracles on Hydration EVM | Upstream source + 13/19 Wormhole guardians | Broadcaster publishes VAA → relayer → renounced `OracleReceiver` (`maxPriceAge` 300 s, monotonic) → `setPrice` | Money market (non-DIA reserves) | Receivers + emitters **renounced** (no pause, no upgrade); Solana config owner = deployer key; target oracles / CheckedOracle wrappers owned by `POOL_ADMIN` = aave-manager `0xaa7e…aa7e0` per money-market deployments (referendum speed; verify on chain) |
| 5 | **External EMA pushers** — `bifrosto` (Bifrost, `update_bifrost_oracle`), any `register_external_source` + `add_authorized_account` | vDOT / vASTR rates (Bifrost); whatever pairs are authorized on chain | The pushing account | Signed extrinsic, `Pays::No` on success | Anything reading those EMA entries (pegs, MM via #2, HSM) | Register / authorize / revoke: `Root \| EconomicParameters` (no TC fast path) |
| 6 | **`AaveOracle`** (per market: main, GIGAHDX, BIL) | Maps each reserve → source contract (#2, #3, #4) + fallback oracle | Aave pool admin | `setAssetSources` / `setFallbackOracle` | All borrow / collateral / liquidation math | Pool / ACL admin — via `AaveManagerAccount` (`Root \| EconomicParameters`); emergency roles unverified |
| 7 | **Stableswap peg sources** (`PegSource`) | Drifting-peg pools: `Value` (fixed), `Oracle(source, period, asset)` (EMA), `MMOracle(EvmAddress)` (an MM oracle contract) | Whatever #1/#3/#4/#5 it points to | Read at trade time | Stableswap pools with drifting pegs (LST / aToken / strategy-token pools), HSM arbitrage | Pool config (`AuthorityOrigin`) |
| 8 | **Money-market wrapper oracles** (money-market repo) | `ManagedOracle` — manual `setPrice` (PRIME, SIGIL **fixed $1**, apyUSD, jitoSOL, wstETH, vDOT-discount, plus legacy-bridge-era copies still deployed); `CheckedOracle` — passes a feed only within `maxDiffBps` of a pool Day-EMA (apyUSD 100, PRIME 100, wstETH 50, jitoSOL 75), otherwise **REJECT** (last good price kept); `USDOracleAdapter`, `OraclesAggregator`, `HybridOracleAggregator`, `BILOracleAdapter` (vault `exchangeRate`), stHDX adapter (`gigahdxs` × HDX EMA) | Whoever can call `setPrice` / owner | Manual tx or wraps #1–#4 | `AaveOracle` (#6) | Owner `POOL_ADMIN` = aave-manager (`Root \| EconomicParameters`) |

**Indirect trust roots** (what #3/#4 themselves rely on): DIA's venue set; Kamino Scope's own aggregation (Pyth/Switchboard-style sources); Jito stake-pool accounting; Lido's oracle committee (`stEthPerToken` moves on Lido reports); the apyUSD vault's share price; Chainlink's SPY feed (US market hours); Wormhole guardians; Solana / Ethereum / Robinhood liveness.

**Outbound:** Hydration *exports* its HDX EMA price (`0x0000010200…0a00000000`, `TenMinutes`) to Robinhood BandHook pools **without a CheckedOracle-style guard** — anything that manipulates the HDX EMA also hits whoever consumes it there.

## 2. What can go wrong

### A. Staleness / liveness

1. **Pusher down** — DIA signers, WHM broadcaster or relayer, Bifrost pusher stop updating. Price freezes at the last value while the market moves → under-collateralised borrows, missed liquidations.
2. **Heartbeat too long for the asset** — DIA's 24 h heartbeat with 0.5% deviation can sit unchanged through a slow drift on a quiet feed.
3. **WHM relay** — `maxPriceAge` 300 s rejects stale VAAs, but the **target oracle keeps its last price**: consumers see a stale value, not an error.
4. **Upstream chain halts** — Solana (PRIME/SOL/jitoSOL), Robinhood (SPY), Ethereum (wstETH/apyUSD); Wormhole guardian delays.
5. **Market-hours feeds** — SPY is flat outside US trading hours, then gaps at open → sudden liquidation wave at 15:30 CET.
6. **EMA goes stale when trading stops** — delisted / paused / frozen assets stop updating; consumers (fee pricing, DCA, HSM, MM via #2) keep reading the old value. Freezing an asset as an incident response can itself freeze its price.
7. **Oracle entry missing → revert** — precompile returns "Price not available" (removed whitelist entry, new asset, no route) → MM borrow / repay / **liquidate** revert → positions can't be closed → bad debt.

### B. Wrong price

8. **EMA manipulation** — sustained trades over several blocks on thin pools; `Short` and `LastBlock` periods are cheap to move; 2 s blocks shrink every period in wall-clock time.
9. **Derived-route compounding** — empty-source precompile prices and ICE `DerivedRouteShortPrice` chain several EMA hops; one thin hop poisons the whole route.
10. **Thin new venues** — curated Uniswap v3 HOLLAR pools feed `uniswpv3` EMA entries; low liquidity = cheap to move.
11. **Exchange-rate (share-price) oracles** — apyUSD `convertToAssets`, jitoSOL stake-pool rate, wstETH `stEthPerToken` are live accounting values: vault donation / inflation attacks, accounting bugs, or slashing move them instantly; they also **don't reflect market depeg** (rate says 1.10, market pays 0.95 → collateral over-valued).
12. **Representation gap** — feeds price the *canonical* asset (BTC-ish tBTC on Ethereum, ETH, USDC) while we hold a *bridged representation* (Snowbridge tBTC, NTT USDC…). If the bridge is drained, the oracle still says full value → unbacked collateral borrows good assets (Kelp rsETH → Aave pattern).
13. **Denomination / decimals** — WHM rates are in native units (stETH per wstETH, SOL per jitoSOL) and must be multiplied by a USD leg; the adapter normalises to 8 decimals (precision loss on tiny prices); inverted pairs; wrong asset id in a peg source.
14. **Upstream manipulation** — DIA venue set or Kamino Scope sources gamed via thin CEX/DEX markets.
15. **Peg-source mistakes** — a stableswap peg bound to the wrong source / period, or to an MM oracle that is itself EMA-derived → reflexive loop (pool price → EMA → peg → pool price).

### C. Compromise / authority

16. **Signer key compromise** — DIA update keys, Bifrost pusher, any authorized external-oracle account can post arbitrary prices. Revoking an EMA pusher needs a referendum (`Root \| EconomicParameters`).
17. **Wormhole guardian quorum** — forged VAA → arbitrary WHM price; receivers can't be paused.
18. **Target-oracle owner = aave-manager** — the WHM target oracles and CheckedOracle wrappers are owned by `POOL_ADMIN` (aave-manager dispatch account): a governance-path compromise can authorise a new writer or loosen `maxDiffBps`; a legitimate fix is referendum-speed. Authorised writers still unverified.
19. **`AaveOracle` source swap** — pool admin calls `setAssetSources` / `setFallbackOracle` → any reserve repriced; a governance-path compromise is an oracle compromise.
20. **Governance attack on the EMA whitelist** — adding a malicious external source / authorized account via referendum.
21. **DIA contract upgrade / owner change** (third-party admin).

### D. Coupling with liquidations

22. **PEPL fast path acts on mempool oracle txs** — a bad DIA update gets liquidations submitted in the same block, before anyone can react.
23. **Liquidation cascade on gap moves** — SPY open, LST slashing, depeg prints.
24. **Self-referential collateral** — GIGAHDX (stHDX via `gigahdxs`) and HDX-related prices come from our own pools: pump HDX on Omnipool → borrow HOLLAR against GIGAHDX; also exported to Robinhood.

### E. Process

25. **Configured ≠ live** — the per-reserve map is now recorded from repo config in [[wiki/note-collateral-map\|note-collateral-map]], but nobody checks it against `AaveOracle.getSourceOfAsset` on chain.
26. **Silent config drift** — runtime upgrades change oracle periods / whitelist; WHM `*-checked` redeploys silently swapped target oracles before.

### F. Money-market pricing choices (from the reserve config)

27. **Manual and fixed prices** — SIGIL is a `ManagedOracle` fixed at **$1** (collateral LTV 85, Treasury holds ~1.1M aSIGIL); PRIME and others have manual oracles behind CheckedOracle. A fixed price never reflects a depeg; a manual one is only as fresh as its last `setPrice`.
28. **HOLLAR assumed $1** — the four HOLLAR stableswap pool reserves (HUSDC/HUSDT/HUSDS/HUSDE, ids 110–113) price the HOLLAR leg at $1: a HOLLAR depeg leaves that collateral over-valued.
29. **Fiat proxy** — EURC priced with DIA **EUR/USD**: an EURC depeg or a frozen bridged EURC is invisible (same class as #12).
30. **CheckedOracle freezes on REJECT** — when the feed and pool EMA diverge past `maxDiffBps` the wrapper keeps the last accepted price (apyUSD has been rejecting while its pool basis > 1%). Correct for manipulation, wrong for a real repricing; money-market docs ask for a >150 bps divergence alert + staleness alert.
31. **Legacy oracles still deployed** — legacy-bridge-era `*oracle` copies remain with live owners; not a reserve source today, but a single `setAssetSources` re-points to them.
32. **tBTC / PAXG feeds price the wrapper market** — confirmed live: DIA "tBTC/USD" (77,209 vs WBTC 76,840) and "PAXG/USD". Better than BTC/gold spot for a depeg, but still blind to *our* bridged copy (#12).

### G. Found on the live chain (2026-11-02, [[wiki/note-live-chain-check\|note-live-chain-check]])

33. **DIA owner = updater EOA** — each DIA feed contract is owned by the EOA that pushes it: `0x33a5…` for 4 feeds (~63.7k txs), `0xFf0c…` for 9 (~83.7k txs). One hot key per group can reprice those reserves and also change the contract.
34. **SPY CheckedOracle has no check** — `checkOracle = 0x0`, so any relayed price is accepted.
35. **apyUSD frozen ~47 days** — last accepted price 1.3672; the route check reads 1.4247 (+4.2%). The same frozen oracle is the **peg** of stableswap pool 146.
36. **Reflexive checks** — the PRIME and apyUSD CheckedOracles check against route prices that run through pools 143 / 146, whose pegs are those same CheckedOracles.
37. **Constant HOLLAR** — HOLLAR's source in all three markets is a 168-byte constant ($1) with no owner.
38. **Stale manual pieces by design** — vDOT discount = 1.0 (set ~111 days ago, aave-manager); SIGIL $1 (~215 days ago). USDC and USDT DIA feeds were ~20.6 h old against a 24 h heartbeat.

## 3. Unknowns to resolve first (chain reads)

- [x] `AaveOracle.getSourceOfAsset(reserve)` + `getFallbackOracle()` for all three markets → [[wiki/note-live-chain-check\|note-live-chain-check]] §3; no fallback oracle on any market.
- [ ] Confirm owner (expected aave-manager) and authorised writers of the six WHM target oracles (PRIME `0x0922…`, SOL `0xf832…`, jitoSOL `0x64b7…`, wstETH `0xBE6B…`, apyUSD `0x80Fa…`, SPY `0x8028…`) and of every ManagedOracle (SIGIL `0xe50A…`, PRIME `0xDEe5…`, apyUSD `0x286B…`, legacy copies).
- [x] What the tBTC (`0xe5Ac…D9c1`) and PAXG (`0x8fB6…`) sources price → DIA tBTC/USD and PAXG/USD.
- [x] `EmaOracle.ExternalSources` + `AuthorizedAccounts` → only `bifrosto`, one authorized account (Bifrost sibling sovereign, DOT/vDOT).
- [x] Stableswap pegs for money-market pools → [[wiki/note-live-chain-check\|note-live-chain-check]] §3 (all `MMOracle`; none use EMA `Oracle`).
- [x] DIA contract owner = updater EOA (`0x33a5…`, `0xFf0c…`); still open: confirm feed ↔ Hydration asset id mapping (e.g. which WETH / tBTC representation).

## Sources

- [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/hydration-precompiles\|hydration-precompiles]], [[wiki/whm-oracle-relay\|whm-oracle-relay]], [[wiki/pepl-worker\|pepl-worker]], [[wiki/pallet-hsm\|pallet-hsm]], [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/note-dependency-registry\|note-dependency-registry]]
- `runtime/hydradx/src/assets.rs` (oracle periods per consumer, `InternalOracleSources`), `primitives/src/constants.rs` (source ids), `runtime/hydradx/src/evm/precompiles/chainlink_adapter.rs` (empty-source route pricing), `runtime/hydradx/src/ice_oracle_routes.rs`, `pallets/stableswap/src/types.rs` (`PegSource`) — `raw/hydration-node` @ a35840f
- `galacticcouncil/money-market` @ 9ba0c52 — `markets/hydration/` (reserve configs, gigahdx, bil), `deployments/`, oracle docs
- DIA Hydration feeds — https://nexus.diadata.org/how-to-guides/fetch-price-data/chain-specific-guide/hydration
- Aave tBTC onboarding discussion (BTC/USD vs market feed, PoR-triggered LTV 0) — https://governance.aave.com/t/arfc-onboard-tbtc-to-aave-v3-on-ethereum-arbitrum-and-optimism/17686/11
