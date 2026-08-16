---
{"dg-publish":true,"permalink":"/wiki/pallet-referrals/","title":"pallet-referrals","tags":["referrals","rewards","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-referrals","repo":"hydration-node","paths":["pallets/referrals/src/lib.rs","pallets/referrals/src/traits.rs","pallets/referrals/src/migration.rs","pallets/referrals/src/weights.rs"],"symbols":["Pallet","Config","Level","FeeDistribution","register_code","link_code","convert","claim_rewards","set_reward_percentage","process_trade_fee","pot_account_id","ReferralCodes","ReferralAccounts","LinkedAccounts","ReferrerShares","TraderShares","TotalShares","Referrer","AssetRewards","PendingConversions"],"traits_impl":[],"depends_on":["pallet-broadcast","pallet-fee-processor"],"runtime_index":75,"tags":["referrals","rewards","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-referrals

**TL;DR:** On-chain referral program. Users register a code, others link via `link_code`; the referrals slice of the trade fee is minted into referrer / trader shares. Tiered by accumulated referred volume. Runtime index = 75.

## Role

Implements [[wiki/referrals\|referrals]]. Since the fee refactor it is a **raw-asset `FeeReceiver`** of [[wiki/pallet-fee-processor\|pallet-fee-processor]] (5% slice, `accepts_raw = true`): the processor offers a slice in the original trade-fee asset, `process_trade_fee` reports how much of it it actually wants, and the processor transfers exactly that into the referrals pot. No trade pallet calls this pallet directly any more.

## Config trait (excerpt)

```rust
// pallets/referrals/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Currency: GetByKey<Self::AssetId, Balance> + MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId>;
    /// `hydradx_traits::fee_processor::Convert` — the local `traits::Convert` was
    /// removed and hoisted into hydradx-traits so pallet-fee-processor shares it.
    type Convert: Convert<Self::AccountId, Self::AssetId, Balance, Error = DispatchError>;
    type LevelVolumeAndRewardPercentages: GetByKey<Level, (Balance, FeeDistribution)>;
    #[pallet::constant] type RewardAsset: Get<Self::AssetId>;
    type RegistrationFee: Get<(Self::AssetId, Balance, Self::AccountId)>;
    #[pallet::constant] type CodeLength: Get<u32>;
    #[pallet::constant] type TierVolume: Get<Balance>;
    #[pallet::constant] type MinTradingAmount: Get<Balance>;
    #[pallet::constant] type SeedNativePrice: Get<(Balance, Balance)>;
    type PriceProvider: SpotPriceProvider<Self::AssetId, Price = FixedU128>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `ReferralCodes` | StorageMap | `ReferralCode → AccountId` (owner) |
| `ReferralAccounts` | StorageMap | `AccountId → ReferralCode` |
| `LinkedAccounts` | StorageMap | `AccountId (trader) → AccountId (referrer)` |
| `ReferrerShares` | StorageMap | `AccountId → Balance` (unclaimed shares) |
| `TraderShares` | StorageMap | `AccountId → Balance` |
| `TotalShares` | StorageValue | `Balance` |
| `Referrer` | StorageMap | `AccountId → (Level, Balance)` — level + volume accumulator toward next level (getter `referrer_level`) |
| `AssetRewards` | StorageDoubleMap | `(AssetId, Level) → FeeDistribution` (per-asset override) |
| `PendingConversions` | CountedStorageMap | `AssetId → ()` — pot assets awaiting conversion to `RewardAsset` |

## Events

`CodeRegistered`, `CodeLinked`, `Converted`, `ConversionFailed { asset_id, reason }`, `Claimed`, `LevelUp`.

## Errors

`TooLong`, `TooShort`, `InvalidCharacter`, `AlreadyExists`, `InvalidCode`, `AlreadyLinked`, `ZeroAmount`, `LinkNotAllowed`, `IncorrectRewardCalculation`, `IncorrectRewardPercentage`, `AlreadyRegistered`, `PriceNotFound`, `ConversionMinTradingAmountNotReached`, `ConversionZeroAmountReceived`.

## Extrinsics

| Name | Description |
|------|-------------|
| `register_code` | Reserve a referral code (pays `RegistrationFee`) |
| `link_code` | Link account to a referrer's code (one-time, immutable) |
| `convert` | Permissionless: convert one pending pot asset to `RewardAsset` |
| `claim_rewards` | Referrer or trader claims accumulated rewards; first drains all `PendingConversions` |
| `set_reward_percentage` | Governance sets per-(asset, level) reward split |

## Hooks

`on_idle` — converts up to a weight-bounded number of `PendingConversions` entries to `RewardAsset`. Best effort: a failed conversion emits `ConversionFailed` and the entry is removed regardless.

## Integration

- **Traits implemented:** none. `process_trade_fee` is a plain associated function invoked by the runtime's `ReferralsFeeReceiver::on_raw_fee_received`.
- **Traits consumed:** `hydradx_traits::fee_processor::Convert`, `PriceProvider`, `Currency`, `GetByKey`
- **Pallets depended on:** [[wiki/pallet-fee-processor\|pallet-fee-processor]] (fee source), [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-omnipool\|pallet-omnipool]] (conversion venue via `Convert`)

## Key logic: `process_trade_fee`

```rust
// pallets/referrals/src/lib.rs
/// Mint referral shares for an offered trade-fee slice and report how much of it is used.
#[transactional]
pub fn process_trade_fee(
    trader: T::AccountId,
    asset_id: T::AssetId,
    amount: Balance,          // the slice offered by pallet-fee-processor
) -> Result<Balance, DispatchError> {
    let Some(price) = T::PriceProvider::get_price(T::RewardAsset::get(), asset_id.clone()) else {
        return Ok(0);         // no price → consume nothing
    };
    let (level, ref_account) = /* Level::None + None when the trader is unlinked */;
    let rewards = Self::asset_rewards(asset_id.clone(), level)
        .unwrap_or_else(|| T::LevelVolumeAndRewardPercentages::get(&level).1);
    let referrer_reward = if ref_account.is_some() { rewards.referrer.mul_floor(amount) } else { 0 };
    let trader_reward = rewards.trader.mul_floor(amount);
    let used = referrer_reward.saturating_add(trader_reward);
    // ... mint shares valued in RewardAsset at `price`, bump TotalShares
    if used > 0 && asset_id != T::RewardAsset::get() {
        PendingConversions::<T>::insert(asset_id, ());
    }
    Ok(used)                  // fee-processor transfers exactly `used` into the pot
}
```

## Key concept: Level tiers

```rust
// pallets/referrals/src/lib.rs
pub enum Level { None, Tier0, Tier1, Tier2, Tier3, Tier4 }
// Volume accumulates in `Referrer[account].1`; `Level::increase` promotes on `TierVolume`.
// Higher levels = larger referrer/trader share of the referrals slice.
```

## Gotchas

- Once linked, a trader's referrer is **immutable** — no re-linking.
- `FeeDistribution` lost its `external` field, and `Config::ExternalAccount` is gone. The "external" cut (historically staking) is now a first-class receiver of [[wiki/pallet-fee-processor\|pallet-fee-processor]] instead. Percentages in `FeeDistribution` are shares **of the referrals slice**, not of the whole trade fee.
- `process_trade_fee` **no longer transfers anything and no longer takes a `source`**. It only mints shares and returns the consumed amount; the caller moves the funds. An unlinked trade with a zero trader-rebate tier consumes 0 and the slice stays with the fee source (not socialized).
- The pot-balance-delta tolerance check (`actual_taken.abs_diff(total_taken) <= 1`, an AToken rounding workaround) was removed together with the transfer.
- Conversion is best effort everywhere: both `claim_rewards` and `on_idle` skip an un-convertible asset and emit `ConversionFailed` instead of reverting. Funds stay in the pot and re-queue on the next fee.
- The local `traits::Convert` was deleted; the shared definition lives in `traits/src/fee_processor.rs`.
- Rewards accumulate as **shares**, redeemed against the pot's `RewardAsset` ([[wiki/hdx\|hdx]]) balance at claim time — not as a fixed HDX amount.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/referrals\|referrals]]
