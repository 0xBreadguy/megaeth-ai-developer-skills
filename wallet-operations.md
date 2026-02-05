# Wallet Operations on MegaETH

## Chain Configuration

| Parameter | Mainnet | Testnet |
|-----------|---------|---------|
| Chain ID | 4326 | 6343 |
| RPC | `https://mainnet.megaeth.com/rpc` | `https://carrot.megaeth.com/rpc` |
| Native Token | ETH | ETH |
| Explorer | `https://mega.etherscan.io` | `https://megaeth-testnet-v2.blockscout.com` |

## Wallet Setup

### Using viem

```typescript
import { createWalletClient, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { megaeth } from './chains'; // See frontend-patterns.md

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

### Using evm-wallet-skill (CLI)

```bash
# Generate wallet (one-time)
node src/setup.js --json
# Stores key at ~/.evm-wallet.json (chmod 600)
```

## Check Balance

### Native ETH

```typescript
// viem
const balance = await publicClient.getBalance({ address: '0x...' });

// ethers
const balance = await provider.getBalance('0x...');
```

```bash
# CLI
node src/balance.js megaeth --json
```

### ERC20 Tokens

```typescript
const balance = await publicClient.readContract({
  address: tokenAddress,
  abi: erc20Abi,
  functionName: 'balanceOf',
  args: [walletAddress]
});
```

```bash
# CLI (WETH example)
node src/balance.js megaeth 0x4200000000000000000000000000000000000006 --json
```

## Send Transactions

### Instant Receipts

MegaETH supports synchronous transaction submission — get receipts in <10ms.

**Two equivalent methods:**
- `realtime_sendRawTransaction` — MegaETH original
- `eth_sendRawTransactionSync` — EIP-7966 standard (recommended)

MegaETH created `realtime_*` first; `*Sync` was later standardized as EIP-7966. Both are proxied and functionally identical. Use `eth_sendRawTransactionSync` for cross-chain compatibility.

```typescript
// Sign transaction
const signedTx = await wallet.signTransaction({
  to: recipient,
  value: parseEther('0.1'),
  gas: 60000n,                // MegaETH intrinsic gas (not 21000)
  maxFeePerGas: 1000000n,     // 0.001 gwei
  maxPriorityFeePerGas: 0n
});

// Send with instant receipt
const receipt = await client.request({
  method: 'eth_sendRawTransactionSync',
  params: [signedTx]
});

console.log('Confirmed in block:', receipt.blockNumber);
```

### Standard Send (polling)

```typescript
// viem
const hash = await walletClient.sendTransaction({
  to: recipient,
  value: parseEther('0.1')
});
const receipt = await publicClient.waitForTransactionReceipt({ hash });

// ethers
const tx = await wallet.sendTransaction({
  to: recipient,
  value: parseEther('0.1')
});
const receipt = await tx.wait();
```

```bash
# CLI
node src/transfer.js megaeth 0xRecipient 0.1 --yes --json
```

## Gas Configuration

MegaETH has stable, low gas costs but different intrinsic gas than standard EVM:

```typescript
const tx = {
  to: recipient,
  value: parseEther('0.1'),
  gas: 60000n,                    // MegaETH intrinsic gas (not 21000!)
  maxFeePerGas: 1000000n,         // 0.001 gwei (base fee)
  maxPriorityFeePerGas: 0n        // Not needed unless congested
};
```

**Tips:**
- Base fee is stable at 0.001 gwei
- Simple ETH transfers need **60,000 gas** on MegaETH (not 21,000)
- Don't add buffers (viem adds 20% by default — override it)
- When in doubt, use `eth_estimateGas` — MegaEVM costs differ from standard EVM
- Hardcode gas limits for known operations

## Token Operations

### Token Addresses

**Official token list:** https://github.com/megaeth-labs/mega-tokenlist

Common tokens (Mainnet):

| Token | Address |
|-------|---------|
| WETH | `0x4200000000000000000000000000000000000006` |
| MEGA | `0x28B7E77f82B25B95953825F1E3eA0E36c1c29861` |
| USDM | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` |

For the full list of verified tokens, logos, and metadata, see the [mega-tokenlist](https://github.com/megaeth-labs/mega-tokenlist) repo.

### Transfer ERC20

```typescript
const hash = await walletClient.writeContract({
  address: tokenAddress,
  abi: erc20Abi,
  functionName: 'transfer',
  args: [recipient, amount]
});
```

```bash
# CLI (send 100 USDM)
node src/transfer.js megaeth 0xRecipient 100 0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7 --yes --json
```

### Approve Spending

```typescript
const hash = await walletClient.writeContract({
  address: tokenAddress,
  abi: erc20Abi,
  functionName: 'approve',
  args: [spenderAddress, maxUint256]
});
```

## Token Swaps (Kyber Network)

MegaETH uses **Kyber Network** as the DEX aggregator for best-route swaps.

### API Integration

```typescript
const KYBER_API = 'https://aggregator-api.kyberswap.com/megaeth/api/v1';

// Get quote
const quoteRes = await fetch(
  `${KYBER_API}/routes?` + new URLSearchParams({
    tokenIn: '0x...', // or 'ETH' for native
    tokenOut: '0x...',
    amountIn: amount.toString(),
    gasInclude: 'true'
  })
);
const quote = await quoteRes.json();

// Build transaction
const buildRes = await fetch(`${KYBER_API}/route/build`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    routeSummary: quote.data.routeSummary,
    sender: walletAddress,
    recipient: walletAddress,
    slippageTolerance: 50 // 0.5% = 50 bips
  })
});
const { data } = await buildRes.json();

// Execute swap
const hash = await walletClient.sendTransaction({
  to: data.routerAddress,
  data: data.data,
  value: data.value,
  gas: BigInt(data.gas)
});
```

### Kyber Resources

- **Docs**: https://docs.kyberswap.com/kyberswap-solutions/kyberswap-aggregator
- **MegaETH Router**: Check Kyber docs for current address

## Bridging ETH to MegaETH

### Canonical Bridge (from Ethereum)

Send ETH directly to the bridge contract on Ethereum mainnet:

```typescript
// On Ethereum mainnet
const bridgeAddress = '0x0CA3A2FBC3D770b578223FBB6b062fa875a2eE75';

const tx = await wallet.sendTransaction({
  to: bridgeAddress,
  value: parseEther('0.1') // Will appear on MegaETH
});
```

For programmatic bridging with gas control:

```typescript
const iface = new ethers.Interface([
  'function depositETH(uint32 _minGasLimit, bytes _extraData) payable'
]);

const data = iface.encodeFunctionData('depositETH', [
  61000,        // Gas limit on MegaETH side
  '0x'          // Extra data (optional)
]);

const tx = await wallet.sendTransaction({
  to: bridgeAddress,
  value: parseEther('0.1'),
  data
});
```

## Transaction Confirmation

MegaETH has ~10ms block times. Transactions confirm almost instantly.

```typescript
// With eth_sendRawTransactionSync — instant
const receipt = await client.request({
  method: 'eth_sendRawTransactionSync',
  params: [signedTx]
});
// receipt is immediately available

// Standard — still very fast
const hash = await wallet.sendTransaction(tx);
const receipt = await tx.wait(); // ~100-200ms total
```

## Nonce Management

For single transactions, the RPC handles nonces automatically. For **rapid/batch transactions**, manage nonces locally to avoid "already known" errors:

```typescript
import { useRef } from 'react';

// Track last used nonce per chain
const lastNonceRef = useRef<Record<number, number>>({});

async function getNextNonce(
  client: PublicClient, 
  address: `0x${string}`,
  chainId: number
): Promise<number> {
  // Fetch pending nonce from network
  let nonce = await client.getTransactionCount({ 
    address, 
    blockTag: 'pending' 
  });
  
  // If we've used this nonce already, increment
  const lastUsed = lastNonceRef.current[chainId] ?? -1;
  if (lastUsed >= nonce) {
    nonce = lastUsed + 1;
  }
  
  // Track for next call
  lastNonceRef.current[chainId] = nonce;
  return nonce;
}
```

### Why This Matters

MegaETH's ~10ms blocks mean transactions confirm quickly, but if you send multiple txs in rapid succession (e.g., trading bot, batch operations), the second tx may hit the RPC before the first confirms. Without local tracking, both get the same nonce → "already known" error.

### Backend/Bot Pattern

```typescript
class NonceManager {
  private nonces: Map<string, number> = new Map();
  private locks: Map<string, Promise<void>> = new Map();

  async getNextNonce(client: PublicClient, address: string): Promise<number> {
    // Serialize nonce fetches per address
    const existing = this.locks.get(address);
    if (existing) await existing;

    let resolve: () => void;
    this.locks.set(address, new Promise(r => resolve = r));

    try {
      const networkNonce = await client.getTransactionCount({ 
        address: address as `0x${string}`, 
        blockTag: 'pending' 
      });
      
      const lastUsed = this.nonces.get(address) ?? -1;
      const nonce = Math.max(networkNonce, lastUsed + 1);
      
      this.nonces.set(address, nonce);
      return nonce;
    } finally {
      resolve!();
      this.locks.delete(address);
    }
  }

  // Call after tx confirms or fails
  reset(address: string) {
    this.nonces.delete(address);
  }
}
```

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| "nonce too low" | Tx already executed | Check receipt, don't retry |
| "already known" | Tx pending or duplicate nonce | Use nonce manager, increment locally |
| "insufficient funds" | Not enough ETH | Check balance, fund wallet |
| "intrinsic gas too low" | Gas limit too low | Increase gas or use remote estimation |

## Security Notes

1. **Never expose private keys** — store at `~/.evm-wallet.json` with chmod 600
2. **Confirm before sending** — always show recipient, amount, gas before execution
3. **Use hardware wallets** for large amounts
4. **Verify contract addresses** — check explorer before interacting
