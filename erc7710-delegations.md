# ERC-7710 Delegations

## Overview

ERC-7710 defines scoped, revocable, composable on-chain permissions. A **delegator** grants a **delegate** authority to act on their behalf, constrained by **caveats** (enforcer contracts that restrict what the delegate can do).

Key properties:
- **Scoped** — permissions limited by caveat enforcers (targets, methods, values, time)
- **Revocable** — delegator can revoke at any time
- **Composable** — delegations can be chained (redelegation)
- **Off-chain signing** — delegations are signed off-chain, redeemed on-chain

## MegaETH Chain Configuration

| Network | Chain ID | RPC |
|---------|----------|-----|
| Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` |
| Testnet | 6343 | `https://carrot.megaeth.com/rpc` |

```typescript
import { defineChain } from 'viem';

export const megaeth = defineChain({
  id: 4326,
  name: 'MegaETH',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://mainnet.megaeth.com/rpc'] },
  },
});
```

## Contract Addresses

All addresses are **deterministic deployments** — identical across every chain where the Delegation Framework is deployed.

> **Note:** Verify these contracts exist on MegaETH before using. If not yet deployed, use `deploySmartAccountsEnvironment()` from `@metamask/smart-accounts-kit`.

### Core

| Contract | Address |
|----------|---------|
| DelegationManager | `0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3` |

### Caveat Enforcers

| Enforcer | Address | Purpose |
|----------|---------|---------|
| AllowedCalldataEnforcer | `0xc2b0d624c1c4319760C96503BA27C347F3260f55` | Restrict calldata patterns |
| AllowedMethodsEnforcer | `0x2c21fD0Cb9DC8445CB3fb0DC5E7Bb0Aca01842B5` | Restrict callable function selectors |
| AllowedTargetsEnforcer | `0x7F20f61b1f09b08D970938F6fa563634d65c4EeB` | Restrict target contract addresses |
| TimestampEnforcer | `0x1046bb45C8d673d4ea75321280DB34899413c069` | Time-bound permissions (start/end) |
| ValueLteEnforcer | `0x92Bf12322527cAA612fd31a0e810472BBB106A8F` | Cap msg.value per call |
| LimitedCallsEnforcer | `0x04658B29F6b82ed55274221a06Fc97D318E25416` | Limit total number of calls |
| NativeTokenPeriodTransferEnforcer | `0x9BC0FAf4Aca5AE429F4c06aEEaC517520CB16BD9` | Recurring native token allowance |
| ERC20PeriodTransferEnforcer | `0x474e3Ae7E169e940607cC624Da8A15Eb120139aB` | Recurring ERC-20 allowance |
| NativeTokenTransferAmountEnforcer | `0xF71af580b9c3078fbc2BBF16FbB8EEd82b330320` | One-time native token cap |
| ERC20TransferAmountEnforcer | `0xf100b0819427117EcF76Ed94B358B1A5b5C6D2Fc` | One-time ERC-20 cap |
| RedeemerEnforcer | `0xE144b0b2618071B4E56f746313528a669c7E65c5` | Restrict who can redeem |
| NonceEnforcer | `0xDE4f2FAC4B3D87A1d9953Ca5FC09FCa7F366254f` | Single-use delegations |
| IdEnforcer | `0xC8B5D93463c893401094cc70e66A206fb5987997` | Identify delegations for selective revocation |
| BlockNumberEnforcer | `0x5d9818dF0AE3f66e9c3D0c5029DAF99d1823ca6c` | Block-range restrictions |
| ERC20StreamingEnforcer | `0x56c97aE02f233B29fa03502Ecc0457266d9be00e` | Linear ERC-20 streaming allowance |
| NativeTokenStreamingEnforcer | `0xD10b97905a320b13a0608f7E9cC506b56747df19` | Linear native token streaming allowance |
| NativeTokenPaymentEnforcer | `0x4803a326ddED6dDBc60e659e5ed12d85c7582811` | Require payment to redeem |
| MultiTokenPeriodEnforcer | `0xFB2f1a9BD76d3701B730E5d69C3219D42D80eBb7` | Recurring multi-token allowance |

## Delegation Lifecycle

### 1. Get the environment

```typescript
import { getSmartAccountsEnvironment } from '@metamask/smart-accounts-kit';

// Returns all contract addresses for the chain
const environment = getSmartAccountsEnvironment(4326); // MegaETH mainnet
```

### 2. Create a delegation

```typescript
import { createDelegation } from '@metamask/smart-accounts-kit';
import { parseUnits } from 'viem';

// ERC-20 spending limit: 100 USDC
const delegation = createDelegation({
  from: smartAccount.address,
  to: delegateAddress,
  environment,
  scope: {
    type: 'erc20TransferAmount',
    tokenAddress: USDC_ADDRESS,
    maxAmount: parseUnits('100', 6),
  },
  caveats: [
    { type: 'timestamp', afterThreshold: now, beforeThreshold: expiry },
    { type: 'limitedCalls', limit: 10 },
    { type: 'redeemer', redeemers: [delegateAddress] },
  ],
});
```

### 3. Sign the delegation

```typescript
// Via smart account method
const signature = await smartAccount.signDelegation({ delegation });
const signedDelegation = { ...delegation, signature };
```

### 4. Redeem the delegation

The delegate (or anyone allowed by RedeemerEnforcer) submits on-chain:

```typescript
import { DelegationManager } from '@metamask/smart-accounts-kit/contracts';
import {
  createExecution,
  ExecutionMode,
} from '@metamask/smart-accounts-kit';
import { encodeFunctionData, erc20Abi } from 'viem';

const callData = encodeFunctionData({
  abi: erc20Abi,
  functionName: 'transfer',
  args: [recipientAddress, parseUnits('50', 6)],
});

const execution = createExecution({ target: USDC_ADDRESS, callData });

const redeemCalldata = DelegationManager.encode.redeemDelegations({
  delegations: [[signedDelegation]],
  modes: [ExecutionMode.SingleDefault],
  executions: [[execution]],
});

// Via smart account + bundler
const userOpHash = await bundlerClient.sendUserOperation({
  account: delegateSmartAccount,
  calls: [{ to: delegateSmartAccount.address, data: redeemCalldata }],
});

// Via EOA — use eth_sendRawTransactionSync for instant receipt on MegaETH
const txHash = await delegateWalletClient.sendTransaction({
  to: environment.DelegationManager,
  data: redeemCalldata,
});
```

## Common Patterns

### Time-bound + spending limit

```typescript
const delegation = createDelegation({
  from: delegator.address,
  to: delegateAddress,
  environment,
  scope: {
    type: 'erc20TransferAmount',
    tokenAddress: TOKEN_ADDRESS,
    maxAmount: parseUnits('1000', 18),
  },
  caveats: [
    { type: 'timestamp', afterThreshold: startTimestamp, beforeThreshold: endTimestamp },
    { type: 'limitedCalls', limit: 50 },
  ],
});
```

### Function call scoping (specific methods on specific targets)

```typescript
const delegation = createDelegation({
  from: delegator.address,
  to: delegateAddress,
  environment,
  scope: {
    type: 'functionCall',
    targets: [CONTRACT_ADDRESS],
    selectors: ['swap(address,uint256)'],
  },
  caveats: [
    { type: 'allowedMethods', selectors: ['swap(address,uint256)'] },
  ],
});
```

### Redelegation chains

A delegate can redelegate their authority (with equal or narrower caveats):

```typescript
const redelegation = createDelegation({
  from: delegate1Address,
  to: delegate2Address,
  environment,
  parentDelegation: originalDelegation,
  scope: {
    type: 'erc20TransferAmount',
    tokenAddress: TOKEN_ADDRESS,
    maxAmount: parseUnits('50', 6), // narrower than parent
  },
  caveats: [
    { type: 'timestamp', afterThreshold: now, beforeThreshold: expiry },
  ],
});
```

When redeeming, pass the full chain: `delegations: [[redelegation, originalDelegation]]`.

### Periodic native token allowance

```typescript
const delegation = createDelegation({
  from: delegator.address,
  to: delegateAddress,
  environment,
  scope: {
    type: 'nativeTokenPeriodTransfer',
    periodAmount: parseEther('1'),
    periodDuration: 86400, // 1 ETH per day
    startDate: now,
  },
});
```

## MegaETH Advantage

Use `eth_sendRawTransactionSync` for instant delegation redemption — get the receipt in <10ms instead of polling. This is critical for real-time delegation flows (AI agents, automated trading, session keys).

> **Note:** MegaETH caps `eth_call` and `eth_estimateGas` at 10M gas per call. Heavy delegation operations (large caveat chains) should account for this limit.

## Safe Multisig Integration

Smart accounts created via `Implementation.MultiSig` use a DeleGator-compatible Safe factory. Delegations work the same way — the multisig threshold must be met to sign delegations, but redemption is permissionless (or restricted via RedeemerEnforcer).

## Revocation

```typescript
import { DelegationManager } from '@metamask/smart-accounts-kit/contracts';

// Revoke a specific delegation
const tx = DelegationManager.encode.disableDelegation({ delegation });
```
