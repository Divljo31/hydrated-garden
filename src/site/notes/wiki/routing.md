---
{"dg-publish":true,"permalink":"/wiki/routing/","title":"Task Routing","dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"routing","title":"Task Routing","last_updated":"2026-04-20"}}
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
- "Schedule DCA / TWAP" → [[wiki/pallet-dca\|pallet-dca]], [[wiki/dca\|dca]]
- "OTC / peer-to-peer trades" → [[wiki/pallet-otc\|pallet-otc]], [[wiki/otc-trading\|otc-trading]]
- "ICE intent engine" → [[wiki/ice\|ice]]

## Trading SDK (TypeScript)

- "Build a trade in the SDK" → [[wiki/sdk-next\|sdk-next]], [[wiki/smart-order-router\|smart-order-router]], [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- "Debug a pool-math discrepancy" → [[wiki/sdk-next\|sdk-next]], [[wiki/source-sdk-codebase\|source-sdk-codebase]] (math-* WASM packages)
- "Add a new pool type to the router" → [[wiki/sdk-next\|sdk-next]], [[wiki/smart-order-router\|smart-order-router]], [[wiki/route-suggester\|route-suggester]]
- "Use SDK from a new frontend" → [[wiki/sdk-next\|sdk-next]], [[wiki/sdk-descriptors\|sdk-descriptors]], [[wiki/sdk-common\|sdk-common]], [[wiki/papi-getting-started\|papi-getting-started]]

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
- "Bridge a token via Basejump / Wormhole / Snowbridge (multi-bridge selection)" → [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-package\|xc-package]]
- "Track a pending cross-chain transfer" → [[wiki/xc-scan\|xc-scan]]
- "Cross-chain from SDK frontend" → [[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]]
- "EVM interop (MetaMask, Solidity)" → [[wiki/pallet-frontier\|pallet-frontier]]

## Lending & stablecoin

- "Debug HOLLAR peg / HSM" → [[wiki/hollar\|hollar]], [[wiki/pallet-hsm\|pallet-hsm]], [[wiki/stableswap\|stableswap]]
- "Work on borrowing / liquidations" → [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/pallet-frontier\|pallet-frontier]]
- "Use Omnipool LP as collateral" → [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/nft-lp-positions\|nft-lp-positions]]

## Yield / strategy tokens

- "Understand GDOT / GETH / GSOL composition" → [[wiki/gdot\|gdot]], [[wiki/geth\|geth]], [[wiki/gsol\|gsol]]

## Governance & tokenomics

- "Governance referendum / parameter change" → [[wiki/opengov\|opengov]], [[wiki/hdx\|hdx]]
- "HDX staking rewards" → [[wiki/pallet-staking\|pallet-staking]], [[wiki/bonding-curve\|bonding-curve]]
- "Referral system" → [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/referrals\|referrals]]

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
- "Work on the wallet / assets UI" → [[wiki/hydration-ui-modules\|hydration-ui-modules]] (wallet section), [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]
- "Transaction signing / submission flow" → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/papi-signers\|papi-signers]]
- "Review a pending multisig transaction in the UI / track multisig approvals" → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]]
- "Add or integrate a new wallet" → [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/papi-signers\|papi-signers]]
- "Style / component additions" → [[wiki/hydration-ui-design-system\|hydration-ui-design-system]]
- "Generate / update GraphQL types (indexer, squid, snowbridge)" → [[wiki/hydration-ui-indexer\|hydration-ui-indexer]]
- "Historical data / analytics queries" → [[wiki/hydration-ui-indexer\|hydration-ui-indexer]], [[wiki/hydration-ui-modules\|hydration-ui-modules]] (stats section)

## Runtime / node-level

- "Understand how pallets are wired" → [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- "EVM precompiles" → [[wiki/hydration-precompiles\|hydration-precompiles]], [[wiki/pallet-frontier\|pallet-frontier]]
- "XCM config on the node" → [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/xcm\|xcm]]
- "Runtime upgrade / migration" → [[wiki/hydration-runtime\|hydration-runtime]]

## Reference

- "What does X audit firm cover?" → [[wiki/runtime-verification\|runtime-verification]], [[wiki/blockscience\|blockscience]], [[wiki/code4rena\|code4rena]], [[wiki/immunefi\|immunefi]]
- "Polkadot / parachain basics" → [[wiki/polkadot\|polkadot]], [[wiki/asset-hub\|asset-hub]]
- "All source pages" → [[wiki/source-hydration-general-context\|source-hydration-general-context]], [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]], [[wiki/source-sdk-codebase\|source-sdk-codebase]], [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]], [[wiki/source-papi-docs\|source-papi-docs]], [[wiki/source-polkadot-api-codebase\|source-polkadot-api-codebase]], [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
