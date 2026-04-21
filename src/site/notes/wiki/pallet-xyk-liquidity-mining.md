---
{"dg-publish":true,"permalink":"/wiki/pallet-xyk-liquidity-mining/","title":"pallet-xyk-liquidity-mining","tags":["liquidity-mining","xyk","farming","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-xyk-liquidity-mining","repo":"hydration-node","paths":["pallets/xyk-liquidity-mining/src/lib.rs"],"symbols":["Pallet","Config","create_global_farm","create_yield_farm","deposit_shares","redeposit_shares","claim_rewards","withdraw_shares","join_farms","add_liquidity_and_join_farms","exit_farms","XYKLiquidityMiningInstance"],"traits_impl":[],"depends_on":["pallet-xyk","pallet-liquidity-mining","pallet-nft"],"runtime_index":95,"tags":["liquidity-mining","xyk","farming","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-xyk-liquidity-mining

**TL;DR:** LM wrapper for [[wiki/pallet-xyk\|pallet-xyk]] pools. Users stake their XYK LP share tokens into a yield farm and receive periodic rewards. Delegates accounting to the shared [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] warehouse (Instance2). Runtime index = 95 (warehouse instance runtime index = 96).

## Role

Binds generic LM primitives to XYK pools. Tracks XYK pool TVL through [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] and distributes rewards to LPs holding XYK share tokens. Parallel twin of [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]] for the isolated-pool AMM.

## Config trait (excerpt)

```rust
// pallets/xyk-liquidity-mining/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: ...;
    type Currencies: MultiCurrency<Self::AccountId, CurrencyId = AssetId, Balance = Balance>;
    type AMM: AMM<Self::AccountId, AssetId, AssetPair, Balance>
        + AMMAddLiquidity<Self::AccountId, AssetId, Balance>;
    type CreateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type PalletId: Get<PalletId>;
    type NFTCollectionId: Get<CollectionId>;
    type NFTHandler: Mutate<Self::AccountId>;
    type LiquidityMiningHandler: LiquidityMiningMutate<...>;
    type OracleSource: Get<Source>;
    type OraclePeriod: Get<OraclePeriod>;
    type LiquidityOracle: AggregatedOracle<AssetId, Balance, BlockNumberFor<Self>, EmaPrice>;
    type NonDustableWhitelistHandler: DustRemovalAccountWhitelist<Self::AccountId, Error = DispatchError>;
    type AssetRegistry: RegistryInspect<AssetId = AssetId>;
    type MaxFarmEntriesPerDeposit: Get<u32>;
    type WeightInfo: WeightInfo;
}
```

## Storage

None directly — storage lives in the [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] warehouse instance (`Instance2`).

## Events

`GlobalFarmCreated`, `GlobalFarmUpdated`, `YieldFarmCreated`, `YieldFarmUpdated`, `YieldFarmStopped`, `YieldFarmResumed`, `YieldFarmTerminated`, `GlobalFarmTerminated`, `SharesDeposited`, `SharesRedeposited`, `RewardClaimed`, `SharesWithdrawn`, `DepositDestroyed`.

## Errors

Delegated to warehouse + pair-specific: `XykPoolDoesntExist`, `AssetNotRegistered`, `InsufficientXYKSharesBalance`.

## Extrinsics

| Name | Description |
|------|-------------|
| `create_global_farm` | Create reward pot |
| `update_global_farm` | Update distribution parameters |
| `terminate_global_farm` | Close global farm |
| `create_yield_farm` | Bind farm to a specific XYK asset pair |
| `update_yield_farm` | Update multiplier |
| `stop_yield_farm` / `resume_yield_farm` | Pause/resume distribution |
| `terminate_yield_farm` | Close yield farm |
| `deposit_shares` | Lock XYK LP shares into farm (mints deposit NFT) |
| `redeposit_shares` | Reuse deposit across farms |
| `claim_rewards` | Claim accrued rewards |
| `withdraw_shares` | Unstake shares + claim remaining rewards |
| `join_farms` | Convenience: deposit + join multiple farms |
| `add_liquidity_and_join_farms` | Convenience: add XYK liquidity + deposit + join farms in one call |
| `exit_farms` | Convenience: exit multiple farms + burn deposit NFT |

## Hooks

None directly.

## Integration

- **Traits implemented:** none exposed; farm state consumed by the warehouse
- **Traits consumed:** `AMM`, `AMMAddLiquidity`, `NFTHandler`, `MultiCurrency`, `AggregatedOracle`, `LiquidityMiningMutate`
- **Pallets depended on:** [[wiki/pallet-xyk\|pallet-xyk]], [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] (instance 2), [[wiki/pallet-nft\|pallet-nft]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/pallet-asset-registry\|pallet-asset-registry]]

## Runtime wiring

```rust
// runtime/hydradx/src/assets.rs
pub type XYKLiquidityMiningInstance = warehouse_liquidity_mining::Instance2;
pub const XYKWarehouseLMPalletId: PalletId = PalletId(*b"xykLMpID");

// runtime/hydradx/src/lib.rs
XYKLiquidityMining: pallet_xyk_liquidity_mining = 95,
XYKWarehouseLM: warehouse_liquidity_mining::<Instance2> = 96,
```

## Gotchas

- Unlike Omnipool LM (which locks an LP position NFT), this pallet locks **XYK share tokens** (fungible) — balance-based rather than NFT-based.
- Each deposit still produces a **deposit NFT** from the `NFTCollectionId` collection, keyed by `DepositId`, tracking which farms that deposit is enrolled in.
- `MaxFarmEntriesPerDeposit` bounds the number of yield farms a single deposit can be enrolled in simultaneously.
- `add_liquidity_and_join_farms` is the common user path — one tx adds XYK liquidity, mints shares, deposits them, and joins farms.
- Position value for reward weighting comes from [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] pool price × shares, not instantaneous spot — prevents flash-loan reward manipulation.
- Warehouse uses `Instance2` to separate storage from the Omnipool LM warehouse (`Instance1`).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
