---
{"dg-publish":true,"permalink":"/wiki/stableswap/","title":"StableSwap Pools","tags":["amm","stablecoin","trading"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"StableSwap Pools","tags":["amm","stablecoin","trading"],"source_count":1,"last_updated":"2026-08-15"}}
---


# StableSwap Pools

**TL;DR:** StableSwap pools are [[wiki/hydration\|hydration]] AMM pools optimized for stablecoin-to-stablecoin swaps with low slippage using a Curve-style algorithm. [[wiki/hollar\|hollar]] has four dedicated StableSwap pools seeded by the protocol.

## Virtual share issuance (storage v1 → v2, Aug 2026)

Share supply is now tracked by the pallet itself in `ShareIssuance: StorageMap<AssetId, Balance>`, mutated through `mint_shares` / `burn_shares` (`pallets/stableswap/src/lib.rs`). Migration: `pallets/stableswap/src/migrations/v2.rs` (`MigrateV1ToV2`).

**Anything deriving a pool's share supply from `Tokens.TotalIssuance(shareAssetId)` is now wrong.** All `StableMath.*` inputs must come from `ShareIssuance`. New errors: `InsufficientShareIssuance`, `UnaccountedShareIssuance`.

> Known hazard: [[wiki/sdk-next\|sdk-next]]'s `StableSwapClient` still reads `Tokens.TotalIssuance` — unfixed at sdk HEAD `c57d172`.

See [[wiki/pallet-stableswap\|pallet-stableswap]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
