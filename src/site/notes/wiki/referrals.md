---
{"dg-publish":true,"permalink":"/wiki/referrals/","title":"Referrals","tags":["growth","incentives","fees"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Referrals","tags":["growth","incentives","fees"],"source_count":1,"last_updated":"2026-08-15"}}
---


# Referrals

**TL;DR:** [[wiki/hydration\|hydration]]'s referral system pays a share of trade fees back to referrers and to the traders they referred. Since Aug 2026 it is **one of four fee receivers of [[wiki/pallet-fee-processor\|pallet-fee-processor]]**, taking a fixed **5% slice** of every Omnipool asset fee — trade pallets no longer call referrals directly.

## Fee path (Aug 2026)

`OmnipoolHookAdapter::on_trade_fee` → `pallet_fee_processor::process_trade_fee` → `ReferralsFeeReceiver` (`runtime/hydradx/src/assets.rs`).

- Referrals is the only **raw-asset** receiver (`accepts_raw_asset() == true`) — it takes its slice in the traded asset and self-converts to HDX later (`PendingConversions`, `on_idle`).
- `Pallet::process_trade_fee(trader, asset_id, amount)` now returns the **used** `Balance`; the processor transfers exactly that into the referrals pot and leaves the remainder with the fee source. An unlinked trader with no reward mints nothing and consumes nothing.
- `FeeDistribution.referrer` / `.trader` are percentages **of the referrals slice**, not of the whole fee, and sum to 100% per tier (Tier0 60/40 → Tier4 weighted further toward the referrer; `Level::None` = 0/0).
- **Removed:** `FeeDistribution.external` and `Config::ExternalAccount` (the old staking cut — staking is now its own `FeeReceiver`). `pallet_referrals::traits::Convert` was deleted in favour of `hydradx_traits::fee_processor::Convert`.
- Failed conversions no longer abort `claim_rewards` / `on_idle`; they emit `ConversionFailed` and skip.

Runtime split of every Omnipool asset fee: gigahdx 15% + gigahdx-rewards 25% + staking 5% + **referrals 5%**.

See [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/pallet-fee-processor\|pallet-fee-processor]], [[wiki/gigahdx\|gigahdx]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
