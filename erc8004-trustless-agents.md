# ERC-8004: Trustless Agents on MegaETH

AI coding skill for implementing [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) (Trustless Agents) on MegaETH. Covers agent identity registration, reputation feedback, validation requests, and integration with A2A/MCP endpoints.

## What This Skill Is For

Use this skill when the user asks for:
- Registering an AI agent identity on-chain (ERC-721 based)
- Setting up agent metadata, wallet verification, and registration files
- Giving, reading, or revoking reputation feedback for agents
- Requesting and recording validation of agent work
- Building agent discovery and trust systems
- Integrating ERC-8004 with A2A, MCP, or x402 payment protocols
- Deploying ERC-8004 registries on MegaETH

## Architecture Overview

ERC-8004 defines three singleton registries that can be deployed per chain:

| Registry | Purpose |
|----------|---------|
| **Identity Registry** | ERC-721 NFT-based agent handles. Each agent gets a `tokenId` (agentId) and a `tokenURI` (agentURI) pointing to a registration file. |
| **Reputation Registry** | Standardized feedback signals — clients rate agents with signed fixed-point values, filterable by tags. |
| **Validation Registry** | Third-party verification hooks — agents request validation, validators respond with scores (0-100). |

## Chain Configuration

| Network | Chain ID | RPC |
|---------|----------|-----|
| MegaETH Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` |
| MegaETH Testnet | 6343 | `https://carrot.megaeth.com/rpc` |
| Ethereum Mainnet | 1 | (reference deployment) |

### Contract Addresses (All Chains — CREATE2 Deterministic)

Deployed via CREATE2 — same addresses on every chain including MegaETH.

| Contract | Mainnet Address | Testnet Address |
|----------|----------------|-----------------|
| Identity Registry | `0x8004A169FB4a3325136EB29fA0ceB6D2e539a432` | `0x8004A818BFB912233c491871b3d84c89A494BD9e` |
| Reputation Registry | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` | `0x8004B663056A597Dffe9eCcC1965A193B7388713` |

**Explorers:**
- MegaETH Mainnet: [Identity](https://megaeth.blockscout.com/address/0x8004A169FB4a3325136EB29fA0ceB6D2e539a432) · [Reputation](https://megaeth.blockscout.com/address/0x8004BAa17C55a88189AE136b182e5fdA19dE9b63)
- MegaETH Testnet: [Identity](https://megaeth-testnet-v2.blockscout.com/address/0x8004A818BFB912233c491871b3d84c89A494BD9e) · [Reputation](https://megaeth-testnet-v2.blockscout.com/address/0x8004B663056A597Dffe9eCcC1965A193B7388713)

Full deployment list: [erc-8004/erc-8004-contracts](https://github.com/erc-8004/erc-8004-contracts)

## Default Stack Decisions (Opinionated)

### 1. Use `eth_sendRawTransactionSync` for all writes
MegaETH returns receipts in <10ms via EIP-7966. No polling needed after registration or feedback calls.

### 2. Use viem over ethers.js
The `@agentic-trust/8004-sdk` supports both. Prefer viem for consistency with MegaETH patterns.

### 3. Store registration files on IPFS or as on-chain data URIs
For fully on-chain agents, use `data:application/json;base64,...` as the agentURI. For off-chain, use IPFS for easy subgraph indexing.

### 4. Always verify agent wallet ownership
Use `setAgentWallet()` with an EIP-712 signature (EOA) or ERC-1271 (smart contract wallet) to prove control. The wallet auto-clears on NFT transfer.

## SDK Installation

```bash
npm install @agentic-trust/8004-sdk viem
```

## Agent Registration File

The agentURI resolves to a JSON registration file:

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "MyAgent",
  "description": "An autonomous trading agent on MegaETH",
  "image": "https://example.com/agent.png",
  "services": [
    {
      "name": "A2A",
      "endpoint": "https://agent.example/.well-known/agent-card.json",
      "version": "0.3.0"
    },
    {
      "name": "MCP",
      "endpoint": "https://mcp.agent.example/",
      "version": "2025-06-18"
    }
  ],
  "x402Support": true,
  "active": true,
  "registrations": [
    {
      "agentId": 1,
      "agentRegistry": "eip155:4326:0x8004A169FB4a3325136EB29fA0ceB6D2e539a432"
    }
  ],
  "supportedTrust": ["reputation"]
}
```

**Required fields:** `type`, `name`, `description`, `image`
**Recommended:** At least one `registrations` entry and one service endpoint.

## Identity Registry

### Register an Agent

```typescript
import { createWalletClient, http, encodePacked } from 'viem'

const IDENTITY_REGISTRY = '0x8004A169FB4a3325136EB29fA0ceB6D2e539a432'

// Minimal registration (set URI later)
const agentId = await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'register',
  args: []
})

// Registration with URI
const agentId = await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'register',
  args: ['ipfs://QmYourRegistrationFile']
})

// Registration with URI + metadata
const agentId = await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'register',
  args: [
    'ipfs://QmYourRegistrationFile',
    [{ metadataKey: 'version', metadataValue: '0x01' }]
  ]
})
```

### Update Agent URI

```typescript
await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'setAgentURI',
  args: [agentId, 'ipfs://QmNewRegistrationFile']
})
```

### Fully On-Chain Registration

```typescript
const registrationJson = JSON.stringify({
  type: "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  name: "OnChainAgent",
  description: "Fully on-chain agent identity",
  image: "data:image/svg+xml;base64,...",
  services: [],
  active: true,
  registrations: [{ agentId: 1, agentRegistry: `eip155:4326:${IDENTITY_REGISTRY}` }]
})

const dataUri = `data:application/json;base64,${btoa(registrationJson)}`

await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'setAgentURI',
  args: [agentId, dataUri]
})
```

### Set & Verify Agent Wallet

The `agentWallet` is where the agent receives payments. It requires a signature to prove ownership:

```typescript
// For EOA wallets (EIP-712 signature)
await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'setAgentWallet',
  args: [agentId, newWalletAddress, deadline, signature]
})

// Read wallet
const wallet = await publicClient.readContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'getAgentWallet',
  args: [agentId]
})
```

### On-Chain Metadata

```typescript
// Set custom metadata
await walletClient.writeContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'setMetadata',
  args: [agentId, 'version', '0x0100'] // arbitrary bytes
})

// Read metadata
const data = await publicClient.readContract({
  address: IDENTITY_REGISTRY,
  abi: identityRegistryAbi,
  functionName: 'getMetadata',
  args: [agentId, 'version']
})
```

### Endpoint Domain Verification (Optional)

Prove control of an HTTPS endpoint by hosting a `.well-known` file:

```
GET https://agent.example/.well-known/agent-registration.json

{
  "registrations": [
    {
      "agentId": 1,
      "agentRegistry": "eip155:4326:0x8004A169FB4a3325136EB29fA0ceB6D2e539a432"
    }
  ]
}
```

## Reputation Registry

### Give Feedback

Any address (except the agent owner) can give feedback:

```typescript
const REPUTATION_REGISTRY = '0x8004BAa17C55a88189AE136b182e5fdA19dE9b63'

await walletClient.writeContract({
  address: REPUTATION_REGISTRY,
  abi: reputationRegistryAbi,
  functionName: 'giveFeedback',
  args: [
    agentId,            // uint256 — target agent
    87n,                // int128 — value (e.g., 87/100)
    0,                  // uint8 — valueDecimals (0 = integer)
    'starred',          // string — tag1 (category)
    '',                 // string — tag2 (optional)
    '',                 // string — endpoint (optional)
    'ipfs://QmFeedback', // string — feedbackURI (optional)
    '0x0000000000000000000000000000000000000000000000000000000000000000' // bytes32 — feedbackHash (optional, not needed for IPFS)
  ]
})
```

### Common Feedback Patterns

| tag1 | What it measures | Example value | valueDecimals |
|------|-----------------|---------------|---------------|
| `starred` | Quality rating (0-100) | 87 | 0 |
| `reachable` | Endpoint reachable (binary) | 1 | 0 |
| `uptime` | Endpoint uptime (%) | 9977 | 2 (= 99.77%) |
| `successRate` | Success rate (%) | 89 | 0 |
| `responseTime` | Response time (ms) | 560 | 0 |

### Read Feedback

```typescript
// Read specific feedback
const feedback = await publicClient.readContract({
  address: REPUTATION_REGISTRY,
  abi: reputationRegistryAbi,
  functionName: 'readFeedback',
  args: [agentId, clientAddress, feedbackIndex]
})
// Returns: { value, valueDecimals, tag1, tag2, isRevoked }

// Get aggregated summary (filtered by trusted clients)
const summary = await publicClient.readContract({
  address: REPUTATION_REGISTRY,
  abi: reputationRegistryAbi,
  functionName: 'getSummary',
  args: [agentId, trustedClientAddresses, 'starred', '']
})
// Returns: { count, summaryValue, summaryValueDecimals }

// Get all clients who gave feedback
const clients = await publicClient.readContract({
  address: REPUTATION_REGISTRY,
  abi: reputationRegistryAbi,
  functionName: 'getClients',
  args: [agentId]
})
```

> **Important:** Always filter by `clientAddresses` when reading summaries. Unfiltered results are vulnerable to Sybil attacks.

### Revoke Feedback

```typescript
await walletClient.writeContract({
  address: REPUTATION_REGISTRY,
  abi: reputationRegistryAbi,
  functionName: 'revokeFeedback',
  args: [agentId, feedbackIndex]
})
```

### Append Response

Anyone can append a response to feedback (e.g., agent showing a refund, spam tagger):

```typescript
await walletClient.writeContract({
  address: REPUTATION_REGISTRY,
  abi: reputationRegistryAbi,
  functionName: 'appendResponse',
  args: [agentId, clientAddress, feedbackIndex, 'ipfs://QmResponse', '0x...']
})
```

## Validation Registry

### Request Validation

Agent owners request third-party validation of their work:

```typescript
// Validation Registry not yet deployed — check erc-8004/erc-8004-contracts for updates
const VALIDATION_REGISTRY = '0x...'

await walletClient.writeContract({
  address: VALIDATION_REGISTRY,
  abi: validationRegistryAbi,
  functionName: 'validationRequest',
  args: [
    validatorContractAddress, // address — the validator
    agentId,                  // uint256
    'ipfs://QmRequestData',   // string — requestURI (inputs + outputs for verification)
    requestHash               // bytes32 — keccak256 of request payload
  ]
})
```

### Respond to Validation (Validator Only)

```typescript
// Called by the validator contract
await walletClient.writeContract({
  address: VALIDATION_REGISTRY,
  abi: validationRegistryAbi,
  functionName: 'validationResponse',
  args: [
    requestHash,      // bytes32
    100,              // uint8 — response (0=failed, 100=passed)
    'ipfs://QmAudit', // string — responseURI (optional)
    '0x...',          // bytes32 — responseHash (optional)
    'final'           // string — tag (optional, e.g., "soft-finality", "final")
  ]
})
```

### Read Validation Status

```typescript
const status = await publicClient.readContract({
  address: VALIDATION_REGISTRY,
  abi: validationRegistryAbi,
  functionName: 'getValidationStatus',
  args: [requestHash]
})
// Returns: { validatorAddress, agentId, response, responseHash, tag, lastUpdate }

// Aggregated stats
const summary = await publicClient.readContract({
  address: VALIDATION_REGISTRY,
  abi: validationRegistryAbi,
  functionName: 'getSummary',
  args: [agentId, validatorAddresses, '']
})
// Returns: { count, averageResponse }
```

## Agent Global Identifier

Each agent is globally unique via:

```
agentRegistry: eip155:{chainId}:{identityRegistryAddress}
agentId: {tokenId}
```

For MegaETH mainnet:
```
eip155:4326:0x8004A169FB4a3325136EB29fA0ceB6D2e539a432
```

An agent registered on MegaETH can operate and transact on any chain. Multi-chain registration is also supported.

## Off-Chain Feedback File

Optional detailed feedback stored on IPFS:

```json
{
  "agentRegistry": "eip155:4326:0x8004A169FB4a3325136EB29fA0ceB6D2e539a432",
  "agentId": 1,
  "clientAddress": "eip155:4326:0xClientAddr",
  "createdAt": "2026-03-12T00:00:00Z",
  "value": 95,
  "valueDecimals": 0,
  "tag1": "starred",
  "endpoint": "https://agent.example/api",
  "mcp": { "tool": "GetPrice" },
  "proofOfPayment": {
    "fromAddress": "0x...",
    "toAddress": "0x...",
    "chainId": "4326",
    "txHash": "0x..."
  }
}
```

## MegaETH-Specific Notes

- **Gas:** Registration mints an ERC-721 (new storage slots). Use `eth_estimateGas` via RPC — MegaETH SSTORE costs differ from standard EVM.
- **Instant receipts:** Use `eth_sendRawTransactionSync` (EIP-7966) for all write operations.
- **State growth:** Each new agent identity creates storage slots. Be aware of MegaETH's 1,000 slot per-tx limit for batch operations.
- **Subgraphs:** Feedback data is stored on-chain + emitted as events. Use subgraphs or event indexing for efficient querying.
- **Sybil resistance:** Always filter reputation queries by trusted `clientAddresses`. Unfiltered `getSummary` calls are meaningless.

## Related Standards

| Standard | Relationship |
|----------|-------------|
| [ERC-721](https://eips.ethereum.org/EIPS/eip-721) | Identity Registry is ERC-721 based |
| [EIP-712](https://eips.ethereum.org/EIPS/eip-712) | Wallet verification signatures |
| [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) | Smart contract wallet verification |
| [x402](https://www.x402.org/) | Payment proof in feedback signals |
| [A2A](https://github.com/google/A2A) | Agent-to-agent communication (advertised in registration) |
| [MCP](https://modelcontextprotocol.io/) | Model context protocol (advertised in registration) |

## Resources

- [EIP-8004 Specification](https://eips.ethereum.org/EIPS/eip-8004)
- [ERC-8004 GitHub (contracts + deployments)](https://github.com/erc-8004)
- [@agentic-trust/8004-sdk (npm)](https://www.npmjs.com/package/@agentic-trust/8004-sdk)
- [erc-8004-js (npm)](https://www.npmjs.com/package/erc-8004-js)
- [8004.org](https://8004.org)
