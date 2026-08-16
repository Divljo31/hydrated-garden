---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-utils/","title":"hydration-ui utils","tags":["utilities","helpers","hydration"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui utils","repo":"hydration-ui","paths":["packages/utils/src","packages/utils/src/helpers/neckwork.ts","packages/utils/src/helpers/multix.ts","packages/utils/src/helpers/address.ts","packages/utils/src/helpers/basejumpscan.ts","packages/utils/src/helpers/xcm.ts","packages/utils/src/constants/assets.ts"],"symbols":["uuid","neckwork","multix","ActivityType","getAccountExplorerLink","NearAddr","ZcashAddr","EvmAddr","SolanaAddr","Ss58Addr","SuiAddr","invariant","findNested","isAddressValidOnHydration","basejumpscan","xcscan","suivision","solexplorer","safeConvertAnyToH160","safeConvertPublicKeyToSS58","safeConvertSS58toH160","createQueryString","safeParse","safeStringify","stringEquals","parseHydrationRpcName","formatInterval","shorten","shortenAccountAddress","clampBigInt"],"key_deps":["humanize-duration-ts","@noble/hashes","uuid","remeda","date-fns","@galacticcouncil/xc-core","@galacticcouncil/xc-cfg"],"tags":["utilities","helpers","hydration"],"last_updated":"2026-08-15"}}
---


# hydration-ui utils

**TL;DR:** Shared utilities package (`packages/utils/`). Zero internal deps (build level 0). Barrels: `helpers/`, `constants/`, `hooks/`, `lib/`, `types/` — all re-exported from `src/index.ts` alongside `uuid` (v4, re-exported from the `uuid` package).

## Helpers (`src/helpers/`)

| File | Notable exports | Purpose |
|------|-----------------|---------|
| `address.ts` | `EvmAddr`, `Ss58Addr`, `SolanaAddr`, `SuiAddr`, **`NearAddr`**, **`ZcashAddr`**, **`getAccountExplorerLink`** | Address validators (re-exported from `xc-core` `addr` + local NEAR/Zcash regexes) and a chain-dispatching explorer link |
| `array.ts` | array utilities | |
| `basejumpscan.ts` | `basejumpscan.{baseUrl, transfers, link, tx}` | Basejump bridge scan (`https://bjscan-api.play.hydration.cloud`) |
| `big.ts` | big.js helpers | Decimal arithmetic |
| `device.ts` | device detection | |
| `evm.ts` | EVM helpers | Hex conversions, chain IDs |
| `formatting.ts` | `wsToHttp`, `shorten`, `shortenAccountAddress` | Display formatters |
| `helpers.ts` | `createQueryString` | Generic helpers |
| `html.ts`, `interpolation.ts` | layout/string helpers | |
| `intl.ts` | `formatDate`, `formatInterval` | Intl + humanize-duration |
| **`invariant.ts`** (NEW) | `invariant(cond, msg)` | Assertion helper (`asserts condition`) |
| `jitosol.ts`, `solana.ts`, `sui.ts`, `wormhole.ts`, `xcm.ts`, `xcscan.ts` | bridge/cross-chain helpers | |
| `json.ts` | `safeParse`, `safeStringify` | Defensive JSON |
| `logger.ts` | logging | |
| `math.ts` | `clampBigInt`, … | Numeric helpers |
| `meta.ts` | metadata helpers | |
| **`multix.ts`** (NEW) | `multix.{graphql, link, account}` | Self-hosted Multix app + GraphQL URLs (`multix.lark.hydration.cloud`) — see [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| **`neckwork.ts`** (NEW) | `neckwork.{base, account, contract, block, extrinsic, event, extrinsicHash, activityEvent, activityExtrinsic, activityDca}`, `ActivityType` | Neckwork block explorer (`hydration-explorer.neckwork.net`) — the app's primary explorer link source |
| `object.ts` | `hasOwn`, `propPath`, **`findNested`** | Object traversal |
| `polkadot.ts` | polkadot/substrate helpers | |
| `providers.ts` | RPC provider helpers | |
| `rpc.ts` | `parseHydrationRpcName`, `PingResponse` | RPC name parsing (see below) |
| `serialization.ts` | binary/state serialization | |
| `subscan.ts` | Subscan URL helpers | |
| `types.ts` | `HexString`, shared types | |

**Removed Aug 2026:** `helpers/snowbridge.ts` (deleted alongside the snowbridge GraphQL client, `0ce954f`).

## Constants (`src/constants/`)

`assets.ts`, `chains.ts`, **`evm.ts`** (NEW — `UINT256_MAX`), `query.ts` (`QUERY_KEY_BLOCK_PREFIX = "@block"`), `url.ts`.

New asset-ID constants (Aug 2026): `HDX_ASSET_ID` (`"0"`), `USDC_ASSET_ID` (`"22"`), `APYUSD_ASSET_ID` (`"46"`), **`BIL_ERC20_ID` (`"55"`) / `BIL_ASSET_ID` (`"550"`)** ([[wiki/hydration-ui-modules\|hydration-ui-modules]] `strategies/`), `HOLLAR_BOND_25_08_26_ID` (`"1001351"`), plus the `EXTERNAL_APY_ASSET_IDS` group (`APYUSD`, `USDT_POOL`, `VDOT`, `PRIME`, `…HOLLAR_ASSETS`, `…GIGA_ASSETS`) used by the `externalApy` Zustand store.

## Explorer links: Neckwork replaces Subscan as the default

`getAccountExplorerLink(address)` dispatches by address shape:

```typescript
// packages/utils/src/helpers/address.ts
export function getAccountExplorerLink(address: string) {
  if (isAddressValidOnHydration(address)) return neckwork.account(address)
  if (SolanaAddr.isValid(address)) return solexplorer.account(address)
  if (SuiAddr.isValid(address)) return suivision.account(address)
  return xcscan.search(address)
}
```

`isAddressValidOnHydration()` (new in `helpers/xcm.ts`) looks up `chainsMap.get(HYDRATION_CHAIN_KEY)` from `@galacticcouncil/xc-cfg` and delegates to `isAddressValidOnChain(address, chain)` (parachain → `chain.usesH160Acc ? EvmAddr : Ss58Addr`, etc.) — so `utils` now has a hard dependency on the [[wiki/xc-cfg\|xc-cfg]] chain registry.

**Undeclared dependency gotcha:** `packages/utils/package.json` declares only `humanize-duration-ts`, `@noble/hashes`, `uuid`, `buffer`, `@types/uuid`. `@galacticcouncil/xc-cfg` and `@galacticcouncil/xc-core` are imported by `helpers/{address,xcm,evm,subscan}.ts` and `lib/AssetMetadataFactory.ts` but resolve only through hoisting from the workspace root. Bumping `xc-*` at the root silently changes this package's behaviour. New sibling link helpers: `solexplorer.account()`, `suivision.account()` (`https://suivision.xyz`), `xcscan.search()`.

`ActivityType` (`"cross-chain" | "swap" | "dca" | "lend" | "borrow" | "repay" | "withdraw"`) is defined here and consumed by [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] (`toActivity()` in `Web3Provider.tsx`) and the transactions module to build activity-scoped explorer deep links (`neckwork.activityEvent`, `neckwork.activityExtrinsic`, `neckwork.activityDca`).

## NEAR / Zcash address support

`NearAddr` (regex on `*.near` account names, plus `parseAccountName`) and `ZcashAddr` (transparent `t1`/`t3` + unified `u1` prefixes) were added for the onramp/deposit flows. They feed `WalletMode.Near` / `WalletMode.Zcash` in [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]].

## RPC helpers gutted

`pingRpc()` and `jsonRpcFetch()` were **removed** from `helpers/rpc.ts` (commit `c524135` "Improved RPC resolution", −137 lines). Only `parseHydrationRpcName()` and the `PingResponse` type remain; RPC latency probing now happens in the app's inlined `apps/main/src/utils/rpc-ping.js` head script (injected by `vite-plugin-html`) before the bundle loads.

## `formatInterval` rework

`helpers/intl.ts` no longer holds a module-level `HumanizeDuration` singleton. Each call constructs one, honours an `options.largest` override (default `2`) and an `options.unit` (`UnitName`) restriction. `shorten()` now uses a real ellipsis character (`…`) instead of three dots.

## Use cases

- **Explorer deep links:** account / extrinsic / event / DCA schedule links across Neckwork, Solana Explorer, SuiVision, xcscan
- **Address validation + conversion:** SS58 ⇄ H160 ⇄ public key, plus Solana / Sui / NEAR / Zcash shape checks
- **Duration display:** Staking lock times, unbond periods
- **Defensive parsing:** `safeParse` / `safeStringify` / `invariant` across API adapters

## For developers

Import from `@galacticcouncil/utils` (single flat barrel — there is no `exports` map, `main` is `./src/index.ts`):

```typescript
import { formatInterval, getAccountExplorerLink, neckwork, uuid } from "@galacticcouncil/utils"

const formId = uuid()
const lockTime = formatInterval(lockPeriodMs, { format: "short" }) // "14d 23h"
const link = getAccountExplorerLink(address)
```

Anything added under `helpers/` must also be listed in `helpers/index.ts` (same for `constants/`, `hooks/`, `lib/`, `types/`) — the barrels are explicit, not glob-based.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/xc-cfg\|xc-cfg]]
