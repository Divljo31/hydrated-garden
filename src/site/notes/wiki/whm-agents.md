---
{"dg-publish":true,"permalink":"/wiki/whm-agents/","title":"WHM — Off-chain agents (broadcaster, relayer, scan, intent, utxo)","tags":["whm","agents","relayer","broadcaster","keeper","wormhole","ntt","oracle","monitoring","ops"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — Off-chain agents (broadcaster, relayer, scan, intent, utxo)","repo":"whm","paths":["docs/agents.md","sh/print-agents.sh","agents/broadcaster/src/apps","agents/relayer/src/apps","agents/relayer/src/apps/ntt/routes.ts","agents/relayer/src/apps/psm/config.ts","agents/scan/src/apps","agents/intent/src/apps","agents/utxo/src/apps"],"key_exports":["broadcaster","relayer","scan","intent","utxo","ROUTES","NttRoute","print-agents"],"tags":["whm","agents","relayer","broadcaster","keeper","wormhole","ntt","oracle","monitoring","ops"],"last_updated":"2026-11-02"}}
---


# WHM — Off-chain agents

**TL;DR:** Five TypeScript services that keep the WHM corridors moving. **None holds protocol authority** — every call they make (oracle `send`, VAA submission, `processOrder`) is permissionless; they are liveness, not trust. Losing them stops oracle updates and delays deliveries; it cannot mint or move funds on its own. `pnpm print:agents` discovers each bot wallet from chain activity and reports balance + runway (WARNs on shared signers competing for nonces).

## Services

| Service | Apps | Does |
|---|---|---|
| `broadcaster` | `oracle-solana`, `oracle-ethereum`, `oracle-robinhood`, `oracle-hydration` | Calls the source `OracleEmitter` / Anchor `send_price`/`send_rate` on a `thresholds.json` change-detect loop + full refresh — [[wiki/whm-oracle-relay\|whm-oracle-relay]] |
| `relayer` (wormhole relayer-engine, Redis/BullMQ) | `ntt`, `ntt-near`, `oracle`, `oracle-robinhood`, `basejump`, `intent`, `psm` | Submits VAAs to Hydration transceivers / receivers, to Robinhood `OracleReceiver`, `IntentReceiver.processOrder` on Ethereum; `ntt-near` redeems NTT on NEAR. `psm` = HOLLAR Base vault ↔ Hydration facilitator corridor (contracts not on master yet) |
| `scan` | `feeds`, `intents`, `transfers` (basejump, utxo) | Indexer + UI: oracle feed history, intent lifecycle (`placed / processed / settled / refunded / failed`), transfer flows |
| `intent` | `quoter`, `relayer` | Prices `maxRelayFee` for the SDK; calls 1Click `submitDepositTx` on each `OrderProcessed` — [[wiki/whm-intents\|whm-intents]] |
| `utxo` | `api`, `relayer` | Derives withdraw account `A`; calls `utxo-withdraw.withdraw` and submits the MPC-signed tx — [[wiki/whm-near-corridors\|whm-near-corridors]] |

## NTT routes the relayer serves (Hydration side)

From `agents/relayer/src/apps/ntt/routes.ts` (sourced from the separate NTT repo's `ops/tokens/*/deployment.json`). Hydration-side managers are what [[wiki/note-dependency-registry\|note-dependency-registry]] should key Wormhole assets on.

| Token | Source chain | Hydration NttManager | Hydration transceiver |
|---|---|---|---|
| DAI | Ethereum | `0xcFd576F88C90844AEBF45378Fd09931281D8b14d` | `0xe8660CA4…01eA` |
| WBTC | Ethereum | `0x6BFca089916c045b0Ca4A09B655aF9F926189993` | `0x9a8a1ab2…187F` |
| WETH | Ethereum (+ Robinhood) | `0xB5cEf790D52A57fa619eD96eDd64c5328F3DCFb7` | `0x8acce9CA…00ae` |
| USDC | Ethereum | `0xEcEab64542A875C4472671D9Ed1E690cdD4e28fC` | `0x0d7488B3…eD97` |
| USDT | Ethereum | `0x5E6C488103b47F804824AE15861638af4C436795` | `0xd2a16B73…Ac0F` |
| sUSDS | Ethereum | `0x1973E7044d9A7C7bB2d6ea1693A296a9e4B7E448` | `0x68Ecadd7…00cb` |
| EURC | Base | `0x8dd1286A29dF5a2785FB638d6fB1598144Cfbc4C` | `0x2e84fac3…7375` |
| SOL | Solana | `0x9e200C0f28D92D296b201D96C8269d3CAFFfA9FF` | `0x2F04AcF2…3202` |
| jitoSOL | Solana | `0xcE73C15B9ED02413066DE5B904A36F8e8f9B5331` | `0xF38D9C3b…7B04` |
| PRIME | Solana | `0xFCaF4aA069C565d25539028970703F01e47D3E0B` | `0x4e7b1E55…41D1` |
| SUI | Sui | `0x978443f00cAB6b09445140321EC73a221ebFF5F8` | `0xa224d6f4…2af7` |
| NEAR | NEAR | `0x374E5239a2342Be6Bf31B410EC03984D03A8b7dE` | `0xc74BcE0b…81d1` |
| ZEC | NEAR | `0xe20cED5f5798cD4FeBc7aE182309b9f51345e0cC` | `0x0dd43A80…DE85` |

Tracked by `print-agents.sh` but **not** in relayer routes: HYPE ← HyperEVM (`0xe74f52d5…c160`), SPY ← Robinhood (`0x9faf16e5…97aa`), **HDX → Robinhood** (`0xdde268d8…46b9`, locking), **HOLLAR → Robinhood** (`0xe1d75c3c…4799`, locking). HDX/HOLLAR lock on Hydration and mint as spokes on Robinhood.

## Trust & admin

No agent key has an on-chain role, with one exception: the intent relayer `0x43A8A728e2514EDEAe8e0C9FAb8B79fb5E4ecD5F` is on `IntentReceiver`'s allowlist (5-minute exclusivity, then permissionless). Failure modes are liveness: stale oracles (receivers reject anything older than 300 s, so a dead broadcaster/relayer makes Hydration oracles **stop updating**, not go wrong), queued Basejump payouts, undelivered intents. Retry budgets span days (psm: 250 attempts ≈ 5.1 days); a job that exhausts it parks in `failed` and needs a manual `receiveMessage` of the same bytes, which anyone may send.

Not covered by `print-agents`: the Gamma keeper and the DIA EVM bots (both live elsewhere).

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-contracts\|whm-contracts]], [[wiki/whm-oracle-relay\|whm-oracle-relay]], [[wiki/whm-basejump\|whm-basejump]], [[wiki/whm-intents\|whm-intents]], [[wiki/whm-near-corridors\|whm-near-corridors]]
