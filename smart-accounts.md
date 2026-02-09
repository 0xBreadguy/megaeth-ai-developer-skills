# MetaMask Smart Accounts Kit

## Setup

```bash
npm install @metamask/smart-accounts-kit@0.3.0
```

## MegaETH Chain Configuration

```typescript
import { defineChain } from 'viem';
import { createPublicClient, http } from 'viem';

export const megaeth = defineChain({
  id: 4326,
  name: 'MegaETH',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://mainnet.megaeth.com/rpc'] },
  },
});

export const megaethTestnet = defineChain({
  id: 6343,
  name: 'MegaETH Testnet',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://carrot.megaeth.com/rpc'] },
  },
});

const client = createPublicClient({ chain: megaeth, transport: http() });
```

## Account Types

| Type | Implementation | Use Case |
|------|---------------|----------|
| Hybrid | `Implementation.Hybrid` | EOA + passkey signers, most flexible |
| MultiSig | `Implementation.MultiSig` | Threshold signing (Safe-compatible) |
| Stateless7702 | `Implementation.Stateless7702` | EIP-7702 upgraded EOA, no new address |

## Creating a Smart Account

### EOA Signer (Hybrid)

```typescript
import { Implementation, toMetaMaskSmartAccount } from '@metamask/smart-accounts-kit';
import { privateKeyToAccount } from 'viem/accounts';

const owner = privateKeyToAccount('0x...');

const smartAccount = await toMetaMaskSmartAccount({
  client,
  implementation: Implementation.Hybrid,
  deployParams: [owner.address, [], [], []],
  deploySalt: '0x',
  signer: { account: owner },
});

console.log('Smart account:', smartAccount.address);
```

### Passkey Signer (Hybrid)

```typescript
import { toWebAuthnAccount } from 'viem/account-abstraction';

const credential = await navigator.credentials.create({ publicKey: /* ... */ });
const webAuthnAccount = toWebAuthnAccount({ credential });

const smartAccount = await toMetaMaskSmartAccount({
  client,
  implementation: Implementation.Hybrid,
  deployParams: [
    '0x0000000000000000000000000000000000000000',
    [webAuthnAccount.publicKey.x],
    [webAuthnAccount.publicKey.y],
    [],
  ],
  deploySalt: '0x',
  signer: { account: webAuthnAccount, type: 'webAuthn' },
});
```

### MultiSig

```typescript
const smartAccount = await toMetaMaskSmartAccount({
  client,
  implementation: Implementation.MultiSig,
  deployParams: [
    [signer1.address, signer2.address, signer3.address],
    2n, // threshold
  ],
  deploySalt: '0x',
  signer: { account: signer1 },
});
```

## User Operations

```typescript
import { createBundlerClient } from 'viem/account-abstraction';

const bundlerClient = createBundlerClient({
  account: smartAccount,
  client,
  transport: http('https://your-bundler.example.com'),
});

const hash = await bundlerClient.sendUserOperation({
  calls: [
    {
      to: '0x...',
      value: parseEther('0.1'),
    },
  ],
});

const receipt = await bundlerClient.waitForUserOperationReceipt({ hash });
```

## EOA-Based Delegation Redemption

For EOA signers, bypass the bundler entirely using `eth_sendRawTransactionSync`:

```typescript
import { createWalletClient } from 'viem';
import { DelegationFramework } from '@metamask/smart-accounts-kit';

const walletClient = createWalletClient({
  account: owner,
  chain: megaeth,
  transport: http(),
});

const tx = await walletClient.sendTransaction({
  to: '0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3', // DelegationManager
  data: DelegationFramework.redeemDelegationsCalldata({
    delegations: [[signedDelegation]],
    modes: [0n],
    executions: [execution],
  }),
});
```

## Signer Types

| Signer | Setup |
|--------|-------|
| EOA (private key) | `privateKeyToAccount('0x...')` |
| Passkey (WebAuthn) | `toWebAuthnAccount({ credential })` |
| Dynamic | `useDynamicContext()` → extract wallet client |
| Web3Auth | `web3auth.provider` → wrap with viem |
| Wagmi | `useWalletClient()` → `walletClientToAccount()` |

## ERC-7715 Advanced Permissions

Request permissions via MetaMask extension (Flask 13.5.0+):

```typescript
const grantedPermissions = await provider.request({
  method: 'wallet_grantPermissions',
  params: [{
    permissions: [{
      type: 'native-token-transfer',
      policies: [],
      required: true,
      data: {
        allowance: '0xDE0B6B3A7640000', // 1 ETH
      },
    }],
    expiry: Math.floor(Date.now() / 1000) + 86400, // 24h
  }],
});
```

## Key API Methods

| Method | Purpose |
|--------|---------|
| `toMetaMaskSmartAccount()` | Create smart account instance |
| `createDelegation()` | Build delegation with caveats |
| `DelegationFramework.signDelegation()` | Sign delegation off-chain |
| `DelegationFramework.redeemDelegationsCalldata()` | Build redemption calldata |
| `DelegationFramework.disableDelegationCalldata()` | Build revocation calldata |
| `deploySmartAccountsEnvironment()` | Deploy all framework contracts |

## Core Contract Addresses (Deterministic)

| Contract | Address |
|----------|---------|
| DelegationManager | `0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3` |
| EntryPoint (v0.7) | `0x0000000071727De22E5E9d8BAf0edAc6f37da032` |
| SimpleFactory | `0x6B36dE4e79f28e6252e268E4e1F45F489A0b0985` |
| HybridDeleGator (impl) | `0x48B3fCD8d0E8e403C78fa7B39a3e7aAEff91E689` |
| MultiSigDeleGator (impl) | `0x4cf028573e7050e3d5B7f98aF4eF7F61c22AfDF3` |

> These are deterministic CREATE2 deployments. Verify on MegaETH before use. If missing, call `deploySmartAccountsEnvironment()`.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Account not deployed | First UserOp triggers deployment via initCode |
| Gas estimation fails | Use remote `eth_estimateGas` — MegaEVM costs differ from standard EVM |
| Delegation reverted | Check all caveat enforcer conditions are met |
| Bundler rejects UserOp | Ensure EntryPoint v0.7 is deployed on MegaETH |
| Passkey not working | WebAuthn requires HTTPS origin |
| `deploySmartAccountsEnvironment` fails | Ensure deployer has enough ETH for deterministic deployment |
