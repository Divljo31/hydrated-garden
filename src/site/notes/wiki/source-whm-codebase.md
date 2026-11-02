---
{"dg-publish":true,"permalink":"/wiki/source-whm-codebase/","title":"WHM codebase (galacticcouncil/whm)","tags":["whm","wormhole","ntt","bridge","oracle","intents","governance","near","solana","solidity","source"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"WHM codebase (galacticcouncil/whm)","source_kind":"repo_clone","raw_path":"raw/whm/","upstream":"https://github.com/galacticcouncil/whm","cloned_at":"2026-11-02","repo":"whm","paths":["README.md","AGENTS.md","contracts/src","crates/solana/programs/oracle-emitter","crates/near/contracts","agents","migrations","deployments/prod","deployments/archive/README.md","docs"],"produces_pages":["whm-contracts","whm-oracle-relay","whm-basejump","whm-intents","whm-cross-chain-governance","whm-near-corridors","whm-agents","wormhole"],"tags":["whm","wormhole","ntt","bridge","oracle","intents","governance","near","solana","solidity","source"],"last_updated":"2026-11-02"}}
---


# Source — WHM codebase (`galacticcouncil/whm`)

**TL;DR:** WHM ("Wormhole Messaging") is Hydration's own Wormhole-side monorepo at `36ae52b` (master, 2026-10-21): Solidity contracts (Foundry, UUPS), a Solana Anchor program, two NEAR contracts, and five off-chain agent services. It carries every non-token Wormhole flow Hydration runs — **oracle price relay into Hydration**, **Basejump** instant USDC inbound, **Intents** (Hydration → NEAR Intents via WETH/NTT), **cross-chain OpenGov executors**, and **NEAR / Zcash corridors**. Token NTT managers themselves live in a separate repo (`native-token-transfers/ops/tokens/*/deployment.json`, referenced by the relayer) — WHM only consumes them. Archived deployments are dead.

## File tree (top levels)

```
whm/
├── contracts/            # @whm/contracts — Foundry
│   └── src/
│       ├── MessageEmitter.sol / MessageReceiver.sol   # base UUPS + VAA verify, replay, authorized emitter
│       ├── basejump/      # BasejumpEmitter (Eth), BasejumpReceiver + BasejumpLanding (Hydration)
│       ├── intents/       # IntentEmitter (Hydration), IntentReceiver (Eth), IntentQuoteEmitter, IntentCodec
│       ├── oracles/       # OracleEmitter, OracleReceiver, adapters/AggregatorAdapter(+V3)
│       ├── governance/    # GovernanceDispatcher (Hydration), GovernanceExecutor (EVM dests), GovernanceCodec
│       ├── utxo/          # ZecUtxoEmitter (Hydration)
│       ├── ntt/           # INttManager / IWormholeTransceiver interfaces, NttPayload
│       └── utils/         # ScaleCodec, XcmV4, Blake2b, DerivedAccount, hydration/{HydrationConsts,HydrationRouter,HydrationPolkadotXcm,HydrationUtility}
├── crates/
│   ├── solana/programs/oracle-emitter/   # Anchor: Kamino Scope prices + SPL stake-pool rates → VAA
│   └── near/contracts/{ntt-manager, utxo-withdraw}
├── agents/               # broadcaster, relayer, scan, intent, utxo (TypeScript)
├── migrations/           # actions/ + definitions/ + envs/{fork,prod,testnet}
├── deployments/          # prod/*.json state files, archive/ (dead), testnet/
├── docs/                 # basejump, intents, intents-v2 (draft), oracle, governance, near-ntt, near-utxo, scan, agents
├── chopsticks/           # fork probes
└── sh/                   # fork-*, migrate-*, verify-*, print-agents.sh
```

## Inventory → pages

| Unit | What | Page |
|---|---|---|
| Oracle relay | Solana (Kamino Scope, Jito stake pool), Ethereum (wstETH, apyUSD), Robinhood (SPY) → `OracleReceiver` on Hydration → per-asset oracle `setPrice`; plus Hydration HDX → Robinhood | [[wiki/whm-oracle-relay\|whm-oracle-relay]] |
| Basejump | Ethereum → Hydration instant USDC: NTT settlement (gross) + fast message (net) paid from a pre-funded landing pool | [[wiki/whm-basejump\|whm-basejump]] |
| Intents | Hydration `placeOrder` → sell to WETH → NTT to Ethereum → `IntentReceiver` forwards ETH to a NEAR 1Click deposit address; v2 (ICE-driven, own strategy) is a draft | [[wiki/whm-intents\|whm-intents]] |
| Cross-chain governance | OpenGov Root → `GovernanceDispatcher` on Hydration → Wormhole → `GovernanceExecutor` on Ethereum/Base/Robinhood/HyperEVM, TC Safe veto / fast-track / 30-day recovery | [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]] |
| NEAR corridors | NEAR NTT (NEAR, ZEC) one-hop; ZEC UTXO deposit/withdraw via Omni + NEAR MPC | [[wiki/whm-near-corridors\|whm-near-corridors]] |
| Agents | broadcaster (oracle publish), relayer (ntt, ntt-near, oracle, oracle-robinhood, basejump, intent, psm), scan, intent (quoter/relayer), utxo (api/relayer) | [[wiki/whm-agents\|whm-agents]] |
| Address book | Every prod deployment with owner / upgrade authority | [[wiki/whm-contracts\|whm-contracts]] |

## Deployments summary (`deployments/prod/`, as committed)

| Migration | Completed | Chains | Final owners |
|---|---|---|---|
| `oracle-relay-solana` | 2026-09-22 | Solana → Hydration | program upgrade authority **revoked**; receiver **renounced** |
| `oracle-relay-ethereum` | 2026-09-22 | Ethereum → Hydration | emitter + receiver **renounced** |
| `oracle-relay-robinhood` | 2026-10-12 | Robinhood → Hydration (SPY) | emitter + receiver **renounced** |
| `oracle-relay-hydration` | 2026-10-12 | Hydration → Robinhood (HDX) | emitter + receiver **renounced** |
| `basejump-ethereum` | 2026-08-28 | Ethereum → Hydration | Eth emitter → TC Safe; Hydration receiver → `EmergencyAdminAccount` |
| `basejump-landing-upgrade` | 2026-09-17 | Hydration | landing proxy owner `EmergencyAdminAccount` |
| `intent` | 2026-09-09 | Hydration → Ethereum | Hydration emitter → `EmergencyAdminAccount`; Eth receiver → TC Safe |
| `governance-hydration` | 2026-10-15 | Hydration | no owner; upgrade only by governance caller `0xaa7e…aa7e2` |
| `governance-{ethereum,robinhood,base,hyper-evm}` | 2026-10-15 … 10-21 | destinations | self-upgrade only; vetoer = TC Safe |
| `near-ntt-near`, `near-ntt-zec` | 2026-10-12 | NEAR | `hydration.sputnik-dao.near`, deployer keys deleted |
| `utxo-emitter-zec` | 2026-10-19 | Hydration | `EmergencyAdminAccount` |
| `utxo-withdraw-zec` | 2026-10-20 | NEAR | `hydration.sputnik-dao.near`, keys deleted |

Not in `prod/`: PSM (relayer app exists for `HollarBaseVault` / `HollarBaseFacilitator`, contracts not on master), `IntentQuoteEmitter` / `intent-mpc` (definition only).

`deployments/archive/` = write-once records of retired deployments — all dead, "disarmed rather than deleted".

## Identifiers

Hydration has three ids: Wormhole chain **73**, EVM chain **222222**, para id **2034** (`HydrationConsts.sol`). Hydration Wormhole core: `0x3792a6d63c31941b2805181771795d9176fa82a1`. Dispatch precompile `0x0401`; asset ERC-20 precompile = `0x0100000000 | assetId`; `WETH_ID = 20` (EVM gas asset). Router calls are SCALE-encoded `pallet 67` (`router.sell` = 0, `buy` = 1) dispatched through `0x0401` — see [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/hydration-precompiles\|hydration-precompiles]].

## Sources

- `raw/whm/` at `36ae52b` (`.vault-clone-info`)
- [[wiki/wormhole\|wormhole]], [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]
