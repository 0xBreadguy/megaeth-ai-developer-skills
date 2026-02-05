# Privy Integration for MegaETH

Privy embedded wallets enable headless signing for ultra-low latency transactions. This guide covers optimal patterns for MegaETH dApps using Privy.

## Why Privy + MegaETH?

Standard wallet flows add significant latency:
- UI popup for confirmation: 500ms - 2s (user-dependent)
- Gas estimation RPC call: 50-100ms

With Privy's `secp256k1_sign`, you can:
- Sign transactions without UI popups (~10-20ms)
- Build automated/bot transactions
- Achieve sub-200ms end-to-end confirmation on MegaETH

## Setup

```bash
npm install @privy-io/react-auth viem
```

```typescript
// providers/PrivyProvider.tsx
'use client';

import { PrivyProvider as Privy } from '@privy-io/react-auth';
import { megaeth } from '@/lib/chains';

export function PrivyProvider({ children }: { children: React.ReactNode }) {
  return (
    <Privy
      appId={process.env.NEXT_PUBLIC_PRIVY_APP_ID!}
      config={{
        embeddedWallets: {
          createOnLogin: 'users-without-wallets',
        },
        defaultChain: megaeth,
        supportedChains: [megaeth],
      }}
    >
      {children}
    </Privy>
  );
}
```

## Headless Signing with `secp256k1_sign`

The key to low-latency transactions is `secp256k1_sign` — a Privy-specific method that signs raw hashes without UI confirmation.

### Basic Flow

```typescript
import { useWallets } from '@privy-io/react-auth';
import { serializeTransaction, keccak256, parseGwei } from 'viem';

async function sendHeadlessTransaction() {
  const { wallets } = useWallets();
  const wallet = wallets.find(w => w.walletClientType === 'privy');
  if (!wallet) throw new Error('No Privy embedded wallet');

  const provider = await wallet.getEthereumProvider();
  const address = wallet.address as `0x${string}`;

  // 1. Build unsigned transaction
  const unsignedTx = {
    to: address,
    value: 0n,
    gas: 100000n,
    gasPrice: parseGwei('0.001'),  // Hardcoded for MegaETH
    nonce: await getNonce(address),
    chainId: 6343,  // MegaETH testnet
  };

  // 2. Serialize and hash
  const serializedUnsigned = serializeTransaction(unsignedTx);
  const txHash = keccak256(serializedUnsigned);

  // 3. Sign with secp256k1_sign (headless, no UI)
  const signature = await provider.request({
    method: 'secp256k1_sign',
    params: [txHash],
  }) as `0x${string}`;

  // 4. Parse signature components
  const { r, s, v } = parseSignature(signature, 6343);

  // 5. Serialize signed transaction
  const signedTx = serializeTransaction(unsignedTx, { r, s, v });

  // 6. Send via realtime API
  return await sendRealtimeTransaction(signedTx);
}
```

### Signature Parsing

`secp256k1_sign` returns a compact 65-byte signature that needs parsing:

```typescript
function parseSignature(
  signature: `0x${string}`,
  chainId: number
): { r: `0x${string}`; s: `0x${string}`; v: bigint } {
  // Extract r, s (32 bytes each)
  const r = `0x${signature.slice(2, 66)}` as `0x${string}`;
  const s = `0x${signature.slice(66, 130)}` as `0x${string}`;
  
  // Extract recovery bit (last byte)
  let recoveryBit = parseInt(signature.slice(130, 132), 16);
  if (recoveryBit >= 27) recoveryBit -= 27;
  
  // Calculate v for legacy transactions (EIP-155)
  const v = BigInt(chainId) * 2n + 35n + BigInt(recoveryBit);
  
  return { r, s, v };
}

// For EIP-1559 transactions, use yParity instead of v:
function parseSignatureEIP1559(signature: `0x${string}`): {
  r: `0x${string}`;
  s: `0x${string}`;
  yParity: 0 | 1;
} {
  const r = `0x${signature.slice(2, 66)}` as `0x${string}`;
  const s = `0x${signature.slice(66, 130)}` as `0x${string}`;
  let yParity = parseInt(signature.slice(130, 132), 16);
  if (yParity >= 27) yParity -= 27;
  
  return { r, s, yParity: yParity as 0 | 1 };
}
```

## Critical: Signing Warmup

The **first** `secp256k1_sign` call is slow (~500-700ms) because Privy needs to:
- Load the embedded wallet's private key
- Initialize cryptographic libraries
- Establish secure signing context

**Always warm up on page load:**

```typescript
import { useEffect } from 'react';
import { useWallets } from '@privy-io/react-auth';

function useSigningWarmup() {
  const { wallets } = useWallets();

  useEffect(() => {
    const wallet = wallets.find(w => w.walletClientType === 'privy');
    if (!wallet) return;

    async function warmup() {
      const provider = await wallet.getEthereumProvider();
      
      // Wait for Privy to fully initialize
      await new Promise(r => setTimeout(r, 100));
      
      // Warm up with dummy signature
      const dummyHash = '0x0000000000000000000000000000000000000000000000000000000000000001';
      
      try {
        await provider.request({
          method: 'secp256k1_sign',
          params: [dummyHash],
        });
        console.log('[Privy] Signing path warmed up');
      } catch (e) {
        // Retry once after delay
        await new Promise(r => setTimeout(r, 500));
        await provider.request({
          method: 'secp256k1_sign',
          params: [dummyHash],
        });
      }
    }

    warmup();
  }, [wallets]);
}
```

After warmup, subsequent signatures take only **~10-20ms**.

## Nonce Management for Rapid Transactions

When sending multiple transactions quickly, you may hit "already known" errors. Track nonces locally:

```typescript
const lastNonceRef = useRef<number>(-1);

async function getNextNonce(client: PublicClient, address: `0x${string}`) {
  let nonce = await client.getTransactionCount({ 
    address, 
    blockTag: 'pending' 
  });
  
  // Ensure nonce is always higher than last used
  if (lastNonceRef.current >= nonce) {
    nonce = lastNonceRef.current + 1;
  }
  
  lastNonceRef.current = nonce;
  return nonce;
}
```

## Complete Example: Ultra-Low Latency Send

```typescript
import { useCallback, useRef, useEffect } from 'react';
import { useWallets } from '@privy-io/react-auth';
import { createPublicClient, http, serializeTransaction, keccak256, parseGwei } from 'viem';

const MEGAETH_RPC = 'https://carrot.megaeth.com/rpc';
const CHAIN_ID = 6343;

const client = createPublicClient({
  transport: http(MEGAETH_RPC),
});

export function useMegaETHTransaction() {
  const { wallets } = useWallets();
  const lastNonceRef = useRef(-1);
  const isWarmedUp = useRef(false);

  // Warmup on mount
  useEffect(() => {
    const wallet = wallets.find(w => w.walletClientType === 'privy');
    if (!wallet || isWarmedUp.current) return;

    wallet.getEthereumProvider().then(async (provider) => {
      // Warm up RPC connection
      await client.getBlockNumber();
      
      // Warm up signing
      await new Promise(r => setTimeout(r, 100));
      const dummyHash = '0x0000000000000000000000000000000000000000000000000000000000000001';
      await provider.request({ method: 'secp256k1_sign', params: [dummyHash] });
      
      isWarmedUp.current = true;
      console.log('[Ready] RPC + signing warmed up');
    });
  }, [wallets]);

  const sendTransaction = useCallback(async (to: `0x${string}`, value: bigint) => {
    const wallet = wallets.find(w => w.walletClientType === 'privy');
    if (!wallet) throw new Error('No Privy wallet');

    const provider = await wallet.getEthereumProvider();
    const address = wallet.address as `0x${string}`;

    // Get nonce
    let nonce = await client.getTransactionCount({ address, blockTag: 'pending' });
    if (lastNonceRef.current >= nonce) nonce = lastNonceRef.current + 1;
    lastNonceRef.current = nonce;

    // Build transaction
    const unsignedTx = {
      to,
      value,
      gas: 100000n,
      gasPrice: parseGwei('0.001'),
      nonce,
      chainId: CHAIN_ID,
    };

    // Sign
    const serialized = serializeTransaction(unsignedTx);
    const hash = keccak256(serialized);
    const sig = await provider.request({
      method: 'secp256k1_sign',
      params: [hash],
    }) as `0x${string}`;

    // Parse signature
    const r = `0x${sig.slice(2, 66)}` as `0x${string}`;
    const s = `0x${sig.slice(66, 130)}` as `0x${string}`;
    let recoveryBit = parseInt(sig.slice(130, 132), 16);
    if (recoveryBit >= 27) recoveryBit -= 27;
    const v = BigInt(CHAIN_ID) * 2n + 35n + BigInt(recoveryBit);

    // Send via realtime API
    const signedTx = serializeTransaction(unsignedTx, { r, s, v });
    const response = await fetch(MEGAETH_RPC, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method: 'realtime_sendRawTransaction',
        params: [signedTx],
        id: Date.now(),
      }),
    });

    const { result, error } = await response.json();
    if (error) throw new Error(error.message);
    
    return result; // Receipt with txHash, blockNumber, etc.
  }, [wallets]);

  return { sendTransaction, isReady: isWarmedUp.current };
}
```

## Latency Breakdown

With all optimizations applied:

| Phase | Time |
|-------|------|
| Nonce fetch | ~50-100ms |
| Headless sign | ~10-20ms |
| `realtime_sendRawTransaction` | ~100-150ms |
| **Total** | **~150-250ms** |

Compare to standard wallet flow: 500ms - 2s+

## When to Use Headless Signing

✅ **Good for:**
- Trading bots / automated strategies
- Gaming transactions (fast actions)
- High-frequency operations
- Backend services with embedded wallets

❌ **Not for:**
- Large value transfers (users should confirm)
- First-time users (need to understand what they're signing)
- Compliance-sensitive operations

## SSR Compatibility (Next.js)

Privy cannot run during static generation. Use dynamic imports:

```typescript
import dynamic from 'next/dynamic';

const PrivyProvider = dynamic(
  () => import('./PrivyProvider').then(mod => mod.PrivyProvider),
  { ssr: false }
);
```
