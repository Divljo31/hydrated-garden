---
{"dg-publish":true,"permalink":"/wiki/sdk-descriptors/","title":"descriptors (SDK)","tags":["sdk","metadata","papi","types","ice"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"descriptors (SDK)","repo":"sdk","paths":["packages/descriptors/src/whitelist.ts","packages/descriptors/.papi/whitelist.ts","packages/descriptors/.papi/polkadot-api.json","packages/descriptors/.papi/metadata/hydration.scale","packages/descriptors/.papi/metadata/hydrationIce.scale","packages/descriptors/.papi/metadata/hub.scale","packages/descriptors/wasm/ice/ice.wasm"],"key_exports":["whitelist","HydrationWhitelistEntry","HydrationIceWhitelistEntry","HubWhitelistEntry","WhitelistEntriesByChain"],"tags":["sdk","metadata","papi","types","ice"],"source_count":1,"last_updated":"2026-08-15"}}
---


# descriptors (SDK)

**TL;DR:** `@galacticcouncil/descriptors` (v2.6.0, papi v2, `polkadot-api
{ #2}
.1.7`) provides type-safe papi metadata descriptors for [[hydration]], the **ICE** runtime and AssetHub, generated from a source whitelist. Built on [[papi-codegen]] and [[papi-typed-api]] (see [[papi]] for context). **Aug 2026: all metadata regenerated and a fourth chain entry (`hydrationIce`) added.**

## Generation

Whitelist lives in two places:

- `packages/descriptors/src/whitelist.ts` — typed source of truth (`HydrationWhitelistEntry[]`, `HydrationIceWhitelistEntry[]`, `HubWhitelistEntry[]`)
- `packages/descriptors/.papi/whitelist.ts` — generated artifact consumed by `papi`

`npm run papi:whitelist` runs as `prebuild`. **Do not edit `.papi/descriptors/`** — edit `src/whitelist.ts`.

## Chain entries (`.papi/polkadot-api.json`)

| Key | Source | Metadata |
|---|---|---|
| `hydration` | live RPC `wss://hydration-rpc.n.dwellir.com` (genesis `0xafdc188f…`) | `hydration.scale` |
| `hydrationNext` | local file only | `hydrationNext.scale` |
| **`hydrationIce`** (new) | local file only, generated from the committed runtime blob `wasm/ice/ice.wasm` (~2.4 MB) | `hydrationIce.scale` |
| `hub` | `polkadot_asset_hub` | `hub.scale` |

`hydrationIce` is the descriptor [[wiki/sdk-next\|sdk-next]]'s ICE intent tx builders (`IntentMarketTxBuilder`, `IntentLimitTxBuilder`, `IntentOrderTxBuilder`) run against — the `Intent` / `ICE` pallets are not in the Hydration production runtime, so the descriptor is generated from a checked-in wasm instead of a live chain. Consumers of those builders must build this package first.

## Whitelist changes this cycle

| Chain | Added |
|---|---|
| hydration / hydrationNext | `query.Router.*`, `tx.Ethereum.*`, `tx.TechnicalCommittee.*` |
| hydrationIce | `const.ICE.*`, `const.Intent.*`, `query.ICE.*`, `query.Intent.*`, `tx.ICE.*`, `tx.Intent.*` |
| hub | `tx.Balances.*` |

Nothing was removed.

## New pallets: in metadata, not in the whitelist

`hydration.scale` was regenerated against a newer runtime (532 KB → 558 KB) and now **contains** [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] and [[wiki/pallet-fee-processor\|pallet-fee-processor]] — none of which appeared in the previous blob. **None of the three is whitelisted**, so they are absent from the generated typed API: `api.query.GigaHdx.*`, `GigaHdxRewards.*` and `FeeProcessor.*` are not reachable from [[wiki/sdk-next\|sdk-next]] or [[wiki/xc-cfg\|xc-cfg]] today. Anything the SDK needs to read about liquid staking (stHDX/GIGAHDX exchange rate, pending unstakes) or trade-fee routing requires whitelist entries first.

## Coverage (current whitelist)

- **Runtime APIs:** `AaveTradeExecutor`, `CurrenciesApi`, `EthereumRuntimeRPCApi`, `EvmAccountsApi`, `Metadata`; globally `DryRunApi`, `XcmPaymentApi`
- **Storage:** AssetRegistry, Balances, Bonds, CircuitBreaker, ConvictionVoting, Democracy, DynamicFees, EmaOracle, Ethereum, HSM, Identity, LBP, MultiTransactionPayment, Multisig, Omnipool, OmnipoolLiquidityMining, OmnipoolWarehouseLM, OTC, `ParachainSystem.ValidationData`, Proxy, Referenda, Referrals, **Router**, Stableswap, Staking, Timestamp, Tokens, `Uniques.Account`, XYK, XYKWarehouseLM
- **Events:** `EVM.Log`, `Proxy.PureCreated`, `Router.Executed`, `Stableswap.*`
- **Constants:** Aura, Balances, CircuitBreaker, DCA, DynamicFees, HSM, LBP, Multisig, Omnipool, OmnipoolLiquidityMining, OTC, Proxy, Referenda, Stableswap, Staking, XYK, XYKLiquidityMining, `System` (global)
- **Extrinsics:** ConvictionVoting, Currencies, DCA, Democracy, Dispatcher, **Ethereum**, EVM, EVMAccounts, MultiTransactionPayment, Multisig, Omnipool, OmnipoolLiquidityMining, OTC, Proxy, Referrals, Router, Stableswap, Staking, **TechnicalCommittee**, Tokens, Utility, XYK, XYKLiquidityMining, plus `PolkadotXcm` on every chain
- **Hub:** `query.{ForeignAssets,Assets}.*`, `api.AssetConversionApi.*`, `tx.{Assets,Balances}.*`

## Role in the Stack

Peer dependency of [[wiki/sdk-next\|sdk-next]] (`>=2.6.0`) and imported directly by [[wiki/xc-cfg\|xc-cfg]] for its Snowbridge codecs, XCM builders and circuit-breaker reads. ESM-only, `noDescriptorsPackage: true` — the build copies `.papi/descriptors/dist` into `build/`.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
