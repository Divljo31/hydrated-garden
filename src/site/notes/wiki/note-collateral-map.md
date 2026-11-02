---
{"dg-publish":true,"permalink":"/wiki/note-collateral-map/","title":"Collateral map — every reserve, its dependency chain and what to watch","tags":["collateral","money-market","oracles","dependencies","risk","monitoring"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"note","title":"Collateral map — every reserve, its dependency chain and what to watch","repo":"money-market","paths":["markets/hydration/index.ts","markets/hydration/reservesConfigs.ts","markets/gigahdx/index.ts","markets/bil/index.ts","docs/checked-oracles-hydration.md","docs/checked-oracle-robinhood.md","deployments/hydration/","tasks/proposals/update-manual-oracle-prices.ts","tasks/proposals/sigil-launch.ts"],"depends_on":["note-oracle-inventory","note-dependency-registry","note-risk-register","hydration-borrow","hollar","whm-oracle-relay","snowbridge","wormhole"],"tags":["collateral","money-market","oracles","dependencies","risk","monitoring"],"last_updated":"2026-11-02"}}
---


# Collateral map — every reserve, its dependency chain and what to watch

**TL;DR:** Hydration runs three Aave-v3 markets — **main** (24 configured reserves; **32 live** — see [[wiki/note-live-chain-check\|note-live-chain-check]]), **GIGAHDX** (stHDX only) and **BIL** (vault share only) — and HOLLAR is minted as a GHO-style debt asset in main and BIL. Every reserve is a stack: *issuer/custody → bridge or XCM reserve → Hydration asset id → price source → market params → where else it sits (stablepools, Uniswap v3, HSM)*. Source: `galacticcouncil/money-market` @ `9ba0c52` (2026-10-27) — **deployment config, not live state**; governance has since moved some sources (e.g. PRIME/apyUSD → CheckedOracles). Live mapping needs `AaveOracle.getSourceOfAsset` per market. See [[wiki/note-oracle-inventory\|note-oracle-inventory]] for oracle families and [[wiki/note-risk-register\|note-risk-register]] for the protocol-wide list.

> **Live check 2026-11-02 (block 16,540,332, spec 455) — [[wiki/note-live-chain-check\|note-live-chain-check]] supersedes the config tables below where they differ:**
> - The main market has **32 live reserves**. Missing from the tables below: HOLLAR debt, USDC 21 / USDT 23 (Wormhole copies), ZEC, SPY, 2-Pool-PRIME 143, 2-Pool-apyUSD 146, 2-Pool-BIL 10055, legacy 3-pool 105. stHDX is GIGAHDX-only.
> - **Isolation ceilings:** PRIME $12M, apyUSD $1M.
> - **tBTC and PAXG feeds:** DIA **tBTC/USD** and **PAXG/USD** market feeds, not BTC or gold spot.
> - **SIGIL:** total issuance **0**, but the reserve is still collateral at a fixed $1.
> - **apyUSD:** price frozen ~47 days.
> - **TC role:** none on the main market.

## 1. Main market (`ProviderId 222222`)

Params from `reservesConfigs.ts` (LTV / liquidation threshold in bps; caps in token units as configured; "—" = inherited / not set in file).

| Reserve (id) | What it is / route in | Price source as configured | LTV / LT | Borrowable | Notes |
|---|---|---|---|---|---|
| USDC (22) | Circle USDC, Asset Hub reserve | DIA USDC/USD `0x1771…b954` | 80 / 90 | yes | Stable eMode (90/93) |
| USDT (10) | Tether USDT, Asset Hub reserve | DIA USDT/USD `0x8b0D…d5B` | — | yes | Also the USD leg for 3-POOL, HDX, BNC adapters |
| WBTC (19) | BitGo WBTC via **Wormhole NTT** (Ethereum hub, 4/7 Safe) | DIA WBTC `0xeDD9…04f` | 60 / 70 | — | Frozen per `hype-mm.ts` note; no Omnipool leg |
| DOT (5) | Relay-chain DOT | DIA DOT/USD `0xFBCa…702` | 80 / 85 | yes | DOT eMode (85/92) with vDOT, gDOT |
| VDOT (15) | Bifrost vDOT (reserve: Bifrost para 2030) | `USDOracleAdapter`: **`bifrosto` EMA** DOT/vDOT 10 min × DIA DOT/USD | 60 / 70 | yes | A `vDOT-Discount` HybridOracleAggregator + manual discount oracle also deployed |
| **TBTC (1000765)** | Threshold tBTC via **Snowbridge** | external aggregator `0xe5Ac…D9c1` (DIA — market vs BTC proxy unverified) | 80 / 85 | yes | See §5 worked example |
| 2-POOL-GDOT (690) | stableswap share (aDOT + vDOT) | adapter: stableswap 10 min aDOT/gDOT × DIA DOT/USD | 69 / 75 | no | gDOT supply incentives (asset 69) |
| ETH (34) | ETH via **Snowbridge** | DIA ETH/USD `0x1AF5…94b` | 75 / 85 | yes | ETH eMode (80/90) |
| 2-POOL-GETH (4200) | stableswap share (wstETH + aETH) | adapter: stableswap 10 min aETH/gETH × ETH/USD | 80 / 85 | no | Pool peg = wstETH CheckedOracle (Lido `stEthPerToken` via WHM) |
| 3-POOL (103) | stableswap share (stables) | adapter: aUSDT/3-POOL 10 min × DIA USDT | 75 / 85 | no | |
| 2-POOL-HUSDC / HUSDT / HUSDS / HUSDE (110–113) | HOLLAR stablepool shares (HOLLAR + USDC / USDT / sUSDS / sUSDe) | **precompile only**: `stablesw` 10 min HOLLAR/share — **no USD leg (HOLLAR ≡ $1)** | eMode only | — | PRIME supply incentives on HUSDT/HUSDC |
| PAXG (39) | Paxos gold via **Snowbridge** | external `0x8fB6…105` (DIA PAXG) | 70 / 75 | yes | |
| PRIME (43) | PRIME via **NTT from Solana** | was `PRIMEoracle` **manual** `0xDEe5…307`; target: PRIME **CheckedOracle** `0x0922…2Cd` (Kamino Scope via WHM, ±100 bps vs HOLLAR→PRIME day-EMA) | 85 / 88 | yes | **Isolation**: debt ceiling 1.2B; supply cap 15M |
| APYUSD (46) | apyUSD ERC-4626 share via **Snowbridge** | was `apyUSDoracle` **manual** `0x286B…F96`; target: **CheckedOracle** `0x80Fa…Caf` (`convertToAssets` via WHM) — **REJECTs while pool basis > 1%** (frozen at 1.3672 on 2026-09-16) | 85 / 88 | yes | **Isolation**: debt ceiling 222.2M |
| **SIGIL (816)** | "Sigil Stable", **local asset, no location, `xcmRateLimit: null`** (no deposit fuse) | `SIGILoracle` ManagedOracle **fixed $1** `0xe50A…62F` | 85 / 88 | no | Treasury-held 1.1M supplied as aSIGIL (1816) |
| SOL (1000752) | SOL via **NTT from Solana** | DIA SOL/USD `0x2FAA…2C` | 70 / 75 | yes | SOL eMode (80/85) |
| 2-POOL-GSOL (90001) | stableswap share (jitoSOL + aSOL) | adapter: aSOL/gSOL 10 min × DIA SOL/USD | 60 / 70 | no | Pool peg = jitoSOL CheckedOracle (Jito rate via WHM); `jitoSOLOracle` manual also deployed |
| HYPE (1001355) | HYPE via **NTT from HyperEVM** (our locking manager) | DIA HYPE/USD `0xa2F5…a7` | 70 / 75 | yes | Newest listing (PR #81, 2026-10-27) |
| EURC (44) | Circle EURC via **NTT from Base** | **DIA EUR/USD** `0xaa47…75c` (fiat EUR, not EURC market) | 75 / 80 | yes | Eurozone eMode (80/85) |
| 2-POOL-HEURC (10044) | stableswap share (aEURC + HOLLAR?) | adapter: aEURC/HEURC 10 min × DIA EUR/USD | eMode only | — | Drifting peg uses DIA EUR/USD |
| STHDX (670) | GIGAHDX staking token | adapter: `gigahdxs` stHDX/HDX 10 min × HDX/USD | 40 / 70 | no | Debt ceiling 100M; also the only GIGAHDX-market collateral |
| (WETH 20) | NTT WETH | DIA WETH | — | — | Commented out in `ReserveAssets` |

Incentive reward oracles (not collateral): HDX `0xea63…d13` = adapter USDT/HDX 10 min **Omnipool EMA** × DIA USDT; BNC `0xc94c…cE` = USDT/BNC EMA × DIA USDT.

## 2. GIGAHDX market (`ProviderId 22222269`)

| Reserve | Price | LTV / LT | Debt asset |
|---|---|---|---|
| STHDX (670), `LockableAToken` (non-transferable GIGAHDX) | **fully internal**: `gigahdxs` stHDX/HDX TenMinutes × **Omnipool EMA HDX/USD (Day)** (switched from DIA HDX) | 40 / 70 | HOLLAR (protocol-funded `liquidate_gigahdx` borrows HOLLAR from main) |

**Reflexive:** collateral price, liquidation proceeds and debt are all Hydration-native (HDX pools, HOLLAR). Pumping HDX on Omnipool raises borrowing power; an HDX crash liquidates into HDX liquidity we also provide.

## 3. BIL market

| Reserve | Price | LTV / LT | Debt asset |
|---|---|---|---|
| uBIL (550) — BIL vault share (ERC-4626 / ERC-7540) | `BILOracleAdapter` reading the vault `exchangeRate` (auto-wired at init) | 50 / 85 | HOLLAR (`GhoAToken-HOLLAR-BIL`) |

Instant redeem via BIL/HOLLAR stableswap pool 10055. Trust root = vault NAV accounting + the RWA behind it.

## 4. HOLLAR and exports

- **HOLLAR facilitators:** `GhoAToken-HOLLAR-Main` `0x8C0f…08e`, `GhoAToken-HOLLAR-BIL` `0xEf31…E3E`; HSM collateral (sUSDS etc.) — [[wiki/pallet-hsm\|pallet-hsm]]. HOLLAR is borrowed against *every* collateral above → HOLLAR inherits every reserve's dependency chain.
- **Main-market pricing assumes HOLLAR = $1** for the four HOLLAR stablepool reserves (no USD leg) — a HOLLAR depeg is invisible to their collateral value.
- **Exports:** Hydration's HDX/USD (Omnipool EMA) is pushed over WHM to an **unchecked** `HDX-CheckedOracle` on Robinhood that prices three BandHook pools (HDX/HOLLAR, ETH/HDX) in `galacticcouncil/liquidity`; HDX and HOLLAR are NTT-locked here with spokes on Robinhood.

## 5. Worked example — tBTC end to end

| Hop | What it is | What can go wrong | Signal to watch |
|---|---|---|---|
| 1. Bitcoin custody | BTC in Threshold wallets, 51-of-100 threshold ECDSA by a **permissioned** signer set (Beta Stakers); ~8 live wallets; ~1.03 BTC per tBTC at late-Oct 2026 | Signer collusion / key compromise; wallet liveness (heartbeat, moving-funds failures) | BTC held in tBTC wallets vs tBTC total supply (all chains); wallet state events |
| 2. Minting on Ethereum | Bridge + TBTCVault; **Optimistic Minting** (Minters request, any Guardian can cancel in a ~3 h window) — **paused since a failed malicious-mint attempt on 2026-05-18**; standard sweep minting continues | Mint without BTC (the May attempt); governance / upgrade of Bridge, Vault, token; minter / guardian set change; unpause | `OptimisticMintingRequested/Cancelled/Finalized`, `MinterAdded/Removed`, `GuardianAdded/Removed`, pause flag, proxy `Upgraded`, Threshold DAO proposals |
| 3. Market | tBTC/BTC price on Ethereum DEXes; Aave Ethereum tBTC reserve | Depeg; peer protocols freezing tBTC | tBTC/BTC on main venues; Aave tBTC reserve freeze / LTV changes (peer reaction) |
| 4. Snowbridge | Lock on Ethereum Gateway → BridgeHub → **Asset Hub foreign asset** → reserve-backed XCM to Hydration | Gateway / light-client bug (unbacked AH supply); Snowbridge halt (RFC-0166) or velocity-cap trip (RFC-0167) | Gateway tBTC escrow ≥ AH tBTC supply ≥ Hydration-held tBTC; Snowbridge operating mode |
| 5. Hydration asset 1000765 | XCM deposit into Hydration | Over-limit inflow → deposit fuse (if `xcm_rate_limit` set — **unverified**) | Issuance jumps; `AssetLockdown`; fresh accounts receiving tBTC |
| 6. Price | Aggregator `0xe5Ac…D9c1` (DIA) | If it prices BTC rather than tBTC-market, an unbacked/depegged tBTC keeps full value (Aave's own tBTC onboarding debate) | Oracle answer vs tBTC/BTC market vs BTC/USD; staleness vs heartbeat |
| 7. Money market | TBTC reserve, LTV 80 / LT 85, borrowable; aToken atBTC (1006) | Unbacked tBTC supplied → HOLLAR/stables borrowed out (Kelp pattern) | Large supply followed by borrow within N blocks by same/fresh account; cap utilisation jumps |
| 8. Elsewhere on Hydration | Uniswap v3 **atBTC/HOLLAR** curated pool (Gamma vault), feeds `uniswpv3` EMA | Thin pool → EMA manipulation; LP drain | Pool price vs oracle; large one-sided swaps |

Response levers (from [[wiki/note-emergency-levers\|note-emergency-levers]]): freeze TBTC reserve / LTV 0 (referendum-speed unless the Aave emergency role is granted to TC), `lockdown_asset(1000765)`, tradability flags on any Omnipool/stablepool leg, `suspend_xcm` (blunt).

## 6. Per-asset "weird movement" signals (apply to every reserve)

1. **Inflow → supply → borrow** within a short window by a fresh or newly funded account (the unbacked-collateral pattern).
2. **Supply / borrow cap utilisation** step changes; debt-ceiling usage on isolation assets (PRIME $12M, apyUSD $1M live).
3. **Issuance jumps** of the Hydration asset not matched by an upstream lock (bridge invariant).
4. **Price divergence**: reserve oracle vs Hydration pool EMA vs external market (the CheckedOracle `checkPrice` vs feed alert the money-market doc already asks for: > 150 bps).
5. **Oracle staleness** vs expected heartbeat; manual oracles not updated in N days.
6. **Peg drift** of LST/stable shares (gDOT, gETH, gSOL, HOLLAR pools) vs their peg source.
7. **Liquidation spikes** / health-factor clustering just above 1 for one collateral.
8. **Egress after inflow** — bridged asset arrives and good assets leave via XCM / NTT within minutes.
9. **Peer reactions** — Aave / Morpho / Compound freezing an asset we list; issuer pausing / blacklisting.
10. **Admin / governance** events on the asset's upstream contracts (owner, upgrade, minter, pauser changes).

## 7. Open — needs chain reads

- [x] Live `AaveOracle.getSourceOfAsset` + caps / LTV for every reserve in all three markets → [[wiki/note-live-chain-check\|note-live-chain-check]] §2–3.
- [ ] Which deployed manual oracles are still referenced anywhere — no live reserve reads a legacy-bridge-era copy; pegs and adapters still to sweep.
- [x] `xcm_rate_limit` (deposit fuse) per collateral asset → [[wiki/note-live-chain-check\|note-live-chain-check]] §4.
- [x] What `0xe5Ac…D9c1` (tBTC) and `0x8fB6…105` (PAXG) price → DIA tBTC/USD and PAXG/USD market feeds.
- [ ] SIGIL issuer and backing — issuance is 0 today; who can mint it?

## Sources

- `galacticcouncil/money-market` @ 9ba0c52 — `markets/{hydration,gigahdx,bil}`, `deployments/hydration/*.json`, `docs/checked-oracles-hydration.md`, `docs/checked-oracle-robinhood.md`, `tasks/proposals/{update-manual-oracle-prices,sigil-launch,hype-mm}.ts`, `helpers/well-known-addresses.json` (read 2026-11-02)
- Threshold tBTC: https://docs.threshold.network/applications/tbtc-v2/the-path-to-permissionlessness · https://tbtc.network/developers/tbtc-technical-system-overview · https://docs.threshold.network/app-development/tbtc-contracts-api/tbtc-v2-api/tbtcoptimisticminting · https://threshold.network/blog/may-2026-recap-tightening-security-and-scaling-real-bitcoin-yield · https://phemex.com/academy/what-is-tbtc-bitcoin-bridge
- Aave tBTC pricing debate: https://governance.aave.com/t/arfc-onboard-tbtc-to-aave-v3-on-ethereum-arbitrum-and-optimism/17686/11
- [[wiki/note-oracle-inventory\|note-oracle-inventory]], [[wiki/note-dependency-registry\|note-dependency-registry]], [[wiki/whm-oracle-relay\|whm-oracle-relay]], [[wiki/note-emergency-levers\|note-emergency-levers]], [[wiki/hollar\|hollar]], [[wiki/hydration-borrow\|hydration-borrow]]
