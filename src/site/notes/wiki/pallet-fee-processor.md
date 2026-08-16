---
{"dg-publish":true,"permalink":"/wiki/pallet-fee-processor/","title":"pallet-fee-processor","tags":["fees","distribution","omnipool","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-fee-processor","repo":"hydration-node","paths":["pallets/fee-processor/src/lib.rs","pallets/fee-processor/src/weights.rs","pallets/fee-processor/src/tests/","traits/src/fee_processor.rs","runtime/hydradx/src/assets.rs"],"symbols":["Pallet","Config","PendingConversions","HeldFees","convert","process_trade_fee","do_convert","distribute_proportionally","convert_percentage","pot_account_id","FeeReceiver","FeeDestination","Convert"],"traits_impl":[],"depends_on":["pallet-omnipool","pallet-staking","pallet-referrals","pallet-gigahdx","pallet-gigahdx-rewards"],"runtime_index":207,"tags":["fees","distribution","omnipool","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-fee-processor

**TL;DR:** Central splitter for [[wiki/pallet-omnipool\|pallet-omnipool]] trade (asset) fees. Takes each receiver's slice out of the fee account, converts non-HDX slices to [[wiki/hdx\|hdx]] in `on_idle` via an Omnipool sell, and pays the HDX-target receivers pro-rata. Replaces the old ad-hoc "referrals-then-staking" chain in the `OmnipoolHooks::on_trade_fee` adapter. Runtime index = 207.

## Role

Single choke point for trade-fee distribution. `runtime/adapters/src/lib.rs → OmnipoolHookAdapter::on_trade_fee` now calls exactly one thing:

```rust
// runtime/adapters/src/lib.rs
fn on_trade_fee(fee_account, trader, asset, amount) -> Result<Vec<Option<(Balance, AccountId)>>, _> {
    let trader = pallet_broadcast::Pallet::<Runtime>::get_swapper().unwrap_or(trader);
    if asset == Lrna::get() { return Ok(vec![]); }
    let result = pallet_fee_processor::Pallet::<Runtime>::process_trade_fee(
        fee_account, trader, asset.into(), amount,
    )?;
    Ok(vec![result])
}
```

Everything downstream ([[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]) is a `FeeReceiver` impl in `runtime/hydradx/src/assets.rs`.

## Config trait (excerpt)

```rust
// pallets/fee-processor/src/lib.rs
pub trait Config: frame_system::Config {
    type AssetId: Member + Parameter + Copy + MaybeSerializeDeserialize + MaxEncodedLen + Ord;
    type Currency: Mutate<Self::AccountId, AssetId = Self::AssetId, Balance = Balance>
        + Inspect<Self::AccountId, AssetId = Self::AssetId, Balance = Balance>;
    /// Swaps an asset to HDX (runtime: `ConvertViaOmnipool<Omnipool>`).
    type Convert: Convert<Self::AccountId, Self::AssetId, Balance, Error = DispatchError>;
    #[pallet::constant] type PalletId: Get<PalletId>;          // b"feeproc/"
    #[pallet::constant] type HdxAssetId: Get<Self::AssetId>;
    #[pallet::constant] type LrnaAssetId: Get<Self::AssetId>;  // LRNA fees are skipped
    #[pallet::constant] type MaxConversionsPerBlock: Get<u32>; // runtime: 5
    /// Receivers for the non-HDX fee path (tuple, 1..6 members).
    type FeeReceivers: FeeReceiver<Self::AccountId, Self::AssetId, Balance, Error = DispatchError>;
    /// Receivers for direct HDX fees — may differ (runtime swaps the staking receiver).
    type HdxFeeReceivers: FeeReceiver<Self::AccountId, Self::AssetId, Balance, Error = DispatchError>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `PendingConversions` | CountedStorageMap | `AssetId → ()` — assets sitting in the pot awaiting an HDX swap |
| `HeldFees` | StorageMap | `AccountId → Balance` — HDX earmarked in the pot for a `hold_until_ed` receiver still below ED |

## Events

`FeeReceived { asset, amount, trader }`, `Converted { asset_id, amount_in, hdx_out }`, `ConversionFailed { asset_id, reason }`.

## Errors

`AlreadyHdx`, `ConversionFailed`, `TransferFailed`, `PriceNotAvailable`, `Arithmetic`.

## Extrinsics

| Name | Description |
|------|-------------|
| `convert` | Permissionless manual trigger of `do_convert(asset_id)` — swaps the pot's balance of `asset_id` to HDX and distributes |

## Hooks

| Hook | Behaviour |
|------|-----------|
| `on_idle` | Drains up to `min(MaxConversionsPerBlock, remaining_weight / convert_weight)` entries from `PendingConversions`. Budgeted on **both** `ref_time` and `proof_size` — each conversion is a real Omnipool sell. A failed conversion removes the pending entry (no weight burned on retries) and emits `ConversionFailed`; the next fee in that asset re-inserts it. |
| `integrity_test` | Asserts `FeeReceivers::percentage() <= 100%` and `HdxFeeReceivers::percentage() <= 100%` |

## The `FeeReceiver` trait

Defined in `traits/src/fee_processor.rs` (hydradx-traits), auto-implemented for tuples of 1..6.

```rust
// traits/src/fee_processor.rs
pub struct FeeDestination<AccountId> {
    pub account: AccountId,
    pub percentage: Permill,
    /// Receiver takes its slice in the raw (unconverted) trade-fee asset.
    pub accepts_raw: bool,
    /// Buffer the HDX slice in the pot while `account` is below ED.
    pub hold_until_ed: bool,
}

pub trait FeeReceiver<AccountId, AssetId, Balance> {
    type Error;
    fn destination() -> AccountId;
    fn percentage() -> Permill;
    fn accepts_raw_asset() -> bool { false }
    fn hold_until_ed() -> bool { true }
    fn destinations() -> Vec<FeeDestination<AccountId>>;
    /// Raw receivers only: returns `(destination, amount_used)` — may consume LESS
    /// than the slice offered; the remainder stays with the fee source.
    fn on_raw_fee_received(trader, asset, amount) -> Result<Vec<(AccountId, Balance)>, Self::Error>;
}
```

Two receiver kinds:

| Kind | Payment | Flow |
|------|---------|------|
| **Raw-asset** (`accepts_raw = true`) | original trade-fee asset | `on_raw_fee_received` reports how much it wants; that exact amount is transferred source → destination. Unconsumed remainder is **not** socialized — it stays with the fee source. Only [[wiki/pallet-referrals\|pallet-referrals]] uses this. |
| **HDX-target** (default) | HDX | Combined slice goes into `pot_account_id()`. HDX path: distributed immediately. Non-HDX path: asset marked in `PendingConversions`, swapped to HDX in `on_idle`, then distributed pro-rata by relative `percentage`. |

## Runtime fee split (`runtime/hydradx/src/assets.rs`)

| Receiver | Share | Destination | Notes |
|---|---|---|---|
| `GigaHdxFeeReceiver` | 15% | `pallet_gigahdx::gigapot_account_id()` | plain HDX transfer lifts the [[wiki/gigahdx\|gigahdx]] exchange rate |
| `GigaHdxRewardsFeeReceiver` | 25% | `pallet_gigahdx_rewards::reward_accumulator_pot()` | drained into per-track allocations |
| `StakingFeeReceiver` / `HdxStakingFeeReceiver` | 5% | `pallet_staking::pot_account_id()` | separate types for non-HDX / HDX paths |
| `ReferralsFeeReceiver` | 5% | `pallet_referrals::pot_account_id()` | `accepts_raw = true`; calls `pallet_referrals::process_trade_fee` |

Total = **50% of every Omnipool asset fee leaves the pool**; the remaining 50% stays with LPs.

`type Convert = ConvertViaOmnipool<Omnipool>` — computes a spot-price-derived `min_expected` with 5% tolerance, then `Omnipool::sell`. `ZeroAmountOut` is mapped to `Error::ConversionFailed`.

## Integration

- **Called by:** `OmnipoolHookAdapter::on_trade_fee` (`runtime/adapters/src/lib.rs`) — the only production caller of `process_trade_fee`
- **Traits consumed:** `FeeReceiver`, `Convert` (both `hydradx_traits::fee_processor`), `fungibles::{Inspect, Mutate}`
- **Pallets depended on:** [[wiki/pallet-omnipool\|pallet-omnipool]] (conversion venue), [[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]], [[wiki/pallet-broadcast\|pallet-broadcast]] (swapper resolution upstream)

## Gotchas

- **LRNA fees are skipped entirely** — `process_trade_fee` logs a warning and returns `Ok(None)`. The adapter also short-circuits LRNA before calling in.
- The **HDX** and **non-HDX** receiver tuples are distinct `Config` items. The runtime uses this to swap `StakingFeeReceiver` ↔ `HdxStakingFeeReceiver`; other receivers are shared.
- `hold_until_ed` defaults to `true`. Without it, a trade would revert with `Token::BelowMinimum` when a receiver pot is uninitialized. The buffer flushes as soon as `balance + held + slice >= ED`; the HDX physically lives in the fee-processor pot the whole time, `HeldFees` only earmarks it.
- Pro-rata distribution uses `multiply_by_rational_with_rounding(.., Rounding::Down)` against the **sum of non-raw percentages** (`convert_percentage`), not against 100% — so receivers split the taken slice, not the whole fee.
- Conversion is **best effort**: `on_idle` never blocks a block, and a failure drops the `PendingConversions` entry rather than retrying, so the asset's balance simply waits in the pot for the next fee to re-queue it.
- `pot_account_id()` (`PalletId(b"feeproc/")`) is whitelisted in several runtime helpers (`assets.rs:706`, `assets.rs:2182`, `benchmarking/omnipool_liquidity_mining.rs`) — do not treat it as a user account.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[omnipool\|omnipool]]
