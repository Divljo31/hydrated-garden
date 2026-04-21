---
{"dg-publish":true,"permalink":"/wiki/pallet-liquidation/","title":"pallet-liquidation","tags":["lending","liquidation","money-market","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-liquidation","repo":"hydration-node","paths":["pallets/liquidation/src/lib.rs","pallets/liquidation/src/types.rs"],"symbols":["Pallet","Config","liquidate","BorrowingContract"],"traits_impl":[],"depends_on":["pallet-route-executor","pallet-evm","pallet-currencies"],"runtime_index":76,"tags":["lending","liquidation","money-market","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-liquidation

**TL;DR:** Liquidator adapter for [[wiki/hydration-borrow\|hydration-borrow]] (Aave-fork money market on Hydration EVM). Calls the borrowing contract's `liquidationCall` via EVM, routes the received collateral through [[wiki/pallet-route-executor\|pallet-route-executor]] for profit. Runtime index = 76.

## Role

Bridges Substrate-native liquidation bots to the EVM lending protocol. Users call `liquidate` specifying borrower + debt amount; pallet flashes the debt, calls EVM liquidation, sells collateral through the router, keeps the profit.

## Config trait (excerpt)

```rust
// pallets/liquidation/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    type BorrowingContract: Get<EvmAddress>;
    type Evm: EVM<CallResult>;
    type EvmAccounts: InspectEvmAccounts<Self::AccountId>;
    type Router: TradeExecution<Self::RuntimeOrigin, Self::AccountId, Self::AssetId, Balance>;
    type RouteExecutor: RouteProvider<Self::AssetId>;
    type GasLimit: Get<u64>;
    type GasWeightMapping: GasWeightMapping;
    type ProfitReceiver: Get<Self::AccountId>;
    #[pallet::constant] type MinProfitPercentage: Get<Perquintill>;
    type HollarAssetId: Get<Self::AssetId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `BorrowingContract` | StorageValue | `Option<EvmAddress>` (updatable via governance) |

## Events

`Liquidated`.

## Errors

`ContractNotSet`, `EvmCallFailed`, `NotProfitable`, `InvalidRoute`, `InsufficientCollateral`.

## Extrinsics

| Name | Description |
|------|-------------|
| `liquidate` | Liquidate a borrow position via EVM contract |
| `set_borrowing_contract` | Governance sets/updates the money-market contract address |

## Hooks

None.

## Integration

- **Traits consumed:** `EVM`, `InspectEvmAccounts`, `Router` ([[wiki/pallet-route-executor\|pallet-route-executor]]), `MultiCurrency`
- **Pallets depended on:** [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/pallet-frontier\|pallet-frontier]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]

## Gotchas

- `BorrowingContract` is an Aave-v3 fork deployed at a fixed EVM address.
- Liquidator must hold / borrow debt-asset upfront (no flash-loans wired in this pallet — could be added via a precompile).
- `MinProfitPercentage` enforces round-trip profit or transaction reverts.
- HOLLAR liquidations get special treatment (uses [[wiki/pallet-hsm\|pallet-hsm]] integration instead of router).
- Liquidation penalty / bonus is configured on the EVM contract, not in this pallet.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-borrow\|hydration-borrow]]
