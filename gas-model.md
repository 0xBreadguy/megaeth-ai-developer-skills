# Gas Model

> **Sources**: Gas model validated against [MegaEVM spec](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md) and [BLOCK_AND_TX_LIMITS.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/BLOCK_AND_TX_LIMITS.md).

## Multidimensional Gas Model

MegaETH uses a **dual gas model** — compute gas and storage gas are separate dimensions:

| Dimension | Description |
|-----------|-------------|
| **Compute gas** | Execution cost (opcodes, memory) |
| **Storage gas** | Persistent state modification cost |

Both are paid from your gas limit, but tracked separately for resource accounting.

## Base Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Base fee | 0.001 gwei (10⁶ wei) | Fixed, no EIP-1559 adjustment |
| Priority fee | 0 | Ignored unless congested |
| Intrinsic gas | 60,000 | 21K compute + 39K storage (not 21K like Ethereum) |
| Gas forwarding | 98/100 | More gas available in nested calls than Ethereum's 63/64 |

## Per-Transaction Resource Limits (Rex)

| Resource | Limit |
|----------|-------|
| Compute gas | 200M |
| KV updates | 500K |
| State growth slots | 1,000 |
| Data size | 12.5 MB |
| Contract code | 512 KB |
| Calldata | 128 KB |

## Setting Gas Price

### Correct Approach

```javascript
// MegaETH: use base fee directly
const gasPrice = 1000000n; // 0.001 gwei in wei

// Or fetch from RPC (always returns 0.001 gwei)
const baseFee = await client.request({ method: 'eth_gasPrice' });
```

### Common Mistakes

```javascript
// ❌ Wrong: viem adds 20% buffer
const gasPrice = await publicClient.getGasPrice(); // Returns 1.2M wei

// ❌ Wrong: using maxPriorityFeePerGas
const priority = await client.request({ 
  method: 'eth_maxPriorityFeePerGas' 
}); // Returns 0 (hardcoded)
```

## Gas Estimation

### MegaEVM Intrinsic Gas

> ⚠️ **Important:** MegaEVM has different intrinsic gas costs than standard EVM. A simple ETH transfer costs **60,000 gas** on MegaETH, not 21,000.

If you hardcode gas limits, use MegaETH-specific values:

```javascript
// Common operations - MegaETH gas limits
const gasLimits = {
  transfer: 60000n,       // NOT 21000 like standard EVM
  erc20Transfer: 100000n, // Higher than standard EVM
  erc20Approve: 80000n,
  swap: 350000n,
};

// Send with correct gas limit
await wallet.sendTransaction({
  to: recipient,
  value: amount,
  gasLimit: 60000n,       // MegaETH intrinsic gas
  maxFeePerGas: 1000000n,
});
```

### When to Use Remote Estimation

For any non-trivial operation, use `eth_estimateGas` — MegaEVM opcode costs differ from standard EVM:

```javascript
// ✅ Correct: remote estimation
const gas = await client.request({
  method: 'eth_estimateGas',
  params: [{ from, to, data }]
});

// ❌ Wrong: local simulation (Hardhat/Foundry)
// These use standard EVM costs, not MegaEVM
```

For Foundry:
```bash
# Skip local simulation, use remote
forge script Deploy.s.sol --gas-limit 5000000 --skip-simulation
```

## Volatile Data Access Limit

Accessing block metadata caps the **total** compute gas for the entire transaction to 20M — retroactively. This is not a budget for remaining work; it's an absolute ceiling. If the transaction has already used more than 20M compute gas before the volatile opcode, it immediately reverts with OOG.

```solidity
// Operations that trigger the 20M total compute gas cap:
block.timestamp   // TIMESTAMP
block.number      // NUMBER
blockhash(n)      // BLOCKHASH
block.basefee     // BASEFEE
block.prevrandao  // PREVRANDAO
block.gaslimit    // GASLIMIT
block.coinbase    // COINBASE
// Also: BLOBBASEFEE, BLOBHASH
// Also: any access to the beneficiary account (BALANCE, EXTCODESIZE, etc.)

// Oracle contract SLOAD: also 20M cap (raised from 1M in Rex3)
```

**Key rules:**
- The cap is **retroactive** — it limits total compute gas, not just what's left
- Front-loading heavy computation before `block.timestamp` does NOT help
- If multiple volatile types are accessed, the most restrictive cap wins
- Oracle SLOAD triggers a separate 20M cap (Rex3+); pre-Rex3 was 1M

**Workaround:** Keep total compute gas under 20M in any transaction that touches block metadata, or use the high-precision timestamp oracle which has separate accounting.

## Fee History

Get historical gas prices:

```javascript
const history = await client.request({
  method: 'eth_feeHistory',
  params: ['0x10', 'latest', [25, 50, 75]]
});
// Returns base fees and priority fee percentiles
```

## Priority Fees in Practice

Priority fees only matter during congestion:

```javascript
// During normal operation: any priority fee works
// During congestion: higher priority = faster inclusion

const tx = {
  maxFeePerGas: 1000000n,      // 0.001 gwei
  maxPriorityFeePerGas: 0n,    // Usually sufficient
};
```

## SSTORE Costs (Storage Gas)

The most important gas difference from standard EVM. SSTORE 0→nonzero costs:

```
Compute gas: 22,100
Storage gas: 20,000 × (bucket_multiplier - 1)
```

| Bucket Multiplier | Storage Gas | Total Gas |
|-------------------|-------------|-----------|
| 1 | 0 | 22,100 |
| 2 | 20,000 | 42,100 |
| 10 | 180,000 | 202,100 |
| 100 | 1,980,000 | 2,002,100 |

**Key insight:** When multiplier = 1, storage gas is **zero**. This is why slot reuse patterns (RedBlackTreeLib, transient storage) are so important.

### Transient Storage (EIP-1153)

Use `TSTORE`/`TLOAD` for temporary data within a transaction — avoids storage gas entirely:

```solidity
// Transient storage - no storage gas cost
assembly {
    tstore(0, value)  // Store temporarily
    let v := tload(0) // Load back
}
// Cleared after transaction
```

## Gas Forwarding Ratio

MegaETH uses **98/100** gas forwarding (vs Ethereum's 63/64):

```solidity
// More gas available in nested calls
// Each call depth loses only 2%, not ~1.5%
// Budget accordingly for deep call chains
```

## Gas Refunds

Standard EVM refunds apply (SSTORE clear), but:
- Refund capped at 50% of gas used
- SELFDESTRUCT is disabled on MegaETH

## LOG Opcode Costs

After a DoS attack, LOG opcodes have quadratic cost above 4KB data:

```solidity
// Gas cost for log data:
// < 4KB: linear
// > 4KB: quadratic growth
```

This affects contracts emitting large events.

## Contract Deployment

| Resource | Limit |
|----------|-------|
| Contract code | 512 KB |
| Calldata | 128 KB |
| Deployment gas | Effectively unlimited |

For large contracts, may need VIP endpoint (higher gas limit on `eth_estimateGas`).
