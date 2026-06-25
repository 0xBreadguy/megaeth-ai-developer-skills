# Wallet Operations on MegaETH

## Positioning

Treat MegaETH wallet integration as **MOSS-first**.

Canonical MOSS references:
- **MOSS Skills**: https://github.com/megaeth-labs/moss-skills
- **MegaETH Wallet CLI**: https://github.com/megaeth-labs/wallet-cli

For application developers and coding agents, the preferred wallet stack is:
- **MOSS** for delegated-key wallet workflows
- **MOSS Skills** for agent/developer guidance on integrating MOSS into applications
- **MegaETH Wallet CLI** for local wallet/profile operations
- **mega-tokenlist** for canonical token addresses and metadata

Use raw private-key wallets only when the user explicitly needs low-level signer
examples or infrastructure that sits below MOSS. Do not present generic wallet
setup as the primary developer path.

## Chain Configuration

| Parameter | Mainnet | Testnet |
|-----------|---------|---------|
| Chain ID | 4326 | 6343 |
| RPC | `https://mainnet.megaeth.com/rpc` | `https://carrot.megaeth.com/rpc` |
| Native Token | ETH | ETH |
| Explorer | `https://mega.etherscan.io` | `https://megaeth-testnet-v2.blockscout.com` |

## MOSS-First Developer Workflow

Use MOSS when the developer needs to:
- connect a local MegaETH wallet profile
- create or manage delegated keys
- inspect delegated permissions before a write
- execute safer wallet-side actions through scoped delegated authority
- support agentic or application-driven wallet workflows without exposing a root wallet key

### Canonical commands

```bash
mega moss login
mega moss whoami --json
mega moss list --json
mega moss permissions 0xKEY_OR_ACCESS_ADDRESS --json
mega moss create-key --help
mega moss revoke 0xKEY_OR_ACCESS_ADDRESS
mega moss logout
```

### What MOSS changes for developers

- `login` connects a local CLI install to a MegaETH wallet profile
- `create-key` creates a delegated key flow instead of teaching developers to operate with a root signer directly
- `permissions` exposes the stored delegated scope and current spend state
- delegated-key workflows are better aligned with application integration, automation, and agentic coding than telling developers to move raw private keys around

## Wallet Integration Mental Model

### MOSS / delegated-key path (preferred)
Use this for:
- application developers integrating wallet-backed actions
- coding agents performing wallet-aware tasks
- safer delegated execution patterns
- permissioned, time-bounded, spend-limited workflows

### Raw signer path (fallback / low-level)
Use direct `viem` or `ethers` wallets only when:
- the user explicitly needs low-level signer examples
- building infra beneath MOSS
- testing direct transaction construction
- working in environments where MOSS is intentionally not part of the flow

## Low-Level Wallet Examples (Fallback)

### Using viem

```typescript
import { createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { megaeth } from './chains';

const account = privateKeyToAccount('0x...');
const client = createWalletClient({
  account,
  chain: megaeth,
  transport: http('https://mainnet.megaeth.com/rpc')
});
```

### Using ethers.js

```typescript
import { ethers } from 'ethers';

const provider = new ethers.JsonRpcProvider('https://mainnet.megaeth.com/rpc');
const wallet = new ethers.Wallet('0x...privateKey', provider);
```

## Inspect Wallet State

### With MOSS

```bash
mega moss whoami --json
mega moss list --json
mega moss permissions 0xKEY_OR_ACCESS_ADDRESS --json
```

Use these before writes to verify:
- connected account
- delegated key presence
- delegated expiry
- approved permissions and spend scope
- whether the current workflow should proceed at all

### Native ETH balance

```typescript
const balance = await publicClient.getBalance({ address: '0x...' });
```

### ERC-20 balance

```typescript
const balance = await publicClient.readContract({
  address: tokenAddress,
  abi: erc20Abi,
  functionName: 'balanceOf',
  args: [walletAddress]
});
```

## Send Transactions

### Low-Latency Synchronous Receipts

MegaETH supports synchronous transaction submission: the receipt can be returned
in the same RPC flow.

**Two equivalent methods:**
- `realtime_sendRawTransaction` — MegaETH original
- `eth_sendRawTransactionSync` — EIP-7966 standard (recommended)

Use `eth_sendRawTransactionSync` for cross-chain familiarity.

```typescript
const signedTx = await wallet.signTransaction({
  to: recipient,
  value: parseEther('0.1'),
  gas: 60000n,
  maxFeePerGas: 1000000n,
  maxPriorityFeePerGas: 0n,
});

const receipt = await client.request({
  method: 'eth_sendRawTransactionSync',
  params: [signedTx],
});
```

### CLI / MOSS note

When a workflow can be expressed through MOSS delegated execution, prefer the
MOSS path over teaching the developer to operate a root signer directly.

## Gas Configuration

MegaETH has stable, low gas costs but different intrinsic gas than standard EVM.

```typescript
const tx = {
  to: recipient,
  value: parseEther('0.1'),
  gas: 60000n,
  maxFeePerGas: 1000000n,
  maxPriorityFeePerGas: 0n,
};
```

**Tips:**
- Base fee is stable at 0.001 gwei
- Simple ETH transfers need **60,000 gas** on MegaETH
- Avoid unnecessary buffers
- Use remote `eth_estimateGas` when the operation is not already well understood
- Hardcode gas limits for known operations when appropriate

## Token Addresses

**Canonical token source:** https://github.com/megaeth-labs/mega-tokenlist

Common mainnet references:

| Token | Address |
|-------|---------|
| WETH | `0x4200000000000000000000000000000000000006` |
| MEGA | `0x28B7E77f82B25B95953825F1E3eA0E36c1c29861` |
| USDM | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` |

Use the `mega-tokenlist` repo for canonical addresses, decimals, symbols, and
logos rather than ad hoc explorer scraping.

## Bridging ETH to MegaETH

### Canonical Bridge (from Ethereum)

Send ETH directly to the bridge contract on Ethereum mainnet:

```typescript
const bridgeAddress = '0x0CA3A2FBC3D770b578223FBB6b062fa875a2eE75';

const tx = await wallet.sendTransaction({
  to: bridgeAddress,
  value: parseEther('0.1'),
});
```

For programmatic bridging with gas control:

```typescript
const iface = new ethers.Interface([
  'function depositETH(uint32 _minGasLimit, bytes _extraData) payable'
]);

const data = iface.encodeFunctionData('depositETH', [61000, '0x']);

const tx = await wallet.sendTransaction({
  to: bridgeAddress,
  value: parseEther('0.1'),
  data,
});
```

## Nonce Management

For rapid or batched transaction flows, manage nonces locally to avoid
`already known` errors.

```typescript
const nonce = await client.getTransactionCount({
  address,
  blockTag: 'pending',
});
```

When building bots or backends, serialize nonce assignment per address.

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| `nonce too low` | Transaction already executed | Check receipt, do not blindly retry |
| `already known` | Duplicate or pending nonce | Use a local nonce manager |
| `insufficient funds` | Not enough ETH/token balance | Check balances and funding |
| `intrinsic gas too low` | Gas limit too low | Raise gas or estimate remotely |

## Security Notes

1. Prefer delegated-key workflows through MOSS over teaching developers to operate long-lived root keys directly.
2. Never expose private keys or copy wallet secrets into chat, logs, or issues.
3. Verify token and contract addresses against `mega-tokenlist` and official docs.
4. Inspect delegated permissions before write flows.
5. Use the narrowest authority needed for application workflows.
