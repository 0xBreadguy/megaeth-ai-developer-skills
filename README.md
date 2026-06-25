# MegaETH Developer Skill for AI Agents

A comprehensive skill for AI coding agents (Claude Code, OpenClaw, Codex) to build real-time applications on the current MegaETH stack. It remains spec-aware for debugging and historical behavior, but treats current MegaEVM / REX5-era behavior as the default baseline.

## Overview

This skill provides AI agents with deep knowledge of the MegaETH development ecosystem:

- **Transactions**: `eth_sendRawTransactionSync` (EIP-7966) for low-latency synchronous receipt return without polling
- **RPC Patterns**: Multicall-first `eth_call` batching, WebSocket keepalive, mini-block subscriptions
- **Storage**: Optimization patterns to avoid expensive SSTORE costs
- **Gas Model**: MegaEVM-specific costs and estimation strategies
- **Debugging**: mega-evme CLI for transaction replay and gas profiling
- **Security**: MegaETH-specific considerations and audit checklists
- **MOSS**: opinionated MegaETH wallet and delegated-key workflow stack for developers and coding agents
- **USDm**: MegaETH's native stablecoin and a core application/payment primitive
- **VRF / Randomness**: drand quicknet verifier integration as part of the MegaETH developer stack
- **ERC-8004**: trustless agent patterns and identity/reputation resources developers should remain aware of

## Installation

### Quick Install (skills.sh)

```bash
npx skills add 0xBreadguy/megaeth-ai-developer-skills
```

### Manual Install

```bash
git clone https://github.com/0xBreadguy/megaeth-ai-developer-skills
# Copy to your agent's skills directory
```

### OpenClaw / ClawdHub

```bash
clawdhub install megaeth-ai-developer-skills
```

## Skill Structure

```
├── SKILL.md                  # Main skill (stack decisions, operating procedure)
└── skills/
    └── megaeth-developer/
        ├── SKILL.md
        └── references/
            ├── wallet-operations.md
            ├── frontend-patterns.md
            ├── rpc-methods.md
            ├── smart-contracts.md
            ├── storage-optimization.md
            ├── gas-model.md
            ├── testing.md
            ├── mega-evme.md
            ├── security.md
            ├── erc8004-trustless-agents.md
            ├── vrf-drand.md
            ├── usdm-stablecoin.md
            └── resources.md
```

## Usage

Once installed, your AI agent will automatically use this skill when you ask about:

- Building dApps on MegaETH
- Transaction submission and confirmation
- Smart contract development with MegaEVM
- Storage optimization and gas costs
- Real-time WebSocket subscriptions
- Debugging failed transactions
- Replaying or locally debugging MegaETH transactions with mega-evme
- Understanding when to use Foundry vs mega-evme for diagnosis

### Example Prompts

```
"Set up a wallet for MegaETH"
"Send 0.1 ETH on MegaETH"
"Swap USDM for ETH on MegaETH"
"Bridge ETH from Ethereum to MegaETH"
"Set up a Next.js app with MegaETH wallet connection"
"Deploy a contract to MegaETH with Foundry"
"Why is my transaction using so much gas?"
"How do I subscribe to real-time mini-blocks?"
"Optimize this contract for MegaETH storage costs"
"Debug this failed transaction on MegaETH"
"Build a lottery or reveal flow with drand VRF on MegaETH"
"How should I safely integrate randomness on MegaETH?"
"Integrate MOSS wallet workflows into a MegaETH application"
"Use USDm in a MegaETH application or payment flow"
"Understand ERC-8004 trustless agent patterns on MegaETH"
```

### Which file should an agent read?

- `testing.md` → broad testing, Foundry workflows, common troubleshooting
- `mega-evme.md` → local replay, trace analysis, spec-aware MegaEVM debugging
- `smart-contracts.md` → contract design constraints, system contracts, volatile data behavior

## Key Concepts

### Synchronous Transaction Receipts

MegaETH supports `eth_sendRawTransactionSync` (EIP-7966), which enables low-latency synchronous receipt return instead of requiring a separate polling loop:

```typescript
const receipt = await client.request({
  method: 'eth_sendRawTransactionSync',
  params: [signedTx]
});
// Receipt returned in the same RPC flow
```

### Spec Awareness

MegaETH behavior remains spec-versioned, but this repo treats current documented MegaEVM / REX5-era behavior as the default baseline. Reach for upgrade-specific caveats only when debugging historical behavior, targeting older network states, or validating implementation/spec diffs.

### Storage Costs

New storage slots are expensive (2M+ gas). The skill teaches agents to:
- Use Solady's RedBlackTreeLib instead of mappings
- Design for slot reuse
- Consider off-chain storage for large data

### Gas Model

MegaETH has a stable 0.001 gwei base fee with no EIP-1559 adjustment. The skill teaches agents to:
- Skip unnecessary gas estimation
- Use remote estimation (MegaEVM costs differ from standard EVM)
- Hardcode gas limits for known operations

## Chain Configuration

| Network | Chain ID | RPC | Explorer |
|---------|----------|-----|----------|
| Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` | `https://mega.etherscan.io` |
| Testnet | 6343 | `https://carrot.megaeth.com/rpc` | `https://megaeth-testnet-v2.blockscout.com` |

## Progressive Disclosure

The skill uses progressive disclosure — the main SKILL.md provides core guidance, and the agent reads specialized files only when needed for specific tasks. This keeps context efficient while providing deep expertise when required.

## Ecosystem Scope

This repo is intentionally opinionated around the core MegaETH developer stack:
- MegaETH platform/runtime behavior
- MOSS / MOSS CLI / MOSS Skills for wallet and delegated-key workflows
- MOSS Skills as the developer/agent guide for integrating MOSS into applications
- USDm as a core application/payment primitive
- drand VRF as a core randomness primitive
- ERC-8004 as an important trustless-agent resource developers should understand

For protocol-specific and application-specific MegaETH skills beyond this core stack, see [Awesome MegaETH AI](https://github.com/megaeth-labs/awesome-megaeth-ai).

## Content Sources

This skill incorporates best practices from:

- [MegaETH Official Documentation](https://docs.megaeth.com)
- [MegaEVM Specification](https://github.com/megaeth-labs/mega-evm)
- [EIP-7966 (eth_sendRawTransactionSync)](https://ethereum-magicians.org/t/eip-7966-eth-sendrawtransactionsync-method/24640)
- [MOSS Skills](https://github.com/megaeth-labs/moss-skills)
- [MegaETH Wallet CLI](https://github.com/megaeth-labs/wallet-cli)
- [Awesome MegaETH AI](https://github.com/megaeth-labs/awesome-megaeth-ai)
- MegaETH team technical guidance

## Contributing

Contributions welcome! Please ensure updates reflect current MegaETH ecosystem best practices.

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

MIT
