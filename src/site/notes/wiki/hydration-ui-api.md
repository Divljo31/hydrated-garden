---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-api/","title":"hydration-ui API (data layer)","tags":["api","data-fetching","hydration","react-query"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui API (data layer)","repo":"hydration-ui","paths":["apps/main/src/api","apps/main/src/api/rpcClient.ts","apps/main/src/api/rpcConfig.ts","apps/main/src/api/balances","apps/main/src/api/portfolio","apps/main/src/api/gamma","apps/main/src/api/neckwork.ts","apps/main/src/api/neckworkSync.ts","apps/main/src/api/intents.ts","apps/main/src/api/xcm.ts","apps/main/src/api/gigaStake.ts","apps/main/src/providers/rpcProvider.tsx"],"symbols":["TProviderContext","isReady","isFork","rpcProviderQuery","switchRpc","Papi","PapiNext","PROVIDER_LIST","getSortedRpcUrlList","bestSellQuery","bestBuyQuery","dcaOrderQuery","useAccountBalances","accountBalancesQuery","useMultichainPortfolio","setupPortfolioPersistence","useAccountIntents","getIntentDeadline","useVaultSnapshots","useV3Pools","useNeckworkSync","neckworkClient","transfersScanClient","scanJourneysQuery","useNttOutboundLimit","useNttCustody","useBasejumpPoolLiquidity","useCrossChainDepositLimit","useCrossChainGlobalWithdrawLimit","gigaStakeConstantsQuery","useGigaApr","useMoneyMarketEvents","useKrakenOhlc","useInvalidateOnBlock","useReloadOnStaleBlocks","isHydrationFork"],"key_deps":["@tanstack/react-query","@galacticcouncil/sdk-next","@galacticcouncil/indexer","@galacticcouncil/xc-core","polkadot-api"],"tags":["api","data-fetching","hydration","react-query"],"last_updated":"2026-11-02"}}
---


# hydration-ui API (data layer)

**TL;DR:** `apps/main/src/api/` — React Query `queryOptions` factories (`*Query`) and hooks (`use*`) by domain. Backed by [[wiki/papi\|papi]] (`rpc.papi` / `rpc.papiNext`), [[wiki/sdk-next\|sdk-next]] (`rpc.sdk`), the **Neckwork REST API** and scan apps via [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], one Grafana SQL dataset, and external APIs. Inventory below is the exported-name list grepped at `770f868` (2026-11-01).

## Structure

```
src/api/
├── rpcClient.ts      # Papi / PapiNext types, rpcProviderQuery, switchRpc (was provider.ts)
├── rpcConfig.ts      # PROVIDER_LIST, PROVIDER_URLS, getSortedRpcUrlList, data env
├── chainSpec.ts      # chainSpecDataQuery, isHydrationFork, getHydrationGenesisHash
├── chain.ts          # best block, useInvalidateOnBlock, useReloadOnStaleBlocks, block time
├── rpc.ts            # RPC status probes
├── neckwork.ts       # neckworkClient (+ PROXY_URL)
├── neckworkSync.ts   # useNeckworkSync — refresh account queries once the indexer catches up
├── indexer.ts        # useIndexerClient (GraphQL, farms)
├── scan.ts           # transfersScanClient, intentsScanClient, scanJourneysQuery
├── account.ts  assets.ts  metadata.ts  balances/  portfolio/  locks.ts
├── trade.ts  omnipool.ts  stableswap.ts  xyk.ts  pools.ts  spotPrice.ts  otc.ts  intents.ts
├── farms.ts  gamma/  bonds.ts
├── aave.ts  borrow/
├── gigaStake.ts  gigaApr.ts  staking.ts  democracy.ts  constants.ts
├── xcm.ts  wormhole.ts  near.ts
├── payments.ts  transaction.ts  evm.ts  dryRun.ts  errors.ts  multisig.ts  referrals.ts
├── stats.ts  grafana/ (reserveRate only)  external/
└── utils/slippage.ts  mock.ts
```

## Provider context

Every factory takes the rpc context from `useRpcProvider()` (`providers/rpcProvider.tsx`):

- `TProviderContext = TProviderData & { isEndpointSettled, isReady, dataEnv, endpoint, isFork }`; `TProviderData` (in `rpcClient.ts`) carries `papi` (`TypedApi<hydration>`), `papiNext` (`TypedApi<hydrationNext>`), `sdk`, feature flags.
- **`isReady`** (endpoint settled *and* asset registry populated) is the gate — the old `isApiLoaded` is gone. Account queries use `enabled: isReady && !!address`.
- `isFork` (`isHydrationFork(genesisHash)`) disables indexer-dependent behaviour on Chopsticks/lark forks.
- The provider's default value is load-bearing: the boot `LayoutSkeleton` renders outside the provider and only reads `featureFlags` / `isReady: false`. Don't make `useRpcProvider()` throw.

## Patterns

```typescript
// apps/main/src/api/trade.ts
export const bestSellQuery = ({ sdk, isReady }: TProviderContext, { assetIn, assetOut, amountIn, debug }: BestSellArgs) =>
  queryOptions({
    queryKey: [QUERY_KEY_BLOCK_PREFIX, "trade", "bestSell", assetIn, assetOut, amountIn],
    queryFn: async () => {
      const swap = await sdk.api.router.getBestSell(Number(assetIn), Number(assetOut), amountIn)
      // …
```

- Block-scoped reads key on `QUERY_KEY_BLOCK_PREFIX` ([[wiki/hydration-ui-utils\|hydration-ui-utils]]); `useInvalidateOnBlock()` (`chain.ts`) invalidates them each block.
- Indexer-backed account queries key under `NECKWORK_ACCOUNT_KEY` and are refreshed by `useNeckworkSync` after a tx ([[wiki/hydration-ui-indexer\|hydration-ui-indexer]]).
- Real-time state uses rxjs via `usePapiValue` / `usePapiEntries` / `useObservableQuery` (`src/hooks/`).
- Portfolio queries are persisted across reloads: `portfolio/persistence.ts` → `setupPortfolioPersistence(queryClient)` (called in `App.tsx`), `shouldDehydratePortfolioQuery`.
- Tx-producing hooks return the extrinsic and hand it to `createTransaction` ([[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]]).

## Domain inventory (`770f868`)

### Account, balances, portfolio
- `account.ts` — `useAccountInfo`, `omnipoolPositionsQuery` / `useAccountOmnipoolPositions`, `omnipoolMiningPositionsQuery` / `useAccountOmnipoolMiningPositions`, `xykMiningPositionsQuery` / `useAccountXykMiningPositions`, `useAccountPermitNonce`, `useAccountPendingPermit`
- `balances/` (split from the old `balances.ts`): `account.hooks.ts` — `useAccountBalances`, `useAccountBalance`, `useAccountBalancesWithPriceByAssetType`; `account.queries.ts` — `accountBalancesQuery`, `useAccountBalanceFilter`, `maxWithdrawAllQuery`; `account.utils.ts` — `mergeBalances`, `watchFilteredAccountBalances`, ERC-20 balance snapshots, `withMaxWithdraw`; `utils.ts` — `nativeTokenLocksQuery`, `tokenReservesQuery`, `tokenBalanceQuery`, `HDXStakingBalanceQuery`, `HDXIssuanceQuery`, `fetchHydrationRegistryAssetAmounts`
- `portfolio/` — `useMultichainPortfolio` (`MultichainValuedBalance`, balances across tracked wallets/chains), persistence helpers, `portfolioBalanceQueryKey`
- `locks.ts` — `useUnlockableNativeTokens`
- `assets.ts` — `assetsQuery` (feeds `AssetRegistryGate`); `metadata.ts` — `assetMetadataQuery`, `useAssetMetadata`

### Trading
- `trade.ts` — `bestSellQuery`, `bestSellTxQuery`, `bestSellWithTxQuery`, `bestSellTwapQuery`, `bestBuyQuery`, `bestBuyTxQuery`, `bestBuyWithTxQuery`, `dcaOrderQuery`, `minimumOrderBudgetQuery`, `tradeOrderDurationQuery`; `TradeType`, `TradeOrderError`
- `intents.ts` ([[wiki/ice\|ice]]) — `useAccountIntents`, `maxIntentDurationQuery`, `getIntentDeadline`
- `omnipool.ts` — `useOmnipoolAssetsData`, `useMaxAddLiquidityLimit`, `useOmnipoolMinLiquidity`, `useMinWithdrawalFee`, `useAssetFeeParameters`, `useOraclePrice`, `useTotalOmnipoolLiquidity`
- `spotPrice.ts` — `usePriceSubscriber`, `spotPriceQuery`, `getSpotPrice`, `spotPriceKeyQuery`, `scSpotPriceKeyQuery`, `useSubscribedPriceKeys`, `useShareTokenPrices`
- `stableswap.ts` — `useStableswap`, `useStableSwapTradability`; `xyk.ts` — `xykPoolWithLiquidityQuery`, `useXYKConsts`, `useXykShareTokenEntries`; `otc.ts` — `otcExistentialDepositorMultiplierQuery`

### Liquidity, pools, vaults
- `pools.ts` — `PoolType`, `allPools`, `stablePoolsQuery`, `hubTokenQuery`, `omnipoolTokensQuery`, `xykPoolQuery`, `useXykPools`, `useStablePools`, `useOmnipoolIds`, **Uniswap V3:** `useV3Pools`, `v3LpFeeShare`, `V3PoolBase`, `V3Tick`
- `gamma/` — Gamma-managed V3 vaults: `GAMMA_STACKS`, `GAMMA_BOOTSTRAP_HYPERVISOR`, `loadBootstrapV3Pools`, `vaultIdentityQuery`, `useVaultSnapshots`, `useVaultShares`, `useVaultDepositChecks`, `estimateVaultCallGas`, ABIs (`HYPERVISOR_ABI`, `UNIPROXY_ABI`, …)
- `farms.ts` — `useOmnipoolFarms`, `useIsolatedPoolsFarms`, `useOmnipoolActiveFarm`, `useFarmRewards`, `useNewFarms`
- `bonds.ts` — `useBondData`, `useRedeemBond`

### Lending
- `aave.ts` — `healthFactorQuery`, `healthFactorAfterSupplyQuery` / `AfterWithdrawQuery` / `AfterSwapQuery`, `aaveSummaryQuery`, `maxWithdrawQuery`, `useTransfarebleATokenBalance`, `AAVE_GAS_LIMIT`
- `borrow/queries.ts` — `lendingPoolAddressProvider`, `gigaLendingPoolAddressProvider`, `borrowReserve(s)Query`, `userBorrowReservesQuery`, `borrowIncentivesQuery`, `borrowUserIncentivesQuery`, `ghoReserveDataQuery`, `ghoUserDataQuery`, `userBorrowSummaryQuery`, `useUserGigaBorrowSummary`, `facilitatorBucketQuery`, `useGetClaimAllBorrowRewardsTx`, `convertEvmTxRawToPapiTx`, `estimateGasLimit`; `borrow/contracts.ts` — pool / data / incentives / GHO contract hooks, `gigaBorrowPoolContractQuery`; `borrow/hooks.ts` — `useBorrowAssetsApy`, `useBorrowClaimableRewards`, `useApproveErc20`; `borrow/moneyMarketEvents.ts` — `useMoneyMarketEvents` (Neckwork history)

### Staking & governance
- `gigaStake.ts` — `gigaStakeConstantsQuery`, `gigaUnstakePositionsQuery`, `gigaAccountStakesQuery`, `gigaAccountBalanceQuery`, `gigaTotalLockedQuery`, `gigaHDXIssuanceQuery`, `gigapotBalanceQuery`, `useGigaStakeExchangeRate`, `getCooldownExpiresAt`, `gigaTwoSecBlocksSinceQuery`, `referendaRewardPoolQuery`, `referendaTotalWeightedVotesQuery`, `claimableVotingRewardsQuery`, `accumulatorPotBalanceQuery`, `gigaRewardPoolEstimateQuery`, `STAKE_GIGAPOT_ADDRESS`, `REWARD_ACCUMULATOR_POT_ADDRESS` → [[wiki/gigahdx\|gigahdx]]
- `gigaApr.ts` — `gigaAprQuery` / `useGigaApr` now **fetch from Neckwork** (`hydration-api.neckwork.net/v1/staking/gigahdx/apr` → `{ base, voting, total }`); the on-chain 28-day-baseline computation is gone.
- `staking.ts` — legacy HDX staking (`stakeQuery`, `stakingPositionsQuery`, `stakingRewardsQuery`, `processedVotesQuery`, `pendingVotesQuery`, `potBalanceQuery`, `subscanHDXSupplyQuery`) — serves `/staking-old` only
- `democracy.ts` — OpenGov (conviction-voting/referenda pallets, despite the filename): `ongoingReferendaQuery`, `referendumInfoQuery`, `accountOpenGovVotesQuery`, `accountUnlockClassesQuery`, `openGovUnlockedTokensQuery`, `voteLockingPeriodQuery`, `CONVICTIONS`, `VOTE_WEIGHT_BY_CONVICTION`, `REWARD_MULTIPLIER_BY_CONVICTION`
- `constants.ts` — `stakingConstsQuery`, `referendaTracksQuery`, `insufficientFeeQuery`, `uniquesIds`

### Cross-chain
- `xcm.ts` — `useCrossChainConfig`, `useCrossChainConfigService`, `useCrossChainWallet`, `useHydrationAssetId`, `useHydrationDisplayAssetId`, `useCrossChainBalance`, `useCrossChainBalancesFetch`, `useCrossChainBalanceSubscription`, `xcmTransferQuery`, `xcmTransferCallQuery`, `xcmTransferReportQuery`, `xcmDestinationFeeQuery`; limits: `useNttOutboundLimit`, `useNttInboundLimit`, `useNttCustody`, `useBasejumpPoolLiquidity`, `useCrossChainDepositLimit`, `useCrossChainGlobalWithdrawLimit` ([[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]])
- `scan.ts` — `scanJourneysQuery` (transfers scan); `wormhole.ts` — `wormholeVaaQuery`; `near.ts` — `nearAccountQuery`

### Chain, fees, accounts
- `payments.ts` — fee-payment asset hooks, `getEvmAccountClaimMessage`; `transaction.ts` — `paymentInfoQuery`; `evm.ts` — `evmAccountBindingQuery`, `useBindEvmAccount`, `usePermitNonce`, `usePendingPermit`, `weightToEvmFeeQuery`, `useErc20Allowance`; `dryRun.ts` — `papiDryRunErrorQuery`; `errors.ts` — `parseChainErrorState`, `decodePjsErrorQuery`; `multisig.ts` — `useMultisigDeposit`, `useDecodedMultisigTx`, `useMultisigSignerBalance`; `referrals.ts` — `accountReferralSharesQuery`

### Analytics & external
- `stats.ts` — `useFeesChartsData`, `VIEW_MODES` (Neckwork `feesChartQuery`)
- `grafana/` — `fetchGrafana`, `reserveRateChartDataQuery`, `supplyRateChartDataQuery`, `variableBorrowRateChartDataQuery`
- `external/` — `assethub.ts`, `common.ts` (`assethub_cex` fee location), `defillama.ts`, `kamino.ts`, `kraken.ts` (`krakenOhlcQuery`, `useKrakenOhlc`, `useKrakenSpotPrice` — foreign-chain price reference for the cross-chain swap chart / provider), `pendulum.ts`, `subsquare.ts` (`accountVotesQuery`)

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hydration-ui-utils\|hydration-ui-utils]], [[wiki/sdk-next\|sdk-next]], [[wiki/papi\|papi]], [[wiki/xc-core\|xc-core]], [[wiki/gigahdx\|gigahdx]], [[wiki/ice\|ice]]
