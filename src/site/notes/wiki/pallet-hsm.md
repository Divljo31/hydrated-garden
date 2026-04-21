---
{"dg-publish":true,"permalink":"/wiki/pallet-hsm/","title":"pallet-hsm","tags":["stablecoin","hsm","hollar","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-hsm","repo":"hydration-node","paths":["pallets/hsm/src/lib.rs","pallets/hsm/src/types.rs","pallets/hsm/src/trade_execution.rs","pallets/hsm/src/provider.rs"],"symbols":["Pallet","Config","CollateralInfo","Collaterals","sell","buy","add_collateral_asset","update_collateral_asset","remove_collateral_asset","set_flash_minter","execute_arbitrage","MMOracle"],"traits_impl":["TradeExecution","MMOracle"],"depends_on":["pallet-stableswap","pallet-broadcast","pallet-evm"],"runtime_index":82,"tags":["stablecoin","hsm","hollar","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-hsm

**TL;DR:** Hollar Stability Module. Mint/burn [[wiki/hollar\|hollar]] against approved collateral (gDOT, USDT, etc.) at oracle-driven prices. Arbitrage bots can flash-mint HOLLAR to rebalance [[wiki/stableswap\|stableswap]] pools against the peg. Runtime index = 82.

## Role

Peg-enforcement engine for HOLLAR. Provides a primary-market mint/burn path that clamps HOLLAR price to $1 by routing arbitrage through stableswap pools.

## Config trait (excerpt)

```rust
// pallets/hsm/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    #[pallet::constant] type HollarId: Get<Self::AssetId>;
    #[pallet::constant] type PalletId: Get<PalletId>;
    type GhoContractAddress: Get<EvmAddress>;
    type EvmAccounts: InspectEvmAccounts<Self::AccountId>;
    type Evm: EVM<CallResult>;
    type Stableswap: StableswapAddLiquidityAndSwap<Self::AssetId, Balance, Self::AccountId>;
    type GasLimit: Get<u64>;
    type GasWeightMapping: GasWeightMapping;
    type FlashMinter: GetFlashMinterAddress;
    type OraclePriceProvider: PriceOracle<Self::AssetId, Price = EmaPrice>;
    type OraclePeriod: Get<OraclePeriod>;
    #[pallet::constant] type MaxAllowedCollaterals: Get<u32>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Collaterals` | StorageMap | `AssetId → CollateralInfo` (pool_id, purchase_fee, max_buy_price_coefficient, buy_back_rate, buyback_fee, max_in_holding) |
| `HollarAmountReceived` | StorageMap | `AssetId → Balance` (holding tracker per collateral) |
| `FlashMinter` | StorageValue | `Option<EvmAddress>` |

## Events

`CollateralAdded`, `CollateralUpdated`, `CollateralRemoved`, `SellExecuted`, `BuyExecuted`, `ArbitrageExecuted`, `FlashMinterSet`.

## Errors

`AssetNotApproved`, `CollateralLimitExceeded`, `MaxBuyPriceCoefficientExceeded`, `OraclePriceNotAvailable`, `InvalidEVMCall`, `InvalidHollarMintCall`, `InvalidCallParam`, `FlashMinterNotSet`, `MaxBuyBackExceeded`, `InsufficientHollarLiquidity`, `NoArbitrageOpportunity`, `PoolNotFound`, `InvalidCollateralPrice`.

## Extrinsics

| Name | Description |
|------|-------------|
| `add_collateral_asset` | Approve new collateral (AuthorityOrigin) |
| `update_collateral_asset` | Adjust fees/limits per collateral |
| `remove_collateral_asset` | Remove collateral |
| `set_flash_minter` | Set the EVM flash-mint contract address |
| `sell` | Sell collateral → receive HOLLAR at oracle price minus purchase_fee |
| `buy` | Buy collateral with HOLLAR (limited by max buy-back rate) |
| `execute_arbitrage` | Bot-callable: flash-mint HOLLAR, trade through stableswap, burn profit |

## Hooks

None.

## Integration

- **Traits implemented:** `TradeExecution`, `MMOracle` (feeds stableswap's peg oracle)
- **Traits consumed:** `StableswapAddLiquidityAndSwap`, `EVM`, `InspectEvmAccounts`, `PriceOracle`
- **Pallets depended on:** [[wiki/pallet-stableswap\|pallet-stableswap]], [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]], [[wiki/pallet-frontier\|pallet-frontier]]

## Key concept: arbitrage flow

```
1. HSM observes stableswap HOLLAR/USDT pool price below peg.
2. Bot calls `execute_arbitrage(pool_id, collateral, amount)`.
3. HSM flash-mints HOLLAR via GHO contract (on EVM).
4. HSM sells HOLLAR into the stableswap pool, receives USDT.
5. HSM burns the debt HOLLAR; keeps USDT profit as protocol reserve.
6. Pool price pushed back toward peg.
```

## Gotchas

- HOLLAR is implemented as an ERC-20 via the GHO contract (Aave's GHO forked); HSM calls it through EVM.
- `max_buy_price_coefficient` caps how far off-peg HSM is willing to buy collateral (prevents paying too much during depeg).
- `buyback_fee` + `buy_back_rate` limit how fast protocol reserves can be drawn down.
- `MMOracle` trait: feeds per-peg price to [[wiki/pallet-stableswap\|pallet-stableswap]] for drifting-peg pools.
- `max_in_holding` per-collateral prevents over-concentration (governance-tunable).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hollar\|hollar]]
- [[wiki/stableswap\|stableswap]]
