---
{"dg-publish":true,"permalink":"/wiki/source-omnipool-deep-context/","title":"Omnipool — Deep Context","tags":["omnipool","amm","liquidity","defi","math","security"],"dg-note-properties":{"type":"source","title":"Omnipool — Deep Context","author":"Hydration team (internal AI context document)","date_ingested":"2026-04-13","source_date":"2025-09-22","tags":["omnipool","amm","liquidity","defi","math","security"]}}
---


# Source: Omnipool — Deep Context

**Raw file:** `raw/omnipool-deep-context.md`

## Summary

**TL;DR:** Detailed technical reference for the [[omnipool\|omnipool]] covering AMM model, swap math, liquidity mechanics, fees, on-chain data model, extrinsics, math module, risk controls, governance, and audits — the most technically dense source in the wiki.

## Key Points

**Architecture:** The Omnipool consolidates all liquidity into one pool. Every asset is internally paired with [[wiki/lrna\|lrna]] (the hub token) in a virtual sub-pool. Traders swap any asset for any other in a single transaction — internally it routes TKN1→LRNA→TKN2. Implemented as a Substrate pallet (`pallet-omnipool`) using `u128` balances.

**LRNA mechanics:** [[wiki/lrna\|lrna]] is minted when liquidity is added and burned when removed. It is never a user-held asset in the traditional sense. LRNA can only be sold into the pool (not bought). The only way to obtain it is as partial [[wiki/impermanent-loss\|IL]] compensation when withdrawing an LP position where the asset price has risen. A global `HubAssetImbalance` tracks the net deviation from ideal LRNA supply.

**LRNA imbalance recovery:** Three mechanisms counteract negative LRNA imbalance: protocol fee burning (burns collected fees until 2× the sold LRNA is recovered), dynamic protocol fees (higher volatility → higher fees → faster burn), and [[wiki/protocol-owned-liquidity\|POL]] as liquidity of last resort.

**AMM model:** Each virtual sub-pool uses a constant product invariant (`Q_i * T_i = k`). Asset price = `Q_i / T_i`. Asset weight = ratio of sub-pool LRNA to total LRNA.

**Swap math:** Two fees per trade — asset fee `fA` (in output token, stays in pool for LPs) and protocol fee `fP` (in LRNA, for HDX buybacks). The sell formula: `ΔQ₁ = Q₁ * (-ΔT₁ / T₁⁺)`, `ΔQ₂ = -ΔQ₁ * (1 - fP)`, `ΔT₂ = T₂ * (-ΔQ₂ / Q₂⁺) * (1 - fA)`.

**LP positions:** Single-sided deposits. Positions represented as NFTs storing shares, entry price (`p₀`), and asset ID. On withdrawal: if price fell, the protocol claims a share of the LP's position (becomes POL); if price rose, the LP receives TKN + LRNA compensation, and excess LRNA is burned.

**Impermanent loss:** IL formula: `IL = 2*sqrt(p * p₀) / (p₀ + p) - 1`. Structurally identical to classic two-asset CFMM IL but sensitive only to TKN/LRNA divergence. Because LRNA is a weighted index of the whole pool, market-correlated assets experience lower IL than a TKN/stablecoin XYK pool. An asset's weight in the pool affects its IL exposure — larger weight → lower IL.

**Fee structure:** Dynamic trading fees adjust based on volatility. Dynamic withdrawal fee (0.01%–1%) based on spot vs. oracle price deviation, to deter manipulation. Transaction fees payable in any Omnipool asset.

**Risk controls:** Weight caps per asset (bounds maximum exposure), [[wiki/circuit-breaker\|circuit-breaker]] (50% per-block trade volume limit), [[wiki/price-barrier\|price-barrier]] (1% spot/oracle deviation threshold for liquidity ops), dynamic withdrawal fee, [[wiki/ema-oracle\|EMA oracle]], and targeted function pausing by the Technical Committee.

**On-chain data model:** `AssetState` stores hub_reserve, shares, protocol_shares, cap, and tradability bitflags (SELL/BUY/ADD_LIQUIDITY/REMOVE_LIQUIDITY). `safe_withdrawal` state inferred when both SELL and BUY are disabled.

**Pallet extrinsics:** `add_token`, `add_liquidity`, `remove_liquidity`, `sell`, `buy`, `set_asset_tradable_state`, `set_asset_weight_cap`, `refund_refused_asset`, `sacrifice_position`.

**Math module:** Separated into `hydra-dx-math` crate. Core functions: `calculate_sell_state_changes`, `calculate_buy_state_changes`, `calculate_add_liquidity_state_changes`, `calculate_remove_liquidity_state_changes`, `calculate_withdrawal_fee`, `calculate_cap_difference`. All use `FixedU128` and checked arithmetic.

**Governance:** Listings require OpenGov referendum. Initial liquidity pre-transferred, then `add_token` called if approved. `refund_refused_asset` returns funds if rejected.

**Audits:** Runtime Verification (Rust, Sept 2022), BlockScience (math/economics, March 2022), Code4rena competitive audit (full node, Feb 2024). Active Immunefi bug bounty.

## Entities Mentioned

- [[omnipool\|omnipool]] — the AMM itself
- [[wiki/lrna\|lrna]] — hub token
- [[wiki/hydration\|hydration]] — the protocol
- [[wiki/hdx\|hdx]] — governance token (fee buybacks)
- [[wiki/runtime-verification\|runtime-verification]], [[wiki/blockscience\|blockscience]], [[wiki/code4rena\|code4rena]] — auditors
- [[wiki/immunefi\|immunefi]] — bug bounty platform

## Concepts Mentioned

- [[wiki/impermanent-loss\|impermanent-loss]] — IL mechanics and formula
- [[wiki/circuit-breaker\|circuit-breaker]] — per-block volume limits
- [[wiki/price-barrier\|price-barrier]] — spot/oracle deviation check
- [[wiki/ema-oracle\|ema-oracle]] — exponential moving average oracle
- [[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]] — POL and its role
- [[wiki/dynamic-fees\|dynamic-fees]] — volatility-adjusted trading and withdrawal fees
- [[wiki/nft-lp-positions\|nft-lp-positions]] — NFT-based LP accounting
- [[wiki/tradability-flags\|tradability-flags]] — per-asset operation permissions
