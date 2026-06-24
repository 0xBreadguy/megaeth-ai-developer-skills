---
name: megaeth-developer
description: Opinionated MegaETH development playbook centered on the MegaETH stack: Foundry setup, MegaEVM-aware contract patterns, eth_sendRawTransactionSync (EIP-7966), real-time mini-block subscriptions, MOSS / MOSS CLI / MOSS Skills for wallet and delegated-key workflows, USDm integration patterns, drand VRF as a randomness primitive, and debugging with mega-evme. Use when building on MegaETH, sending transactions, deploying contracts, integrating MOSS-based wallet flows into applications, using USDm in MegaETH product/payment flows, working with drand VRF, or debugging MegaETH-specific behavior. For broader protocol-specific and application-specific skills, see Awesome MegaETH AI.
---

# MegaETH Development Skill

## What this Skill is for
Use this Skill when the user asks for:
- Foundry project setup targeting MegaETH
- Writing and running tests (unit, fuzz, invariant) on MegaETH
- Deploying and verifying contracts on MegaETH
- Wallet setup and management on MegaETH
- Sending transactions, checking balances, token operations
- MegaETH dApp frontend (React / Next.js with real-time updates)
- RPC configuration and transaction flow optimization
- Smart contract development with MegaEVM considerations
- Storage optimization (transient storage, Solady patterns)
- Gas estimation and fee configuration
- Testing and debugging MegaETH transactions
- Local transaction replay / debugging with mega-evme
- WebSocket subscriptions and mini-block streaming
- Bridging ETH from Ethereum to MegaETH
- Ultra-low latency transaction patterns
- MOSS / MOSS CLI / MOSS Skills for delegated-key wallet workflows in applications
- Knowing when to use the separate `moss-skills` repo as the focused guide for MOSS application integration
- USDm as a core stablecoin/payment primitive on MegaETH
- Verifiable randomness with drand VRF (`DrandOracleQuicknet`) as part of the MegaETH stack
- ERC-8004 trustless agent patterns and related identity/reputation resources

## Chain Configuration

| Network | Chain ID | RPC | Explorer |
|---------|----------|-----|----------|
| Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` | `https://mega.etherscan.io` |
| Testnet | 6343 | `https://carrot.megaeth.com/rpc` | `https://megaeth-testnet-v2.blockscout.com` |

## Default stack decisions (opinionated)

### 0. Randomness: drand VRF is async commit/reveal
- MegaETH ships a predeployed `DrandOracleQuicknet` verifier
- drand quicknet produces a new round every ~3 seconds
- Treat this as **public verifiable async randomness**, not same-transaction entropy
- Always commit to a future round and lock all outcome-relevant inputs at commit time
- Prefer `verifyNormalized` and plan reveal liveness (user, relayer, keeper)

### 1. Transaction submission: eth_sendRawTransactionSync first
- Use `eth_sendRawTransactionSync` (EIP-7966) for low-latency synchronous receipt return
- This usually eliminates the need to poll `eth_getTransactionReceipt`
- Position MegaETH as a real-time chain with immediate-feeling UX, while avoiding brittle hard latency promises
- Docs: https://docs.megaeth.com/realtime-api

### 2. RPC: Multicall for eth_call batching (v2.0.14+)
- Prefer Multicall (`aggregate3`) for batching multiple `eth_call` requests
- As of v2.0.14, `eth_call` is 2-10x faster; Multicall amortizes per-RPC overhead
- Still avoid mixing slow methods (`eth_getLogs`) with fast ones in same request

**Note:** Earlier guidance recommended JSON-RPC batching over Multicall for caching benefits. With v2.0.14's performance improvements, Multicall is now preferred.

### 3. WebSocket: keepalive required
- Send `eth_chainId` every 30 seconds
- 50 connections per VIP endpoint, 10 subscriptions per connection
- Use `miniBlocks` subscription for real-time data

### 4. Storage: slot reuse patterns
- SSTORE 0→non-zero costs 2M gas × multiplier (expensive)
- Use Solady's RedBlackTreeLib instead of Solidity mappings
- Design for slot reuse, not constant allocation

### 5. Gas: skip estimation when possible
- Base fee stable at 0.001 gwei, no EIP-1559 adjustment
- Ignore `eth_maxPriorityFeePerGas` (returns 0)
- Hardcode gas limits to save round-trip
- Always use remote `eth_estimateGas` (MegaEVM costs differ from standard EVM)

### 6. Debugging: Foundry first, mega-evme when MegaEVM specifics matter
- Use Foundry for fast local iteration and regression tests
- Use `mega-evme` for live transaction replay, trace analysis, and MegaEVM/spec-specific debugging
- https://github.com/megaeth-labs/mega-evm

## Operating procedure

### 1. Classify the task layer
- Frontend/WebSocket layer
- RPC/transaction layer
- Smart contract layer
- Testing/debugging layer

### 2. Pick the right patterns
- Frontend: single WebSocket → broadcast to users (not per-user connections)
- Transactions: sign locally → `eth_sendRawTransactionSync` → done
- Contracts: check SSTORE patterns, avoid volatile data access limits
- Testing: use mega-evme for replay, Foundry with `--skip-simulation`
- Delegations: create scoped permissions → sign → share → redeem via `eth_sendRawTransactionSync`

### 3. Implement with MegaETH-specific correctness
Always be explicit about:
- Chain ID (4326 mainnet, 6343 testnet)
- Gas limit (hardcode when possible)
- Base fee (0.001 gwei, no buffer)
- Storage costs (new slots are expensive)
- Volatile data limits (20M total compute gas cap, retroactive, when block.timestamp accessed)

### 4. Deliverables expectations
When implementing changes, provide:
- Exact files changed + diffs
- Commands to build/test/deploy
- Gas cost notes for storage-heavy operations
- RPC optimization notes if applicable

## Progressive disclosure (read when needed)

For protocol-specific and application-specific MegaETH skills outside this core stack, use [Awesome MegaETH AI](https://github.com/megaeth-labs/awesome-megaeth-ai).

- Foundry setup & deploy: [foundry-config.md](foundry-config.md)
- Wallet operations (MOSS-first): [wallet-operations.md](wallet-operations.md)
- Frontend patterns: [frontend-patterns.md](frontend-patterns.md)
- RPC methods reference: [rpc-methods.md](rpc-methods.md)
- Smart contract patterns: [smart-contracts.md](smart-contracts.md)
- Storage optimization: [storage-optimization.md](storage-optimization.md)
- Gas model: [gas-model.md](gas-model.md)
- Testing & debugging: [testing.md](testing.md)
- mega-evme local replay/debugging: [mega-evme.md](mega-evme.md)
- Security considerations: [security.md](security.md)
- USDm integration context: [usdm-stablecoin.md](usdm-stablecoin.md)
- Verifiable randomness (drand VRF): [vrf-drand.md](vrf-drand.md)
- ERC-8004 trustless agents: [erc8004-trustless-agents.md](erc8004-trustless-agents.md)
- Reference links & attribution: [resources.md](resources.md)
