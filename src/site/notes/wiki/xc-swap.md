---
{"dg-publish":true,"permalink":"/wiki/xc-swap/","title":"xc-swap","tags":["sdk","cross-chain","swap","near","intents","wormhole","typescript"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"xc-swap","repo":"sdk","paths":["packages/xc-swap/src/index.ts","packages/xc-swap/src/factory.ts","packages/xc-swap/src/client.ts","packages/xc-swap/src/types.ts","packages/xc-swap/src/trade/swap.ts","packages/xc-swap/src/trade/builder.ts","packages/xc-swap/src/trade/abi.ts","packages/xc-swap/src/quote/oneClick.ts","packages/xc-swap/src/quote/relayFee.ts","packages/xc-swap/src/registry/consts.ts","packages/xc-swap/README.md","packages/xc-swap/docs/spec.md"],"key_exports":["createXcSwap","XcSwapClient","XcSwapTrade","XcSwapRequest","XcSwapParams","XcSwapOpts","XcSwapError","swap","buildCalls","buildRoutes","SWAP_AND_BRIDGE_ABI","fetchMaxRelayFee","getOneClickQuote","fetchOneClickTokens","configureOneClick","WETH_ID","GLMR_ID","MIN_WETH","ONE_CLICK_ORIGIN_ASSET"],"depends_on":["sdk-next","xc-core","xc-sdk","sdk-common"],"tags":["sdk","cross-chain","swap","near","intents","wormhole","typescript"],"source_count":1,"last_updated":"2026-08-15"}}
---


# xc-swap

**TL;DR:** `@galacticcouncil/xc-swap` (v0.6.0, new Aug 2026) is a **NEAR Intent Routing (NIR)** cross-chain *swap* SDK — sell any [[wiki/hydration\|hydration]] asset for a NEAR-ecosystem asset in **one Hydration EVM transaction**. It is **not** part of the [[wiki/xc-package\|xc-package]] transfer stack: no `ConfigService`, no `AssetRoute`, no `Wallet`. It drives [[wiki/sdk-next\|sdk-next]]'s `TradeRouter`, the on-chain `IntentEmitter` contract, and the off-chain [1Click](https://docs.near-intents.org/) API directly.

## How it relates to the rest of the XC stack

| Package | Relationship |
|---|---|
| [[wiki/sdk-next\|sdk-next]] | **Hard dependency.** `XcSwapOpts.sdk` is a whole `SdkCtx`. Uses `sdk.api.router` (`getBestBuy` / `getBestSell`), `sdk.client.asset.getSupported()` (runtime asset registry), `sdk.client.evm` (allowance read). |
| [[wiki/xc-core\|xc-core]] | **Types only** — `AssetAmount` (human amounts) and the `CallType` enum. No `Chain`, no `AssetRoute`, no `ConfigService`. |
| [[wiki/xc-sdk\|xc-sdk]] | **Types only** — the `EvmCall` interface, so emitted calls plug into the existing signer path. No `Wallet`, no `PlatformAdapter`. |
| [[wiki/sdk-common\|sdk-common]] | `erc20.ERC20.fromAssetId()` to derive the Hydration ERC-20 precompile address of an asset id. |
| [[wiki/xc-cfg\|xc-cfg]] | **None.** Deliberately bypassed — xc-cfg keys assets by *display* key (`weth_wh`), not the runtime ids the emitter takes. |

Published as a **sibling** of `@galacticcouncil/xc`, not a layer over it. Peer deps: `common >=1.1.0`, `sdk-next >=1.5.0`, `xc-core >=2.0.0`, `xc-sdk >=2.0.0`, `viem
{ #2}
.38.3`. Runtime dep: `@defuse-protocol/one-click-sdk-typescript
{ #0}
.1.17`.

## The NIR flow

1. On Hydration: buy the GLMR XCM fee with ≤ `maxFeeIn` of asset `A`, then sell the remaining `A` → WETH via [[omnipool\|omnipool]] / the [[wiki/smart-order-router\|smart-order-router]].
2. Reserve-transfer `[GLMR, WETH]` to the MDA on Moonbeam, bridging WETH → native ETH to a **1Click deposit address** on Ethereum.
3. 1Click swaps ETH → the destination NEAR asset and delivers to `recipient`.

Steps 1–2 are a single EVM call: `IntentEmitter.swapAndBridge(...)` on Hydration EVM. The SDK reproduces the off-chain pieces (relay-fee ceiling, 1Click quote, `intentId`, slippage bounds).

> Note the asymmetry with [[wiki/xc-cfg\|xc-cfg]]: the *transfer* stack deleted the Moonbeam hop in the MRL→NTT migration, but the `IntentEmitter` contract still bridges through Moonbeam. That hop is on-chain (WHM contracts), not SDK config, so it is invisible to xc-cfg.

## Public API

```ts
// packages/xc-swap/src/factory.ts + client.ts
export function createXcSwap(opts: XcSwapOpts): XcSwapClient;

class XcSwapClient {
  getChains(): XcSwapChain[];                       // [hydration, near, zec]
  getOriginAssets(): Promise<XcSwapAsset[]>;        // every Hydration runtime asset, unfiltered
  getDestinationAssets(): Promise<XcSwapAsset[]>;   // 1Click /v0/tokens ∩ allowlist
  getRoutes(): Promise<XcSwapRoute[]>;
  swap(params: XcSwapParams): Promise<XcSwapTrade>; // dry quote — amounts only
}
```

`XcSwapTrade.buildCall()` requests a **firm** 1Click quote (yields the deposit address) and returns `XcSwapRequest { calls, depositAddress, intentId, correlationId, deadline }`. `calls` is `[approve, swapAndBridge]`, or just `[swapAndBridge]` when the emitter already has sufficient allowance.

### `XcSwapOpts`

| Field | Default | Notes |
|---|---|---|
| `sdk` | — | required `SdkCtx` from `createSdkContext` |
| `emitter` | — | required `IntentEmitter` proxy address on Hydration EVM |
| `quoterUrl` | `https://quoter-api.play.hydration.cloud` | relay-fee quoter |
| `relayMargin` | `20` (percent) | sent to the quoter as `marginBps` |
| `slippage` | `1` (percent) | same unit as `sdk-next`'s `withSlippage` |
| `xcmFee` | `DEFAULT_XCM_FEE` = 3 GLMR | reserved for the XCM leg |
| `destinationAssets` | `['nep141:wrap.near', 'nep141:zec.omft.near']` | 1Click asset id allowlist |
| `oneClick` | `{}` | `{ baseUrl, jwt }`; default base `https://1click.chaindefuser.com`, default JWT is a baked-in `partner_id: hydration` distribution channel |

## Registry constants

```ts
// packages/xc-swap/src/registry/consts.ts
export const WETH_ID = 20;   // Hydration runtime asset id, bridged out by the emitter
export const GLMR_ID = 16;   // pays the XCM fee
export const MIN_WETH = 400_000_000_000_000n;               // 0.0004 WETH viability floor
export const DEFAULT_XCM_FEE = 3_000_000_000_000_000_000n;  // 3 GLMR
export const ONE_CLICK_ORIGIN_ASSET = 'nep141:eth.omft.near';
export const WRAP_NEAR_ASSET = 'nep141:wrap.near';
export const ZEC_ASSET = 'nep141:zec.omft.near';
```

Asset ids are **Hydration runtime asset ids** — the same id space `sdk-next`'s `TradeRouter` and `IntentEmitter.swapAndBridge` key on (mirrored from WHM `HydrationConsts.sol`). `WETH_ID = 20` cross-checks against `apps/src/rescue/rescue.ts`, which names asset 20 as the Hydration EVM gas asset. Chains: `hydration` (origin) → `near` / `zec` (destination); platform enum `'hydration' | 'near' | 'zec'`.

## Quote orchestration (`src/trade/swap.ts → swap()`)

1. Fee leg — `router.getBestBuy(A, GLMR_ID, xcmFee)` → `maxFeeIn` (padded up by slippage). Skipped when `A == GLMR` (the fee is withheld instead).
2. Sell leg — `router.getBestSell(A, WETH_ID, amountIn - feeSpent)` → `wethOut`, `priceImpactPct`. Skipped when `A == WETH`.
3. Reference leg — full `A → WETH` with no fee carve-out (`idealWeth`), used only for the fee breakdown.
4. In parallel: `fetchMaxRelayFee()` (`GET {quoter}/relay-fee?chain=ethereum&marginBps=…` → `feeRequested`), asset resolution, and a **dry** 1Click quote (`swapType=FLEX_INPUT`, `depositType=ORIGIN_CHAIN`, origin `nep141:eth.omft.near`, `referral: 'hydration'`).
5. `swapAmount = wethOut - maxRelayFee`; the dry quote's outputs were priced at the full `wethOut` and are **linearly scaled** down to `swapAmount`.
6. `fee = idealWeth - swapAmount` (GLMR XCM fee + relay fee), valued in WETH and in USD via the quote's `amountInUsd`.

```ts
// packages/xc-swap/src/trade/swap.ts → buildCall()
const intentId = keccak256(
  encodePacked(['address', 'uint256'], [depositAddress as `0x${string}`, amountIn])
);
```

This is the **shipped** form; `docs/spec.md` still describes the older 4-field `abi.encode(correlationId, depositAddress, amountIn, deadline)` variant.

## Errors are reported, not thrown

`XcSwapTrade.errors: XcSwapError[]` (router-style). `buildCall()` throws only if the array is non-empty.

| `XcSwapError` | Meaning |
|---|---|
| `MinWethNotMet` | `minEthOut < MIN_WETH` (0.0004 WETH) |
| `RelayFeeTooHigh` | `wethOut < 2 × maxRelayFee` |
| `AmountTooLow` | 1Click rejected the amount (message match on `"too low"`) |
| `RecipientInvalid` | 1Click rejected the recipient (message match on `"recipient"`) |
| `QuoteFailed` | any other 1Click failure |

## Files

| Path | Contents |
|---|---|
| `src/client.ts` | `XcSwapClient` — listings, memoized asset/token registries, `swap()` |
| `src/trade/swap.ts` | quote orchestration + `buildCall()` closure |
| `src/trade/builder.ts` | `buildCalls()` → `[approve, swapAndBridge]` as `EvmCall[]` |
| `src/trade/abi.ts` | `SWAP_AND_BRIDGE_ABI` (`uint32 assetIn, uint256 amountIn, uint256 minEthOut, uint256 maxFeeIn, bytes32 intentId, address intentDepositAddress, uint256 maxRelayFee`) |
| `src/trade/utils.ts` | `amount()`, `padUp()` / `padDown()` (bps slippage pads) |
| `src/quote/relayFee.ts` | `fetchMaxRelayFee()` |
| `src/quote/oneClick.ts` | `configureOneClick()`, `getOneClickQuote()`, `fetchOneClickTokens()` |
| `src/registry/` | consts, chains, assets, route metadata + `registry.spec.ts` |
| `docs/spec.md` | original design plan (partly superseded — `estimateTrade` → `swap`, `slippageBps` → `slippage` percent) |

Demo: `examples/xc-transfer/src/swap.ts` + `examples/xc-transfer/public/swap/index.html`.

## Gotchas

- **`refundTo` is overloaded.** It is documented as the Ethereum-side refund address, but `swap()` also uses it as the Hydration EVM `from` of both calls and as the allowance *owner* in the `erc20.allowance(refundTo, emitter)` read. Callers must pass the sender's Hydration EVM (h160) address, not a NEAR/Ethereum-only refund address.
- **`XcSwapOpts.xcmFee` doc comment is wrong** — it says "Default 1 GLMR"; the code applies `DEFAULT_XCM_FEE` = 3 GLMR.
- The relay-fee quoter default points at a **play** (non-prod) endpoint — override `quoterUrl` for mainnet.
- `emitter` has no default; the deployed Hydration-side `IntentEmitter` proxy address must be supplied by the caller.
- `getOriginAssets()` returns the entire runtime registry with no viability filter — an asset with no `A → WETH` route only fails at `swap()` time.
- The dry quote is priced pre-relay-fee and scaled linearly; the firm quote in `buildCall()` is sized to the exact net `swapAmount`, so `amountOut` can move slightly between the two.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
