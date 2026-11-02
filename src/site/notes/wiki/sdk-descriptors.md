---
{"dg-publish":true,"permalink":"/wiki/sdk-descriptors/","title":"descriptors (SDK)","tags":["sdk","papi","descriptors","metadata","typescript"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"descriptors (SDK)","repo":"sdk","paths":["packages/descriptors/.papi/whitelist.ts","packages/descriptors/.papi/polkadot-api.json","packages/descriptors/.papi/metadata","packages/descriptors/wasm","packages/descriptors/package.json"],"key_exports":["hydration","hydrationNext","hydrationIce","hub"],"tags":["sdk","papi","descriptors","metadata","typescript"],"last_updated":"2026-11-02"}}
---


# descriptors (SDK)

**TL;DR:** `@galacticcouncil/descriptors` (v2.9.0, `polkadot-api
{ #2}
.1.7`) — type-safe papi metadata descriptors for [[hydration]] (`hydration`, `hydrationNext`, `hydrationIce`) and Asset Hub (`hub`), generated from a single whitelist. Built on [[papi-codegen]] / [[papi-typed-api]]. **Edit `.papi/whitelist.ts`** — it is now the only whitelist (the old `src/whitelist.ts` source-of-truth was deleted; `src/index.ts` is an empty stub).

## Generation

- `.papi/whitelist.ts` — `hydrationWhitelist`, `hydrationIceWhitelist`, `hubWhitelist`, plus a `'*'` block applied to every chain (`api.DryRunApi.*`, `api.XcmPaymentApi.*`, `const.System.*`, `query.System.*`, `tx.PolkadotXcm.*`). `hydrationNext` reuses `hydrationWhitelist`.
- `prebuild` = `npm run papi:whitelist` (`papi`); `build` copies `.papi/descriptors/dist` → `build/`. **Do not edit `.papi/descriptors/`.**
- Runtime blobs checked in under `wasm/`: `wasm/ice/ice.wasm` and `wasm/v45.0.0/hydradx_runtime.compact.compressed.wasm`.

## Chain entries (`.papi/polkadot-api.json`)

| Key | Source | Metadata |
|---|---|---|
| `hydration` | live RPC `wss://hydration-rpc.n.dwellir.com` (genesis `0xafdc188f…`) | `hydration.scale` |
| `hydrationNext` | local file only | `hydrationNext.scale` |
| `hydrationIce` | local file only (from `wasm/ice/ice.wasm`) | `hydrationIce.scale` |
| `hub` | `polkadot_asset_hub` | `hub.scale` |

`ICE` / `Intent` are now in the **main** `hydration` whitelist too (`const/query/tx.ICE.*`, `const/query/tx.Intent.*`, `event.Intent.*`) — the pallets ship in the production runtime ([[wiki/pallet-ice\|pallet-ice]], [[wiki/pallet-intent\|pallet-intent]]). `hydrationIce` remains for the ICE-only build.

## Coverage (`hydrationWhitelist`, d16ff85)

- **Runtime APIs:** `AaveTradeExecutor`, `CurrenciesApi`, `EthereumRuntimeRPCApi`, `EvmAccountsApi`, `Metadata`
- **Constants:** Aura, Balances, CircuitBreaker, ConvictionVoting, DCA, DynamicFees, HSM, ICE, Intent, LBP, Multisig, Omnipool, OmnipoolLiquidityMining, OTC, Proxy, Referenda, Stableswap, Staking, XYK, XYKLiquidityMining, GigaHdx
- **Events:** Broadcast, DCA, EVM, Intent, LBP, Omnipool, Proxy, Router, Stableswap, XYK (all `.*` — previously only `EVM.Log`, `Proxy.PureCreated`, `Router.Executed`)
- **Storage:** AssetRegistry, Balances, Bonds, CircuitBreaker, ConvictionVoting, DCA, Democracy, DynamicFees, EmaOracle, Ethereum, HSM, ICE, Identity, Intent, LBP, MultiTransactionPayment, Multisig, Omnipool, OmnipoolLiquidityMining, OmnipoolWarehouseLM, OTC, `ParachainSystem.ValidationData`, **Parameters** (read for `UniswapV3Factory`), Proxy, Referenda, Referrals, Router, Stableswap, Staking, Timestamp, Tokens, `Uniques.Account`, XYK, XYKWarehouseLM, GigaHdx, GigaHdxRewards
- **Extrinsics:** Bonds, ConvictionVoting, Currencies, DCA, Democracy, Dispatcher, Ethereum, EVM, EVMAccounts, ICE, Intent, MultiTransactionPayment, Multisig, Omnipool, OmnipoolLiquidityMining, OTC, Proxy, Referrals, Router, Stableswap, Staking, Tokens, TechnicalCommittee, Utility, XYK, XYKLiquidityMining, GigaHdx, GigaHdxRewards
- **Hub:** `query.{ForeignAssets,Assets}.*`, `api.AssetConversionApi.*`, `tx.{Assets,Balances}.*`

## Gotchas

- **`FeeProcessor` is in `hydration.scale` but not whitelisted** — no typed API for fee routing ([[wiki/pallet-fee-processor\|pallet-fee-processor]]). `GigaHdx` / `GigaHdxRewards` *are* whitelisted now.
- **`Democracy` is still whitelisted** (`query.Democracy.*`, `tx.Democracy.*`) and still present in the committed `hydration.scale`, but hydration-node master removed the pallet. Expect these to vanish (and break callers) on the next metadata regen after the runtime upgrade.
- `hydration.scale` also contains `LazyExecutor` — not whitelisted.

## Role in the Stack

Peer dependency of [[wiki/sdk-next\|sdk-next]] (`>=2.7.0`) and imported directly by [[wiki/xc-cfg\|xc-cfg]] (Snowbridge codecs, XCM builders, circuit-breaker reads). ESM-only, `noDescriptorsPackage: true`.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
