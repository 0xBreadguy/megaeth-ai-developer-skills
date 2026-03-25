# x402 Payments on MegaETH

AI coding skill for integrating x402 payments on MegaETH
using the standard Permit2 flow. Covers seller (server)
and buyer (agent/client) implementation.

## What This Skill Is For

Use this skill when the user asks for:
- x402 payments on MegaETH
- Protecting APIs, tools, MCP servers, or agent actions
  behind a paywall
- Permit2-based token transfers on MegaETH
- Seller-side payment verification and settlement
- Buyer-side token approval, signing, and payment

## Verified MegaETH Contracts

### Mainnet (Chain ID: 4326)

| Contract | Address |
|----------|---------|
| x402ExactPermit2Proxy | `0x402085c248EeA27D92E8b30b2C58ed07f9E20001` |
| x402UptoPermit2Proxy | `0x402039b3d6E6BEC5A02c2C9fd937ac17A6940002` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| USDm (token) | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` |

### Testnet (Chain ID: 6343)

| Contract | Address |
|----------|---------|
| x402ExactPermit2Proxy | `0x402085c248EeA27D92E8b30b2C58ed07f9E20001` |
| x402UptoPermit2Proxy | `0x402039b3d6E6BEC5A02c2C9fd937ac17A6940002` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |

All contracts are deployed to the same canonical addresses
on both networks via deterministic CREATE2.

### RPC Endpoints

| Network | RPC |
|---------|-----|
| Mainnet | `https://mainnet.megaeth.com/rpc` |
| Testnet | `https://carrot.megaeth.com/rpc` |

## Mental Model

x402 uses HTTP 402 (Payment Required) as the payment
trigger. The flow:

```
1. Client requests protected resource
2. Server returns 402 + payment requirements
3. Client signs Permit2 authorization (off-chain)
4. Client retries request with payment header
5. Server settles on-chain via Permit2Proxy
6. Server returns the resource
```

The client never submits a transaction. The server
(or facilitator) pays gas and executes on-chain. On
MegaETH, gas is negligible (<$0.001/tx) and settlement
takes <50ms.

### Two Proxy Variants

- **Exact** (`0x4020...0001`): Transfers the full
  permitted amount. Simpler. Anyone can submit the
  settlement tx. Use this for standard payments.

- **Upto** (`0x4020...0002`): Transfers up to the
  permitted amount. The facilitator chooses the final
  amount at settlement. Use this for variable pricing
  or when the exact cost isn't known upfront.

## Seller / Server Side

### Payment Requirements

The seller defines what payment is required via a
`PaymentRequirements` object returned in the 402
response:

```typescript
const paymentRequirements = {
  scheme: "exact",
  network: "eip155:4326",
  maxAmountRequired: "1000000000000000000", // 1 USDm
  resource: "https://api.example.com/resource",
  description: "API access",
  mimeType: "application/json",
  payTo: "0xYOUR_WALLET_ADDRESS",
  maxTimeoutSeconds: 300,
  asset: "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
  extra: {
    name: "USDm",
    version: "1"
  }
};
```

Key fields:
- `scheme`: `"exact"` or `"upto"`
- `network`: `"eip155:4326"` (mainnet) or
  `"eip155:6343"` (testnet)
- `maxAmountRequired`: Amount in base units (18
  decimals for USDm — 1 USDm = 10^18)
- `asset`: The ERC-20 token address
- `payTo`: Your wallet address that receives payment

### Server Middleware (Express)

```typescript
import express from "express";

const app = express();

app.get("/api/resource", (req, res) => {
  const authHeader = req.headers["x-payment"];
  if (!authHeader) {
    return res.status(402).json({
      paymentRequirements: [{
        scheme: "exact",
        network: "eip155:4326",
        maxAmountRequired: "1000000000000000000",
        resource: `${req.protocol}://${req.get("host")}${req.originalUrl}`,
        description: "1 USDm for API access",
        payTo: process.env.WALLET_ADDRESS,
        maxTimeoutSeconds: 300,
        asset: "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
        extra: { name: "USDm", version: "1" }
      }]
    });
  }

  // Verify and settle payment
  // (use x402 facilitator or self-settle)
  // Then return the resource
  res.json({ data: "protected content" });
});
```

### Self-Settlement

To settle payments yourself without a facilitator,
call the Permit2Proxy directly:

```typescript
import { createWalletClient, http, parseAbi } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const EXACT_PROXY = "0x402085c248EeA27D92E8b30b2C58ed07f9E20001";

const settleAbi = parseAbi([
  "function settle((address token, uint256 amount) permit, address owner, (address to, uint256 validAfter) witness, bytes signature) external"
]);

const walletClient = createWalletClient({
  account: privateKeyToAccount(process.env.PRIVATE_KEY),
  chain: {
    id: 4326,
    name: "MegaETH",
    nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
    rpcUrls: { default: { http: ["https://mainnet.megaeth.com/rpc"] } }
  },
  transport: http()
});

async function settle(paymentPayload) {
  const { permit, owner, witness, signature } = paymentPayload;
  const hash = await walletClient.writeContract({
    address: EXACT_PROXY,
    abi: settleAbi,
    functionName: "settle",
    args: [permit, owner, witness, signature]
  });
  return hash;
}
```

## Buyer / Client Side

### One-Time Setup: Approve Permit2

Before any x402 payment, the buyer must approve the
Permit2 contract to spend their tokens. This is a
one-time operation per token:

```typescript
import { createWalletClient, http, parseAbi, maxUint256 } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const USDM = "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7";
const PERMIT2 = "0x000000000022D473030F116dDEE9F6B43aC78BA3";

const account = privateKeyToAccount(process.env.PRIVATE_KEY);

const walletClient = createWalletClient({
  account,
  chain: { id: 4326, rpcUrls: { default: { http: ["https://mainnet.megaeth.com/rpc"] } } },
  transport: http()
});

// Approve Permit2 (one-time per token)
await walletClient.writeContract({
  address: USDM,
  abi: parseAbi(["function approve(address,uint256) returns (bool)"]),
  functionName: "approve",
  args: [PERMIT2, maxUint256]
});
```

For production, consider using a bounded approval
instead of `maxUint256`.

### Signing a Payment

When the client receives a 402, it signs a Permit2
`PermitWitnessTransferFrom` message:

```typescript
import { SignatureTransfer } from "@uniswap/permit2-sdk";

const EXACT_PROXY = "0x402085c248EeA27D92E8b30b2C58ed07f9E20001";

function createPaymentSignature(
  paymentRequirements,
  signerAddress,
  nonce
) {
  const permit = {
    permitted: {
      token: paymentRequirements.asset,
      amount: paymentRequirements.maxAmountRequired
    },
    spender: EXACT_PROXY,
    nonce,
    deadline: Math.floor(Date.now() / 1000) + paymentRequirements.maxTimeoutSeconds
  };

  const witness = {
    to: paymentRequirements.payTo,
    validAfter: 0
  };

  const witnessType = {
    Witness: [
      { name: "to", type: "address" },
      { name: "validAfter", type: "uint256" }
    ]
  };

  // Build EIP-712 typed data for signing
  const { domain, types, values } = SignatureTransfer.getPermitData(
    permit,
    EXACT_PROXY,
    4326, // chainId
    witness,
    witnessType
  );

  return { permit, witness, domain, types, values };
}
```

### Full Buyer Flow

```typescript
async function payForResource(url) {
  // 1. Request the resource
  let res = await fetch(url);

  if (res.status !== 402) return res;

  // 2. Parse payment requirements
  const { paymentRequirements } = await res.json();
  const req = paymentRequirements[0];

  // 3. Sign Permit2 authorization
  const { permit, witness, domain, types, values } =
    createPaymentSignature(req, account.address, Date.now());

  const signature = await account.signTypedData({
    domain, types, primaryType: "PermitWitnessTransferFrom",
    message: values
  });

  // 4. Build payment payload
  const paymentPayload = {
    signature,
    permit: {
      permitted: permit.permitted,
      nonce: permit.nonce.toString(),
      deadline: permit.deadline.toString()
    },
    witness: {
      to: witness.to,
      validAfter: witness.validAfter.toString()
    },
    owner: account.address
  };

  // 5. Retry with payment
  res = await fetch(url, {
    headers: {
      "X-PAYMENT": JSON.stringify(paymentPayload),
      "X-PAYMENT-REQUIREMENTS": JSON.stringify(req)
    }
  });

  return res;
}
```

## Amount Handling

USDm on MegaETH uses **18 decimals**. This differs from
USDC (6 decimals) on most other chains.

| Amount | Base Units (18 decimals) |
|--------|--------------------------|
| 0.01 USDm | `10000000000000000` |
| 0.10 USDm | `100000000000000000` |
| 1.00 USDm | `1000000000000000000` |
| 10.00 USDm | `10000000000000000000` |

Do not use 6-decimal USDC math. Always use 18 decimals
for USDm amounts.

## Settlement Speed

MegaETH's 10ms block times enable settlement in under
50ms end-to-end. Use `realtime_sendRawTransaction` for
inline receipt confirmation:

```typescript
const receipt = await client.request({
  method: "realtime_sendRawTransaction",
  params: [signedTx]
});
// Receipt returned inline — no polling needed
```

## Legacy: Meridian / EIP-3009 Flow

Prior to the Permit2Proxy deployment, MegaETH x402
payments used Meridian's EIP-3009 forwarder as a
workaround. This flow is now **deprecated** in favor
of the standard Permit2 path described above.

If you encounter references to the Meridian facilitator
(`0x8E7769D440b3460b92159Dd9C6D17302b036e2d6`) or the
USDm forwarder (`0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4`)
in existing code, migrate to the Permit2 flow.

See `meridian.md` for the legacy implementation details
if you need to maintain backward compatibility.

## Common Mistakes

1. **Wrong decimals**: USDm is 18 decimals, not 6.
   `1 USDm = 1000000000000000000` (1e18), not 1000000.

2. **Approving the wrong contract**: Approve Permit2
   (`0x00000000...78BA3`), not the proxy or the seller.

3. **Using the standard x402 library without config**:
   The x402 TypeScript SDK defaults to USDC/6 decimals.
   Override the asset address and decimal handling for
   USDm.

4. **Forgetting the one-time Permit2 approval**: The
   buyer's first payment will fail if they haven't
   approved Permit2 to spend their USDm.

5. **Using EIP-3009 / Meridian for new integrations**:
   Use the Permit2 flow. The Meridian path is legacy.
