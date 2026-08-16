---
{"dg-publish":true,"permalink":"/wiki/routing/","title":"Task Routing","dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"routing","title":"Task Routing","last_updated":"2026-08-15"}}
---


# Routing

Task → pages cheat sheet for Claude Code working on [[wiki/hydration\|hydration]]. When starting a task, find the best-matching entry and read the listed pages first.

## Trading / AMM

- "Work on Omnipool swap / fee math" → [[wiki/pallet-omnipool\|pallet-omnipool]], [[omnipool\|omnipool]], [[wiki/dynamic-fees\|dynamic-fees]], [[wiki/impermanent-loss\|impermanent-loss]], [[wiki/price-barrier\|price-barrier]], [[wiki/circuit-breaker\|circuit-breaker]]
- "Add or list a new asset on Omnipool" → [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/tradability-flags\|tradability-flags]], [[wiki/opengov\|opengov]]
- "Debug liquidity add / remove" → [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/nft-lp-positions\|nft-lp-positions]], [[wiki/price-barrier\|price-barrier]], [[wiki/impermanent-loss\|impermanent-loss]], [[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]]
- "Work on Stableswap (Curve-style) pools" → [[wiki/pallet-stableswap\|pallet-stableswap]], [[wiki/stableswap\|stableswap]]
- "Work on XYK constant-product pools" → [[wiki/pallet-xyk\|pallet-xyk]], [[wiki/xyk-pools\|xyk-pools]]
- "Launch a token via LBP" → [[wiki/pallet-lbp\|pallet-lbp]], [[wiki/lbp\|lbp]]
- "Route a multi-hop trade on-chain" → [[wiki/pallet-route-executor\|pallet-route-executor]], [[wiki/smart-order-router\|smart-order-router]]
- "Schedule DCA / TWAP" (note: **buy schedules can no longer be created** — `Error::NoLongerSupported`; existing ones keep executing) → [[wiki/pallet-dca\|pallet-dca]], [[wiki/dca\|dca]]
- "OTC / peer-to-peer trades" → [[wiki/pallet-otc\|pallet-otc]], [[wiki/otc-trading\|otc-trading]]
- "ICE intent engine" → [[wiki/ice\|ice]]
- "Where do trade fees actually go / change the fee split" → [[wiki/pallet-fee-processor\|pallet-fee-processor]], [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/dynamic-fees\|dynamic-fees]]

## Trading SDK (TypeScript)

- "Build a trade in the SDK" → [[wiki/sdk-next\|sdk-next]], [[wiki/smart-order-router\|smart-order-router]], [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- "Debug a pool-math discrepancy" → [[wiki/sdk-next\|sdk-next]], [[wiki/source-sdk-codebase\|source-sdk-codebase]] (math-* WASM packages)
- "Add a new pool type to the router" → [[wiki/sdk-next\|sdk-next]], [[wiki/smart-order-router\|smart-order-router]], [[wiki/route-suggester\|route-suggester]]
- "Use SDK from a new frontend" → [[wiki/sdk-next\|sdk-next]], [[wiki/sdk-descriptors\|sdk-descriptors]], [[wiki/sdk-common\|sdk-common]], [[wiki/papi-getting-started\|papi-getting-started]]
- "Compute a trade route from an indexed snapshot (no live RPC)" → [[wiki/sdk-next\|sdk-next]] (`SnapshotPoolCtxProvider`), [[wiki/smart-order-router\|smart-order-router]]
- "Read pool state at a historical block / replay a swap" → [[wiki/sdk-next\|sdk-next]] (`createSdkContext` `at` option)
- "Simulate a farm reward claim before submitting" → [[wiki/sdk-next\|sdk-next]] (`RewardClaimSimulator`)
- "Route a trade through HSM (HOLLAR) or Aave (aTokens)" → [[wiki/sdk-next\|sdk-next]], [[wiki/pallet-hsm\|pallet-hsm]], [[wiki/hollar\|hollar]], [[wiki/hydration-borrow\|hydration-borrow]]
- "Submit an ICE intent (market / limit / DCA)" → [[wiki/sdk-next\|sdk-next]], [[wiki/sdk-descriptors\|sdk-descriptors]], [[wiki/ice\|ice]]
- "Bulk-scan historical blocks / build an indexer on top of the SDK" → [[wiki/sdk-next\|sdk-next]]
- "Run the Chopsticks integration tests" → [[wiki/source-sdk-codebase\|source-sdk-codebase]]

## polkadot-api (papi)

- "Set up papi in a new project / first client" → [[wiki/papi-getting-started\|papi-getting-started]], [[wiki/papi-codegen\|papi-codegen]], [[wiki/papi-providers\|papi-providers]]
- "Read storage / call a runtime API / decode an event" → [[wiki/papi-typed-api\|papi-typed-api]], [[wiki/papi-client\|papi-client]]
- "Build and submit a transaction" → [[wiki/papi-typed-api\|papi-typed-api]], [[wiki/papi-signers\|papi-signers]]
- "Regenerate Hydration descriptors / add a new pallet to the whitelist" → [[wiki/sdk-descriptors\|sdk-descriptors]], [[wiki/papi-codegen\|papi-codegen]]
- "Work with chain types (SS58, Binary, Enum, Option, Result)" → [[wiki/papi-types\|papi-types]], [[wiki/papi-typed-codecs\|papi-typed-codecs]]
- "Pick a provider (smoldot vs WebSocket) / add provider enhancers" → [[wiki/papi-providers\|papi-providers]]
- "Use browser-extension signer / build a raw signer from a seed" → [[wiki/papi-signers\|papi-signers]]
- "Sign a tx offline / airgap / serverless" → [[wiki/papi-offline\|papi-offline]]
- "Dynamic chain access without codegen" → [[wiki/papi-unsafe-static\|papi-unsafe-static]]
- "Integrate with ink! contracts" → [[wiki/papi-ink\|papi-ink]], [[wiki/papi-sdks\|papi-sdks]]
- "Use a papi plugin SDK (staking, multisig, accounts, governance, statement)" → [[wiki/papi-sdks\|papi-sdks]]
- "Common patterns (transfer, multi-chain client, runtime upgrade, metadata cache)" → [[wiki/papi-recipes\|papi-recipes]]

## Cross-chain

- "Add / modify a cross-chain route" → [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xcm\|xcm]]
- "Move assets from Ethereum" → [[wiki/snowbridge\|snowbridge]], [[wiki/wormhole\|wormhole]], [[wiki/xc-sdk\|xc-sdk]]
- "Bridge a token via NTT / Basejump / Snowbridge V1 or V2 (multi-bridge selection)" → [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-package\|xc-package]]
- "Track a pending cross-chain transfer" → [[wiki/xc-scan\|xc-scan]]
- "Cross-chain from SDK frontend" → [[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]]
- "EVM interop (MetaMask, Solidity)" → [[wiki/pallet-frontier\|pallet-frontier]]
- "Check whether a cross-chain deposit will be rejected by Hydration's circuit breaker" → [[wiki/xc-cfg\|xc-cfg]] (`HydrationDepositLimitValidation` / `HydrationWithdrawLimitValidation`), [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]]
- "Bridge something over NTT (Ethereum / Base / Solana / Sui)" → [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-package\|xc-package]]
- "My NTT transfer is stuck and the executor never delivered it" → [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-sdk\|xc-sdk]]
- "Check an NTT rate limit before I fire the transfer" → [[wiki/xc-cfg\|xc-cfg]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]
- "Swap a Hydration asset into something in the NEAR ecosystem" → [[wiki/xc-swap\|xc-swap]], [[wiki/sdk-next\|sdk-next]]

## Lending & stablecoin

- "Debug HOLLAR peg / HSM" → [[wiki/hollar\|hollar]], [[wiki/pallet-hsm\|pallet-hsm]], [[wiki/stableswap\|stableswap]]
- "Work on borrowing / liquidations" → [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/pallet-frontier\|pallet-frontier]]
- "Use Omnipool LP as collateral" → [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/nft-lp-positions\|nft-lp-positions]]
- "Why didn't this liquidation fire?" → [[wiki/pepl-worker\|pepl-worker]], [[wiki/pallet-liquidation\|pallet-liquidation]], [[wiki/gigahdx\|gigahdx]]
- "Liquidate a GigaHDX position / protocol-funded liquidation" → [[wiki/pallet-liquidation\|pallet-liquidation]], [[wiki/gigahdx\|gigahdx]], [[wiki/hydration-borrow\|hydration-borrow]]

## Yield / strategy tokens

- "Understand GDOT / GETH / GSOL composition" → [[wiki/gdot\|gdot]], [[wiki/geth\|geth]], [[wiki/gsol\|gsol]]

## Governance & tokenomics

- "Governance referendum / parameter change" → [[wiki/opengov\|opengov]], [[wiki/hdx\|hdx]]
- "HDX staking rewards" → [[wiki/pallet-staking\|pallet-staking]], [[wiki/bonding-curve\|bonding-curve]]
- "Stake / unstake GigaHDX, or migrate someone off legacy staking" → [[wiki/gigahdx\|gigahdx]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-api\|hydration-ui-api]]
- "How are referendum rewards calculated and claimed?" → [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]], [[wiki/opengov\|opengov]]
- "Referral system" → [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/referrals\|referrals]]
- "Issue a new bond (now requires Treasurer track or Root)" → [[wiki/pallet-bonds\|pallet-bonds]], [[wiki/opengov\|opengov]]
- "Set an on-chain identity / request a registrar judgement (reg_index, max_fee)" → [[wiki/runbook-request-judgement\|runbook-request-judgement]], [[wiki/hydration-runtime\|hydration-runtime]]
- "Which registrars exist / how does identity verification work" → [[wiki/runbook-request-judgement\|runbook-request-judgement]]

## Risk / infrastructure

- "Adjust risk parameters (caps, thresholds)" → [[wiki/circuit-breaker\|circuit-breaker]], [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]], [[wiki/price-barrier\|price-barrier]], [[wiki/dynamic-fees\|dynamic-fees]], [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]]
- "EMA oracle lookup / config" → [[wiki/ema-oracle\|ema-oracle]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]]
- "Protocol-owned liquidity (POL)" → [[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]]
- "Tradability flags / pause an asset" → [[wiki/tradability-flags\|tradability-flags]], [[wiki/pallet-omnipool\|pallet-omnipool]]
- "Transaction fee payment in alt assets" → [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]]

## Frontend (`hydration-ui`)

- "Get oriented in the UI monorepo" → [[wiki/hydration-ui\|hydration-ui]], [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]]
- "Add or modify a route / module in the main app" → [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-modules\|hydration-ui-modules]]
- "Add a new domain data hook (react-query)" → [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/papi-typed-api\|papi-typed-api]], [[wiki/sdk-next\|sdk-next]]
- "Work on the trade UI" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (trade section), [[wiki/sdk-next\|sdk-next]], [[wiki/smart-order-router\|smart-order-router]]
- "Work on the liquidity UI" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (liquidity section), [[wiki/nft-lp-positions\|nft-lp-positions]], [[wiki/pallet-omnipool\|pallet-omnipool]]
- "Work on the borrow UI" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (borrow section), [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hydration-borrow\|hydration-borrow]]
- "Work on the XCM transfer UI" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (xcm section), [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-package\|xc-package]], [[wiki/xc-cfg\|xc-cfg]]
- "Add a new bridge route in the UI / multi-bridge selector" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (xcm/transfer `BridgeSelector`), [[wiki/hydration-ui-utils\|hydration-ui-utils]] (basejumpscan), [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] (icons), [[wiki/xc-cfg\|xc-cfg]]
- "Investigate Basejump bridge scan history / SSE journey timeline" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (xcm/history), [[wiki/hydration-ui-utils\|hydration-ui-utils]] (basejumpscan), [[wiki/hydration-ui-indexer\|hydration-ui-indexer]]
- "Work on the wallet / assets UI" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (wallet section), [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]
- "Transaction signing / submission flow" → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/papi-signers\|papi-signers]]
- "Sign an EVM permit and submit it as a papi extrinsic" → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] (`useSignAndSubmit`), [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] (`EthereumSigner.getPermit`)
- "Review a pending multisig transaction in the UI / track multisig approvals" → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]]
- "Add or integrate a new wallet" → [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/papi-signers\|papi-signers]]
- "Style / component additions" → [[wiki/hydration-ui-design-system\|hydration-ui-design-system]]
- "Add a top-of-page banner / app announcement" → [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] (`BannerTop`), [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] (`banners.ts`), [[wiki/hydration-ui-modules\|hydration-ui-modules]] (`NewFarmsBanner`)
- "Subscribe to a chain storage value with rxjs/sharing" → [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] (`usePapiValue`)
- "Generate / update GraphQL types (three clients now: indexer, squid, multix — the Snowbridge client was deleted)" → [[wiki/hydration-ui-indexer\|hydration-ui-indexer]]
- "Historical data / analytics queries" → [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-modules\|hydration-ui-modules]] (stats section)
- "Build a yield strategy page — BIL vault or Hollar bonds" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (strategies section), [[wiki/hollar\|hollar]], [[wiki/pallet-otc\|pallet-otc]]
- "Wire up a CEX or fiat deposit / withdraw flow" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (onramp section), [[wiki/xc-cfg\|xc-cfg]]
- "Add a lending market to the money-market UI" → [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/hydration-borrow\|hydration-borrow]]
- "Add a chart (squid → Grafana SQL)" → [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-design-system\|hydration-ui-design-system]]
- "Work on the address book / contacts" → [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]
- "Render long-form MDX content" → [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]]

## Runtime / node-level

- "Understand how pallets are wired" → [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- "EVM precompiles" → [[wiki/hydration-precompiles\|hydration-precompiles]], [[wiki/pallet-frontier\|pallet-frontier]]
- "Call the lock-manager precompile / why won't GIGAHDX transfer?" → [[wiki/hydration-precompiles\|hydration-precompiles]], [[wiki/pallet-gigahdx\|pallet-gigahdx]]
- "XCM config on the node" → [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/xcm\|xcm]]
- "Runtime upgrade / migration" → [[wiki/hydration-runtime\|hydration-runtime]]

## Operations / collators

- "Run a Hydration collator (machine, rewards, onboarding)" → [[wiki/runbook-run-collator\|runbook-run-collator]], [[wiki/pallet-collator-rewards\|pallet-collator-rewards]], [[wiki/pallet-collator-rotation\|pallet-collator-rotation]], [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/opengov\|opengov]]
- "How are collators paid / how much do they earn" → [[wiki/pallet-collator-rewards\|pallet-collator-rewards]], [[wiki/runbook-run-collator\|runbook-run-collator]]
- "Why did my collator stop earning for one session? / odd-session benching" → [[wiki/pallet-collator-rotation\|pallet-collator-rotation]], [[wiki/runbook-run-collator\|runbook-run-collator]]
- "Get added to the Invulnerables set" → [[wiki/runbook-run-collator\|runbook-run-collator]], [[wiki/opengov\|opengov]]

## Reference

- "What does X audit firm cover?" → [[wiki/runtime-verification\|runtime-verification]], [[wiki/blockscience\|blockscience]], [[wiki/code4rena\|code4rena]], [[wiki/immunefi\|immunefi]]
- "Audit a PR or triage a locked-down asset using the upstream AI skills" → [[wiki/note-ai-skills\|note-ai-skills]]
- "Polkadot / parachain basics" → [[wiki/polkadot\|polkadot]], [[wiki/asset-hub\|asset-hub]]
- "All source pages" → [[wiki/source-hydration-general-context\|source-hydration-general-context]], [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]], [[wiki/source-sdk-codebase\|source-sdk-codebase]], [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]], [[wiki/source-papi-docs\|source-papi-docs]], [[wiki/source-polkadot-api-codebase\|source-polkadot-api-codebase]], [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
