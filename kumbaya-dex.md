# Kumbaya DEX on MegaETH

AI coding skill for integrating [Kumbaya DEX](https://kumbaya.xyz), the most liquid DEX on MegaETH. Uniswap V3 fork with Kumbaya-specific SDK packages. Covers swaps, quoting, liquidity provision, pool discovery, and Permit2 flows.

## What This Skill Is For

Use this skill when the user asks for:
- Swapping tokens on MegaETH
- Getting swap quotes (price impact, expected output)
- Adding or removing liquidity on Kumbaya
- Creating new pools
- Building swap frontends or trading bots
- Computing pool addresses (CREATE2 with Kumbaya init code hash)
- Managing liquidity positions (NFT-based)

## Architecture

Kumbaya is a Uniswap V3 fork. If you know Uni V3, you know Kumbaya — with one critical difference: **the pool init code hash is different**, so CREATE2 pool address computation must use Kumbaya's hash.

```
Pool Init Code Hash: 0x851d77a45b8b9a205fb9f44cb829cceba85282714d2603d601840640628a3da7
```

## Chain Configuration

| Network | Chain ID | RPC |
|---------|----------|-----|
| MegaETH Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` |
| MegaETH Testnet | 6343 | `https://carrot.megaeth.com/rpc` |

## Contract Addresses

### Mainnet (Chain ID: 4326)

| Contract | Address |
|----------|---------|
| UniswapV3Factory | `0x68b34591f662508076927803c567Cc8006988a09` |
| SwapRouter02 | `0xE5BbEF8De2DB447a7432A47EBa58924d94eE470e` |
| UniversalRouter | `0xAAB1C664CeaD881AfBB58555e6A3a79523D3e4C0` |
| QuoterV2 | `0x1F1a8dC7E138C34b503Ca080962aC10B75384a27` |
| NonfungiblePositionManager | `0x2b781C57e6358f64864Ff8EC464a03Fdaf9974bA` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| WETH9 | `0x4200000000000000000000000000000000000006` |
| Multicall2 | `0xf6f404ac6289ab8eB1caf244008b5F073d59385c` |
| TickLens | `0x9c22f028e0a1dc76EB895a1929DBc517c9D0593e` |
| UniswapV3Staker | `0x9F393A399321110Fb7D85aCc812b8e48A7c569aC` |

### Testnet (Chain ID: 6343)

| Contract | Address |
|----------|---------|
| UniswapV3Factory | `0x53447989580f541bc138d29A0FcCf72AfbBE1355` |
| SwapRouter02 | `0x8268DC930BA98759E916DEd4c9F367A844814023` |
| UniversalRouter | `0x7E6c4Ada91e432efe5F01FbCb3492Bd3eb7ccD2E` |
| QuoterV2 | `0xfb230b93803F90238cB03f254452bA3a3b0Ec38d` |
| NonfungiblePositionManager | `0x367f9db1F974eA241ba046b77B87C58e2947d8dF` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| WETH9 | `0x4200000000000000000000000000000000000006` |
| Multicall2 | `0xc638099246A98B3A110429B47B3f42CA037BC0a3` |
| TickLens | `0x6D65B4854944Fd93Cd568bb1B54EE22Fe9BF2faa` |
| UniswapV3Staker | `0x511f4EC90936450152895b2C2FD20AF6DC72663b` |

### Common Tokens (Mainnet)

| Token | Address | Decimals |
|-------|---------|----------|
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| USDm | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` | 18 |

## SDK Installation

```bash
npm install @kumbaya_xyz/sdk-core @kumbaya_xyz/v3-sdk @kumbaya_xyz/smart-order-router
```

| Package | Purpose |
|---------|---------|
| `@kumbaya_xyz/sdk-core` | Core SDK, chain definitions, token/currency types |
| `@kumbaya_xyz/v3-sdk` | V3 pool math, position management, tick operations |
| `@kumbaya_xyz/router-sdk` | Swap encoding helpers |
| `@kumbaya_xyz/universal-router-sdk` | UniversalRouter command encoding |
| `@kumbaya_xyz/smart-order-router` | Optimal multi-hop routing |
| `@kumbaya_xyz/default-token-list` | Curated token list |

## Default Stack Decisions (Opinionated)

### 1. Use QuoterV2 for quotes, SwapRouter02 for execution
QuoterV2 is a read-only quoter (use `staticCall`). SwapRouter02 executes swaps. Don't mix them.

### 2. Use `eth_sendRawTransactionSync` for swap execution
Instant receipts via EIP-7966. No polling needed — critical for trading bots.

### 3. Use Permit2 for token approvals
One-time infinite approve of each token to Permit2, then sign per-swap permits. Better UX and security than per-contract approvals.

### 4. Always quote before swapping
Never submit a swap without a fresh quote. Use `quoteExactInputSingle` / `quoteExactOutputSingle` to get `amountOutMinimum` / `amountInMaximum`.

### 5. Fee tiers: know which to use

| Fee | Use Case | Tick Spacing |
|-----|----------|-------------|
| 100 (0.01%) | Stablecoin-stablecoin | 1 |
| 500 (0.05%) | Stable-correlated pairs | 10 |
| 3000 (0.3%) | Standard pairs (most common) | 60 |
| 10000 (1%) | Exotic/volatile pairs | 200 |

## Core Operations

### Get a Quote

```typescript
import { createPublicClient, http, parseEther } from 'viem'
import { QuoterV2Abi } from './abis'

const QUOTER_V2 = '0x1F1a8dC7E138C34b503Ca080962aC10B75384a27'
const WETH = '0x4200000000000000000000000000000000000006'
const USDM = '0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7'

// Quote: how much USDm for 1 WETH?
const [amountOut, sqrtPriceX96After, initializedTicksCrossed, gasEstimate] =
  await publicClient.simulateContract({
    address: QUOTER_V2,
    abi: QuoterV2Abi,
    functionName: 'quoteExactInputSingle',
    args: [{
      tokenIn: WETH,
      tokenOut: USDM,
      fee: 3000,
      amountIn: parseEther('1'),
      sqrtPriceLimitX96: 0n
    }]
  }).then(r => r.result)
```

### Exact Input Swap (SwapRouter02)

```typescript
const SWAP_ROUTER = '0xE5BbEF8De2DB447a7432A47EBa58924d94eE470e'

// 1. Approve token (or use Permit2)
await walletClient.writeContract({
  address: WETH,
  abi: erc20Abi,
  functionName: 'approve',
  args: [SWAP_ROUTER, parseEther('1')]
})

// 2. Swap
await walletClient.writeContract({
  address: SWAP_ROUTER,
  abi: SwapRouter02Abi,
  functionName: 'exactInputSingle',
  args: [{
    tokenIn: WETH,
    tokenOut: USDM,
    fee: 3000,
    recipient: walletClient.account.address,
    amountIn: parseEther('1'),
    amountOutMinimum: amountOut * 99n / 100n, // 1% slippage
    sqrtPriceLimitX96: 0n
  }]
})
```

### Exact Output Swap

```typescript
// "I want exactly 1000 USDm — how much WETH will it cost?"
const [amountIn] = await publicClient.simulateContract({
  address: QUOTER_V2,
  abi: QuoterV2Abi,
  functionName: 'quoteExactOutputSingle',
  args: [{
    tokenIn: WETH,
    tokenOut: USDM,
    fee: 3000,
    amount: parseUnits('1000', 18),
    sqrtPriceLimitX96: 0n
  }]
}).then(r => r.result)

await walletClient.writeContract({
  address: SWAP_ROUTER,
  abi: SwapRouter02Abi,
  functionName: 'exactOutputSingle',
  args: [{
    tokenIn: WETH,
    tokenOut: USDM,
    fee: 3000,
    recipient: walletClient.account.address,
    amountOut: parseUnits('1000', 18),
    amountInMaximum: amountIn * 101n / 100n, // 1% slippage
    sqrtPriceLimitX96: 0n
  }]
})
```

### Multi-Hop Swap (Path Encoding)

```typescript
import { encodePacked } from 'viem'

// WETH → USDm → TOKEN (two hops)
const path = encodePacked(
  ['address', 'uint24', 'address', 'uint24', 'address'],
  [WETH, 3000, USDM, 3000, TOKEN_ADDRESS]
)

await walletClient.writeContract({
  address: SWAP_ROUTER,
  abi: SwapRouter02Abi,
  functionName: 'exactInput',
  args: [{
    path,
    recipient: walletClient.account.address,
    amountIn: parseEther('1'),
    amountOutMinimum: 0n // Set from quote in production!
  }]
})
```

### Swap ETH → Token (Native ETH)

```typescript
// Send ETH as msg.value — router wraps to WETH internally
await walletClient.writeContract({
  address: SWAP_ROUTER,
  abi: SwapRouter02Abi,
  functionName: 'exactInputSingle',
  args: [{
    tokenIn: WETH,
    tokenOut: USDM,
    fee: 3000,
    recipient: walletClient.account.address,
    amountIn: parseEther('1'),
    amountOutMinimum: amountOut * 99n / 100n,
    sqrtPriceLimitX96: 0n
  }],
  value: parseEther('1') // Send native ETH
})
```

## Liquidity Provision

### Add Liquidity (New Position)

```typescript
const NFT_MANAGER = '0x2b781C57e6358f64864Ff8EC464a03Fdaf9974bA'

// 1. Approve both tokens to NonfungiblePositionManager
await walletClient.writeContract({
  address: WETH, abi: erc20Abi, functionName: 'approve',
  args: [NFT_MANAGER, parseEther('10')]
})
await walletClient.writeContract({
  address: USDM, abi: erc20Abi, functionName: 'approve',
  args: [NFT_MANAGER, parseUnits('10000', 18)]
})

// 2. Mint position
await walletClient.writeContract({
  address: NFT_MANAGER,
  abi: NonfungiblePositionManagerAbi,
  functionName: 'mint',
  args: [{
    token0: USDM,       // token0 < token1 (sorted by address)
    token1: WETH,
    fee: 3000,
    tickLower: -60000,   // Price range lower bound
    tickUpper: 60000,    // Price range upper bound
    amount0Desired: parseUnits('5000', 18),
    amount1Desired: parseEther('5'),
    amount0Min: 0n,
    amount1Min: 0n,
    recipient: walletClient.account.address,
    deadline: BigInt(Math.floor(Date.now() / 1000) + 600)
  }]
})
```

> **Important:** token0 must be the lower address. Sort tokens before calling mint.

### Increase Liquidity

```typescript
await walletClient.writeContract({
  address: NFT_MANAGER,
  abi: NonfungiblePositionManagerAbi,
  functionName: 'increaseLiquidity',
  args: [{
    tokenId: positionTokenId,
    amount0Desired: parseUnits('1000', 18),
    amount1Desired: parseEther('1'),
    amount0Min: 0n,
    amount1Min: 0n,
    deadline: BigInt(Math.floor(Date.now() / 1000) + 600)
  }]
})
```

### Collect Fees

```typescript
await walletClient.writeContract({
  address: NFT_MANAGER,
  abi: NonfungiblePositionManagerAbi,
  functionName: 'collect',
  args: [{
    tokenId: positionTokenId,
    recipient: walletClient.account.address,
    amount0Max: maxUint128,
    amount1Max: maxUint128
  }]
})
```

### Remove Liquidity

```typescript
// 1. Decrease liquidity
await walletClient.writeContract({
  address: NFT_MANAGER,
  abi: NonfungiblePositionManagerAbi,
  functionName: 'decreaseLiquidity',
  args: [{
    tokenId: positionTokenId,
    liquidity: positionLiquidity, // Full amount to remove all
    amount0Min: 0n,
    amount1Min: 0n,
    deadline: BigInt(Math.floor(Date.now() / 1000) + 600)
  }]
})

// 2. Collect the tokens
await walletClient.writeContract({
  address: NFT_MANAGER,
  abi: NonfungiblePositionManagerAbi,
  functionName: 'collect',
  args: [{
    tokenId: positionTokenId,
    recipient: walletClient.account.address,
    amount0Max: maxUint128,
    amount1Max: maxUint128
  }]
})
```

## Pool Discovery

### Compute Pool Address (CREATE2)

```typescript
import { getCreate2Address, keccak256, encodePacked } from 'viem'

const FACTORY = '0x68b34591f662508076927803c567Cc8006988a09'
const INIT_CODE_HASH = '0x851d77a45b8b9a205fb9f44cb829cceba85282714d2603d601840640628a3da7'

function getPoolAddress(tokenA: string, tokenB: string, fee: number): string {
  // Sort tokens
  const [token0, token1] = tokenA.toLowerCase() < tokenB.toLowerCase()
    ? [tokenA, tokenB]
    : [tokenB, tokenA]

  return getCreate2Address({
    from: FACTORY,
    salt: keccak256(encodePacked(
      ['address', 'address', 'uint24'],
      [token0 as `0x${string}`, token1 as `0x${string}`, fee]
    )),
    bytecodeHash: INIT_CODE_HASH as `0x${string}`
  })
}
```

> **Critical:** Use Kumbaya's init code hash (`0x851d77...`), NOT the standard Uniswap hash. Using the wrong hash will compute wrong pool addresses.

### Read Pool State

```typescript
const poolAddress = getPoolAddress(WETH, USDM, 3000)

// Get current price and tick
const [sqrtPriceX96, tick, , , , ,] = await publicClient.readContract({
  address: poolAddress as `0x${string}`,
  abi: UniswapV3PoolAbi,
  functionName: 'slot0'
})

// Get liquidity
const liquidity = await publicClient.readContract({
  address: poolAddress as `0x${string}`,
  abi: UniswapV3PoolAbi,
  functionName: 'liquidity'
})
```

## Permit2 Flow

Kumbaya's UniversalRouter uses Permit2 for token approvals:

```typescript
const PERMIT2 = '0x000000000022D473030F116dDEE9F6B43aC78BA3'

// One-time: approve Permit2 to spend your token
await walletClient.writeContract({
  address: USDM,
  abi: erc20Abi,
  functionName: 'approve',
  args: [PERMIT2, maxUint256]
})

// Per-swap: sign a Permit2 signature (handled by Kumbaya SDK)
// The UniversalRouter pulls tokens via Permit2
```

## Foundry Integration (Solidity)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ISwapRouter02} from "@kumbaya_xyz/swap-router-contracts/interfaces/ISwapRouter02.sol";

address constant SWAP_ROUTER = 0xE5BbEF8De2DB447a7432A47EBa58924d94eE470e;
address constant WETH = 0x4200000000000000000000000000000000000006;
address constant USDM = 0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7;

contract MySwapper {
    function swapExactInput(uint256 amountIn, uint256 minOut) external returns (uint256) {
        IERC20(WETH).transferFrom(msg.sender, address(this), amountIn);
        IERC20(WETH).approve(SWAP_ROUTER, amountIn);

        return ISwapRouter02(SWAP_ROUTER).exactInputSingle(
            ISwapRouter02.ExactInputSingleParams({
                tokenIn: WETH,
                tokenOut: USDM,
                fee: 3000,
                recipient: msg.sender,
                amountIn: amountIn,
                amountOutMinimum: minOut,
                sqrtPriceLimitX96: 0
            })
        );
    }
}
```

## MegaETH-Specific Notes

- **Instant receipts:** Use `eth_sendRawTransactionSync` for swap execution — get confirmation in <10ms. Critical for MEV and trading bots.
- **Gas:** Use `eth_estimateGas` via RPC. MegaETH SSTORE costs differ — first-time Permit2 approval is expensive (new storage slot), subsequent swaps are cheap.
- **No mempool:** MegaETH has no public mempool. Transactions are ordered by the sequencer. Front-running risk is structurally lower than Ethereum.
- **Multicall:** Use Multicall2 (`0xf6f404ac6289ab8eB1caf244008b5F073d59385c`) to batch read calls (pool state + quote + balances in one RPC call).

## Resources

- [Kumbaya Integrator Kit](https://github.com/Kumbaya-xyz/integrator-kit) — Addresses, ABIs, and integration tests
- [Kumbaya SDK packages](https://www.npmjs.com/org/kumbaya_xyz) — npm packages
- [Uniswap V3 Docs](https://docs.uniswap.org/contracts/v3/overview) — Core concepts (Kumbaya is a fork)
