---
{"dg-publish":true,"permalink":"/wiki/whm-contracts/","title":"WHM — Contract address book & admin map (prod)","tags":["whm","deployments","addresses","admin","upgradeability","safe","emergency-admin","sputnik-dao","security"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — Contract address book & admin map (prod)","repo":"whm","paths":["deployments/prod","deployments/archive","migrations/envs/prod","contracts/src/MessageEmitter.sol","contracts/src/MessageReceiver.sol","contracts/src/utils/hydration/HydrationConsts.sol"],"key_exports":["MessageEmitter","MessageReceiver","_authorizeUpgrade","setOwner","HydrationConsts"],"tags":["whm","deployments","addresses","admin","upgradeability","safe","emergency-admin","sputnik-dao","security"],"last_updated":"2026-11-02"}}
---


# WHM — Contract address book & admin map

**TL;DR:** Every prod WHM deployment as recorded in `deployments/prod/*.json` (state at `36ae52b`, 2026-10-21), with its **current** owner. All EVM contracts are UUPS proxies whose `_authorizeUpgrade` is the owner check (governance contracts excepted), so **owner = upgrade authority**, and a renounced owner (`0x0`) means the code is frozen forever. No WHM contract has a `pause()`. Read live state before relying on this — owners can change after the last migration record.

## Owner legend

| Owner | Who | Speed |
|---|---|---|
| **`EmergencyAdminAccount` `0xaa7e0000000000000000000000000000000aa7e1`** | Hydration TC majority via `dispatch_as_emergency_admin` ([[wiki/pallet-dispatcher\|pallet-dispatcher]]) | TC motion |
| **TC Safe `0xD557AeAf1e0cB3D226BfF3B7a10C2cdA9dA081E7`** | Gnosis Safe 4-of-7, same address on Ethereum / Base / Robinhood / HyperEVM; signers in [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]] | off-chain signers |
| `CrossChainGovernanceAccount` `0xaa7e…aa7e2` | OpenGov Root only (via `Utility::dispatch_as`) | referendum |
| `hydration.sputnik-dao.near` | Sputnik DAO on NEAR | DAO proposal |
| `0x0` (renounced) | nobody — frozen | — |
| `0x71FeB8b2849101a6E62e3369eaAfDc6154CD0Bc0` | Hydration-side deployer EOA (deploy-time only; transferred/renounced afterwards in every prod record) | — |
| `0x6d116C3E43Bc1bFd4EEBe5c7BF043a342Ea6bBB2` | Source-chain deployer EOA (same) | — |
| `6db8TCH51fK4Pq2LkBdMaMFPfXD17GjfLD8TaMPWNuiq` | Solana deployer — **still `Config.owner`** of `oracle-emitter` (feed creation only) | — |

## Hydration (EVM 222222 / Wormhole 73)

| Contract | Address | Owner (= upgrade) | Can move funds / push data? |
|---|---|---|---|
| Wormhole core | `0x3792a6d63c31941b2805181771795d9176fa82a1` | Wormhole (guardian set 7, 19 keys) | verifies all VAAs |
| `OracleReceiver` (Solana) | `0x582e2fac5af62dc024396b5e7f549c72273a69c3` | **renounced** | `setPrice` on PRIME / SOL / jitoSOL oracles |
| `OracleReceiver` (Ethereum) | `0x6913770466fed4dbc24337cd7f1ae92af4321083` | **renounced** | `setPrice` on wstETH / apyUSD oracles |
| `OracleReceiver` (Robinhood) | `0x3b6e3469d8e64e306f235838e0fdf7b4d60a39ce` | **renounced** | `setPrice` on SPY oracle |
| `OracleEmitter` (HDX → Robinhood) | `0x3f5cc44141a52529323f9be42dbb98fda7c1d066` | **renounced** | publishes HDX price |
| `AggregatorAdapter` (EMA HDX feed) | `0xbc6758319ecacf59142de0d6e9d945ab0ac0856f` | immutable | — |
| `BasejumpReceiver` | `0x35bf3a1b9ac564c8f66c97cea1ee410cd3f97c8a` | EmergencyAdmin | instructs pool payouts |
| `BasejumpLanding` | `0x70e9b12c3b19cb5f0e59984a5866278ab69df976` | EmergencyAdmin | **holds the USDC pool**; `withdraw` |
| `IntentEmitter` | `0x98f1ebc9dcc8ab7ba54d83c98500e9e313f793f2` | EmergencyAdmin | swaps + NTT; `sweep` |
| `ZecUtxoEmitter` | `0xb3e365a6241683d5d4ef2701de6bedc16dcc259e` | EmergencyAdmin | burns ZEC via NTT; `sweep` |
| `GovernanceDispatcher` | `0x00302167f9597b9d90bd342311c5c4be39bf0fa0` | governance caller `…aa7e2` (no owner) | publishes cross-chain actions |
| NTT managers / transceivers | see [[wiki/whm-agents\|whm-agents]] | NTT repo (not here) | mint/burn |

## Ethereum (Wormhole 2)

| Contract | Address | Owner |
|---|---|---|
| `OracleEmitter` (wstETH, apyUSD) | `0xfbf682642a6a28760e717b637f12d014bd5db4b9` | **renounced** |
| `BasejumpEmitter` | `0xa72e2bf29c840eb93adbb9ee1aa41580f01c9944` | TC Safe |
| `IntentReceiver` | `0x2173f6ece25768e7efc5199f70f8783d88ba63c8` | TC Safe (relayer allowlist: `0x43A8…cD5F`) |
| `GovernanceExecutor` | `0xCEaab5A5AF21141a501FF5b561b6Cc3d4c55eF83` | self (vetoer TC Safe) |
| USDC NttManager (Basejump rail) | `0x447b2c7485A3d6813F8197E605b10BcCD8dd8398` | NTT repo — owner+pauser = TC Safe per [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] |

## Robinhood (Wormhole 72, EVM 4663)

| Contract | Address | Owner |
|---|---|---|
| `OracleEmitter` (SPY) + `AggregatorAdapter` `0xb01d3f48…3649` | `0x9fbd9f16ce7fa17097e91cc36dc1b7b47adca9de` | **renounced** |
| `OracleReceiver` (HDX) | `0x060f1ef6bb1c7ab31d2bbc3d2ae47e590952b958` | **renounced** → oracle `0xD1a2198C…901D` |
| `GovernanceExecutor` | `0xceaab5a5af21141a501ff5b561b6cc3d4c55ef83` | self (vetoer TC Safe) |

## Base (30) / HyperEVM (47)

| Contract | Address | Owner |
|---|---|---|
| `GovernanceExecutor` (Base) | `0x8830b6eb2e015184b86cfa3a0a759cd55ff47a4b` | self (vetoer TC Safe) |
| `GovernanceExecutor` (HyperEVM) | `0x102225852eca1937ae57d2fb23409877aee50699` | self (vetoer TC Safe) |

## Solana

| Program | Address | Authority |
|---|---|---|
| `oracle-emitter` | `2knjvgqZKtU3nV1r9RaLshS9JYWMerZ1ysSFcDyTVsdn` (emitter PDA `3FVzVAw8…jwJ2`) | upgrade authority **revoked**; `Config.owner` = `6db8TCH5…uiq` |

## NEAR

| Contract | Owner |
|---|---|
| `ntt-near.hydration-whm.near`, `ntt-zec.hydration-whm.near`, `withdraw-zec.hydration-whm.near` | `hydration.sputnik-dao.near`; deployer access keys deleted |

## Archived deployments

Everything under `deployments/archive/` is retired — ignore it. Basejump is now a direct Ethereum → Hydration route ([[wiki/whm-basejump\|whm-basejump]]).

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-oracle-relay\|whm-oracle-relay]], [[wiki/whm-basejump\|whm-basejump]], [[wiki/whm-intents\|whm-intents]], [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]], [[wiki/whm-near-corridors\|whm-near-corridors]], [[wiki/whm-agents\|whm-agents]]
- [[wiki/note-emergency-levers\|note-emergency-levers]], [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]
