---
name: megaeth-developer
description: End-to-end MegaETH development playbook (Feb 2026). Covers Foundry project setup with MegaETH-specific config, wallet operations, token swaps (Kyber Network), eth_sendRawTransactionSync (EIP-7966) for instant receipts, JSON-RPC batching, real-time mini-block subscriptions, storage-aware contract patterns (Solady RedBlackTreeLib, transient storage), MegaEVM multidimensional gas model, WebSocket keepalive, bridging from Ethereum, Privy headless signing for ultra-low latency, and debugging with mega-evme. Use when building on MegaETH, using Foundry, managing wallets, sending transactions, deploying contracts, or integrating Privy embedded wallets, ERC-7710 delegation framework for scoped on-chain permissions, MetaMask Smart Accounts Kit for smart account creation and delegation management.
---

# MegaETH Development Skill

## What this Skill is for
Use this Skill when the user asks for:
- Foundry project setup targeting MegaETH
- Writing and running tests (unit, fuzz, invariant) on MegaETH
- Deploying and verifying contracts on MegaETH
- Wallet setup and management on MegaETH
- Sending transactions, checking balances, token operations
- Token swaps via Kyber Network aggregator
- MegaETH dApp frontend (React / Next.js with real-time updates)
- RPC configuration and transaction flow optimization
- Smart contract development with MegaEVM considerations
- Storage optimization (transient storage, Solady patterns)
- Gas estimation and fee configuration
- Testing and debugging MegaETH transactions
- WebSocket subscriptions and mini-block streaming
- Bridging ETH from Ethereum to MegaETH
- Privy integration for headless/automated signing
- Ultra-low latency transaction patterns
- ERC-7710 delegations (scoped permissions, spending limits, redelegation chains)
- MetaMask Smart Accounts (ERC-4337 accounts, signers, user operations)
- Advanced permissions (ERC-7715) via MetaMask

## Chain Configuration

| Network | Chain ID | RPC | Explorer |
|---------|----------|-----|----------|
| Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` | `https://mega.etherscan.io` |
| Testnet | 6343 | `https://carrot.megaeth.com/rpc` | `https://megaeth-testnet-v2.blockscout.com` |

## Default stack decisions (opinionated)

### 1. Transaction submission: eth_sendRawTransactionSync first
- Use `eth_sendRawTransactionSync` (EIP-7966) — returns receipt in <10ms
- Eliminates polling for `eth_getTransactionReceipt`
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

### 6. Debugging: mega-evme CLI
- Replay transactions with full traces
- Profile gas by opcode
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
- Volatile data limits (20M gas after block.timestamp access)

### 4. Deliverables expectations
When implementing changes, provide:
- Exact files changed + diffs
- Commands to build/test/deploy
- Gas cost notes for storage-heavy operations
- RPC optimization notes if applicable

## Progressive disclosure (read when needed)
- Foundry setup & deploy: [foundry-config.md](foundry-config.md)
- Wallet operations: [wallet-operations.md](wallet-operations.md)
- Frontend patterns: [frontend-patterns.md](frontend-patterns.md)
- Privy integration: [privy-integration.md](privy-integration.md)
- RPC methods reference: [rpc-methods.md](rpc-methods.md)
- Smart contract patterns: [smart-contracts.md](smart-contracts.md)
- Storage optimization: [storage-optimization.md](storage-optimization.md)
- Gas model: [gas-model.md](gas-model.md)
- Testing & debugging: [testing.md](testing.md)
- Security considerations: [security.md](security.md)
- ERC-7710 delegations: [erc7710-delegations.md](erc7710-delegations.md)
- MetaMask Smart Accounts: [smart-accounts.md](smart-accounts.md)
- Reference links & attribution: [resources.md](resources.md)
