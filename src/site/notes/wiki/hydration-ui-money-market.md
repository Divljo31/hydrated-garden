---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-money-market/","title":"hydration-ui money-market","tags":["money-market","aave","lending","hydration","gigahdx","bil"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui money-market","repo":"hydration-ui","paths":["packages/money-market/src","packages/money-market/src/hooks","packages/money-market/src/ui-config/marketsConfig.ts","packages/money-market/src/ui-config/addresses.ts","packages/money-market/src/utils/convertTx.ts","packages/money-market/src/components/MoneyMarketProvider.tsx"],"key_exports":["MoneyMarketProvider","useMoneyMarketData","useUserData","useWalletData","useMarketAssetsData","useSupplyAssetsData","useSuppliedAssetsData","useBorrowAssetsData","useBorrowedAssetsData","useAggregatedMarketStats","useFormattedHealthFactor","useFormattedLtv","useAssetCaps","useApprovedAmount","useModal","useProtocolDataContext","useReserveActionState"],"symbols":["CustomMarket","marketsData","AaveV3HydrationMainnet","AaveV3GIGAHDXPool","AaveV3BILVaultMainnet","ExtendedEvmCall","ExtendedFormattedUser","MoneyMarketTxFn","UseMaxBalanceFn","ExternalApyData","convertPopulatedTransactionToEvmCall","calculateMaxWithdrawAmount","useSupplyEstimationTx","useWithdrawEstimationTx","useRepayEstimationTx","gasLimitRecommendations","HEALTH_FACTOR_RISK_THRESHOLD","GIGAHDX_LAUNCH_BLOCK","GIGAHDX_ANNUAL_BASE_INCENTIVES_HDX","GIGAHDX_ANNUAL_VOTING_INCENTIVES_HDX"],"key_deps":["@aave/contract-helpers","@aave/math-utils","ethers","@tanstack/react-query","@galacticcouncil/web3-connect","@galacticcouncil/xc-core","@galacticcouncil/xc-sdk"],"tags":["money-market","aave","lending","hydration","gigahdx","bil"],"last_updated":"2026-08-15"}}
---


# hydration-ui money-market

**TL;DR:** Vendored Aave v3 UI integration (`packages/money-market/`) — a fork of the Aave interface's store/hooks/modals wired to Hydration's EVM precompiles. As of Aug 2026 it serves **three markets**, not one: `hydration_v3` (main money market), `gigahdx_v3` ([[wiki/gigahdx\|gigahdx]] staking pool) and `bil_v3` (the BIL vault behind the new `strategies/` module). Consumed by `borrow/` and `strategies/` ([[wiki/hydration-ui-modules\|hydration-ui-modules]]).

## Purpose

Abstracts Aave v3 protocol interactions (supply collateral, borrow, repay, withdraw, e-mode, claim rewards). Exposes a `MoneyMarketProvider` that owns the Zustand root store + all tx modals, plus React Query hooks over the Aave contract helpers.

Package `exports` map is `"./*": "./src/*/index.ts"` — import from `@galacticcouncil/money-market/hooks`, `/components`, `/utils`, `/ui-config`, `/types`.

## Dependencies

- `@aave/contract-helpers` v1.23.1 — Aave v3 contract ABI wrappers
- `@aave/math-utils` v1.23.1 — Aave math calculations (risk, APY, etc.)
- `ethers` v5.7.0 — the only ethers user in the monorepo
- `reflect-metadata` v0.1.13 — Decorator support for Aave helpers
- `@tanstack/react-query` `^5.100.14` (bumped from `^5.59.15`)
- `@galacticcouncil/ui`, `/utils`, `/web3-connect` (build level 3)

**Undeclared:** `@galacticcouncil/xc-sdk` (`types/index.ts` → `EvmCall`) and `@galacticcouncil/xc-core` (`utils/convertTx.ts` → `CallType`) are imported but not listed in `packages/money-market/package.json`; they resolve by hoisting from the root `dependencies`. Root `xc-*` bumps therefore reach this package unannounced.

## Markets (`ui-config/marketsConfig.ts` + `ui-config/addresses.ts`)

```typescript
// packages/money-market/src/ui-config/marketsConfig.ts
export enum CustomMarket {
  hydration_v3 = "hydration_v3",
  hydration_testnet_v3 = "hydration_testnet_v3",
  bil_v3 = "bil_v3",          // NEW — BIL vault
  gigahdx_v3 = "gigahdx_v3",  // NEW — GIGAHDX staking pool
}
```

| Market | `marketTitle` | Addresses export | Used by |
|---|---|---|---|
| `hydration_v3` | "Hydration" | `AaveV3HydrationMainnet` | `borrow/` module |
| `hydration_testnet_v3` | testnet | `AaveV3HydrationTestnet` | dev only |
| `gigahdx_v3` | "GIGAHDX" | `AaveV3GIGAHDXPool` (`POOL 0x2Ce2…8923`) | [[wiki/gigahdx\|gigahdx]] staking UI at `/staking` |
| `bil_v3` | "BIL" | `AaveV3BILVaultMainnet` (`POOL 0x6931…7a53`) | `strategies/` module |

**Breaking provider change.** `MoneyMarketProvider` no longer takes `env: MoneyMarketEnv` and derives the market from it — the caller passes the market directly, plus two new injection points:

```typescript
// packages/money-market/src/components/MoneyMarketProvider.tsx
export type MoneyMarketProviderProps = AppFormattersProvidersContextType & {
  children: React.ReactNode
  provider: ExternalProvider
  market: CustomMarket                              // was: env: MoneyMarketEnv
  squidClient: SquidSdk
  onCreateTransaction: MoneyMarketTxFn
  useMaxBalance: UseMaxBalanceFn                    // NEW — app-supplied max-balance-with-fee
  getRelatedATokenId: (id: string) => string | undefined  // NEW — aToken ↔ asset mapping
  externalApyData: ExternalApyData
}
```

`setProvider(new Web3Provider(externalProvider))` dropped its `env` argument; `setCurrentMarket(market)` runs in its own effect, so the app can switch markets at runtime (GIGAHDX ↔ Hydration ↔ BIL).

## GIGAHDX APR constants (`ui-config/misc.ts`)

Governance ref 358 numbers, used to compute a guaranteed APR **floor** and to gate the measured rate-delta APR (`max(guaranteed, measured)`, commit `2d6de3d`):

| Constant | Value | Meaning |
|---|---|---|
| `GIGAHDX_LAUNCH_BLOCK` | `12959351` | Block where the gigapot was swept to ~0, so GIGAHDX:HDX = 1. Anchors the passive-APR baseline and gates when the measured rate-delta becomes meaningful (~7 days of pallet age) |
| `GIGAHDX_ANNUAL_BASE_INCENTIVES_HDX` | `36_000_000` | Annual HDX into the `gigahdx!` pot (4,109.59 HDX / 600 blocks × 8,760) |
| `GIGAHDX_ANNUAL_VOTING_INCENTIVES_HDX` | `54_000_000` | Annual HDX into the voting-rewards pot `gigarwd!` (6,164.38 HDX / tick) |

Also new here: `HDX_ERC20_ASSET_ID = "67"`, `STHDX_ASSET_ID = "670"`. `HEALTH_FACTOR_RISK_THRESHOLD` was **lowered 1.2 → 1.1**. See [[wiki/gigahdx\|gigahdx]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]].

## Main hooks (`src/hooks/index.ts`)

| Hook | Purpose |
|------|---------|
| `useMoneyMarketData()` | Aggregate market + user snapshot |
| `useUserData()` / `useWalletData()` | User reserves, wallet balances |
| `useMarketAssetsData()` | Reserve list + rates for the current market |
| `useSupplyAssetsData()` / `useSuppliedAssetsData()` | Supply-side dashboards |
| `useBorrowAssetsData()` / `useBorrowedAssetsData()` | Borrow-side dashboards |
| `useAggregatedMarketStats()` | Protocol totals |
| `useFormattedHealthFactor()` / `useFormattedLtv()` | Risk display |
| `useAssetCaps()` | Supply / borrow cap state |
| `useApprovedAmount()` | ERC-20 allowance |
| `useReserveActionState()`, `usePermissions()`, `useModal()`, `useProtocolDataContext()` | Modal + permission plumbing |
| `useProtocolActionToasts()` | Toast copy per `ProtocolAction` |
| `useCurrentTimestamp()`, `useLocalStorageBool()` | Misc |

**Internal (not in the barrel):** `useSupplyEstimationTx`, `useWithdrawEstimationTx`, `useRepayEstimationTx` (all NEW) are deep-imported by their own modal contents (`components/transactions/{supply,withdraw,repay}/…ModalContent.tsx` via `@/hooks/useXEstimationTx`). They build + gas-estimate the populated tx ahead of submission, returning an `ExtendedEvmCall` used to show a real fee and to compute max-balance-minus-fee.

There is no `useAaveMarketData` / `useAaveUserData` / `useLiquidationRisk` / `useSupplyCapacity` — those names never existed in this fork.

## Integration with borrow + strategies modules

The `borrow/` and `strategies/` modules ([[wiki/hydration-ui-modules\|hydration-ui-modules]]) use these hooks to:
1. Display available reserves (collateral + borrow options)
2. Fetch user's current positions
3. Calculate health factor for risk dashboard
4. Show collateral prices and liquidation thresholds

`ExtendedFormattedUser` (`src/hooks/commonTypes.ts`) — its `{earnedAPY, debtAPY, netAPY}` fields are now `number | null` (previously `number`) — an unknown APY renders as "—" rather than `0` (`884768d` "Fix Net APY"). Consumers must null-check.

## Aave math utils

Re-exported / wrapped in `src/utils/`: `getMaxAmountAvailableToBorrow`, `getMaxAmountAvailableToSupply`, **`calculateMaxWithdrawAmount`** (moved out of the withdraw modal into `utils/calculateMaxWithdrawAmount.ts`, `b50094f`), `hfUtils`, `interestRateModel`, `eMode`, `ghoUtilities`, `dashboard`, `formatters`.

## EVM chain interaction

On [[wiki/pallet-frontier\|pallet-frontier]] (EVM precompiles for Hydration), calls to Aave contracts are routed through the EVM. `ethers` builds a `PopulatedTransaction`, which is converted to the SDK's call shape by the newly extracted `utils/convertTx.ts` (lifted out of `Web3Provider.tsx`, `2afb311`):

```typescript
// packages/money-market/src/utils/convertTx.ts
export const convertPopulatedTransactionToEvmCall = (
  tx: PopulatedTransaction,
  action?: ProtocolAction,
) => {
  const isClaimTx = tx.data?.startsWith(CLAIM_ALL_METHOD_HASH)
    || tx.data?.startsWith(CLAIM_METHOD_HASH)
  const abi = isClaimTx ? getClaimTransactionAbi(tx) : getPoolTransactionAbi(action)
  return { data, from, to, type: CallType.Evm, abi,
           gasLimit, maxFeePerGas, maxPriorityFeePerGas, dryRun } as ExtendedEvmCall
}
```

`Web3Provider.tsx` now also tags each tx with an **`ActivityType`** from [[wiki/hydration-ui-utils\|hydration-ui-utils]] (`supply → "lend"`, `borrow → "borrow"`, `repay → "repay"`, `withdraw → "withdraw"`), which flows through `MoneyMarketTxFn` into the transactions module so the toast/history entry can deep-link into the Neckwork explorer's activity views.

`ExtendedEvmCall` (`src/types/index.ts`) extends `EvmCall` from `@galacticcouncil/xc-sdk` with `nonce?`, `gasLimit?`, `maxFeePerGas?`, `maxPriorityFeePerGas?`.

## Gas limits (`ui-config/gasLimit.ts`)

`gasLimitRecommendations` values are **strings**. `ProtocolAction.supply` got its own entry (`"1000000"`, `c60e3c3`), and `ProtocolAction.borrow` was raised `"1000000" → "1300000"` (`1b1195f`) after on-chain borrow reverts; the default and `setUsageAsCollateral` stay at `"1000000"`.

## Data flow

```
borrow/ + strategies/ modules
    ↓ (useMoneyMarketData, useUserData, useMarketAssetsData)
money-market hooks (Zustand root store + React Query)
    ↓
@aave/contract-helpers (contract calls)
    ↓
[EVM RPC (Frontier precompiles)]
    ↓
Aave v3 contracts (read-only)
```

Write operations (supply, borrow, withdraw, repay) are converted to an `ExtendedEvmCall` and handed to `onCreateTransaction`, then submitted via [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]].

## React Query integration

Hooks read from the Zustand root store (`src/store/root.ts`) and wrap Aave service calls in React Query, keyed via `ui-config/queries.ts` (`queryKeysFactory`). The estimation hooks are the clearest example of the current pattern:

```typescript
// packages/money-market/src/hooks/useSupplyEstimationTx.ts (excerpt)
const [supply, estimateGasLimit, account] = useRootStore((store) => [
  store.supply, store.estimateGasLimit, store.account,
])

return useQuery({
  enabled: enabled && !!account && !!amount && !!poolAddress,
  queryKey: ["supplyEstimationTx", poolAddress, amount],
  queryFn: async (): Promise<ExtendedEvmCall> => {
    let tx = supply({ amount: parseUnits(amount, decimals).toString(), reserve: poolAddress })
    tx = await estimateGasLimit(tx, ProtocolAction.supply)
    return convertPopulatedTransactionToEvmCall(tx, ProtocolAction.supply)
  },
})
```

Queries are cached and invalidated on block finalization (`queryKeysFactory` + the app's `@block` prefix).

## Aave v3 features supported

- **Collateral supply** — Deposit assets as collateral
- **Variable borrowing** — Borrow with floating rates
- **Collateral switching** — Change collateral types
- **E-mode** — Efficiency mode selection (`emode/` modal group)
- **Claim rewards** — Incentive claiming (`claim/` modal group)
- **Liquidation monitoring** — Track health factor, liquidation price
- **Risk parameters** — LTV, liquidation threshold, penalty, caps, debt ceiling, isolation mode

## For developers

To add a lending feature:

```typescript
import { useUserData, useFormattedHealthFactor } from "@galacticcouncil/money-market/hooks"

export const BorrowForm = () => {
  const { user } = useUserData()
  const healthFactor = useFormattedHealthFactor()
  // user.netAPY may be null — null-check before formatting
}
```

Deposit and Withdrawal forms were reworked in Aug 2026 (`d3224c1`, `bcdec5f`) around the new estimation hooks + `useMaxBalance` injection; GHO is now accepted as a repay asset (`c413acd`) and GHO borrow validation was fixed (`ea857e6`, `3fa51f2` "fix inflated hollar max borrow"). Health-factor display shows an infinity icon above 1000 (`4877015`).

Tx submission is handled by [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] after the `ExtendedEvmCall` is handed to `onCreateTransaction`.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/hydration-ui-utils\|hydration-ui-utils]], [[wiki/gigahdx\|gigahdx]], [[wiki/pallet-frontier\|pallet-frontier]], [[wiki/pallet-liquidation\|pallet-liquidation]]
