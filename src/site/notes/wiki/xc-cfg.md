---
{"dg-publish":true,"permalink":"/wiki/xc-cfg/","title":"xc-cfg","tags":["sdk","cross-chain","configuration","routing","validation","ntt","wormhole","snowbridge"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-cfg","repo":"sdk","paths":["packages/xc-cfg/src/index.ts","packages/xc-cfg/src/assets.ts","packages/xc-cfg/src/tags.ts","packages/xc-cfg/src/chains","packages/xc-cfg/src/configs","packages/xc-cfg/src/builders","packages/xc-cfg/src/bridges","packages/xc-cfg/src/clients/ntt","packages/xc-cfg/src/clients/wormhole","packages/xc-cfg/src/resolvers","packages/xc-cfg/src/validations"],"key_exports":["assetsMap","chainsMap","routesMap","validations","builders","clients","dex","tags","HydrationConfigService"],"tags":["sdk","cross-chain","configuration","routing","validation","ntt","wormhole","snowbridge"],"source_count":1,"last_updated":"2026-08-15"}}
---


# xc-cfg

**TL;DR:** `@galacticcouncil/xc-cfg` (v2.3.0) is the **data + builder layer** of the XC stack — concrete chain / asset / route definitions, DEX integrations for fee conversion, contract/extrinsic/program/move builders per bridge, and the transfer-validation framework. **Aug 2026: the legacy Wormhole TokenBridge / TokenRelayer / MRL stack was ripped out and replaced by Wormhole NTT** (Native Token Transfers), which removed the Moonbeam hop entirely.

> 131 files changed since the May 2026 sync — the single biggest churn in the repo. It is a **bridge-model replacement**, not a chain expansion: 5 chains deleted, 1 added.

## Headline change: MRL → NTT

| Before (≤ May 2026) | After (Aug 2026) |
|---|---|
| Wormhole via **MRL** — Hydration → Moonbeam → Ethereum, XCM `Transact` payload | Wormhole **NTT** — Hydration ↔ Ethereum / Base / Solana / Sui **direct** |
| `builders/contracts/Wormhole/TokenBridge.ts`, `TokenRelayer.ts` | `builders/contracts/Wormhole/Ntt.ts` (+ `programs/`, `moves/` twins) |
| `xc-core/utils/mrl.ts`, `evm/abi/{TokenBridge,TokenRelayer,Gmp}.ts` | `xc-core/bridge/ntt.ts`, `evm/abi/{NttManager,NttManagerWithExecutor,WormholeTransceiver,Executor}.ts` |
| `Tag.Mrl` | `Tag.Ntt`, `Tag.NttExecutor` |
| assets keyed `*_mwh` (moonbeam-wormhole) | assets keyed `*_wh` |
| `AssetRoute.transact` (intermediate-chain hop) | **removed** — see [[wiki/xc-core\|xc-core]] |
| `HydrationMrlFeeValidation` (GLMR balance) | **removed**; `NttRateLimitValidation` added |

Hydration is now a first-class Wormhole chain (wormhole id `73`, EVM chain id `222222`, registered in `@wormhole-foundation/sdk-base`, pinned at exactly `6.1.4` across xc-core). Long-form reference: `packages/xc/docs/ntt.md`.

### NTT model

No shared bridge contract — each token has its own deployment per chain:

- **NttManager** — locks/burns on source, mints/unlocks on destination; holds the per-direction rate limits.
- **WormholeTransceiver** — emits/verifies the VAA. **The VAA emitter is the transceiver**, not the manager (on Solana a PDA of it, on Sui its `EmitterCap` object id — declared explicitly via the optional `emitter` field).

Registry lives inline in each chain def under `wormhole.ntt`, keyed by asset key, typed as `NttTokenDef` in [[wiki/xc-core\|xc-core]]. Lookups: `Ntt.fromChain()`, `Ntt.isKnown()`, `Ntt.find()`, `Ntt.findByEmitter()`.

**Registry contract:** one asset key per token across all chains — `WormholeTransfer` resolves the destination deployment with the *source* asset key, so chain-local key variants would silently break the redeem callback. Chain-side cross-check: `pallet-evm-accounts` keeps `NttMinters: StorageMap<AssetId, EvmAddress>`.

### NTT deployments (as configured)

| Chain | wormhole id | NTT assets |
|---|---|---|
| Hydration | 73 | `dai_wh`, `eurc_wh`, `jito_sol`, `prime`, `sol`, `sui`, `susds_wh`, `usdc_wh`, `usdt_wh`, `wbtc_wh`, `weth_wh` — **burning** managers; `token` = the erc20 precompile of the asset id |
| Ethereum | 2 | `dai`, `susds`, `usdc`, `usdt`, `wbtc`, `weth`, `eth` — locking. `eth` shares the `weth` deployment (the manager only locks the erc20, so a native source is wrapped upfront via `ContractConfig.wrapNative`); registered **after** `weth` so a VAA from the shared transceiver resolves back to the erc20 key |
| Base | 30 | `eurc` — locking |
| Solana | 1 | `sol`, `wsol` (same deployment, keyed separately), `jito_sol`, `prime` |
| Sui | 21 | `sui` — locking; `manager` / `transceiver.wormhole` are State **object ids**, `emitter` is the transceiver's `EmitterCap` object id |

### Configured NTT routes (`configs/polkadot/hydration/index.ts`)

| Direction | Pairs |
|---|---|
| Hydration → Ethereum | `dai_wh→dai`, `susds_wh→susds`, `usdc_wh→usdc`, `usdt_wh→usdt`, `wbtc_wh→wbtc`, `weth_wh→eth` |
| Hydration → Base | `eurc_wh→eurc` |
| Hydration → Solana | `sol→sol`, `jito_sol→jito_sol`, `prime→prime` (the executor variant sends `sol→wsol`) |
| Hydration → Sui | `sui→sui` |

Every pair above is registered twice — plain (`viaNttTemplate`) and executor (`viaNttExecutorTemplate`). Reverse legs live in `configs/evm/{ethereum,base}`, `configs/solana`, `configs/sui/sui.ts`. Sui is wired **both ways at HEAD**, though `packages/xc/docs/ntt.md` still describes those two route lines as commented out — the doc lags the config.

### Delivery: self-redeem vs Executor

Each NTT pair carries **two** routes, picked by tag (same shape as the Snowbridge V1/V2 pair):

- `Tag.Wormhole + Tag.Ntt` — plain `NttManager.transfer`; **self-redeem**, the user completes via `receiveMessage` on the destination.
- `Tag.Wormhole + Tag.Ntt + Tag.NttExecutor` — the sender pays `deliveryPrice + estimatedCost` and the Wormhole Executor service delivers.

**The executor route is built two different ways depending on source chain** — `ContractBuilder().Wormhole().Ntt()` exposes both:

| Builder | Used by | Shape |
|---|---|---|
| `transferWithExecutor()` | Ethereum / Base templates (`configs/evm/*/templates.ts`) | one call into the `NttManagerWithExecutor` shim (selector `0xce972e0e`), which pulls the erc20 and forwards |
| `transferViaExecutor()` | **Hydration** template (`viaNttExecutorTemplate`) | two calls the sender owns: `NttManager.transfer` (value = `deliveryPrice`) then `Executor.requestExecution` (value = `estimatedCost`), with an exact-amount approve as prerequisite |

**Why Hydration can't use the shim:** `NttManagerWithExecutor` approves the manager for `type(uint256).max`, and Hydration's erc20 precompile carries a **u128** balance — anything above `uint128` max reverts with `"value too big for type"`. Inside `EVM.call` the revert is swallowed and surfaces as `Utility.BatchCompleted` over an `EVM.ExecutedFailed`; `allowance(shim → manager)` on Hydration has never been non-zero.

Two distinct addresses per chain def: `wormhole.executor` (the Executor service) and `wormhole.nttExecutor` (the shim). The Hydration executor template also prices its destination fee as `FeeAmountBuilder().Wormhole().quoteExecutorCost()` denominated in `weth_wh`, and decorates the extrinsic path with a fee swap (`ExtrinsicDecorator(isDestinationFeeSwapSupported, swapExtrinsicBuilder).prior(...)`).

`Executor.requestExecution` is generic — it moves no tokens and knows nothing about NTT — so `requestBytes` (`encodeNttRequest`, 70-byte `ERN1` layout) is what names the message. The sequence comes from `nextMessageSequence()` read at build time: **another transfer through the same manager landing in between leaves the request pointing at the wrong message**, nothing is relayed, and the transfer stays claimable by hand.

EVM amounts are floored to NTT wire precision (`TrimmedAmount`, `min(8, decimals)` dp) or the manager reverts with `TransferAmountHasDust`. Sui trims internally and hands the remainder back.

## Chain inventory

`chainsMap` (`src/chains/`). **Removed** this cycle: `moonbeam`, `interlay`, `crust`, `ajuna`, `laos`. **Added:** `assethub_cex`.

| Ecosystem | Chains |
|---|---|
| Polkadot | `hydration`, `assethub`, **`assethub_cex`** (new), `astar`, `bifrost`, `energywebx`, `mythos`, `neuroweb`, `pendulum`, `polkadot`, `unique` |
| Kusama | `assethub_kusama`, `basilisk` |
| EVM | `ethereum`, `base` |
| Solana | `solana` |
| Sui | `sui` |

`assethub_cex` is defined at the bottom of `src/chains/polkadot/assethub.ts` — it shares AssetHub's config but is `isTestChain`, `usesSignerFee`, and clears `balanceOverrides` (CEX forwarding reads *every* asset incl. DOT from the `assets` pallet). It reuses AssetHub's DEX via the new `dexAlias` map in [[wiki/xc-package\|xc-package]] (`packages/xc/src/dex.ts`). Routes into it are the `toCex` block on the Hydration config: `dot`, `usdt`, `usdc` via `toHubTemplate`.

RPC: Bifrost's IBP endpoint (`wss://bifrost-polkadot.ibp.network`) dropped in favour of `wss://hk.p.bifrost-rpc.liebi.com/ws`.

## Asset key rename

All Moonbeam-wormhole assets renamed and multilocations updated:

`dai_mwh → dai_wh`, `eurc_mwh → eurc_wh`, `susds_mwh → susds_wh`, `usdc_mwh → usdc_wh`, `usdt_mwh → usdt_wh`, `wbtc_mwh → wbtc_wh`, `weth_mwh → weth_wh`.

New: `apyusd`, `wsol` (Executor delivery stops at the ATA and leaves SOL wrapped, so that route names `wsol` as what arrives). Removed: `laos`.

## Tags

```ts
// packages/xc-cfg/src/tags.ts
export enum Tag {
  Basejump = 'Basejump',
  Ntt = 'Ntt',
  NttExecutor = 'NttExecutor',
  Wormhole = 'Wormhole',
  Relayer = 'Relayer',
  Snowbridge = 'Snowbridge',
  SnowbridgeV1 = 'SnowbridgeV1',
}
```

`ConfigBuilder.build(assetOnDest, tag?)` still selects among multiple routes sharing a `sourceAsset-destChain-destAsset` key.

## Route templates

`src/configs/polkadot/hydration/templates.ts`, `src/configs/evm/{ethereum,base}/templates.ts`:

| Template | Tags |
|---|---|
| `viaNttTemplate` | `Wormhole`, `Ntt` |
| `viaNttExecutorTemplate` | `Wormhole`, `Ntt`, `NttExecutor` |
| `viaSnowbridgeTemplate` | `Snowbridge` (V2) |
| `viaSnowbridgeV1Template` | `Snowbridge`, `SnowbridgeV1` |
| `toTransferTemplate`, `toParaTemplate`, `toHubTemplate`, `toHubExtTemplate`, `toKusamaHubTemplate`, `toParaErc20Template` | plain XCM |

Snowbridge is unchanged in shape but now runs V1 and V2 side by side: V2 routes (`eth`, `aave`, **`apyusd`** (new), `cfg_new`, `ena`, `paxg`, `susde`, `tbtc`, `trac`, `lbtc`, `ldo`, `link`, `sky`, `wsteth`, `usdc_eth`, `usdt_eth`) are registered **first** so default / `Tag.Snowbridge` selection resolves to V2; the cheaper legacy V1 route for the same pairs is reachable only via `Tag.SnowbridgeV1` (a UI switch). Note `usdc_eth`/`usdt_eth` (Snowbridge) and `usdc_wh`/`usdt_wh` (NTT) are **different asset keys for the same underlying token** — the destination asset key, not the chain pair, is what picks the bridge.

### Dual signer origin

NTT / Snowbridge templates declare **both** a `contract` and an `extrinsic`:

```ts
// packages/xc-cfg/src/configs/polkadot/hydration/templates.ts → viaNttTemplate
const transfer = ContractBuilder().Wormhole().Ntt().transfer();
return new AssetRoute({
  source: { asset: assetIn, fee: fee() },
  destination: { chain: to, asset: assetOut, fee: { amount: 0, asset: assetIn } },
  contract: transfer,
  extrinsic: ExtrinsicBuilder().evm().call(transfer), // new: src/builders/extrinsics/evm.ts
  tags: [Tag.Wormhole, Tag.Ntt],
});
```

`DataOriginProcessor` ([[wiki/xc-sdk\|xc-sdk]]) hands the `contract` to an h160 signer and the `extrinsic` to everyone else. The extrinsic path wraps the EVM calls in `EVM.call` + `Utility.batch_all` — one signature, fee quoted by the runtime in the sender's fee currency. Requires the account to be **bound** on chain (`EVMAccounts.bind_evm_address`); otherwise `EnsureAddressTruncated` resolves to the unrelated `ETH\0` phantom account. `HydrationEvmResolver` (`src/resolvers/hydration.ts`) throws rather than deriving silently.

Contract builders now return an **ordered vector** of configs (`ContractConfigBuilder.build → Promise<ContractConfig[]>`) — `[manager.transfer]` for self-redeem, `[manager.transfer, executor.requestExecution]` for the executor bypass.

## Clients

`src/clients/` (exported as `clients`):

| Client | Purpose |
|---|---|
| `nttClient(chain, asset?)` → `NttEvmClient` / `NttSolanaClient` / `NttSuiClient` | `getOutboundLimit()`, `getInboundLimit(from)`, `getRedeemBudget(recipient?)` — the `recipient` arg only matters on Solana, where an unopened ATA changes the budget |
| `ExecutorClient` (`clients/wormhole/`) | signed delivery quotes (`signedQuote`, `estimatedCost`, `relayInstructions`), `ExecutorBudget { gasLimit, msgValue }`, `QUOTE_SAFETY_MS = 5 min` quote-reuse window |
| `HydrationClient` (`clients/chain/hydration/`) | ED, token balances, circuit-breaker limits (unchanged) |
| `AssethubClient` | AssetHub ED / frozen-asset state |

`NttRateLimit { capacity, limit, windowMs, capacityAtLastTx, lastTxMs }`; `windowMs === 0` means the direction is unmetered (`UNMETERED` sentinel); `capacityAt()` mirrors the manager's linear refill (`rate_limit::capacity_at`). Sources per platform:

- **EVM** — `rateLimitDuration()`, `getCurrent{Out,In}boundCapacity()`, `get{Out,In}boundLimitParams()` (packed `TrimmedAmount`s needing untrimming).
- **Solana** — `outbox_rate_limit` / inbound rate-limit PDAs fetched with `getAccountInfo` and decoded locally at a fixed offset, because upstream returns the un-refilled stored value.
- **Sui** — manager `State.fields.outbox.rate_limit`, inbound via the `peers` table (`inbound_rate_limit`).

## Bridge constants

New `src/bridges/`:

- `bridges/wormhole/constants.ts` — `NTT_DEFAULT_INSTRUCTIONS = '0x00'` (must match what the fee builder quoted), `NTT_TRIMMED_DECIMALS = 8`, `EXECUTOR_API = 'https://executor.labsapis.com'`, `encodeNttRequest(srcWormholeId, nttManager, sequence)` (70-byte `ERN1` Executor request layout, hand-written because the wormhole sdk ships the prefix enum but no body layout).
- `bridges/snowbridge/constants.ts` — `SNOWBRIDGE_BASE_DISPATCH_GAS = 80_000n`, `SNOWBRIDGE_BASE_VERIFICATION_GAS = 120_000n`, `SNOWBRIDGE_TOKEN_DELIVERY_GAS = 100_000n`, `SNOWBRIDGE_SUBMIT_GAS = 2_000_000n`, `ASSETHUB_ETHER_ED = 15_000_000_000_000n`.

Snowbridge builders moved to `builders/contracts/snowbridge/` with a dedicated `codec.ts`; the V2 XCM path adds `extrinsics/xcm/builder/Snowbridge.ts` and `buildParaERC20ReceivedV5.ts`.

## Transfer validations

`src/validations/` exports `validations: TransferValidation[]`, consumed by [[wiki/xc-sdk\|xc-sdk]]'s `TransferValidator`. Matchers unchanged (`isAny`, `isHydration` = para 2034, `isHub` = para 1000, `isEvm`).

| Validator | Source → Dest | Purpose |
|---|---|---|
| `FeeValidation` | any → any | source fee balance |
| `DestFeeValidation` | any → any | destination fee asset balance |
| `HubEdValidation` | any → hub | AssetHub ED |
| `HubFrozenValidation` | hub → any | AssetHub frozen-asset gating |
| `HydrationEdValidation` | any → hydration | Hydration ED via `HydrationClient` |
| `HydrationDepositLimitValidation` | any → hydration | `CircuitBreaker.AssetLockdownState` + per-asset deposit headroom |
| `HydrationWithdrawLimitValidation` | hydration → any | global withdraw lockdown |
| `NttRateLimitValidation` | any → any | **NEW** — outbound (source) + inbound (destination) NTT rate limits |
| ~~`HydrationMrlFeeValidation`~~ | — | **REMOVED** with Moonbeam |

`NttRateLimitValidation` skips unless **both** ends are NTT-registered — the chain pair alone doesn't identify an NTT route, only the destination asset key does (`usdc_wh` = NTT vs `usdc_eth` = Snowbridge, same source asset). Errors: `Ntt_Outbound_Limit_Exceeded`, `Ntt_Inbound_Limit_Exceeded`, `Ntt_Limit_Unreachable` (fails closed — an over-limit *inbound* transfer is already burned on source and parks in the destination queue with no cancel).

### Circuit-breaker integration (unchanged)

`src/clients/chain/hydration/circuit-breaker.ts` — `getAssetDepositLimit()`, `getGlobalWithdrawLimit()`, `getAllAssetDepositLimits()`, `ASSET_LOCKDOWN_PERIOD_BLOCKS = 14400`. Reads [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] storage via [[wiki/sdk-descriptors\|sdk-descriptors]].

## DEX integrations

`HydrationDex` (`src/dex/hydration.ts`) and `AssethubDex`, used for destination-fee asset conversion via the [[wiki/smart-order-router\|smart-order-router]] from [[wiki/sdk-next\|sdk-next]] (peer `>=1.5.0`). Fallback pricing via `MultiTransactionPayment.AcceptedCurrencies` — bug fixed this cycle: the lookup now resolves `chain.getAssetId(asset)` and passes `Number(id)`, instead of querying with the `Asset` object.

## Public exports

```ts
// packages/xc-cfg/src/index.ts
export { assetsMap } from './assets';
export { chainsMap } from './chains';
export { routesMap } from './configs';
export { validations } from './validations';
export * as builders from './builders';
export * as clients from './clients';
export * as dex from './dex';
export * as tags from './tags';
export { HydrationConfigService } from './configs/HydrationConfigService';
```

`BalanceBuilder` / `AssetMinBuilder` and `src/builders/balance/*` were **deleted** — balance and minimum reads moved onto the chain itself in [[wiki/xc-core\|xc-core]] (`chain.getBalance()` / `subscribeBalance()`), declared per chain via `balance:` + `balanceOverrides:`. See the "Chain-native Balance Reads" and "Unidirectional Route Resolution" specs under `packages/xc/docs/` for the design and the phased migration.

## Reference docs (new, in `packages/xc/docs/`)

| Doc | Covers |
|---|---|
| `ntt.md` (~500 lines) | the definitive NTT reference — model, registry, transfer flow, signer paths, rate limits, tracking/claim, executor vs self-redeem, support matrix, open items |
| `overview.md` | XC stack architecture, route DSL, package layouts, navigation cheatsheet |
| `chain-native-balances.md` | declarative balance/min model, per-platform typing, adding a chain |
| `unidirectional-routes.md` | spec for one-way routes + the `reversible` flag on the Transfer DTO |

## Dependencies

`@galacticcouncil/xc-core
{ #2}
.3.0`, `@thi.ng/memoize`; peer `@galacticcouncil/sdk-next >=1.5.0`.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
