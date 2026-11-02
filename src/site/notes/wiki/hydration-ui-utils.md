---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-utils/","title":"hydration-ui utils","tags":["utilities","helpers","hydration"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui utils","repo":"hydration-ui","paths":["packages/utils/src","packages/utils/src/helpers/index.ts","packages/utils/src/helpers/neckwork.ts","packages/utils/src/helpers/intentscan.ts","packages/utils/src/helpers/transferscan.ts","packages/utils/src/helpers/address.ts","packages/utils/src/helpers/xcm.ts","packages/utils/src/helpers/wormhole.ts","packages/utils/src/constants/assets.ts","packages/utils/src/lib/index.ts"],"symbols":["uuid","neckwork","ActivityType","intentscan","transferscan","nearblocks","zcashexplorer","wormholescan","getVaaHeader","multix","xcscan","solexplorer","suivision","getAccountExplorerLink","parseNearAccountName","toNearAccountId","EvmAddr","Ss58Addr","SolanaAddr","SuiAddr","NearAddr","ZecAddr","isAddressValidOnHydration","isAddressValidOnChain","hasRouteToHydration","resolveHydrationAssetId","isNearChain","isZecChain","safeConvertAddressSS58","normalizeSS58Address","withTimeout","arraySearch","formatInterval","formatCurrency","formatAssetAmount","formatRelativeTime","invariant","parseHydrationRpcName","AssetMetadataFactory","createZustandStorage","patchBigJs","polkadotIcon","useAfterFirstRender","useCopy","useDevice","useNow","useStableArray","HOLLAR_ASSET_ID","GIGA_ASSETS","SELL_ONLY_ASSETS","ISOLATED_MODE_ASSETS"],"key_deps":["humanize-duration-ts","@noble/hashes","@scure/base","uuid","buffer","@galacticcouncil/xc-core (undeclared)","@galacticcouncil/xc-cfg (undeclared)"],"tags":["utilities","helpers","hydration"],"last_updated":"2026-11-02"}}
---


# hydration-ui utils

**TL;DR:** `@galacticcouncil/utils` (`packages/utils/`) — shared helpers, constants, hooks and libs, build level 0. Single flat barrel: `src/index.ts` re-exports `helpers/`, `constants/`, `hooks/`, `lib/`, `types/` plus `uuid` (v4). No `exports` map; `main` is `./src/index.ts`. Barrels are explicit — a new file must be added to its folder's `index.ts`.

## Helpers (`src/helpers/`, 34 files)

| File | Exports | Purpose |
|------|---------|---------|
| `address.ts` | `EvmAddr`, `Ss58Addr`, `SolanaAddr`, `SuiAddr`, `NearAddr`, `ZecAddr` (all from `xc-core` `addr`), `getAccountExplorerLink`, `parseNearAccountName`, `toNearAccountId` | Address validators + explorer link dispatch |
| `xcm.ts` | `isAddressValidOnHydration`, `isAddressValidOnChain`, `getChainId`, `getChainAssetId`, `isParachain` / `isEvmParachain` / `isEvmChain` / `isSolanaChain` / `isSuiChain` / `isNearChain` / `isZecChain`, `formatSourceChainAddress`, `formatDestChainAddress`, `hasRouteToHydration`, `resolveHydrationAssetId`, `resolveHydrationDisplayAssetId` | Chain typing and Hydration route/asset resolution over [[wiki/xc-cfg\|xc-cfg]] `chainsMap` |
| `polkadot.ts` | `safeConvertAddressSS58`, `safeConvertPublicKeyToSS58`, `safeConvertSS58toPublicKey`, `isSS58Address`, `normalizeSS58Address`, `accountPublicKey`, `isBinary` | SS58 / public-key conversion |
| `neckwork.ts` | `neckwork.{base, account, contract, block, extrinsic, event, extrinsicHash, activityEvent, activityExtrinsic, activityDca, intent}`, `ActivityType` | Neckwork explorer (`hydration-explorer.neckwork.net`) — default explorer for Hydration |
| `intentscan.ts` | `intentscan.{apiUrl, order}` | Intents scan app (`intents.scan.play.hydration.cloud`, UI `scan.play.hydration.cloud`) |
| `transferscan.ts` | `transferscan.transfer(id)` | Transfers scan app page links |
| `nearblocks.ts`, `zcashexplorer.ts` | `nearblocks.tx(chainKey, hash)`, `zcashexplorer.tx(chainKey, txId)` | NEAR / Zcash tx links from `chainsMap` explorer |
| `wormhole.ts` | `wormholescan.api(path, query)`, `getVaaHeader(vaaRaw)` | Wormholescan API URL + VAA header parsing |
| `solana.ts`, `sui.ts`, `xcscan.ts`, `subscan.ts`, `multix.ts` | `solexplorer`, `suivision`, `xcscan.search`, Subscan links, `multix.{graphql, link, account}` | Other explorers |
| `intl.ts` | `formatNumber`, `formatPercent`, `formatCurrency`, `formatAssetAmount`, `formatDate`, `formatRelativeTime`, `formatInterval`, `getDecimalSeparator`, `MAX_USD_FRACTION_DIGITS` | Intl formatting; `formatInterval` builds a HumanizeDuration per call (`options.largest`, default 2; `options.format: "short"`) |
| `promise.ts` | `withTimeout` | Promise timeout |
| `array.ts` | `arraySearch` (+ tests) | |
| `rpc.ts` | `parseHydrationRpcName`, `PingResponse` | RPC name parsing only |
| `invariant.ts`, `json.ts`, `object.ts`, `math.ts`, `big.ts`, `formatting.ts`, `helpers.ts`, `evm.ts`, `device.ts`, `html.ts`, `interpolation.ts`, `logger.ts`, `meta.ts`, `providers.ts`, `serialization.ts`, `types.ts` | `invariant`, `safeParse` / `safeStringify`, `findNested`, `clampBigInt`, `shorten`, `createQueryString`, … | General |

```typescript
// packages/utils/src/helpers/address.ts
export function getAccountExplorerLink(address: string) {
  if (isAddressValidOnHydration(address)) return neckwork.account(address)
  if (SolanaAddr.isValid(address)) return solexplorer.account(address)
  if (SuiAddr.isValid(address)) return suivision.account(address)
  return xcscan.search(address)
}
```

`ActivityType` = `cross-chain | swap | dca | lend | borrow | repay | withdraw | intent-place | intent-fill | intent-cancel | intent-expire | intent-dca-trade`. `neckwork.intent(id)` takes a **string/bigint** — intent ids are u128.

## Constants (`src/constants/`)

`assets.ts`, `chains.ts` (`HYDRATION_PARACHAIN_ID = 2034`, `HYDRATION_CHAIN_KEY`), `query.ts` (`QUERY_KEY_BLOCK_PREFIX = "@block"`), `url.ts` (HDX supply, Subsquare referenda URLs).

`assets.ts` highlights: `HDX_ASSET_ID "0"`, `H2O_ASSET_ID "1"`, `HOLLAR_ASSET_ID "222"`, `USDT "10"`, `USDC "22"`, `DOT "5"`, `VDOT "15"` (ERC20 `1005`), `ETH "34"`, `WSTETH "1000809"`, `SUSDE "1000625"`, `SUSDS "1000745"`, GIGA set (`GDOT 690`/ERC20 `69`, `GETH 4200`/`420`, `GSOL 90001`/`9001`), `BIL 550`/ERC20 `55`, HOLLAR pools `HUSDC 110`, `HUSDT 111`, `HUSDS 112`, `HUSDE 113`, `HEURC 10044`, `PRIME 43` (ERC20 `1043`, stableswap `143`), bonds `HOLLAR_BOND_25_08_26_ID 1001351`, `HOLLAR_BOND_24_11_26_ID 1001353`. Groups: `HOLLAR_ASSETS`, `GIGA_ASSETS`, `GIGA_ERC20`, `GIGA_STABLESWAP_TO_ERC20`, `EXTERNAL_APY_ASSET_IDS`, `MONEY_MARKET_STRATEGY_ASSETS`, `ISOLATED_MODE_ASSETS = [PRIME]`, `SELL_ONLY_ASSETS`.

## Hooks, lib, types

- `hooks/`: `useAfterFirstRender`, `useCopy`, `useDevice`, `useNow`, `useStableArray`.
- `lib/`: `AssetMetadataFactory` (+ `METADATA_CDN_URL`), `createZustandStorage` (zod-validated persisted Zustand storage), `patchBigJs`, `polkadotIcon`.
- `types/`: `HexString`.

## Gotchas

- **Undeclared deps:** `helpers/{address,xcm,evm,subscan,nearblocks,zcashexplorer}.ts` and `lib/AssetMetadataFactory.ts` import `@galacticcouncil/xc-core` / `xc-cfg`, which `packages/utils/package.json` does not declare (declared: `humanize-duration-ts`, `@noble/hashes`, `@scure/base`, `uuid`, `buffer`). They resolve via root hoisting; bumping `xc-*` at the root changes this package's behaviour.
- `isAddressValidOnHydration` depends on the xc-cfg chain registry; addresses are validated per chain (`usesH160Acc` → `EvmAddr`, else `Ss58Addr`).

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]]
