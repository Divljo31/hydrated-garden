---
{"dg-publish":true,"permalink":"/wiki/omnipool/","title":"Omnipool","tags":["amm","trading","liquidity","defi","core-product"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"Omnipool","tags":["amm","trading","liquidity","defi","core-product"],"source_count":2,"last_updated":"2026-04-13"}}
---


# Omnipool

**TL;DR:** The Omnipool is [[wiki/hydration\|hydration]]'s flagship AMM — a novel multi-asset pool consolidating all liquidity into one pool with virtual [[wiki/lrna\|lrna]]-based sub-pools for any-to-any trading, implemented as a Substrate pallet with dynamic fees, IL mitigation, and multiple risk controls.

## Design

The core insight: instead of fragmenting liquidity across N*(N-1)/2 pairs, every asset is paired with [[wiki/lrna\|lrna]] (the synthetic hub token) internally. From a trader's perspective, they swap any asset for any other in a single transaction with no multi-hop routing overhead. Internally, the trade routes TKN1→LRNA→TKN2.

Each TKN/LRNA virtual sub-pool uses a constant product invariant (`Q_i * T_i = k`). Asset price in LRNA terms = `Q_i / T_i`. Asset weight = ratio of sub-pool LRNA to total LRNA across all sub-pools.

## Key Properties

- Single-pool architecture — no liquidity fragmentation
- Single-sided LP provisioning — LPs deposit one asset only
- LP positions represented as NFTs (see [[wiki/nft-lp-positions\|nft-lp-positions]])
- Listings permissioned via [[wiki/opengov\|opengov]] governance vote
- All arithmetic at runtime level (Substrate pallet), not EVM
- Audited by [[wiki/runtime-verification\|runtime-verification]], [[wiki/blockscience\|blockscience]], and [[wiki/code4rena\|code4rena]]

## Swap Mechanics

Two fees per trade:
- **Asset fee (`fA`)** — charged in the output asset, stays in the pool as LP rewards
- **Protocol fee (`fP`)** — charged in LRNA, flows to the protocol for [[wiki/hdx\|hdx]] buybacks

**Sell formula:**
```
ΔQ₁ = Q₁ * (-ΔT₁ / T₁⁺)
ΔQ₂ = -ΔQ₁ * (1 - fP)
ΔT₂ = T₂ * (-ΔQ₂ / Q₂⁺) * (1 - fA)
```

Fees are [[wiki/dynamic-fees\|dynamic]] — they adjust based on asset volatility to make price manipulation less profitable.

## Liquidity Provisioning

LPs call `add_liquidity(asset, amount)` to deposit a single asset. The pallet mints corresponding LRNA, creates shares, and issues an NFT position storing shares, entry price (`p₀`), and asset ID.

**Withdrawal depends on price movement since entry:**
- **Price fell:** LP receives only TKN. The protocol claims a portion of shares (becomes [[wiki/protocol-owned-liquidity\|POL]]).
- **Price rose:** LP receives TKN + LRNA as [[wiki/impermanent-loss\|IL]] compensation. Excess LRNA is burned.

A [[wiki/price-barrier\|price-barrier]] check (1% spot/oracle deviation threshold) gates both add and remove liquidity operations.

## Impermanent Loss Profile

IL formula: `IL = 2*sqrt(p * p₀) / (p₀ + p) - 1`. Because [[wiki/lrna\|lrna]] is a weighted price index of the whole pool basket, market-correlated assets experience lower IL than a TKN/stablecoin XYK pool. An asset's weight in the pool affects IL — larger weight → lower IL. See [[wiki/impermanent-loss\|impermanent-loss]] for worked examples.

## Risk Controls

- [[wiki/circuit-breaker\|circuit-breaker]] — 50% per-block trade volume limit per asset
- [[wiki/price-barrier\|price-barrier]] — 1% spot/oracle deviation threshold for liquidity ops
- [[wiki/dynamic-fees\|dynamic-fees]] — volatility-adjusted trading and withdrawal fees
- Weight caps — bounds maximum exposure per asset
- [[wiki/ema-oracle\|ema-oracle]] — time-weighted average price data
- [[wiki/tradability-flags\|tradability-flags]] — per-asset operation permissions (SELL/BUY/ADD_LIQUIDITY/REMOVE_LIQUIDITY)
- Targeted function pausing by Technical Committee

## On-Chain Data Model

Each asset tracked in `Assets` storage map as `AssetState<Balance>`:
- `hub_reserve` — LRNA in this sub-pool
- `shares` — total LP shares
- `protocol_shares` — protocol-owned shares (from IL claims)
- `cap` — maximum weight cap (`Permill`)
- `tradable` — bitflags controlling permitted operations

Global state: `HubAssetImbalance` (net LRNA deviation) and `HubAssetTradability` (LRNA tradability — SELL only by default).

## Pallet Extrinsics

`add_token`, `add_liquidity`, `remove_liquidity`, `sell`, `buy`, `set_asset_tradable_state`, `set_asset_weight_cap`, `refund_refused_asset`, `sacrifice_position`.

## Math Module

Separated into `hydra-dx-math` crate for auditability. Core functions: `calculate_sell_state_changes`, `calculate_buy_state_changes`, `calculate_add_liquidity_state_changes`, `calculate_remove_liquidity_state_changes`, `calculate_withdrawal_fee`, `calculate_cap_difference`. All use `FixedU128` and checked/saturating arithmetic.

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
