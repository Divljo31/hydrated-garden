---
{"dg-publish":true,"permalink":"/wiki/xc-core/","title":"xc-core","tags":["sdk","cross-chain","types","definitions","ntt","balances"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-core","repo":"sdk","paths":["packages/xc-core/src/chain/Chain.ts","packages/xc-core/src/chain/balance","packages/xc-core/src/bridge/ntt.ts","packages/xc-core/src/bridge/wormhole.ts","packages/xc-core/src/config/definition/AssetRoute.ts","packages/xc-core/src/config/definition/ChainRoutes.ts","packages/xc-core/src/config/ConfigBuilder.ts","packages/xc-core/src/evm/EvmCall.ts","packages/xc-core/src/evm/abi"],"key_exports":["Chain","Parachain","EvmChain","EvmParachain","SolanaChain","SuiChain","Asset","AssetAmount","AssetRoute","ChainRoutes","ConfigService","ConfigBuilder","Ntt","NttTokenDef","Wormhole","Snowbridge","Basejump","SubstrateBalanceType","EvmBalanceType","SolanaBalanceType","SuiBalanceType","SubstrateMinType","CallType","EvmCallData","EvmPrerequisite","getEvmPrerequisites","ContractConfig","ContractConfigBuilder","TransferValidation","TransferValidationError"],"tags":["sdk","cross-chain","types","definitions","ntt","balances"],"source_count":1,"last_updated":"2026-08-15"}}
---


# xc-core

**TL;DR:** `@galacticcouncil/xc-core` (v2.3.0, papi v2) provides the foundational types for cross-chain transfers — chain models, asset definitions, route configuration and bridge primitives — depended on by every other XC package. **Aug 2026: two structural changes** — balance/min reads moved from route-level builders onto the chain itself, and the MRL/TokenBridge primitives were replaced by NTT.

## Chain Model

**Ecosystems:** Ethereum, Polkadot, Kusama, Solana, Sui
**Chain types:** `Parachain`, `EvmParachain`, `EvmChain`, `SolanaChain`, `SuiChain`

Each chain maps assets with chain-specific ids and decimals, can have a DEX attached (`registerDex()`), tracks minimum deposit amounts, and provides type-checking methods (`isEvm()`, `isSubstrate()`, …). EVM chains can optionally declare bridge integrations: `basejump`, `snowbridge`, `wormhole`.

### Chain-native balance reads (new)

Balance and minimum reads are now **declared on the chain**, not built per route. `Chain` is generic over its platform's balance type and exposes the read API directly:

```ts
// packages/xc-core/src/chain/Chain.ts
export interface ChainParams<T extends ChainAssetData, B extends BalanceType = BalanceType> {
  assetsData: T[];
  /** Default balance storage type for assets on this chain. */
  balance: NoInfer<B>;
  balanceOverrides?: Record<string, NoInfer<B>>;
  // …
}

abstract getBalance(asset: Asset, address: string): Promise<AssetAmount>;
abstract subscribeBalance(asset: Asset, address: string): Observable<AssetAmount>;
getBalanceType(asset: Asset): B  // = balanceOverrides?.[asset.key] ?? balance
getAssetMin(asset: Asset): number
```

`NoInfer<B>` pins `B` to each chain's declared type, so `new Parachain({ balance: EvmBalanceType.Erc20 })` is a compile error rather than a silent widening.

Storage-type enums (`src/chain/balance/types.ts`):

| Enum | Values |
|---|---|
| `SubstrateBalanceType` | `System`, `Tokens`, `OrmlTokens`, `Assets`, `ForeignAssets` |
| `EvmBalanceType` | `EvmNative`, `EvmErc20` |
| `SolanaBalanceType` | `SolanaNative`, `SolanaToken` |
| `SuiBalanceType` | `SuiNative` |
| `SubstrateMinType` | `Assets` — optional dynamic-minimum storage; chains with static minimums use `assetsData[*].min` |

Per-platform read implementations live in `src/chain/balance/{substrate,evm,solana,sui}.ts`. Subscriptions retry 3× with 1 s delay (`BALANCE_RETRY`) and coalesce the post-change update burst over a 500 ms window (`BALANCE_DEBOUNCE`), using `rx.debounceAfterFirst` from [[wiki/sdk-common\|sdk-common]]. `subscribeBalances()` isolates a dead stream so one bad asset doesn't error the whole combined observable.

Design + migration spec: `packages/xc/docs/chain-native-balances.md`, `packages/xc/docs/unidirectional-routes.md`.

## Asset Model

Assets have a unique `key` (e.g. `'eth'`, `'dai'`) and an `originSymbol` (e.g. `'ETH'`). `AssetAmount` extends this with `amount` (bigint), decimals, and origin chain reference.

## Route Configuration

Routes are hierarchical: `ChainRoutes` → `AssetRoute[]`, keyed `${sourceAsset.key}-${destChain.key}-${destAsset.key}`. `AssetRoute` shrank considerably:

```ts
// packages/xc-core/src/config/definition/AssetRoute.ts
export interface SourceConfig {
  asset: Asset;
  // Optional fee-asset override. When unset, the destination fee is paid in
  // `destination.fee.asset`. Balance/min are read from the chain registry.
  destinationFee?: Asset;
  fee?: FeeConfig;
}
export interface AssetRouteParams {
  source: SourceConfig;
  destination: DestinationConfig;      // { chain, asset, fee }
  contract?: ContractConfigBuilder;    // EVM
  extrinsic?: ExtrinsicConfigBuilder;  // Substrate
  move?: MoveConfigBuilder;            // Sui
  program?: ProgramConfigBuilder;      // Solana
  tags?: string[];
}
```

**Removed from `AssetRoute`:** `source.balance`, `source.min`, the `source.destinationFee` *builder*, and `transact` (the intermediate-chain hop that MRL needed). `BalanceConfigBuilder` / `MinConfigBuilder` / `SubstrateQueryConfig` / `SolanaQueryConfig` / `SuiQueryConfig` are deleted.

`ChainRoutes.getAssetDestinationRoutesOrEmpty(asset, destination)` is new — probes for a **reverse** route without using exceptions for control flow, which is what backs `Transfer.reversible` in [[wiki/xc-sdk\|xc-sdk]].

`ConfigBuilder.build(assetOnDest, tag?)` still disambiguates when several routes share a key.

## Bridge Primitives

`src/bridge/` — `Wormhole`, `Snowbridge`, `Basejump`, and the new `Ntt`.

```ts
// packages/xc-core/src/bridge/ntt.ts
export type NttTokenDef = {
  /** Token address: erc20 contract, spl mint or sui coin type */
  token: string;
  /** NttManager: contract address, program id or sui state object id */
  manager: string;
  transceiver: { wormhole: string };
  /** VAA emitter, when different from the transceiver address (Solana pda) */
  emitter?: string;
};
export type NttDef = Record<string, NttTokenDef>;

class Ntt {
  static find(chain, assetKey): NttTokenDef | undefined;
  static findByEmitter(chain, emitter): { assetKey; def } | undefined;
  static fromChain(chain, asset): NttTokenDef;   // throws when unregistered
  static isKnown(chain, asset): boolean;
}
```

Address comparison is case-insensitive for `0x…` and exact for base58 (Solana). The per-chain registry lives inline on each chain def under `wormhole.ntt` in [[wiki/xc-cfg\|xc-cfg]].

**Deleted with MRL:** `utils/mrl.ts` (+ spec), `evm/abi/Gmp.ts`, `evm/abi/TokenBridge.ts`, `evm/abi/TokenRelayer.ts`.
**Added:** `evm/abi/{NttManager,NttManagerWithExecutor,WormholeTransceiver,Executor}.ts`, `evm/EvmCall.ts`, `utils/{codec,solana,sui}.ts`.

## Contract config: ordered calls + prerequisites

`ContractConfigBuilder.build()` now returns **`Promise<ContractConfig[]>`** (was a single config) — one route emits an ordered call vector, e.g. `[NttManager.transfer(...), Executor.requestExecution(...)]`. `EvmPlatform` signs them in order; `SubstrateEvm` batches the lot ([[wiki/xc-sdk\|xc-sdk]]).

`ContractConfig` gained `token` (explicit allowance target when not derivable from `args[0]`) and `wrapNative`. Account-dependent prerequisites are derived separately in `src/evm/EvmCall.ts`:

```ts
// packages/xc-core/src/evm/EvmCall.ts
export async function getEvmPrerequisites(
  client: EvmClient, owner: string, amount: bigint, config: ContractConfig
): Promise<EvmPrerequisite[]>   // ordered: [WETH.deposit?, erc20.approve?]
```

Ordered and self-shortening: a native source is wrapped into the erc20 the contract pulls, which is then approved for the **exact** amount; each step drops off once executed, so re-deriving after signing yields a shorter sequence. Precompile targets and native-ETH bridges (Snowbridge V1 zero-address `sendToken`, V2 `v2_sendMessage` with an empty assets array) need nothing. Gas ceilings are split into `Gas` (fat `EVM.call` declarations: `approve` 200k, `deposit` 200k, `transfer` 1.2M, `redeem` 2M, `queued` 600k) and `FeeGas` (realistic bounds used only for fee estimation: `deposit` 60k, `approve` 70k, `transfer` 500k).

> `packages/xc/docs/ntt.md` describes `ContractConfig.prior` / `.follow` slots — those fields do **not** exist at HEAD; ordering is the array plus `getEvmPrerequisites()`.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
