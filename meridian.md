# Meridian x402 Payments on MegaETH

AI coding skill for integrating Meridian x402 payments on MegaETH. Covers the seller/server setup that owns the Meridian organization and API key, plus the buyer/agent flow that approves USDm, signs the MegaETH forwarder authorization, and sends a raw `paymentPayload` JSON object back to the seller.

## What This Skill Is For

Use this skill when the user asks for:
- Meridian or x402 payments on MegaETH
- Protecting APIs, tools, MCP servers, or agent actions with paid access
- Seller/server-side Meridian settlement with `/v1/settle`
- Buyer/agent-side USDm approval plus EIP-712 `TransferWithAuthorization`
- MegaETH-specific Meridian wiring: USDm, forwarder, facilitator, and payment payload shape

## Verified MegaETH Constants

| Item | Value | Use it for |
|------|-------|------------|
| x402 network | `megaeth` | `paymentRequirements.network` and `paymentPayload.network` |
| Chain ID | `4326` | EIP-712 domain |
| Meridian API base | `https://api.mrdn.finance` | Seller settlement calls |
| Meridian dashboard | `https://mrdn.finance/dev/api-keys` | Seller org + API key setup |
| Facilitator | `0x8E7769D440b3460b92159Dd9C6D17302b036e2d6` | `paymentRequirements.payTo` and `authorization.to` |
| USDm forwarder | `0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4` | `paymentRequirements.asset`, EIP-712 domain, spender for approval |
| USDm token | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` | Buyer balance and `approve()` target token |
| Forwarder domain name | `USDm Forwarder` | `paymentRequirements.extra.name` |
| Forwarder domain version | `1` | `paymentRequirements.extra.version` |

## Mental Model

On MegaETH, Meridian still uses the normal facilitator flow:

`buyer signs auth -> seller receives payment payload -> seller settles with Meridian -> facilitator uses forwarder -> forwarder moves USDm`

The MegaETH-specific twist is:
- The buyer approves **USDm** to the **forwarder**
- The buyer signs typed data against the **forwarder EIP-712 domain**
- The signed authorization still targets the **facilitator**
- The seller sends `paymentPayload` plus `paymentRequirements` to Meridian with its own API key

Do not use the standard x402 library for this MegaETH Meridian flow.
- The stock x402 tooling is designed around USDC assumptions
- Its amount handling assumes 8 decimals
- USDm on MegaETH uses 18 decimals
- Build the MegaETH `paymentRequirements` and `paymentPayload` manually

Do not collapse these addresses into one value:
- User-facing token: USDm
- Settlement asset in `paymentRequirements.asset`: forwarder
- Recipient target in `payTo` and `authorization.to`: facilitator

## Seller / Server Side

### 1. Create the Meridian organization and API key

Seller-side setup starts in Meridian:

1. Connect the seller wallet at `https://mrdn.finance`
2. Open `https://mrdn.finance/dev/api-keys`
3. Create an organization-scoped API key
4. Store the public `pk_...` key on the server

Use the public key in the `Authorization: Bearer <pk_...>` header. Meridian docs state that the public key is used for API authentication; the secret is shown once but is not the header value for `/v1/settle`.

The API key can also be created programmatically with a valid SIWE session cookie:

```bash
curl -X POST https://api.mrdn.finance/v1/api_keys \
  -H "Content-Type: application/json" \
  -H "Cookie: siwe-session=your_session" \
  -d '{
    "name": "My Application Key",
    "test_net": true
  }'
```

### 2. Resolve the forwarder EIP-712 domain

Query the deployed forwarder instead of guessing. The live contract currently returns:

- `name`: `USDm Forwarder`
- `version`: `1`
- `chainId`: `4326`
- `verifyingContract`: `0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4`

The forwarder implementation is open source and can be inspected here:
- `https://github.com/TheGreatAxios/eip3009-forwarder`

It is also verified on the block explorer:
- `https://megaeth.blockscout.com/address/0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4`

Example:

```typescript
const FORWARDER = "0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4" as const;

const domainTuple = await publicClient.readContract({
  address: FORWARDER,
  abi: parseAbi([
    "function eip712Domain() view returns (bytes1 fields,string name,string version,uint256 chainId,address verifyingContract,bytes32 salt,uint256[] extensions)",
  ]),
  functionName: "eip712Domain",
});

const forwarderDomain = {
  name: domainTuple[1],
  version: domainTuple[2],
  chainId: Number(domainTuple[3]),
  verifyingContract: domainTuple[4],
};
```

### 3. Build `paymentRequirements`

For the working MegaETH flow, use:

```typescript
const FACILITATOR = "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6" as const;

const paymentRequirements = {
  scheme: "exact",
  network: "megaeth",
  asset: FORWARDER,
  payTo: FACILITATOR,
  maxAmountRequired: amountWei.toString(),
  resource: "https://seller.example/api/tool",
  description: "Paid access to MegaETH agent action",
  mimeType: "application/json",
  maxTimeoutSeconds: 300,
  extra: {
    name: forwarderDomain.name,
    version: forwarderDomain.version,
  },
};
```

Critical rules:
- `asset` is the forwarder address, not the USDm token address
- `payTo` is the facilitator address
- `maxAmountRequired` uses USDm base units with 18 decimals
- `extra.name` and `extra.version` must match the forwarder domain

If you expose a standard HTTP `402` challenge, return `x402Version: 1` plus `accepts: [paymentRequirements]`.

### 4. Receive the buyer `paymentPayload`

This MegaETH integration passes the raw JSON object, not a base64 blob:

```json
{
  "x402Version": 1,
  "scheme": "exact",
  "network": "megaeth",
  "payload": {
    "signature": "0x...",
    "authorization": {
      "from": "0x...",
      "to": "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6",
      "value": "1000000000000000000",
      "validAfter": "0",
      "validBefore": "1741885200",
      "nonce": "0x..."
    }
  }
}
```

Keep bigint fields as decimal strings when you serialize to JSON.

### 5. Settle on the seller backend

Minimal working flow:

```typescript
const response = await fetch("https://api.mrdn.finance/v1/settle", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.MERIDIAN_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    paymentPayload,
    paymentRequirements,
  }),
});

const result = await response.json();

if (!response.ok || !result.success) {
  throw new Error(result.errorReason ?? "Meridian settlement failed");
}
```

Recommended production flow:
- Keep the API key on the server, even if older docs show `NEXT_PUBLIC_MERIDIAN_PK`
- Treat the seller server as the settlement authority

## Buyer / Agent Side

### 1. Approve USDm to the forwarder

The buyer approves the USDm token contract, with the forwarder as spender:

```typescript
import { erc20Abi, maxUint256 } from "viem";

const USDM = "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7" as const;
const FORWARDER = "0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4" as const;

await walletClient.writeContract({
  address: USDM,
  abi: erc20Abi,
  functionName: "approve",
  args: [FORWARDER, maxUint256],
});
```

This is a one-time prerequisite per wallet unless the allowance is later reduced.

### 2. Sign `TransferWithAuthorization`

Sign against the forwarder domain, but set `to` to the facilitator.

```typescript
import { randomBytes } from "node:crypto";

const FACILITATOR = "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6" as const;

const types = {
  TransferWithAuthorization: [
    { name: "from", type: "address" },
    { name: "to", type: "address" },
    { name: "value", type: "uint256" },
    { name: "validAfter", type: "uint256" },
    { name: "validBefore", type: "uint256" },
    { name: "nonce", type: "bytes32" },
  ],
} as const;

const authorization = {
  from: account.address,
  to: FACILITATOR,
  value: amountWei,
  validAfter: 0n,
  validBefore: BigInt(Math.floor(Date.now() / 1000) + 300),
  nonce: `0x${randomBytes(32).toString("hex")}` as `0x${string}`,
};

const signature = await walletClient.signTypedData({
  account,
  domain: forwarderDomain,
  types,
  primaryType: "TransferWithAuthorization",
  message: authorization,
});
```

Rules that matter:
- Domain comes from `forwarder.eip712Domain()`
- `authorization.to` is the facilitator, not the forwarder
- `value` is a USDm amount with 18 decimals
- Use a fresh random `bytes32` nonce each time
- Keep `validBefore` tight

### 3. Send `paymentPayload` JSON to the seller

After signing, send this object to the seller:

```typescript
const paymentPayload = {
  x402Version: 1,
  scheme: "exact",
  network: "megaeth",
  payload: {
    signature,
    authorization: {
      from: authorization.from,
      to: authorization.to,
      value: authorization.value.toString(),
      validAfter: authorization.validAfter.toString(),
      validBefore: authorization.validBefore.toString(),
      nonce: authorization.nonce,
    },
  },
};
```

This guide assumes a direct JSON handoff from buyer to seller. If you later wrap it in an `X-PAYMENT` header, the underlying authorization data is the same.

## Common Mistakes

- Approving the facilitator instead of the forwarder
- Using the USDm token address as `paymentRequirements.asset`
- Setting `payTo` or `authorization.to` to the forwarder
- Using the standard x402 library and inheriting USDC decimal assumptions
- Reusing old 6-decimal USDC assumptions instead of 18-decimal USDm units
- Sending BigInts directly through JSON without stringifying them
- Exposing the seller API key to untrusted clients in production

## Source Material

- API keys: `https://docs.mrdn.finance/essentials/api-keys`
- Smart contracts / MegaETH forwarder flow: `https://docs.mrdn.finance/payments/smart-contracts`
- Settle endpoint: `https://docs.mrdn.finance/api-reference/endpoint/settle-payment`
- Manual integration: `https://docs.mrdn.finance/api-reference/manual-integration`
- Forwarder source: `https://github.com/TheGreatAxios/eip3009-forwarder`
