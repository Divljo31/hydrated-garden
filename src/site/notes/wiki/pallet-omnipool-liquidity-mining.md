---
{"dg-publish":true,"permalink":"/wiki/pallet-omnipool-liquidity-mining/","title":"pallet-omnipool-liquidity-mining","tags":["liquidity-mining","omnipool","farming","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-omnipool-liquidity-mining","repo":"hydration-node","paths":["pallets/omnipool-liquidity-mining/src/lib.rs"],"symbols":["Pallet","Config","create_global_farm","create_yield_farm","deposit_shares","redeposit_shares","claim_rewards","withdraw_shares","OmnipoolLmInstance"],"traits_impl":[],"depends_on":["pallet-omnipool","pallet-liquidity-mining","pallet-nft"],"runtime_index":63,"tags":["liquidity-mining","omnipool","farming","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-omnipool-liquidity-mining

**TL;DR:** LM wrapper for [[wiki/pallet-omnipool\|pallet-omnipool]] positions. Users lock their NFT LP position into a yield farm and receive periodic rewards. Delegates accounting to the shared [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] warehouse. Runtime index = 63 (warehouse instance runtime index = 62).

## Role

Binds generic LM primitives to the Omnipool-specific position NFT (from [[wiki/pallet-omnipool\|pallet-omnipool]]). Tracks Omnipool position value through [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] and distributes rewards to stakers.

## Config trait (excerpt)

```rust
// pallets/omnipool-liquidity-mining/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: ...;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type CreateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type Currency: MultiCurrencyExtended<Self::AccountId, CurrencyId = AssetId>;
    type Omnipool: AssetInspect<Self::AccountId, AssetId = AssetId>;
    type OmnipoolHooks: OmnipoolHooks<...>;
    type AssetRegistry: Inspect<AssetId = AssetId>;
    type OracleSource: Get<Source>;
    type OraclePeriod: Get<OraclePeriod>;
    type NFTCollectionId: Get<CollectionId>;
    type NFTHandler: Mutate<Self::AccountId, ItemId = DepositId, CollectionId = CollectionId>;
    type LMPalletId: Get<PalletId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

None directly — storage lives in the [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] warehouse instance.

## Events

`GlobalFarmCreated`, `GlobalFarmUpdated`, `YieldFarmCreated`, `YieldFarmUpdated`, `YieldFarmStopped`, `YieldFarmResumed`, `YieldFarmTerminated`, `GlobalFarmTerminated`, `SharesDeposited`, `SharesRedeposited`, `RewardClaimed`, `SharesWithdrawn`, `DepositDestroyed`.

## Errors

Delegated to warehouse + position-specific: `OmnipoolPositionNotFound`, `InvalidOmnipoolPosition`, `AssetNotRegistered`.

## Extrinsics

| Name | Description |
|------|-------------|
| `create_global_farm` | Create reward pot |
| `update_global_farm` | Update distribution parameters |
| `terminate_global_farm` | Close global farm |
| `create_yield_farm` | Bind farm to a specific asset in the omnipool |
| `update_yield_farm` | Update multiplier |
| `stop_yield_farm` / `resume_yield_farm` | Pause/resume distribution |
| `terminate_yield_farm` | Close yield farm |
| `deposit_shares` | Lock omnipool NFT into farm (mints deposit NFT) |
| `redeposit_shares` | Reuse deposit across farms |
| `claim_rewards` | Claim accrued rewards |
| `withdraw_shares` | Unstake NFT + claim remaining rewards |
| `join_farms` | Convenience: deposit + join multiple farms in one call |
| `exit_farms` | Convenience: exit multiple farms + burn deposit NFT |

## Hooks

None directly (omnipool emits events this pallet consumes via `OmnipoolHooks`).

## Integration

- **Traits implemented:** `OmnipoolHooks` — notified on Omnipool liquidity changes
- **Traits consumed:** `AssetInspect`, `NFTHandler`, `MultiCurrencyExtended`
- **Pallets depended on:** [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] (instance 1), [[wiki/pallet-nft\|pallet-nft]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]]

## Gotchas

- Locking an omnipool NFT prevents withdrawal from [[wiki/pallet-omnipool\|pallet-omnipool]] until unstaked.
- Each deposit produces a new "deposit NFT" (different collection from the omnipool LP NFT).
- Position value for reward weighting is derived from [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] snapshot price × position.shares, not current spot.
- `join_farms` / `exit_farms` are user-friendly wrappers over primitive deposit/withdraw — prefer them in tooling.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
