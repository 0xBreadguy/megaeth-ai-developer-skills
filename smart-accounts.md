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

## Environment Setup

```typescript
import { getSmartAccountsEnvironment } from '@metamask/smart-accounts-kit';

// Get all contract addresses for a chain
const environment = getSmartAccountsEnvironment(4326); // MegaETH mainnet

// If contracts aren't deployed yet on MegaETH:
import { deploySmartAccountsEnvironment } from '@metamask/smart-accounts-kit';
await deploySmartAccountsEnvironment({ client, account: deployer });
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

For EOA signers, bypass the bundler entirely. On MegaETH, use `eth_sendRawTransactionSync` for instant receipts:

```typescript
import { createWalletClient } from 'viem';
import { DelegationManager } from '@metamask/smart-accounts-kit/contracts';
import { createExecution, ExecutionMode } from '@metamask/smart-accounts-kit';

const walletClient = createWalletClient({
  account: owner,
  chain: megaeth,
  transport: http(),
});

const execution = createExecution({ target: tokenAddress, callData });

const redeemCalldata = DelegationManager.encode.redeemDelegations({
  delegations: [[signedDelegation]],
  modes: [ExecutionMode.SingleDefault],
  executions: [[execution]],
});

const tx = await walletClient.sendTransaction({
  to: environment.DelegationManager,
  data: redeemCalldata,
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
import { erc7715ProviderActions } from '@metamask/smart-accounts-kit/actions';

const walletClient = createWalletClient({
  transport: custom(window.ethereum),
}).extend(erc7715ProviderActions());

const grantedPermissions = await walletClient.requestExecutionPermissions([
  {
    chainId: megaeth.id,
    expiry: Math.floor(Date.now() / 1000) + 86400,
    signer: {
      type: 'account',
      data: { address: sessionAccount.address },
    },
    permission: {
      type: 'native-token-transfer',
      data: { allowance: parseEther('1') },
    },
    isAdjustmentAllowed: true,
  },
]);
```

## Key API Methods

| Method | Import | Purpose |
|--------|--------|---------|
| `toMetaMaskSmartAccount()` | `@metamask/smart-accounts-kit` | Create smart account instance |
| `createDelegation()` | `@metamask/smart-accounts-kit` | Build delegation with scopes/caveats |
| `createExecution()` | `@metamask/smart-accounts-kit` | Build execution struct for redemption |
| `getSmartAccountsEnvironment()` | `@metamask/smart-accounts-kit` | Get contract addresses for a chain |
| `DelegationManager.encode.redeemDelegations()` | `@metamask/smart-accounts-kit/contracts` | Build redemption calldata |
| `DelegationManager.encode.disableDelegation()` | `@metamask/smart-accounts-kit/contracts` | Build revocation calldata |
| `smartAccount.signDelegation()` | Smart account method | Sign delegation off-chain |
| `erc7715ProviderActions()` | `@metamask/smart-accounts-kit/actions` | Wallet extension for ERC-7715 |

## Core Contract Addresses (Deterministic)

| Contract | Address |
|----------|---------|
| DelegationManager | `0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3` |
| EntryPoint (v0.7) | `0x0000000071727De22E5E9d8BAf0edAc6f37da032` |
| SimpleFactory | `0x69Aa2f9fe1572F1B640E1bbc512f5c3a734fc77c` |
| HybridDeleGator (impl) | `0x48dBe696A4D990079e039489bA2053B36E8FFEC4` |
| MultiSigDeleGator (impl) | `0x56a9EdB16a0105eb5a4C54f4C062e2868844f3A7` |

> These are deterministic CREATE2 deployments. Verify on MegaETH before use. If missing, call `deploySmartAccountsEnvironment()`.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Account not deployed | First UserOp triggers deployment via initCode |
| Gas estimation fails | Use remote `eth_estimateGas` — MegaEVM costs differ; 10M gas cap per call |
| Delegation reverted | Check all caveat enforcer conditions are met |
| Bundler rejects UserOp | Ensure EntryPoint v0.7 is deployed on MegaETH |
| Passkey not working | WebAuthn requires HTTPS origin |
| `deploySmartAccountsEnvironment` fails | Ensure deployer has enough ETH for deterministic deployment |
