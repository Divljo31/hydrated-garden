---
{"dg-publish":true,"permalink":"/wiki/note-live-chain-check/","title":"Live chain check — 2026-11-02","tags":["security","monitoring","oracles","money-market","live-state","snapshot"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Live chain check — 2026-11-02","tags":["security","monitoring","oracles","money-market","live-state","snapshot"],"depends_on":["note-collateral-map","note-oracle-inventory","note-risk-register","note-emergency-levers","note-security-hardening-plan","note-dependency-registry"],"last_updated":"2026-11-02"}}
---


# Live chain check — 2026-11-02

**TL;DR:** First read of live state, block **16,540,332**, runtime **spec 455**. The repo master is spec 458, so some levers documented from master are **not live yet**. Reads were made from a browser on Div's machine against `rpc.hydradx.cloud`: the sandboxes are blocked from every Hydration RPC. Each finding below either corrects [[wiki/note-collateral-map\|note-collateral-map]] / [[wiki/note-oracle-inventory\|note-oracle-inventory]] or answers a Phase-0 item in [[wiki/note-security-hardening-plan\|note-security-hardening-plan]]. This is a snapshot, so re-read before acting on any number.

## 1. Biggest findings

1. **The TC fast path cannot touch the main money market.** On the main and BIL markets, `EmergencyAdminAccount` (`0xaa7e…aa7e1`, TC majority) holds **no** Aave role; on GIGAHDX it holds only `EMERGENCY_ADMIN`. Pausing or freezing a main-market reserve therefore needs aave-manager (`Root | EconomicParameters`), which is referendum speed.
2. **`suspend_xcm` is not live.** Live `Parameters` has only `setUniswapV3Addresses` and `setAaveGasLimits`. The single-member XCM kill switch exists on master only. Live XCM halts are `PolkadotXcm.force_suspension` (master `AdminOrigin` = Root | TC 2/3 | GeneralAdmin; the barrier still has to be verified to honour it), tx-pause on outbound XCM calls (TC majority), and the circuit-breaker lockdown and deposit fuse (TC majority).
3. **The apyUSD price has been frozen for ~47 days.** The `APYUSD-CheckedOracle` last accepted a price on ~2026-09-16, at **1.3672**. Its check (Day-EMA route HOLLAR→apyUSD) now reads **1.4247 (+4.2%)**, beyond `maxDiffBps` 100, so every push is rejected. Exposure: 1.42M apyUSD supplied (~$1.94M) at LTV 85, plus the 2-Pool-apyUSD reserve, whose stableswap **peg is this same frozen oracle**.
4. **SIGIL reserve is live with zero issuance.** `Tokens.TotalIssuance(816)` = 0 and aSIGIL supply = 0. The reserve is still collateral: fixed $1 `ManagedOracle` (last set ~215 days ago), LTV 85 / LT 88, supply cap 1.1M. Anyone able to mint SIGIL (a local token, so a governance-level action) could borrow ~$0.9M against it.
5. **One EOA updates nine DIA feeds.** Each DIA feed contract is owned by the same EOA that pushes its updates:
   - `0x33a5…054e` (~63.7k txs): USDC, USDT, WBTC, DOT.
   - `0xFf0c…07d3` (~83.7k txs): tBTC, ETH, PAXG, SOL, EUR/USD (EURC), HYPE, ZEC, plus the sUSDS and sUSDe peg feeds.

   Compromising one key reprices that whole group.
6. **SPY has no cross-check.** `SPY-CheckedOracle.checkOracle` = `0x0`, so any price pushed by the Robinhood receiver is accepted. Exposure is small (~$20k supplied).
7. **HOLLAR is a constant $1 in all three markets.** Its source `0x6096…34F8` is a 168-byte contract with no owner and no inputs.
8. **The GIGAHDX HOLLAR facilitator is at its cap**: 999,998 of 1,000,000, so no new HOLLAR borrows there.

## 2. Money market — live vs repo config

| | Main | GIGAHDX | BIL |
|---|---|---|---|
| PoolAddressesProvider | `0xf3Ba…A691` | `0x3C7D…3973` | `0x653D…990B` |
| Reserves | **32** (config map had 24) | stHDX, HOLLAR | uBIL, HOLLAR |
| Supplied / borrowed | ~$42.3M / ~$16.7M (HOLLAR $10.09M) | ~$9.14M / ~$1.01M | ~$3.31M / ~$0.76M |
| Fallback oracle | none | none | none |
| ACL — aave-manager `0xaa7e…aa7e0` | DEFAULT_ADMIN, POOL_ADMIN, EMERGENCY_ADMIN | + RISK_ADMIN | + RISK_ADMIN |
| ACL — EmergencyAdmin `0xaa7e…aa7e1` (TC) | **none** | EMERGENCY_ADMIN | **none** |

Other role holders could not be enumerated: `eth_getLogs` on the public RPC times out at 10 s even for a 100k-block range. Use an indexer, or Blockscout, next time.

**Reserves the config map missed** (main): HOLLAR (debt only), USDC 21 and USDT 23 (Wormhole copies), ZEC 1001357, SPY 1001356, 2-Pool-PRIME 143, 2-Pool-apyUSD 146, 2-Pool-BIL 10055, and the legacy 3-pool 105 (USDC 21 / USDT 23 / HOLLAR). stHDX (670) is a GIGAHDX reserve, not a main one. Pools 143, 146, 10055 and 105 have **supply cap 1**, closed to new deposits, but still hold ~$3.3M of collateral.

**Corrections to isolation debt ceilings:** PRIME **$12M** (not 1.2B), apyUSD **$1M** (not 222.2M), stHDX none.

**eModes (live = config):** Stablecoins 90/93, DOT 85/92, ETH 80/90, SOL 80/85, EUROZONE 80/85.

**Largest collateral (main):**
- PRIME ~$7.1M, with $2.68M PRIME borrowed.
- DOT ~$4.1M.
- 2-Pool-GETH ~$3.8M.
- 2-Pool-GDOT ~$3.5M.
- tBTC ~$3.2M (41 tBTC).
- ETH ~$2.4M.
- 2-Pool-HUSDT / HUSDC ~$2.0M / $1.8M.
- apyUSD ~$1.9M.

## 3. Price sources — what each actually reads

| Source | Used by | What it is | Owner / updater | Age at read |
|---|---|---|---|---|
| DIA `0x1771…`, `0x8b0D…`, `0xeDD9…`, `0xFBCa…` | USDC (22, 21), USDT (10, 23), WBTC, DOT | "X/USD Oracle" | EOA `0x33a5…` | USDC/USDT **~20.6 h** (24 h heartbeat), DOT 11 min |
| DIA `0xe5Ac…D9c1` | tBTC | **tBTC/USD market** (not BTC/USD): 77,209 vs WBTC 76,840 | EOA `0xFf0c…` | 2.5 h |
| DIA `0x8fB6…c105` | PAXG | **PAXG/USD market** (not gold spot) | EOA `0xFf0c…` | 4.6 h |
| DIA `0x1AF5…`, `0x2FAA…`, `0xaa47…`, `0xa2F5…`, `0xD76F…` | ETH, SOL, EURC (**EUR/USD**), HYPE, ZEC | X/USD | EOA `0xFf0c…` | 12 min – 1.8 h |
| CheckedOracle PRIME `0x0922…` | PRIME | WHM push (pusher `0x582e…`), check = route HOLLAR→PRIME, 100 bps | aave-manager | 15 min, within band |
| CheckedOracle apyUSD `0x80Fa…` | apyUSD | WHM push (pusher `0x6913…`), check = route HOLLAR→apyUSD, 100 bps | aave-manager | **~47 days (rejecting)** |
| CheckedOracle SPY `0x8028…` | SPY | WHM push (pusher `0x3b6E…`), **no check oracle** | aave-manager | 1.8 h |
| ManagedOracle SIGIL `0xe50A…` | SIGIL | fixed 1.00 | aave-manager | ~215 days |
| Constant `0x6096…` | HOLLAR ×3 markets | fixed 1.00 | none | — |
| USDOracleAdapters | vDOT, GDOT, GETH, 3-Pool, GSOL, HEURC | EMA leg (`bifrosto` DOT/vDOT TenMinutes; stableswap share vs aToken TenMinutes) × DIA USD leg | none | live |
| Precompile `0x00000102 'stablesw' …DE…` | HUSDC/T/S/e, 2-Pool-PRIME/apyUSD/BIL, 105 | stableswap EMA (TenMinutes) priced **in HOLLAR** → $1 | — | live |
| stHDX adapter `0x645C…` | stHDX | `gigahdxs` stHDX/HDX × route USDT/HDX (Day) | none | live |
| BILOracleAdapter `0xB994…` | uBIL | vault `0x6a21…E810C` exchange rate | none | live |

**Stableswap pegs that read money-market oracles:**
- 143 PRIME/HOLLAR → PRIME CheckedOracle
- 146 apyUSD/HOLLAR → apyUSD CheckedOracle (frozen)
- 90001 GSOL → jitoSOL CheckedOracle
- 4200 GETH → wstETH CheckedOracle
- 690 GDOT → vDOT-discount aggregator (manual discount 1.0, set ~111 days ago, × Bifrost rate)
- 10044 HEURC → DIA EUR/USD
- 10055 BIL → BIL vault rate
- 112 → DIA sUSDS
- 113 → DIA sUSDe

The checks for 143 and 146 read prices through the same pools those oracles peg, so the cross-check is not independent.

## 4. Runtime state

- **TC:** 7 members, no prime, no open motions. Majority = 4, 2/3 = 5.
- **Circuit breaker:**
  - No asset in lockdown and no global withdraw lockdown.
  - The global withdraw limit is set with a 6 h window and the accumulator is at ~1% of the limit.
  - 11 egress accounts are registered (sibling sovereigns).
- **Deposit fuse** (`xcm_rate_limit`, per day on issuance increase):
  - Every bridged collateral has one.
  - No fuse on: HOLLAR, uBIL, stHDX, SIGIL, and pool shares 143, 146, 90001.
  - Fuse vs money-market exposure:

    | Asset | Fuse per day | MM exposure | Fuse vs exposure |
    |---|---|---|---|
    | DOT | 5M DOT (~$4.7M) | $4.1M | fuse exceeds all of it |
    | USDC / USDT (Asset Hub) | $5M each | — | — |
    | USDC (Wormhole) | $1.5M | — | — |
    | tBTC | 1.5 (~$116k) | $3.2M | tight |
    | ETH | 150 (~$338k) | $2.4M | tight |
    | PRIME | ~449k (~$480k) | $7.1M | tight |
    | apyUSD | 500k (~$684k) | $1.9M | tight |
- **Tx-pause list:**
  - `Stableswap.buy`
  - `EVM.create` / `create2` (deploys go through the 7-entry deployer whitelist)
  - `PolkadotXcm.claim_assets`
  - `OtcSettlements.settle_otc_order`
  - a stale `Elections.vote`
- **EMA external sources:** only `bifrosto`, with a single authorized account: the Bifrost sibling sovereign for DOT/vDOT. No other pushers.
- **NTT minters:** set for 15 assets, including 19, 20, 21, 23, 43, 44, SOL, HYPE, SPY and ZEC.
- **HSM:** collaterals are aUSDT (1002, pool 111) and aUSDC (1003, pool 110), `maxInHolding` 8M each. The flash minter is `0xb328…cf4`.
- **HOLLAR facilitators** (total supply 11.95M):

  | Facilitator | Level / cap |
  |---|---|
  | Main | 9.97M / 13M |
  | GIGAHDX | 1.0M / 1.0M |
  | BIL | 0.75M / 0.95M |
  | HSM | 0.23M / 18M |
  | FlashMinter | 0 / 0.1M |
- **Pallets live:** ICE (99), Intent (98) and LazyExecutor (97) are live, and Democracy (19) is still present. `Liquidation.borrowingContract` = main pool.

## 5. Still open

- [ ] Enumerate all Aave role holders (RoleGranted logs via an indexer or Blockscout).
- [ ] Confirm live `PolkadotXcm` `AdminOrigin`, and that the XCM barrier honours `force_suspension`.
- [ ] Global withdraw limit units (reference currency) — confirm what "100" means.
- [ ] What the WHM receivers last pushed for apyUSD (the rejected values), to size the drift.
- [ ] Who could mint SIGIL, and whether the reserve should be frozen until it exists.

## Sources

- Live reads 2026-11-02 ~20:30 CET via `https://rpc.hydradx.cloud` (Substrate + EVM JSON-RPC, chain id 222222). Addresses from `galacticcouncil/money-market` `deployments/{hydration,gigahdx,bil}` and resolved on chain through each `PoolAddressesProvider`.
- Runtime origins from `raw/hydration-node` master (spec 458): `runtime/hydradx/src/{assets.rs,xcm.rs,system.rs,governance/mod.rs}`.
