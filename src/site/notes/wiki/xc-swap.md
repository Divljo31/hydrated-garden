---
{"dg-publish":true,"permalink":"/wiki/xc-swap/","title":"xc-swap","tags":["sdk","cross-chain","near-intents","1click","ntt","swap","typescript"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"xc-swap","repo":"sdk","paths":["packages/xc-swap/src/client.ts","packages/xc-swap/src/factory.ts","packages/xc-swap/src/types.ts","packages/xc-swap/src/trade/swap.ts","packages/xc-swap/src/trade/rail.ts","packages/xc-swap/src/trade/builder.ts","packages/xc-swap/src/trade/abi.ts","packages/xc-swap/src/quote/oneClick.ts","packages/xc-swap/src/quote/relayFee.ts","packages/xc-swap/src/registry/consts.ts","packages/xc-swap/docs/spec.md"],"key_exports":["createXcSwap","XcSwapClient","XcSwapOpts","XcSwapParams","XcSwapTrade","XcSwapRequest","XcSwapSettlement","XcSwapError","fetchRailConfig","fetchRailState","ORDER_PLACED_ABI","WETH_ID","TRIM_UNIT","MIN_WETH"],"tags":["sdk","cross-chain","near-intents","1click","ntt","swap","typescript"],"last_updated":"2026-11-02"}}
---


# xc-swap

**TL;DR:** `@galacticcouncil/xc-swap` (**v0.9.0**) — buy any NEAR-Intents asset (wNEAR, ZEC, …) with any [[wiki/hydration\|hydration]] asset in **one Hydration EVM transaction**. Asset `A` is sold for WETH on Hydration, settled to Ethereum **over Wormhole NTT**, and forwarded into a NEAR **1Click** deposit address. On-chain half: `IntentEmitter.placeOrder(...)` / `IntentReceiver.processOrder(...)` in [[wiki/source-whm-codebase\|WHM]] (`contracts/src/intents/`) — see [[wiki/whm-intents\|whm-intents]]. Not part of the [[wiki/xc-package\|xc-package]] transfer stack (no `ConfigService`, `AssetRoute`, `Wallet`).

## Relation to the XC stack

| Package | Relationship |
|---|---|
| [[wiki/sdk-next\|sdk-next]] | hard dependency — `XcSwapOpts.sdk` is a `SdkCtx`; uses `router.getBestSell`, `client.asset`, `client.evm` (provider for rail reads, allowance) |
| [[wiki/xc-core\|xc-core]] / [[wiki/xc-sdk\|xc-sdk]] | types only (`AssetAmount`, `CallType`, `EvmCall`) |
| [[wiki/sdk-common\|sdk-common]] | `erc20.ERC20.fromAssetId()` |
| [[wiki/xc-cfg\|xc-cfg]] | none |

Peers: `common >=1.1.0`, `sdk-next >=2.0.0`, `xc-core >=2.0.0`, `xc-sdk >=2.0.0`, `viem
{ #2}
.38.3`; dep `@defuse-protocol/one-click-sdk-typescript
{ #0}
.1.17`.

## Flow (`docs/spec.md`)

```
HYDRATION  placeOrder(assetIn, amountIn, minEthOut, depositAddress, maxRelayFee) → transferSequence
             sell assetIn → WETH (router) · deduct cost = NTT deliveryPrice + wormhole messageFee
             quantize to TRIM_UNIT (1e10) · nttManager.transfer(amount, 2, IntentReceiver)
             publishMessage(abi.encode(transferSequence, depositAddress, amount, maxRelayFee))
             → OrderPlaced
ETHEREUM   IntentReceiver.processOrder(nttVaa, instructionVaa, feeRequested)
             pairs both messages on the NTT sequence, forwards (amount − feeRequested) to depositAddress
             → OrderProcessed, RelayFeePaid
NEAR       1Click solvers settle ETH → destination asset → recipient
```


## Fee model

```
wethOut    = router.getBestSell(assetIn, WETH_ID, amountIn)
cost       = nttManager.quoteDeliveryPrice(2, 0x00).total + wormhole.messageFee()
bridged    = trim(wethOut − cost)
minEthOut  = trim(padDown(bridged, slippageBps))
swapAmount = bridged − maxRelayFee        // lands at depositAddress
fee        = wethOut − swapAmount
```

`cost` is **zero on the prod rail** today (the WHM relayer is paid via `maxRelayFee` on Ethereum) but is read at runtime. Trim dust (≤ 1e10 wei per order) stays in the emitter as sweepable dust.

## Public API

`createXcSwap(opts) → XcSwapClient`: `getChains()` (`hydration`, `near`, `zec`), `getOriginAssets()`, `getDestinationAssets()` (1Click tokens ∩ allowlist), `getRoutes()`, `swap(params) → XcSwapTrade` (dry quote). `XcSwapTrade.buildCall()` requests a **firm** 1Click quote (mints the deposit address) and returns `XcSwapRequest { calls: [approve?, placeOrder], depositAddress, correlationId, deadline }`. `XcSwapTrade.settlement` exposes `wethOut`, `bridged`, `amount`, `minEthOut`, `maxRelayFee`.

### `XcSwapOpts`

| Field | Default |
|---|---|
| `sdk` | required |
| `emitter` | required — `IntentEmitter` proxy on Hydration EVM |
| `quoterUrl` | `https://quoter-intent.play.hydration.cloud` (play endpoint — override for mainnet) |
| `relayMargin` | 20 % |
| `slippage` | 1 % |
| `railTtl` | 30 000 ms |
| `destinationAssets` | `nep141:wrap.near`, `nep141:zec.omft.near` |
| `oneClick` | `{ baseUrl = https://1click.chaindefuser.com, jwt }` |

## Rail reads (`trade/rail.ts`)

`fetchRailConfig(provider, emitter)` reads `nttManager` / `wormhole` from the emitter (survives `setNttManager` / `setIntentReceiver`; cached for client lifetime). `fetchRailState()` reads `quoteDeliveryPrice`, `messageFee`, `isPaused`, `getCurrentOutboundCapacity` (cached `railTtl`). Failed reads are not cached.

## Viability errors (`XcSwapError`, reported on the trade, thrown only by `buildCall()`)

`BelowDeliveryPrice`, `BelowTrimUnit`, `RelayFeeExceedsAmount`, `RailPaused`, `RailRateLimited` (emitter pins `shouldQueue = false` → over-capacity **reverts**), `MinWethNotMet` (< 0.0004 WETH), `RelayFeeTooHigh` (`bridged < 2 × maxRelayFee`), `AmountTooLow`, `RecipientInvalid`, `QuoteFailed`.

## Tracking

Before submission the correlation key is `depositAddress` (indexed on `OrderPlaced` and `OrderProcessed`); after, `transferSequence` (decode from the receipt with `ORDER_PLACED_ABI`).

## Constants (`registry/consts.ts`)

`WETH_ID = 20`, `ETHEREUM_WORMHOLE_ID = 2`, `TRIM_UNIT = 1e10`, `NO_TRANSCEIVER_INSTRUCTIONS = '0x00'`, `MIN_WETH = 4e14`, `ONE_CLICK_ORIGIN_ASSET = 'nep141:eth.omft.near'`, `WRAP_NEAR_ASSET`, `ZEC_ASSET`.

## UI consumer

[[wiki/hydration-ui\|hydration-ui]] drives the trade page's **Market** tab (XcSwap section) through `XcSwapClient`; emitter / receiver addresses are pinned in `apps/main/src/config/xcSwap.ts` (see [[wiki/hydration-ui-modules\|hydration-ui-modules]]).

## Gotchas

- **`refundTo` is overloaded** — also the Hydration EVM `from` of both calls and the allowance owner. Pass the sender's h160.
- **Ethereum-side inbound NTT limit is not checked** (needs an Ethereum connection): an oversized settlement queues in `IntentReceiver` and `processOrder` reverts `SettlementNotReleased` until the queued transfer completes.
- Dry quote is scaled linearly from full `wethOut`; the firm quote is sized to `swapAmount`, so `amountOut` can move slightly.
- Zcash *transfers* (not swaps) live in [[wiki/xc-cfg\|xc-cfg]] (Omni corridor); 1Click is used here only for swaps.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/source-whm-codebase\|source-whm-codebase]]
