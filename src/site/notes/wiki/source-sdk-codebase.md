---
{"dg-publish":true,"permalink":"/wiki/source-sdk-codebase/","title":"Galactic SDK — Codebase","tags":["sdk","codebase","typescript","wasm","cross-chain","trading"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"Galactic SDK — Codebase","author":"Galactic Council (galacticcouncil/sdk)","source_kind":"repo_clone","raw_path":"raw/sdk/","upstream":"https://github.com/galacticcouncil/sdk","date_ingested":"2026-04-13","source_date":"2026-10-28","last_refreshed":"2026-11-02","last_commit":"d16ff85 RELEASE: Releasing 1 package(s) (2026-10-28)","prev_commit":"c57d172 apps: revert grid styles (2026-08-12)","cloned_at":"2026-11-02","produces_pages":["sdk-next","sdk-common","sdk-descriptors","xc-package","xc-sdk","xc-cfg","xc-core","xc-scan","xc-swap","route-suggester"],"tags":["sdk","codebase","typescript","wasm","cross-chain","trading"],"last_updated":"2026-11-02"}}
---


# Source: Galactic SDK — Codebase

**Raw location:** `raw/sdk/` (snapshot of `galacticcouncil/sdk` master, `.git` stripped — see `raw/sdk/.vault-clone-info`)

## Summary

**TL;DR:** TypeScript monorepo for building on [[wiki/hydration\|hydration]] — pool math, trading, cross-chain tooling — on polkadot-api v2, **17 packages** under `@galacticcouncil/*`, plus unpublished ops apps and a Chopsticks test workspace.

## Since the Aug 2026 sync (`c57d172..d16ff85`, 142 commits, 419 files)

1. **[[wiki/sdk-next\|sdk-next]] v2:** pools moved to `src/pool/amm/*` behind one event-driven sync engine (`PoolSync` + `EventBus`, best-chain, reorg-healing) — the torn-snapshot bug is fixed; new **Uniswap v3** venue (curated HOLLAR pools); multiple Aave markets (main, BIL, GIGAHDX); 2-second block time; TWAP buy removed; EVM reads batched through multicall.
2. **XC expansion:** NTT to **HyperEVM**, **Robinhood**, **NEAR**; **Zcash** via the Omni (NEAR One) corridor; new `NearChain` / `ZecChain`, `NearPlatform` / `ZecPlatform`; `NttCustodyValidation`, `OmniValidation`, `HydrationEvmValueValidation`; multichain balances in [[wiki/xc-sdk\|xc-sdk]]; inbound routes dropped for assets no longer accepted as fee currency. See [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]].
3. **[[wiki/xc-swap\|xc-swap]]** runs on an **NTT rail** (`IntentEmitter.placeOrder` → Ethereum `IntentReceiver`).
4. **Descriptors:** single whitelist file; ICE/Intent, GigaHdx(Rewards), Parameters, broad `event.*` coverage.
5. **Ops apps** (`apps/`): AMM monitor, circuit breakers, claim, dispatch, TC voting, rescue.

## Architecture

```
Your dApp
├── sdk-next ──── Trade routing (SOR), pool sync engine, venues (Omnipool, Stableswap, XYK,
│   │             LBP, HSM, Aave, Uniswap v3), DCA/TWAP, tx builders incl. ICE intents,
│   │             staking, farms, MM oracles, block indexer
│   └── math-* ── Pool math (WASM, compiled from Rust)
├── xc ────────── Cross-chain transfers (batteries-included)
│   ├── xc-sdk ── Wallet, ordered call sequences, platforms (Substrate, EVM, Solana, Sui, NEAR, Zcash)
│   ├── xc-cfg ── Chains/routes, bridge builders (XCM, Snowbridge, NTT, Omni, Basejump), validations
│   └── xc-core ─ Core types, chain & asset definitions, bridge primitives
├── xc-swap ───── Hydration → NEAR Intents swaps over the NTT rail (sibling of xc)
├── xc-scan ───── Journey tracking (standalone)
├── common ────── Shared utilities (big numbers, XCM, EVM, substrate, rxjs ops)
└── descriptors ─ papi v2 metadata for hydration / hydrationNext / hydrationIce / hub
```

## File tree (top 2 levels)

```
sdk/
├── packages/     17 packages
├── chopsticks/   Chopsticks integration tests (@galacticcouncil/chopsticks 2.3.0 fork)
│   └── src/      call.spec.ts, e2e.spec.ts, omni-tear.spec.ts, lib/ (incl. eth/), probes/nearNtt.ts, spec/
├── apps/         Unpublished ops tools (see below)
├── examples/     sdk-next-cjs, sdk-next-esm, xc-transfer
├── crates/       route-suggester (Rust)
├── scripts/      build-math, changeset-*, rpc-latency*.sh
└── CLAUDE.md     repo-level agent instructions
```

npm workspaces: `packages/*`, `examples/*`, `chopsticks`.

## Package versions (d16ff85)

| Package | Version |
|---|---|
| `@galacticcouncil/sdk-next` | 2.5.0 |
| `@galacticcouncil/descriptors` | 2.9.0 |
| `@galacticcouncil/common` | 1.2.0 |
| `@galacticcouncil/xc` | 2.1.0 |
| `@galacticcouncil/xc-cfg` | 2.9.1 |
| `@galacticcouncil/xc-core` | 2.8.0 |
| `@galacticcouncil/xc-sdk` | 2.7.0 |
| `@galacticcouncil/xc-swap` | 0.9.0 |
| `@galacticcouncil/xc-scan` | 0.5.0 |
| `math-stableswap` | 2.5.0 |
| `math-omnipool` | 1.4.0 |
| `math-ema`, `math-lbp`, `math-liquidity-mining`, `math-staking`, `math-xyk` | 1.3.0 |
| `math-hsm` | 1.2.0 |

## Packages

- **[[wiki/sdk-next\|sdk-next]]** — trading SDK (see above)
- **[[wiki/sdk-common\|sdk-common]]** (`common`) — big-number ops, XCM transforms, EVM/ERC20 helpers, substrate RPC, rxjs operators
- **[[wiki/sdk-descriptors\|sdk-descriptors]]** (`descriptors`) — papi metadata
- **[[wiki/xc-package\|xc-package]]** (`xc`) — context factory + `docs/` (`ntt.md`, `ntt-reorg-recovery.md`, `overview.md`, `chain-native-balances.md`, `unidirectional-routes.md`)
- **[[wiki/xc-sdk\|xc-sdk]]**, **[[wiki/xc-cfg\|xc-cfg]]**, **[[wiki/xc-core\|xc-core]]**, **[[wiki/xc-swap\|xc-swap]]**, **[[wiki/xc-scan\|xc-scan]]**
- **Math (WASM):** `math-omnipool`, `math-stableswap`, `math-xyk`, `math-lbp`, `math-hsm`, `math-ema`, `math-staking`, `math-liquidity-mining`
- **[[wiki/route-suggester\|route-suggester]]** — Rust crate (BFS route discovery)

## Ops apps (`apps/`, unpublished, static pages under `apps/public/*`)

| App | What it does |
|---|---|
| `amm` — "AMM Monitor" | live per-pool field diff of the [[wiki/sdk-next\|sdk-next]] sync store (reserves, scalars, pegs) — the `probeAmm` view |
| `breakers` — "Circuit Breakers" | [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] usage vs limits (`used / limit`), 2 s block time aware |
| `claim` | claim a signed NTT transfer VAA by raw wormhole chain ids |
| `dispatch` | run a substrate call via the pallet-evm **dispatch precompile**; [[wiki/pallet-dispatcher\|pallet-dispatcher]] calls hand-encoded and decoded back against live metadata (descriptors predate the running spec) |
| `tc-voting` | Technical Committee (`pallet_collective::<Instance2>`) propose / vote / close through the dispatch precompile — EVM-signed TC actions |
| `rescue` | recover funds stranded on the `ETH\0` phantom account via `EVM.call` through the dispatch precompile, gas in WETH (asset 20) |

`breakers`, `dispatch` and `tc-voting` are directly relevant to [[wiki/note-emergency-levers\|note-emergency-levers]] / [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] (TC motions and circuit-breaker state from a browser).

## Chopsticks (`chopsticks/`)

| Script | Spec | Covers |
|---|---|---|
| `spec:calldata` | `call.spec.ts` | snapshot-verified call data for every route |
| `spec:e2e` | `e2e.spec.ts` | forked-network transfers, event + storage assertions |
| `spec:omni` | `omni-tear.spec.ts` | Omnipool add-liquidity coherence under the event-driven sync (formerly the red test for the torn-snapshot bug) |

`probes/nearNtt.ts` — NEAR NTT fork probe. Uses the Galactic Council Chopsticks fork (`@galacticcouncil/chopsticks 2.3.0`).

> Root `CLAUDE.md` still says `cd integration-tests/xc-test` — that path does not exist; use `chopsticks/`.

## Examples

`examples/xc-transfer/src/`: `balances.ts` (multichain), `circuitbreaker.ts`, `near.ts`, `ntt.ts`, `redeem.ts`, `scan.ts`, `swap.ts` ([[wiki/xc-swap\|xc-swap]] demo).

## Key technical details

- TypeScript strict, esbuild, Turborepo, Jest; Node 25 (`.nvmrc`)
- ESM + CJS for all packages except xc-scan and descriptors (ESM-only)
- Math WASM fetched from an external repo
- Peers: common, descriptors, polkadot-api `^2.1.7`, viem `^2.38.3`
- Releases via Changesets (snapshot flag dropped from changeset publish)

## Cross-repo notes

- `StableSwapClient` still reads `Tokens.TotalIssuance` for share supply; [[wiki/pallet-stableswap\|pallet-stableswap]] uses `Stableswap.ShareIssuance` — see [[wiki/sdk-next\|sdk-next]].
- `Democracy` is still whitelisted in [[wiki/sdk-descriptors\|sdk-descriptors]] though hydration-node master removed the pallet.
- Uniswap v3 factory comes from `Parameters.UniswapV3Factory` on chain.
- NEAR NTT managers, `UtxoEmitter`, `IntentEmitter` / `IntentReceiver` are WHM contracts — [[wiki/source-whm-codebase\|source-whm-codebase]].

## Sources

- Raw: `raw/sdk/` @ `d16ff85` (2026-10-28) — <https://github.com/galacticcouncil/sdk>
- In-repo docs: `packages/xc/docs/`, `packages/sdk-next/docs/`, `packages/xc-swap/docs/spec.md`
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]], [[wiki/source-whm-codebase\|source-whm-codebase]]
