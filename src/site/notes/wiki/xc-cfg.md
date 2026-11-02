---
{"dg-publish":true,"permalink":"/wiki/xc-cfg/","title":"xc-cfg","tags":["sdk","cross-chain","xcm","wormhole","ntt","snowbridge","omni","near","zcash","typescript"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-cfg","repo":"sdk","paths":["packages/xc-cfg/src/index.ts","packages/xc-cfg/src/chains","packages/xc-cfg/src/configs","packages/xc-cfg/src/configs/polkadot/hydration/index.ts","packages/xc-cfg/src/configs/polkadot/hydration/templates.ts","packages/xc-cfg/src/builders","packages/xc-cfg/src/bridges","packages/xc-cfg/src/clients","packages/xc-cfg/src/validations","packages/xc-cfg/src/tags.ts","packages/xc/docs"],"key_exports":["assetsMap","chainsMap","routesMap","validations","builders","clients","dex","tags","HydrationConfigService","Tag","viaNttTemplate","viaNttExecutorTemplate","viaOmniTemplate","viaSnowbridgeTemplate","nttClient","ExecutorClient","OmniClient","OMNI_CONNECTORS","NttRateLimitValidation","NttCustodyValidation","OmniValidation","HydrationEvmValueValidation"],"tags":["sdk","cross-chain","xcm","wormhole","ntt","snowbridge","omni","near","zcash","typescript"],"last_updated":"2026-11-02"}}
---


# xc-cfg

**TL;DR:** `@galacticcouncil/xc-cfg` (**v2.9.1**; deps `xc-core
{ #2}
.8.0`, peer `sdk-next >=2.0.0`) is the **data + builder layer** of the XC stack: chain / asset / route definitions, per-bridge contract / extrinsic / program / move / deposit builders, bridge clients, DEX fee conversion and transfer validations. Bridges at d16ff85: **XCM**, **Snowbridge** (V2 default + V1), **Wormhole NTT** (self-redeem + Executor), **Omni** (NEAR One; Zcash UTXO corridor), and one **Basejump** route on Ethereum. Long-form references live in `packages/xc/docs/` (`ntt.md`, `ntt-reorg-recovery.md`, `overview.md`, `chain-native-balances.md`, `unidirectional-routes.md`).

## Chain inventory (`chainsMap`, `src/chains/`)

| Ecosystem | Chain keys |
|---|---|
| Polkadot | `hydration`, `assethub`, `assethub_cex` (test chain, CEX forwarding), `astar`, `bifrost`, `energywebx`, `mythos`, `neuroweb`, `pendulum`, `polkadot`, `unique` |
| Kusama | `assethub_kusama`, `basilisk` |
| EVM | `ethereum` (wh 2), `base` (wh 30), **`hyperevm`** (EVM 999, wh 47), **`robinhood`** (EVM 4663, wh 72) |
| Solana | `solana` (wh 1) |
| Sui | `sui` (wh 21) |
| NEAR | **`near`** (wh 15), `near_testnet` |
| Zcash | **`zec`** (`ZecChain`) |

Hydration itself is wormhole id **73**, EVM chain id 222222.

## Tags (`src/tags.ts`)

`Basejump`, `Ntt`, `NttExecutor`, **`Omni`**, `Wormhole`, `Relayer`, `Snowbridge`, `SnowbridgeV1`. `ConfigBuilder.build(assetOnDest, tag?)` picks among routes sharing a `sourceAsset-destChain-destAsset` key.

## Outbound routes from Hydration (`configs/polkadot/hydration/index.ts`)

| Bridge | Destination | Pairs |
|---|---|---|
| XCM | Asset Hub | `dot`, `ksm`, `usdt`, `usdc` (`toHubTemplate`), `wud` (`toHubExtTemplate`) |
| XCM | Kusama Asset Hub | `ksm`, `dot` (`toKusamaHubTemplate`) |
| XCM | Asset Hub CEX | `dot`, `usdt`, `usdc` |
| XCM | parachains | astar, bifrost (`bnc`, `vdot`, `vastr`, `astr`, `usdt`, `usdc`), mythos, unique (`unq`), energywebx, pendulum, neuroweb (`neuro`) |
| NTT | Ethereum | `dai_wh→dai`, `susds_wh→susds`, `usdc_wh→usdc`, `usdt_wh→usdt`, `wbtc_wh→wbtc`, `weth_wh→eth` |
| NTT | Base | `eurc_wh→eurc` |
| NTT | HyperEVM | `hype→hype` |
| NTT | Robinhood | `hdx`, `hollar`, `spy`, `weth_wh→eth` |
| NTT | Solana | `sol`, `jito_sol`, `prime` (executor variant delivers `sol→wsol`) |
| NTT | Sui | `sui` |
| NTT | NEAR | `near→wnear`, `zec→zec` — **self-redeem only** (no Executor serves NEAR) |
| Omni | Zcash | `zec` via `viaOmniTemplate(zec, zcash, near_chain)` |
| Snowbridge V2 (default) + V1 (`Tag.SnowbridgeV1`) | Ethereum | `eth`, `aave`, `apyusd`, `cfg_new`, `ena`, `paxg`, `susde`, `tbtc`, `trac`, `lbtc`, `ldo`, `link`, `sky`, `wsteth`, `usdc_eth→usdc`, `usdt_eth→usdt` |

Every NTT pair except NEAR is registered twice: `viaNttTemplate` (`Wormhole+Ntt`, self-redeem) and `viaNttExecutorTemplate` (`+NttExecutor`). The **destination asset key picks the bridge**: `usdc_wh` = NTT, `usdc_eth` = Snowbridge, same underlying.

Inbound routes: `configs/evm/{ethereum,base,hyperevm,robinhood}`, `configs/solana`, `configs/sui`, `configs/near` (`toHydrationViaNttTemplate` for `near`, `wnear`, `zec`), `configs/zcash` (Omni deposit), `configs/polkadot/{assethub,astar,bifrost,energywebx,mythos,pendulum}`, `configs/kusama/*`. **Inbound routes were dropped** (Oct 2026) for assets no longer accepted as Hydration fee currency — e.g. KSM from Kusama Asset Hub, and the whole Neuroweb and Unique inbound configs. Basejump survives only as one Ethereum→Hydration USDC route (`configs/evm/ethereum`, `ContractBuilder().Basejump().bridgeViaWormhole()`).

## NTT registry (`wormhole.ntt` per chain def, `NttTokenDef` in [[wiki/xc-core\|xc-core]])

| Chain | Assets | Mode |
|---|---|---|
| Hydration (73) | `dai_wh`, `eurc_wh`, `jito_sol`, `prime`, `sol`, `sui`, `susds_wh`, `usdc_wh`, `usdt_wh`, `wbtc_wh`, `weth_wh`, `hype`, `near`, `spy`, `zec` — **burning**, `token` = erc20 precompile of the asset id; **`hdx`, `hollar` — locking** (Hydration is the hub) |
| Ethereum (2) | `dai`, `susds`, `usdc`, `usdt`, `wbtc`, `weth`, `eth` (shares `weth`) | locking |
| Base (30) | `eurc` | locking |
| HyperEVM (47) | `hype`, `whype` (native wrapped upfront via `wrapNative`) | locking WHYPE |
| Robinhood (72) | `hdx`, `hollar`, `weth`, `eth` — burning (Hydration hub); `spy` — locking (minted on Hydration) |
| Solana (1) | `sol`, `wsol`, `jito_sol`, `prime` | — |
| Sui (21) | `sui` | locking; object ids |
| NEAR (15) | `near` / `wnear` → `ntt-near.hydration-whm.near` (locks `wrap.near`, peered with Hydration asset 1001358); `zec` → `ntt-zec.hydration-whm.near` (locks `zec.omft.near`, Omni's bridged ZEC, peered with asset 1001357) | locking; manager = transceiver; `platformAddressFormat: 'sha256'` |

**One asset key per token across all chains** — the redeem callback resolves the destination deployment by the *source* key. Hydration-side cross-check: `EVMAccounts.NttMinters`. The NEAR managers are deployed from [[wiki/source-whm-codebase\|WHM]].

## Delivery: self-redeem vs Executor

- `Tag.Wormhole + Tag.Ntt` — `NttManager.transfer`; user redeems on destination (`receiveMessage`; NEAR: `complete`).
- `+ Tag.NttExecutor` — sender prepays; Wormhole Executor delivers.

| Builder | Used by | Shape |
|---|---|---|
| `transferWithExecutor()` | Ethereum / Base templates | one call into the `NttManagerWithExecutor` shim |
| `transferViaExecutor()` | Hydration (`viaNttExecutorTemplate`) | `NttManager.transfer` (value `deliveryPrice`) + `Executor.requestExecution` (value `estimatedCost`) |

Hydration can't use the shim: it approves `uint256.max`, Hydration's erc20 precompile holds u128 → revert, swallowed as `EVM.ExecutedFailed` inside `Utility.BatchCompleted`. Executor cost is charged in **WETH (asset 20)** — `EVM.call { value }` debits WETH regardless of signer — so the template prices it with `FeeAmountBuilder().Wormhole().quoteExecutorCost()` in `weth_wh` and prepends a fee swap for ss58 senders with no WETH. `requestBytes` = `encodeNttRequest` (70-byte `ERN1`) with the sequence read at build time — a concurrent transfer through the same manager can make the request point at the wrong message (transfer then stays claimable by hand). Amounts floored to `min(8, decimals)` dp (`TrimmedAmount`).

Dual signer origin: NTT / Snowbridge / Omni templates declare both `contract` (h160 signer) and `extrinsic` (`ExtrinsicBuilder().evm().call(...)` → `EVM.call` + `Utility.batch_all`, needs a **bound** EVM address; `HydrationEvmResolver` throws otherwise).

## Omni / Zcash corridor (`bridges/omni/`, `builders/{contracts,deposits}/Omni/`)

- **Hydration → Zcash:** `viaOmniTemplate` calls the `UtxoEmitter` contract on Hydration EVM (`OMNI_CONNECTORS.zec.withdrawal.emitter = 0xb3e365a6…259e`), which settles over NTT to a NEAR account derived from the Zcash address; `withdraw-zec.hydration-whm.near` signs the withdrawal via the NEAR MPC signer (`v1.signer`, Ed25519 domain). Fee = `FeeAmountBuilder().Omni().withdrawFee(hub)`; miner fee fixed at 10,000 zat (ZIP-317).
- **Zcash → Hydration:** `DepositBuilder().Omni().Ntt().transfer(near_chain)` — Omni mints `zec.omft.near` into the NEAR NTT manager, which publishes to Hydration. Connector `zcash-connector.bridge.near`, `minDeposit` 100,000 zat (smaller deposits are neither minted nor refunded). Failed publishes refund to `hydration.sputnik-dao.near` (DAO refunds by hand).
- API: `OMNI_BRIDGE_API = https://mainnet.api.bridge.nearone.org/api/v3`; `OmniClient.getConfig()`, `getWithdrawFee()`. The deposit message itself is built in [[wiki/xc-swap\|xc-swap]].
- The earlier NEAR **1Click** path for Zcash was dropped from xc-cfg; 1Click remains only as a quote source in [[wiki/xc-swap\|xc-swap]].

## Clients (`src/clients/`)

| Client | Purpose |
|---|---|
| `nttClient(chain, asset?)` → `NttEvmClient` / `NttSolanaClient` / `NttSuiClient` / **`NttNearClient`** | outbound / inbound rate limits, redeem budget, custody |
| `ExecutorClient` | signed delivery quotes; `QUOTE_SAFETY_MS` = 5 min; checks the executor fee currency |
| `OmniClient` | connector config, withdraw fee |
| `HydrationClient` (+ `circuit-breaker.ts`) | ED, balances, [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] deposit / withdraw limits (`ASSET_LOCKDOWN_PERIOD_BLOCKS`) |
| `AssethubClient` | AssetHub ED / frozen assets |

## Transfer validations (`src/validations/`)

| Validator | Purpose |
|---|---|
| `FeeValidation`, `DestFeeValidation` | source / destination fee balance |
| `HubEdValidation`, `HubFrozenValidation` | AssetHub ED / frozen asset |
| `HydrationEdValidation` | Hydration ED |
| `HydrationDepositLimitValidation` | `CircuitBreaker.AssetLockdownState` + deposit headroom |
| `HydrationWithdrawLimitValidation` | global withdraw lockdown |
| **`HydrationEvmValueValidation`** | fee currency must cover the prepaid delivery `value` — Hydration's EVM runner checks it against the fee currency even though WETH pays (`EVM.BalanceLow` otherwise) |
| `NttRateLimitValidation` | outbound + inbound NTT rate limits (only when both ends are NTT-registered) |
| **`NttCustodyValidation`** | a **locking** destination can only release what it holds; unreadable custody fails closed |
| **`OmniValidation`** | connector minimums, emitter fee, NTT leg via the NEAR hub (rate limits + custody); fails closed |

## DEX / resolvers / constants

`dex/{hydration,assethub}.ts` (destination-fee conversion via [[wiki/sdk-next\|sdk-next]] router; `dexAlias` in [[wiki/xc-package\|xc-package]]). `resolvers/hydration.ts` (`HydrationEvmResolver`). `bridges/wormhole/constants.ts` (`NTT_DEFAULT_INSTRUCTIONS`, `NTT_TRIMMED_DECIMALS = 8`, `EXECUTOR_API`, `encodeNttRequest`), `bridges/snowbridge/constants.ts` (gas constants, `ASSETHUB_ETHER_ED`). `testnet/near.ts` — NEAR testnet NTT config.

## Public exports

`assetsMap`, `chainsMap`, `routesMap`, `validations`, `builders`, `clients`, `dex`, `tags`, `HydrationConfigService`.

## Gotchas

- `configs/polkadot/hydration/templates.ts` still carries two unused leftover helpers — ignore anything there with no callers.
- Executor routes need WETH (or a fee swap into it); `HydrationEvmValueValidation` catches the fee-currency side.
- NEAR routes are self-redeem only.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/source-whm-codebase\|source-whm-codebase]] (NEAR NTT managers, `UtxoEmitter`, withdraw contracts)
