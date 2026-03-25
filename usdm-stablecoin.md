# USDm Stablecoin on MegaETH

AI coding skill for integrating USDm, MegaETH's native stablecoin. Covers token operations, ERC-2612 permit flows, approvals, balance checks, and integration patterns with MegaNames, Kumbaya DEX, and other MegaETH protocols.

## What This Skill Is For

Use this skill when the user asks for:
- Transferring, approving, or checking USDm balances
- Using ERC-2612 permit for gasless approvals
- Integrating USDm payments into contracts or frontends
- Swapping to/from USDm on Kumbaya DEX
- Understanding USDm's role in MegaETH's fee model

## Token Details

| Property | Value |
|----------|-------|
| Name | MegaUSD |
| Symbol | USDm |
| Decimals | 18 |
| Standard | ERC-20 + ERC-2612 (permit) |

### Contract Addresses

| Network | Address |
|---------|---------|
| Mainnet (4326) | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` |
| Testnet (6343) | `0xFd16854D7fDC1399F05d5F22bfa0A2311d54eA07` |

**Explorers:**
- Mainnet: [Blockscout](https://megaeth.blockscout.com/address/0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7) · [Etherscan](https://mega.etherscan.io/address/0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7)

## Background

USDm is issued through Ethena's stablecoin stack (USDtb rails). Reserves are primarily invested in BlackRock's tokenized U.S. Treasury fund (BUIDL) via Securitize. The yield from reserves covers MegaETH's sequencer operating costs, enabling the chain to price gas at-cost rather than extracting margin from users.

USDm is the primary payment token across the MegaETH ecosystem — used by MegaNames, Kumbaya DEX pairs, and other native protocols.

## Default Stack Decisions (Opinionated)

### 1. Use permit over approve when possible
USDm supports ERC-2612 — sign a permit off-chain, submit it with the action in one transaction. Better UX, saves a round-trip.

### 2. Use `eth_sendRawTransactionSync` for all writes
Instant receipts via EIP-7966. No polling.

### 3. Always use 18 decimals
`1 USDm = 1e18 wei`. Use `parseUnits('1', 18)` or `parseEther('1')` — they're equivalent.

### 4. Check allowance before approve
Avoid unnecessary approves — check existing allowance first. Some protocols (Permit2) use a single infinite approve.

## Core Operations

### Check Balance

```typescript
import { formatUnits } from 'viem'

const USDM = '0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7'

const balance = await publicClient.readContract({
  address: USDM,
  abi: erc20Abi,
  functionName: 'balanceOf',
  args: [userAddress]
})

console.log(`${formatUnits(balance, 18)} USDm`)
```

### Transfer

```typescript
await walletClient.writeContract({
  address: USDM,
  abi: erc20Abi,
  functionName: 'transfer',
  args: [recipientAddress, parseUnits('10', 18)] // 10 USDm
})
```

### Approve

```typescript
await walletClient.writeContract({
  address: USDM,
  abi: erc20Abi,
  functionName: 'approve',
  args: [spenderAddress, parseUnits('100', 18)]
})
```

### Check Allowance

```typescript
const allowance = await publicClient.readContract({
  address: USDM,
  abi: erc20Abi,
  functionName: 'allowance',
  args: [ownerAddress, spenderAddress]
})
```

## ERC-2612 Permit (Gasless Approvals)

USDm supports `permit()` — sign an off-chain message to authorize spending without a separate approve transaction.

### Sign a Permit

```typescript
import { parseUnits } from 'viem'

const USDM = '0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7'

// Get current nonce
const nonce = await publicClient.readContract({
  address: USDM,
  abi: [{
    name: 'nonces',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'owner', type: 'address' }],
    outputs: [{ type: 'uint256' }]
  }],
  functionName: 'nonces',
  args: [walletClient.account.address]
})

// Get domain separator
const domainSeparator = await publicClient.readContract({
  address: USDM,
  abi: [{
    name: 'DOMAIN_SEPARATOR',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ type: 'bytes32' }]
  }],
  functionName: 'DOMAIN_SEPARATOR'
})

const deadline = BigInt(Math.floor(Date.now() / 1000) + 3600) // 1 hour

// Sign EIP-712 permit
const signature = await walletClient.signTypedData({
  domain: {
    name: 'MegaUSD',
    version: '1',
    chainId: 4326,
    verifyingContract: USDM
  },
  types: {
    Permit: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value', type: 'uint256' },
      { name: 'nonce', type: 'uint256' },
      { name: 'deadline', type: 'uint256' }
    ]
  },
  primaryType: 'Permit',
  message: {
    owner: walletClient.account.address,
    spender: spenderAddress,
    value: parseUnits('100', 18),
    nonce,
    deadline
  }
})
```

### Submit Permit

```typescript
// Decode signature
const { r, s, v } = parseSignature(signature)

// Call permit on the token
await walletClient.writeContract({
  address: USDM,
  abi: [{
    name: 'permit',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value', type: 'uint256' },
      { name: 'deadline', type: 'uint256' },
      { name: 'v', type: 'uint8' },
      { name: 'r', type: 'bytes32' },
      { name: 's', type: 'bytes32' }
    ],
    outputs: []
  }],
  functionName: 'permit',
  args: [ownerAddress, spenderAddress, parseUnits('100', 18), deadline, v, r, s]
})
```

### Permit + Action in One Transaction (Contract Pattern)

```solidity
// Example: approve + register MegaNames in a single call
function registerWithPermit(
    string calldata label,
    address owner,
    uint256 numYears,
    uint256 deadline,
    uint8 v, bytes32 r, bytes32 s
) external returns (uint256) {
    IERC20Permit(USDM).permit(msg.sender, address(this), type(uint256).max, deadline, v, r, s);
    return _register(label, owner, numYears);
}
```

## Integration Patterns

### Pattern 1: Accept USDm Payments in a Contract

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

address constant USDM = 0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7;

function pay(uint256 amount) external {
    // Requires prior approval or use permit pattern
    IERC20(USDM).transferFrom(msg.sender, address(this), amount);
    // ... process payment
}
```

### Pattern 2: Check USDm Balance in Frontend (wagmi)

```typescript
import { useReadContract } from 'wagmi'
import { erc20Abi, formatUnits } from 'viem'

const USDM = '0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7'

function UsdmBalance({ address }: { address: `0x${string}` }) {
  const { data: balance } = useReadContract({
    address: USDM,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [address]
  })

  return <span>{balance ? formatUnits(balance, 18) : '0'} USDm</span>
}
```

### Pattern 3: Permit2 (Kumbaya DEX)

Kumbaya DEX uses Uniswap's Permit2 contract. One-time infinite approve to Permit2, then sign per-swap permits:

```typescript
const PERMIT2 = '0x000000000022D473030F116dDEE9F6B43aC78BA3'

// One-time: approve Permit2 to spend your USDm
await walletClient.writeContract({
  address: USDM,
  abi: erc20Abi,
  functionName: 'approve',
  args: [PERMIT2, maxUint256]
})

// Then use Permit2 signatures for each swap (handled by Kumbaya SDK)
```

## Where USDm Is Used

| Protocol | Usage |
|----------|-------|
| [MegaNames](https://meganame.market) | Registration fees, subdomain marketplace payments |
| [Kumbaya DEX](https://kumbaya.xyz) | Trading pairs (USDm/WETH, USDm/tokens) |
| x402 (pending) | Pay-per-request API payments |
| Gas sponsorship | Paymasters accept USDm for gas abstraction |

## Other Stablecoins on MegaETH

USDm is the native stablecoin, but others are first-class citizens:

| Token | Description |
|-------|-------------|
| **USDT0** | Canonical USDT representation on MegaETH |
| **cUSD** | Circle's bridged USDC |

These maintain deep liquidity, oracle coverage, and DEX routing alongside USDm.

## MegaETH-Specific Notes

- **Instant receipts:** Use `eth_sendRawTransactionSync` for all USDm transfers/approves
- **Gas costs:** SSTORE for allowance updates follows MegaEVM pricing (first-time approve = new slot = expensive; subsequent = cheap)
- **No ETH needed for approve:** If using a paymaster, users can approve USDm without holding ETH
- **18 decimals:** Same as ETH wei — `parseEther` and `parseUnits('x', 18)` are interchangeable
