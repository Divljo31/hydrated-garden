---
{"dg-publish":true,"permalink":"/wiki/impermanent-loss/","title":"Impermanent Loss","tags":["amm","risk","liquidity","economics"],"dg-note-properties":{"type":"concept","title":"Impermanent Loss","tags":["amm","risk","liquidity","economics"],"source_count":1,"last_updated":"2026-04-13"}}
---


# Impermanent Loss

**TL;DR:** Impermanent loss (IL) is the opportunity cost an LP faces when their deposited asset price diverges from entry price. In the [[omnipool\|omnipool]], IL depends on the asset's divergence from [[wiki/lrna\|lrna]] (the hub token), not from specific pairs, with lower IL for larger-weight assets.

## Formula

For a single-asset LP in the [[omnipool\|omnipool]]:

```
IL = 2 * sqrt(p * p₀) / (p₀ + p) - 1
```

Where `p` is the current price and `p₀` is the price at LP entry. This is structurally identical to the classic two-asset CFMM IL formula, but with sensitivity only to the TKN/[[wiki/lrna\|lrna]] price divergence — not to prices of other assets in the Omnipool (except indirectly via LRNA's aggregate value).

## Omnipool IL vs. Traditional XYK

Because [[wiki/lrna\|lrna]] is a weighted price index of the entire Omnipool basket (stablecoins + volatile assets), the IL profile depends on how correlated the deposited asset is with the broader market:

- **Lower IL than TKN/stablecoin XYK:** A market-correlated asset (e.g., DOT, BTC) moves in the same direction as LRNA, reducing the divergence
- **Higher IL than TKN/DOT XYK:** In an isolated pair where both assets move together, there's minimal divergence. The Omnipool sits between these extremes

## Weight Effect

An asset's weight in the [[omnipool\|omnipool]] (its share of total LRNA reserve) directly affects IL exposure:
- **Larger weight** → asset has stronger influence on LRNA's price → LRNA moves more in line with the asset → **lower IL**
- **Smaller weight** → asset barely influences LRNA → IL profile approaches a standard XYK pair

Reference point: at 1% TKN weight, a 35% price decrease results in approximately 2% IL.

## IL Mitigation on Withdrawal

When an LP withdraws from the [[omnipool\|omnipool]], what they receive depends on price movement:

**Price fell (p < p₀):** LP receives only TKN. The protocol claims a portion of the LP's shares, converting them to [[wiki/protocol-owned-liquidity\|POL]]:
```
ΔB = max(-(p₀ - p)/(p + p₀) * Δs, 0)
```

**Price rose (p > p₀):** LP receives TKN + [[wiki/lrna\|lrna]] as compensation. Excess LRNA (not distributed to the LP) is burned by the protocol.

## Worked Examples

### Price Rose — LRNA Compensation
Bob provides 1,000 DAI at `p₀ = 1`, receiving 500 shares. Total pool: 10,000 shares, 19,000 DAI. He withdraws when `p = 1.2`. Since price rose, Bob receives TKN + LRNA compensation. The protocol burns the excess LRNA not distributed to Bob.

### Price Fell — Protocol Claims Shares
Alice provides 100 DOT at `p₀ = 10`, receiving 200 shares. Total pool: 1,000 shares, 710 DOT. She withdraws when `p = 5`. Protocol claims `≈66.667` shares as POL. Alice retains `≈133.333` shares, receiving `≈94.67` DOT (entered with 100 DOT). The loss could be offset by fee accumulation in the pool.

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
