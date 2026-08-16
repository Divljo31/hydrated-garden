---
{"dg-publish":true,"permalink":"/wiki/sdk-common/","title":"common (SDK)","tags":["sdk","utilities","shared","rxjs"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"common (SDK)","repo":"sdk","paths":["packages/common/src/index.ts","packages/common/src/utils/index.ts","packages/common/src/utils/rx.ts","packages/common/src/substrate/entries.ts","packages/common/src/substrate/SubstrateApis.ts"],"key_exports":["acc","big","enums","erc20","h160","hex","log","meta","rx","xcm","xc","encodeAssetId","encodeLocation","transform","SubstrateApis","changedEntries","debounceAfterFirst"],"tags":["sdk","utilities","shared","rxjs"],"source_count":1,"last_updated":"2026-08-15"}}
---


# common (SDK)

**TL;DR:** `@galacticcouncil/common` (v1.2.0) provides shared utilities across the SDK monorepo — big-number ops, XCM transformations, ERC20/EVM helpers, Substrate RPC abstractions and, new this cycle, **rxjs operators for subscription hygiene**. Aligned with [[wiki/papi\|papi]] v2 (`createWsClient`); peers `polkadot-api
{ #2}
.1.7`, `rxjs
{ #7}
.8.0`.

## Modules

Namespaced re-exports from `src/utils/index.ts`: `acc`, `big`, `enums`, `erc20`, `h160`, `hex`, `log`, `meta`, **`rx`**, `xcm`, `xc` (plus flat `encodeAssetId`, `encodeLocation`, `transform`).

- **big** — BigInt/decimal conversions, power-of-10 calculations
- **xcm** — AccountId32/AccountKey20 conversion, location encoding (papi-v2 `SizedHex`/`Binary`)
- **erc20** — ERC20 helpers, incl. `ERC20.fromAssetId()` (Hydration asset id → precompile address), used by [[wiki/xc-swap\|xc-swap]]
- **h160 / hex / acc** — Ethereum address, hex and account utilities
- **rx** *(new)* — rxjs operators, see below
- **SubstrateApis** (`src/substrate/`) — endpoint probing + RPC utilities on papi-v2 `createWsClient`; `WsProviderOpts` derived from `Parameters<typeof createWsClient>[1]` (sans `getMetadata`/`setMetadata`), cached connections track a single `client: WsClient` (no separate provider)
- **log** — logging utilities

## Subscription operators (new)

Both were lifted out of [[wiki/sdk-next\|sdk-next]]'s `BalanceClient` so every client shares one implementation.

```ts
// packages/common/src/utils/rx.ts
/** Emit the first value immediately, debounce every value after it. */
export function debounceAfterFirst<T>(ms: number): MonoTypeOperatorFunction<T> {
  return connect((shared) =>
    concat(shared.pipe(take(1)), shared.pipe(debounceTime(ms)))
  );
}
```

A plain `debounceTime` delays the *first* value by the full window — pure latency; the window is only useful for coalescing the burst that follows. The second stream deliberately has no `skip(1)`: the multicast subject does not replay, so it never sees the first value and a `skip` would swallow the second one instead.

```ts
// packages/common/src/substrate/entries.ts
/** Drop `watchEntries` emissions that carry no deltas. */
export function changedEntries<T>(): MonoTypeOperatorFunction<T> {
  return distinctUntilChanged((_, curr: any) => !curr?.deltas);
}
```

`watchEntries` re-emits every block whether or not the entries moved; `changedEntries()` gates on the presence of `deltas`. `T` is intentionally unconstrained so it infers from the pipe like rxjs' own operators — constraining it to a `deltas` shape would pin `T` and erase the caller's emission type.

## Role in the Stack

Peer dependency of [[wiki/sdk-next\|sdk-next]] (`>=1.2.0`), [[wiki/xc-core\|xc-core]], [[wiki/xc-cfg\|xc-cfg]] and [[wiki/xc-swap\|xc-swap]] (`>=1.1.0`). Foundation layer below all domain logic — check downstream impact before modifying. Runtime deps: `big.js`, `lru-cache`.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
