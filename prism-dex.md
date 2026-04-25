# Prism DEX

Agent skill for integrating [Prism DEX](https://prismfi.cc) on MegaETH.
Prism is Uniswap V3-compatible with one critical difference for pool address computation:
use `abi.encode` for CREATE2 salt (not `encodePacked`).

Use this skill for swaps, quotes, liquidity positions, pool discovery, and taxable-token routing.

## Quick Start (Agent Path)

1. Set chain to MegaETH (`4326` mainnet, `6343` testnet).
2. Get quote from `QuoterV2`.
3. Choose router:
   - Taxable flow: `SelectiveTaxRouter` (or `SelectiveTaxRouterPermit2`).
   - Non-taxable flow: `UniversalRouter` or `SwapRouter02`.
4. Set slippage bounds (`amountOutMinimum` / `amountInMaximum`).
5. Submit transaction (latency-sensitive: `eth_sendRawTransactionSync`).

## Network

| Network | Chain ID | RPC |
|---------|----------|-----|
| MegaETH Mainnet | `4326` | `https://mainnet.megaeth.com/rpc` |
| MegaETH Testnet | `6343` | `https://carrot.megaeth.com/rpc` |

## Mainnet Contracts

| Contract | Address |
|----------|---------|
| Factory | `0x1adb8f973373505bb206e0e5d87af8fb1f5514ef` |
| QuoterV2 | `0xdd79c72c21f7dcd1d034b55caf9177bc42f5df0c` |
| SwapRouter02 | `0xb1f38c36249834d8e3cd582d30101ff4b864f234` |
| UniversalRouter | `0x955d56f6391a496231509134e0d2beadf82a223f` |
| Permit2 | `0x56783fbf77a33871892a2d66337677555714ffbb` |
| NonfungiblePositionManager | `0xcb91c75a6b29700756d4411495be696c4e9a576e` |
| SelectiveTaxRouter | `0x19956ebe69659c78ad4ee500694287bc6f67c4da` |
| SelectiveTaxRouterPermit2 | `0x4c2c989f20794fa92d3082a3d49d9a430face555` |
| Multicall2 | `0x3064c9b0fd9bf73caf668c9c621bb12ec0cccb0c` |
| WETH9 | `0x4200000000000000000000000000000000000006` |

## Core Tokens

| Token | Address | Decimals |
|-------|---------|----------|
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| USDm | `0xfafddbb3fc7688494971a79cc65dca3ef82079e7` | 18 |
| BTC.b | `0xb0f70c0bd6fd87dbeb7c10dc692a2a6106817072` | 8 |
| USDT0 | `0xb8ce59fc3717ada4c02eadf9682a9e934f625ebb` | 6 |

## Default Decisions

- Quote with `QuoterV2` only (read-only).
- Execute swaps with router contracts only.
- Use `Permit2` for better UX when supported.
- Re-quote right before submit; always apply slippage bounds.
- For low latency flows, submit via `eth_sendRawTransactionSync`.

## Minimal Setup

```bash
npm install viem @uniswap/v3-sdk @uniswap/sdk-core
```

## Quote (Exact Input)

```javascript
const [amountOut] = await publicClient.simulateContract({
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

## Swap (Exact Input)

```javascript
await walletClient.writeContract({
  address: WETH,
  abi: erc20Abi,
  functionName: 'approve',
  args: [SWAP_ROUTER, parseEther('1')]
})

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
  }]
})
```

## Swap (Exact Output)

```javascript
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
    amountInMaximum: amountIn * 101n / 100n,
    sqrtPriceLimitX96: 0n
  }]
})
```

## Multi-hop Swap

```javascript
const path = encodePacked(
  ['address', 'uint24', 'address', 'uint24', 'address'],
  [WETH, 3000, USDM, 3000, TOKEN_OUT]
)

await walletClient.writeContract({
  address: SWAP_ROUTER,
  abi: SwapRouter02Abi,
  functionName: 'exactInput',
  args: [{
    path,
    recipient: walletClient.account.address,
    amountIn: parseEther('1'),
    amountOutMinimum
  }]
})
```

## Native ETH Swap

```javascript
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
    amountOutMinimum,
    sqrtPriceLimitX96: 0n
  }],
  value: parseEther('1')
})
```

## SelectiveTaxRouter (Required for Taxable Flow)

Do not use `swapWithTax` (not the current ABI). Use `exactInputSingle` / `exactOutputSingle`.

```javascript
const isTaxed = await publicClient.readContract({
  address: SELECTIVE_TAX_ROUTER,
  abi: SelectiveTaxRouterAbi,
  functionName: 'willBeTaxed',
  args: [tokenIn, tokenOut, userAddress]
})

const [taxAmount, swapAmount] = await publicClient.readContract({
  address: SELECTIVE_TAX_ROUTER,
  abi: SelectiveTaxRouterAbi,
  functionName: 'calculateTax',
  args: [tokenIn, tokenOut, userAddress, amountIn]
})

await walletClient.writeContract({
  address: tokenIn,
  abi: erc20Abi,
  functionName: 'approve',
  args: [SELECTIVE_TAX_ROUTER, amountIn]
})

await walletClient.writeContract({
  address: SELECTIVE_TAX_ROUTER,
  abi: SelectiveTaxRouterAbi,
  functionName: 'exactInputSingle',
  args: [{
    tokenIn,
    tokenOut,
    fee: 3000,
    recipient: userAddress,
    deadline: BigInt(Math.floor(Date.now() / 1000) + 120),
    amountIn,
    amountOutMinimum,
    sqrtPriceLimitX96: 0n
  }]
})
```

Rules:
- Tax applies if either side is taxable and user is not exempt.
- `exactInputSingle`: quote tax first with `calculateTax`.
- `exactOutputSingle`: set `amountInMaximum` with tax headroom.
- Non-taxable swaps should use `UniversalRouter` or `SwapRouter02`.

## Permit2 Pattern

```javascript
await walletClient.writeContract({
  address: TOKEN,
  abi: erc20Abi,
  functionName: 'approve',
  args: [PERMIT2, maxUint256]
})

// Then sign Permit2 data per swap and execute via UniversalRouter
// or SelectiveTaxRouterPermit2 wrapper.
```

## Liquidity Position (NFT)

```javascript
await walletClient.writeContract({
  address: NFT_MANAGER,
  abi: NonfungiblePositionManagerAbi,
  functionName: 'mint',
  args: [{
    token0,
    token1,
    fee: 3000,
    tickLower,
    tickUpper,
    amount0Desired,
    amount1Desired,
    amount0Min: 0n,
    amount1Min: 0n,
    recipient: walletClient.account.address,
    deadline: BigInt(Math.floor(Date.now() / 1000) + 600)
  }]
})
```

`token0` must be lower-address token, `token1` must be higher-address token.

To remove: call `decreaseLiquidity`, then `collect`.

## Pool Address Computation (Critical)

Pool init code hash:
`0xe34f199b19b2b4f47f68442619d555527d244f78a3297ea89325f843f87b8b54`

```javascript
function getPoolAddress(tokenA, tokenB, fee) {
  const [token0, token1] =
    tokenA.toLowerCase() < tokenB.toLowerCase() ? [tokenA, tokenB] : [tokenB, tokenA]

  const salt = keccak256(
    encodeAbiParameters(
      [
        { type: 'address', name: 'token0' },
        { type: 'address', name: 'token1' },
        { type: 'uint24', name: 'fee' }
      ],
      [token0, token1, fee]
    )
  )

  return getCreate2Address({ from: FACTORY, salt, bytecodeHash: INIT_CODE_HASH })
}
```

Use `abi.encode` semantics (padded). Do not use `encodePacked` for the salt.

## MegaETH Runtime Notes

- Use `eth_sendRawTransactionSync` for instant receipt in latency-sensitive flows.
- Base fee is typically near `0.001 gwei`.
- Prefer batched reads (Multicall `aggregate3`) for quote-heavy screens.
- Keep one WS connection and send keepalive (`eth_chainId`) around every 30s.
- Always use fresh quotes and short deadlines (commonly 30-120 seconds for trading flows).

## Fee Tiers

| Fee | Use Case | Tick Spacing |
|-----|----------|--------------|
| `100` (0.01%) | Stablecoin pairs | 1 |
| `500` (0.05%) | Stable-correlated | 10 |
| `3000` (0.3%) | Standard pairs | 60 |
| `10000` (1%) | Exotic/volatile | 200 |

## Endpoints

```javascript
const tokenList = await fetch('https://prismfi.cc/tokenlist.json').then(r => r.json())
const pools = await fetch('https://prismfi.cc/api/pools/list').then(r => r.json())
const routes = await fetch('https://prismfi.cc/api/swaps/swap-pools').then(r => r.json())
```

## Resources

- Prism: https://prismfi.cc
- Prism Docs: https://prismfi.cc/docs
- MegaETH Docs: https://docs.megaeth.com
- Uniswap V3 Concepts: https://docs.uniswap.org/contracts/v3/overview