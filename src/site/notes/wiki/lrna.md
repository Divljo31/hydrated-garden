---
{"dg-publish":true,"permalink":"/wiki/lrna/","title":"LRNA (H2O)","tags":["token","hub-token","omnipool","amm"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"LRNA (H2O)","tags":["token","hub-token","omnipool","amm"],"source_count":2,"last_updated":"2026-04-13"}}
---


# LRNA (H2O)

**TL;DR:** LRNA (H2O in the math spec) is the internal hub token of the [[omnipool\|omnipool]], enabling single-pool trading between any asset pair. It is minted on liquidity provision, burned on removal, and serves as a weighted price index of the entire pool.

## How It Works

Every asset in the [[omnipool\|omnipool]] is internally paired with LRNA in a virtual TKN/LRNA sub-pool. When a user adds liquidity of asset TKN, a corresponding amount of LRNA is **minted** against it. When liquidity is removed, the corresponding LRNA is **burned**. All trades route through LRNA internally: selling TKN1 → buying TKN2 executes as TKN1→LRNA→TKN2.

LRNA never exists as a user-held asset in the traditional sense. It functions as a **weighted price index** of the entire Omnipool basket (stablecoins + volatile assets), since it has a liquidity pair with every asset in the pool.

## Tradability

LRNA has restricted tradability — it can only be **sold** into the pool (not bought directly). The only way to obtain LRNA is to receive it as partial [[wiki/impermanent-loss\|IL]] compensation when withdrawing an LP position where the asset price has risen. This is enforced at the storage level via `DefaultHubAssetTradability` (SELL only).

## Imbalance Tracking

The protocol tracks a global `HubAssetImbalance` — the net deviation between LRNA minted on liquidity provision and LRNA burned or collected as protocol fees. This is a key input to the IL mitigation mechanism.

When LPs sell received LRNA back into the pool, it creates a negative imbalance. Three mechanisms counteract this:

1. **Protocol fee burning** — collected LRNA fees are burned until 2× the sold LRNA is recovered
2. **[[wiki/dynamic-fees\|dynamic-fees]]** — higher volatility → higher protocol fees → faster burn rate
3. **[[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]]** — acts as liquidity of last resort, setting a floor on LRNA's value

## Role in Impermanent Loss

Because LRNA is a weighted index of the whole pool, [[wiki/impermanent-loss\|impermanent-loss]] for any asset depends on its divergence from LRNA (not from a stablecoin or specific pair). Assets with larger weight in the pool influence LRNA more, resulting in lower IL. See [[wiki/impermanent-loss\|impermanent-loss]] for the full model.

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
