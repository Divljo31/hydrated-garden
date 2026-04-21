---
{"dg-publish":true,"permalink":"/wiki/dynamic-fees/","title":"Dynamic Fees","tags":["fees","economics","security","omnipool"],"dg-note-properties":{"type":"concept","title":"Dynamic Fees","tags":["fees","economics","security","omnipool"],"source_count":2,"last_updated":"2026-04-13"}}
---


# Dynamic Fees

**TL;DR:** The [[omnipool\|omnipool]] uses dynamic fee adjustment for both trading and withdrawals, making price manipulation less profitable.

## Trading Fees

Two components per trade:
- **Asset fee (`fA`)** — charged in the output asset, stays in the pool as LP rewards
- **Protocol fee (`fP`)** — charged in [[wiki/lrna\|lrna]], used for [[wiki/hdx\|hdx]] buybacks distributed to stakers

Both adjust based on current market volatility: higher volatility → higher fees. This accelerates [[wiki/lrna\|lrna]] imbalance recovery during volatile periods.

## Dynamic Withdrawal Fee

Applied when removing liquidity from the [[omnipool\|omnipool]]:
- **Range:** 0.01% to 1% of the withdrawn amount
- **Logic:** Fee equals the percentage difference between spot price and [[wiki/ema-oracle\|ema-oracle]] price
- **Purpose:** Disincentivizes withdrawing immediately after a price manipulation event
- **Safe withdrawal exception:** If an asset's trading is fully disabled (`safe_withdrawal == true` — both SELL and BUY disabled via [[wiki/tradability-flags\|tradability-flags]]), the dynamic fee is waived

The minimum fee floor is set via `MinWithdrawalFee` (0.01%).

## Transaction Fees

All [[wiki/hydration\|hydration]] network transactions (including [[omnipool\|omnipool]] trades) can be paid in **any Omnipool asset** — not just [[wiki/hdx\|hdx]]. This is handled by `pallet-transaction-multi-payment`.

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
