---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-modules/","title":"hydration-ui modules","tags":["modules","products","hydration","react"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui modules","repo":"hydration-ui","paths":["apps/main/src/modules/trade","apps/main/src/modules/liquidity","apps/main/src/modules/borrow","apps/main/src/modules/strategies","apps/main/src/modules/onramp","apps/main/src/modules/portfolio","apps/main/src/modules/xcm","apps/main/src/modules/staking","apps/main/src/modules/stats","apps/main/src/modules/transactions","apps/main/src/modules/submit-transaction","apps/main/src/modules/tutorials","apps/main/src/modules/layout","apps/main/src/modules/governance"],"symbols":["SwapPage","SwapTabPage","XcSwap","XcSwapProvider","Limit","Dca","TradeOrders","TradeOtcPage","PoolsPage","PoolDetails","VaultDetails","AddVaultLiquidity","TransferVaultPosition","BorrowDashboardPage","StrategiesPage","BilVaultPage","StableBondsPage","DepositPage","WithdrawPage","PortfolioOverviewPage","PortfolioTrackedPage","XcmPage","useScanJourneys","GigaStakePage","StakingDashboard","StatsFees","TutorialProvider","TutorialAnchor","SubmitTransactionPage","BreadcrumbBar","UserMenu"],"key_deps":["@tanstack/react-query","@galacticcouncil/sdk-next","@galacticcouncil/xc-swap","@galacticcouncil/money-market","@galacticcouncil/xc-core","viem"],"tags":["modules","products","hydration","react"],"last_updated":"2026-11-02"}}
---


# hydration-ui modules

**TL;DR:** 14 feature folders under `apps/main/src/modules/`, each owning pages, components, hooks and forms; routes in `apps/main/src/routes/` mount them. As of `770f868`: the **Market** swap tab is now `XcSwap` (on-chain *and* cross-chain swaps via [[wiki/xc-swap\|xc-swap]]), **Limit** orders are [[wiki/ice\|ice]] intents, a new **portfolio/** module replaced `wallet/` (`/wallet*` routes only redirect to `/portfolio`), liquidity gained **Gamma-managed Uniswap V3 vaults**, and **tutorials/** adds first-run hints.

## Module map

| Module | Routes | One-liner |
|---|---|---|
| `trade/` | `/trade/swap/$tab` (`market`/`limit`/`twap`), `/trade/otc`, `/trade/swap/xc`, `/trade/yield-dca` | Swap, ICE limit orders, TWAP/DCA, OTC, order history |
| `liquidity/` | `/liquidity`, `/liquidity/$id[/add\|join\|remove]`, `/liquidity/create`, `/liquidity/vault/$address` | Omnipool / stableswap / isolated / money-market LP, farms, V3 vaults |
| `borrow/` | `/borrow/{dashboard,markets,markets/$address,history}` | Aave v3 money market ([[wiki/hydration-ui-money-market\|hydration-ui-money-market]]) |
| `strategies/` | `/strategies`, `/strategies/bil-vault`, `/strategies/hollar-bonds[/$bondId]` | BIL RWA vault + Hollar stable bonds |
| `portfolio/` | `/portfolio`, `/portfolio/tracked` | Balances, positions, bonds, rewards; multichain + tracked wallets |
| `onramp/` | `/deposit`, `/withdraw` | CEX / bank deposit & withdraw wizard |
| `xcm/` | `/cross-chain`, `/cross-chain/history` | Cross-chain transfers + scan-backed history/claims |
| `staking/` | `/staking` (GigaHDX), `/staking-old` (legacy) | [[wiki/gigahdx\|gigahdx]] liquid staking + OpenGov voting |
| `stats/` | `/stats/fees` (others are placeholders) | Fee/revenue analytics |
| `transactions/` | — | Tx queue, review modal, signing, toasts |
| `submit-transaction/` | `/submit-transaction` | Standalone call-data submit page |
| `tutorials/` | — | First-run hint anchors |
| `layout/` | `__root` | Shell, header, breadcrumbs, nav, footer |
| `governance/` | **none** | Unrouted (`GovernancePage` is imported nowhere) — dead until a route lands |

## trade/

`swap/SwapTabPage.tsx` maps the `$tab` param:

```typescript
// apps/main/src/modules/trade/swap/SwapTabPage.tsx
const SWAP_TABS = { market: XcSwap, limit: Limit, twap: Dca } as const
```

- **`swap/sections/XcSwap/`** (Market) — `XcSwapProvider` + `XcSwapClient` from [[wiki/xc-swap\|xc-swap]]. A quote is a union: `{ kind: "oc", swap: Trade, twap?: TradeOrder }` (on-chain via `sdk.api.router`) or `{ kind: "xc", swap: XcSwapTrade }` (cross-chain, e.g. NEAR/Zcash destinations). Hooks: `useXcSwapQuote`, `useSubmitByQuote` → `useSubmitSwap` / `useSubmitTwap` / `useSubmitXcSwap`, `useXcSwapRequiredWalletMode`, `useNearRecipientValidation`, `useXcSwapHealthFactor` (money-market HF when selling aTokens), `useMaxSellAmount`. `lib/iceAmounts.ts` pads ICE buy amounts by slippage. Kraken spot price (`api/external/kraken.ts`) is the reference price for XC pairs.
- **`swap/sections/Limit/`** — ICE limit order: `sdk.tx.intentLimit(trade).withBeneficiary().withMinAmountOut().withPartial().withDeadline(getIntentDeadline(...))` (`useSubmitLimitOrder.ts`); `useLimitCascade` / `cascadeLogic.ts` (price ↔ amount cascade), `useLimitHealthFactor`. Gated by `useIsIceEnabled()` (`featureFlags.isIceEnabled && intents store enabled`).
- **`swap/sections/DCA/`** — TWAP/DCA (`useDcaForm`, `useSubmitDcaOrder`, `DcaLimitPrice`, budget modes).
- `swap/components/` — `TradeChart/`, `SwapChart/`, `XcSwapChart/` (Neckwork candles + Kraken reference), `TradeRoutes/`, `SettingsModal/`, `QuotedPriceField/`, `TradeFormShell/`.
- **`orders/`** — `TradeOrders/` (`OpenOrders`, `OrderHistory`, `MarketTransactions`, `MyRecentActivity`, `PastExecutions`, `PastExecutionsIntent`); detail modals for swap / DCA / limit; `lib/useIntentOrdersData.ts`, `useRemoveIntent.ts` (`Intent.remove_intent`), `useTerminateDcaSchedule.ts` (`DCA.terminate`), `useLimitFillStatus.ts`, `getSwapExplorerLink.ts` (Neckwork). Data from Neckwork `intents` / `dca` / `trades` ([[wiki/hydration-ui-indexer\|hydration-ui-indexer]]).
- **`otc/`** — place / fill / partial-fill / cancel (`OTC.*`), table from `usePapiEntries("OTC.Orders")`.

Links: [[omnipool\|omnipool]], [[wiki/smart-order-router\|smart-order-router]], [[wiki/ice\|ice]], [[wiki/pallet-dca\|pallet-dca]], [[wiki/pallet-otc\|pallet-otc]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-swap\|xc-swap]]

## liquidity/

`PoolsPage` (omnipool+stablepool table, isolated pools, **vaults** table — `Vaults.columns.tsx` / `Vaults.utils.ts` `useVaults` — plus `MyLiquidityTab`), `PoolDetails`, **`VaultDetails`**. Flows under `components/`: `AddLiquidity`, `AddStablepoolLiquidity`, `AddIsolatedLiquidity`, `AddMoneyMarketLiquidity`, `SupplyIsolatedLiquidity`, **`AddVaultLiquidity`**, `RemoveLiquidity` (incl. **`RemoveVaultLiquidity`**), **`TransferVaultPosition`**, `CreateIsolatedPool`, `JoinFarms`, `AvailableFarms`, `Farms`, `PositionsTable`, `NewFarmsBanner`, `PoolDetailsChart`.

**Vaults** = Gamma hypervisors over Uniswap V3 pools on Hydration EVM (`api/gamma/`: `GAMMA_STACKS`, `useVaultSnapshots`, `useVaultDepositChecks`; pools via `api/pools.ts` `useV3Pools`). `VaultDetails/` renders `LiquidityDistribution`, `VaultComposition`, `VaultPriceChart`, `VaultPositions`, `VaultExplainer`; `AutoManagedBadge` marks them.

Links: [[wiki/nft-lp-positions\|nft-lp-positions]], [[omnipool\|omnipool]], [[wiki/stableswap\|stableswap]], [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]]

## borrow/

`dashboard/` (`BorrowDashboardPage`), `markets/` (`BorrowMarketsPage`, market detail), `history/` (Neckwork `useMoneyMarketEvents`), `reserve/`, `hollar/`, `healthfactor/`, `context/ApyContext.tsx`, `BorrowContextProvider.tsx`. New: `components/StrategySupplyModal.tsx`, `hooks/useMoneyMarketPostTxRefresh.ts`.

Links: [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hollar\|hollar]]

## strategies/

`StrategiesPage` renders `StrategyCard`s gated by `featureFlags.bilEnabled` / `hollarBondsEnabled` (all `true` once the RPC context loads).

- **`bil/`** — BIL vault on Hydration EVM: ERC-4626 deposit + ERC-7540 async redeem, Aave layer (supply BIL, borrow/repay HOLLAR), instant redeem via the BIL/HOLLAR stablepool. Everything is viem calls batched through `Dispatcher.dispatch_evm_call`. Addresses in `bil/config/constants.ts` (`VAULT_ADDRESS 0x6a21…810C`, `HOLLAR_ADDRESS 0x531a…f99a`, `DCL_PRECOMPILE_ADDRESS` = asset 550). New: `BilBorrowCapCurrency`, `utils/queueWait.ts` (redemption-queue ETA).
- **`stable-bonds/`** — Hollar bonds sold via OTC offers (`config/bonds.ts` `STABLE_BONDS`: `HOLLAR_BOND_25_08_26_ID` 1.725 % offers `1488/1489`, `HOLLAR_BOND_24_11_26_ID` 2.0165 % offers `1546/1547`). New: **rollover** (`STABLE_BONDS[25_08_26].rollover → toBondId 24_11_26` via OTC offer `1548`; `StableBondsRollover*`, `useStableBondsRollover`), `StableBondsPastSales`, `useStableBonds`. Redeem via `api/bonds.ts`.

Links: [[wiki/hollar\|hollar]], [[wiki/pallet-otc\|pallet-otc]], [[wiki/hydration-precompiles\|hydration-precompiles]]

## portfolio/ (replaces wallet/)

- **`overview/`** (`/portfolio`, search param `category: assets | liquidity | bonds`) — `PortfolioOverviewPage`, `PortfolioSummary`, per-chain `PortfolioChainSection` / `PortfolioChainHeader`, `Balances/` (`NetWorth`, `WalletBalances`), `MyAssets/` (asset rows, unlock via `api/locks.ts`, `DepositToHydrationAction`, `AssetOrigin`), `MyLiquidity/` (omnipool / stableswap / XYK / **vault** positions), `MyBonds/`, `Rewards/` (claim), `Transfer/` (`TransferPositionModal` — transfer an LP position NFT).
- **`tracked/`** (`/portfolio/tracked`) — watch-only **tracked wallets** (`states/trackedWallets.ts`, `ManageTrackedWalletsModal`, `TrackedWalletCard`). Multichain valuation via `api/portfolio/multichain.ts` `useMultichainPortfolio`.
- Portfolio queries persist across reloads (`setupPortfolioPersistence`, [[wiki/hydration-ui-api\|hydration-ui-api]]).

## onramp/

`/deposit` (`DepositPage` + `DepositManager`) and `/withdraw` (`WithdrawPage`). CEX ids `kraken | binance | kucoin | coinbase | gateio` (`types/`), `config/cex.tsx` (`CEX_CONFIG`), persisted wizard state `store/useOnrampStore.ts`, step components under `deposit/steps/` and `withdraw/steps/`. CEX legs route through the `assethub_cex` chain ([[wiki/xc-cfg\|xc-cfg]]); header `DepositButton` shows pending deposits.

## xcm/

- **`transfer/`** — `XcmTransferApp`, `XcmForm`, `XcmSummary`, `XcmLimits`. Bridge choice: `utils/bridge.ts` + `hooks/useBridgeOptions.ts` (Basejump → NttExecutor → Wormhole → Snowbridge priority; `BRIDGE_PROVIDER_TAGS` in `states/transactions.ts`). Limits & alerts: circuit-breaker deposit / global-withdraw (`useCrossChainDepositLimit`, `useCrossChainGlobalWithdrawLimit`), NTT rate limits, **Wormhole custody** (`WormholeCustodySummaryRow`, `useNttCustody`) and **Basejump pool liquidity** (`BasejumpPoolSummaryRow`, `useBasejumpPoolAlerts`); `useHydrationEvmBindingAlert`; `Recipient/RecipientModal`. Kill switches: `VITE_WORMHOLE_DISABLED`, `VITE_BASEJUMP_DISABLED`, `VITE_SNOWBRIDGE_DISABLED`, `VITE_SNOWBRIDGE_INBOUND_DISABLED`.
- **`history/`** — journeys from the **transfers scan app**: `useScanJourneys` (`scanJourneysQuery`) + `useScanJourneysSubscription` (`subscribeScanEvents`, refetch debounced 500 ms, refetch on reconnect) merged with xc-scan (`useXcScan`, `xcScanStore`). Claims: `hooks/useDepositClaim`, `useWithdrawClaim`, `useClaimableTransactions`, `useJourneyRedeemed`, `usePendingClaimsStore`; `utils/scan.ts`, `journey.ts`, `optimistic.ts`, `claim.ts`.

Links: [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-package\|xc-package]], [[wiki/xc-core\|xc-core]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/wormhole\|wormhole]], [[wiki/snowbridge\|snowbridge]], [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]]

## staking/

| Route | Component | Backing |
|---|---|---|
| `/staking` | `gigaStaking/GigaStakePage.tsx` | [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] |
| `/staking-old` | `StakingDashboard.tsx` | [[wiki/pallet-staking\|pallet-staking]] (legacy; migration path to GigaHDX) |

`gigaStaking/`: `stake/`, `unstake/`, `pendingPositions/`, `borrow/` + `repay/` (Aave via `Dispatcher.dispatch_evm_call`), `supplyInfo/`, `GigaStakingMigration` (`GigaHdx.migrate()`), dashboard/headers. Referenda voting (`Referenda*.tsx`, `OngoingReferenda.tsx`, `AyeDetails.tsx`, `components/VoteModal/`) is shared by both pages. APR from Neckwork (`api/gigaApr.ts`).

## stats/

Only **`fees/`** has content (`StatsFees`: `FeeAndRevenueChart/` stacked bars from Neckwork `feesChartQuery`; the Sankey `FeeFlowChart` was removed). `/stats/{overview,treasury,money-market,hollar,amm}` route components return `null`, and `/stats` redirects to `/stats/overview` — i.e. to an empty page; only the fees view is real.

## transactions/

`TransactionProvider` / `TransactionManager` (modal host + toast pump), `review/` (`ReviewTransaction`, `ReviewMultiTransaction`, `ReviewMultisig/`, JSON view, fee / tip / mortality), hooks `useSignAndSubmit`, `useNonce`, `useEstimateFee`, `useBatchTx`, `useMaxBalance*`, **`useIntentToasts`**, **`useTxCallData`**, toast processors. Utils per signer: `polkadot.ts`, `ethereum.ts`, `solana.ts`, `sui.ts`, **`near.ts`**, **`zec.ts`**, **`xcSwap.ts`**, `xcm.ts`, `permitFee.ts`, `tx.ts`; `toasts/` (`extrinsics.ts`, `intents.ts`, `intentOutcomes.ts`). Detail: [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]].

## submit-transaction/

`SubmitTransactionPage.tsx` — decode and submit arbitrary call data. See [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]].

## tutorials/

`TutorialProvider` (mounted in `__root.tsx`) + `TutorialAnchor` wrap UI elements with step hints. `config.ts` `tutorials`: `settings-avatar`, `trade-intents` (shown after the intents modal). State: `states/tutorials.ts` (persisted); `?resetTutorials` param handled by `applyResetTutorialsParam()` in `App.tsx`.

## layout/

`MainLayout`, `SubpageLayout`, `components/` (`Header`, `HeaderMenu`, `HeaderToolbar`, `HeaderWeb3ConnectButton`, **`UserMenu/`**, **`SubNavBar`**, **`BreadcrumbBar`**, `MobileTabBar/`, `Settings/` (+ `AvatarPreference`), `NotificationCenter/`, `Footer`, `DepositButton/`, `NavigationItemLink`, `StrategiesHeaderSubmenu`, `LayoutSkeleton/`), **`crumbs/`** (`AssetCrumb`, `PoolCrumb`) + `hooks/useCrumbs.ts` (routes declare `staticData.crumb`), `hooks/useNavigation`, `useHasTopNavbar`, `useRemoveInitialLoader`.

## Conventions

File suffixes: `.form.ts` (zod + react-hook-form), `.data.ts` / `.utils.ts`, `.tx.ts` / `.submit.ts`, `.columns.tsx`, `Skeleton.tsx`. Shared: react-hook-form + zod v4, Emotion via [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], Zustand in `src/states/`, viem + `Dispatcher.dispatch_evm_call` for EVM.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-utils\|hydration-ui-utils]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-swap\|xc-swap]], [[wiki/gigahdx\|gigahdx]], [[wiki/ice\|ice]]
