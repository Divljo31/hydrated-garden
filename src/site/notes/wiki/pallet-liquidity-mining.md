---
{"dg-publish":true,"permalink":"/wiki/pallet-liquidity-mining/","title":"pallet-liquidity-mining","tags":["liquidity-mining","farming","rewards","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-liquidity-mining","repo":"hydration-node","paths":["pallets/liquidity-mining/src/lib.rs","pallets/liquidity-mining/src/types.rs"],"symbols":["Pallet","Config","GlobalFarm","YieldFarm","Deposit","FarmMultiplier","LoyaltyCurve","create_global_farm","update_global_farm","create_yield_farm","deposit_lp_shares","redeposit_lp_shares","claim_rewards","withdraw_lp_shares"],"traits_impl":[],"depends_on":[],"runtime_index":null,"tags":["liquidity-mining","farming","rewards","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-liquidity-mining

**TL;DR:** Warehouse pallet: generic two-tier farm primitives (GlobalFarm → YieldFarm → Deposit). Not directly exposed; consumed by [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]] and [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]] as a shared library. No runtime index (not a construct_runtime member).

## Role

Implements the core farm accounting: reward-per-share (RPS), loyalty curves, farm multipliers, period rollovers. Called by the two wrapper pallets that bind it to a specific AMM venue.

## Config trait (excerpt)

```rust
// pallets/liquidity-mining/src/lib.rs
pub trait Config<I: 'static = ()>: frame_system::Config {
    type RuntimeEvent: ...;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type MultiCurrency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    type PalletId: Get<PalletId>;
    type TreasuryAccountId: Get<Self::AccountId>;
    type CreateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type MinTotalFarmRewards: Get<Balance>;
    type MinPlannedYieldingPeriods: Get<BlockNumberFor<Self>>;
    type BlockNumberProvider: BlockNumberProvider<BlockNumber = BlockNumberFor<Self>>;
    type PriceAdjustment: PriceAdjustment<GlobalFarmData<Self, I>, PriceAdjustment = FixedU128, Error = DispatchError>;
    type WeightInfo: WeightInfo;
}
```

## Storage (per instance)

| Name | Kind | Key → Value |
|------|------|-------------|
| `GlobalFarm` | StorageMap | `GlobalFarmId → GlobalFarmData` |
| `YieldFarm` | StorageDoubleMap | `(AmmPoolId, YieldFarmId) → YieldFarmData` |
| `Deposit` | StorageMap | `DepositId → DepositData` |
| `ActiveYieldFarm` | StorageDoubleMap | `(AmmPoolId, GlobalFarmId) → YieldFarmId` |
| `FarmSequencer` | StorageValue | `FarmId` |
| `DepositSequencer` | StorageValue | `DepositId` |

## Events (per instance)

`GlobalFarmCreated`, `GlobalFarmUpdated`, `YieldFarmCreated`, `YieldFarmUpdated`, `YieldFarmStopped`, `YieldFarmResumed`, `YieldFarmDestroyed`, `Deposited`, `RewardClaimed`, `SharesWithdrawn`, `AllRewardsDistributed`.

## Extrinsics

None exposed directly — this pallet provides callable *functions* (not dispatchables) consumed by wrapper pallets.

## Key functions (library API)

| Function | Purpose |
|---|---|
| `create_global_farm` | Create reward pot with loyalty curve + reward distribution schedule |
| `create_yield_farm` | Attach AMM pool to global farm with multiplier |
| `deposit_lp_shares` | Record a user's LP shares entering the farm |
| `redeposit_lp_shares` | Reuse existing deposit for another farm (same AMM position) |
| `claim_rewards` | Pay out accumulated rewards with loyalty multiplier |
| `withdraw_lp_shares` | Remove shares; returns remaining entitled rewards |

## Hooks

None.

## Integration

- **Consumed by:** [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]], [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]] — each instantiates this pallet with a different `I` generic.
- **Traits consumed:** `PriceAdjustment` (per-farm price oracle for weighted reward distribution).

## Gotchas

- `LoyaltyCurve` rewards long-term stakers more — new deposits get a fraction of pro-rata rewards; fraction grows with deposit age.
- `FarmMultiplier` lets governance boost/suppress specific pool rewards within a global farm.
- Updates are all lazy: RPS computed only when a user claim/deposit/withdraw triggers.
- `PriceAdjustment` feeds relative TVL pricing so multi-pool farms can weight yield correctly.
- `MinPlannedYieldingPeriods` enforces schedules last at least N periods; prevents griefing tiny farms.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
