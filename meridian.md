# Meridian x402 Payments on MegaETH

AI coding skill for integrating Meridian x402 payments on MegaETH. Covers the seller/server setup that owns the Meridian organization and API key, plus both supported Meridian flows on MegaETH:
- Preferred: Permit2 via the Meridian facilitator V4_2 for any ERC-20 on MegaETH
- Legacy: EIP-3009 via the USDm forwarder for USDm compatibility

## What This Skill Is For

Use this skill when the user needs Meridian-specific MegaETH integration details:
- Seller/server-side Meridian settlement with `/v1/settle`
- Meridian organization and API key setup
- The current Permit2 flow for any ERC-20 bound to the Meridian facilitator
- The legacy EIP-3009 forwarder flow for backward compatibility
- Meridian-specific `paymentRequirements` and `paymentPayload` conventions that differ from the generic `x402-payments.md` guide

Use `x402-payments.md` for general-purpose MegaETH x402 integrations that do not go through Meridian.

## Verified MegaETH Constants

| Item | Value | Use it for |
|------|-------|------------|
| x402 network | `megaeth` | `paymentRequirements.network` and `paymentPayload.network` |
| Chain ID | `4326` | Typed-data domain / signer config |
| Meridian API base | `https://api.mrdn.finance` | Seller settlement calls |
| Meridian dashboard | `https://mrdn.finance/dev/api-keys` | Seller org + API key setup |
| Facilitator | `0x8E7769D440b3460b92159Dd9C6D17302b036e2d6` | `paymentRequirements.payTo`, Permit2 witness `to`, legacy `authorization.to` |
| x402ExactPermit2Proxy | `0x402085c248EeA27D92E8b30b2C58ed07f9E20001` | Spender for the current Meridian Permit2 flow |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` | One-time approval target for the Permit2 flow |
| USDm forwarder | `0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4` | Legacy EIP-3009 asset, domain, and spender |
| USDm token | `0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7` | Example Permit2 asset and legacy approval target |
| Forwarder domain name | `USDm Forwarder` | Legacy EIP-712 domain |
| Forwarder domain version | `1` | Legacy EIP-712 domain |

## Mental Model

Meridian on MegaETH is still facilitator-based:

`buyer signs auth -> seller receives payment payload -> seller settles with Meridian -> facilitator pulls funds -> facilitator distributes proceeds and fees`

The Meridian-specific differences from the generic x402 guide are:
- The seller still settles through Meridian's `/v1/settle` API
- `paymentRequirements.payTo` stays the Meridian facilitator, not the seller wallet
- The Permit2 flow can use any MegaETH ERC-20; amounts use that token's native base units
- Buyer auth is passed around as raw JSON `paymentPayload`, not as a transaction the buyer submits

There are two supported Meridian flows:
- Preferred: Permit2 via facilitator V4_2. `asset` is the chosen ERC-20 token address, `spender` is the exact Permit2 proxy, and `witness.to` is the facilitator.
- Legacy: EIP-3009 via the USDm forwarder. This path remains USDm-only. `asset` is the forwarder address, the buyer approves the forwarder, and `authorization.to` is the facilitator.

Do not blindly reuse stock x402 examples for Meridian:
- Generic x402 examples usually make `payTo` the seller wallet; Meridian does not
- Generic buyer examples often vary on the witness shape; for Meridian V4_2, keep `witness.to` bound to the facilitator
- Older examples often hardcode USDC or USDm decimals; in the Permit2 flow you must use the chosen token's actual decimals

## Shared Seller / Server Setup

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

### 2. Settle on the seller backend

Both Meridian flows settle through the same seller-side API call:

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

## Current Flow: Permit2 via Meridian V4_2

This is the current Meridian path for same-chain MegaETH payments. It supports any ERC-20 token that can be transferred normally on MegaETH.

Token support rules:
- Any ERC-20 works if the buyer can grant Permit2 approval for that token
- ERC-2612 tokens can optionally attach `permit2612` for a first-payment flow without a pre-existing Permit2 approval
- The legacy EIP-3009 section below remains specific to USDm

### 1. Build `paymentRequirements`

For the current Meridian Permit2 flow, use the chosen ERC-20 token as `asset` but keep `payTo` pointed at the facilitator:

```typescript
const FACILITATOR = "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6" as const;
const TOKEN = "0xYourErc20TokenAddress" as const;

const paymentRequirements = {
  scheme: "exact",
  network: "megaeth",
  asset: TOKEN,
  payTo: FACILITATOR,
  maxAmountRequired: amountInTokenBaseUnits.toString(),
  resource: "https://seller.example/api/tool",
  description: "Paid access to MegaETH agent action",
  mimeType: "application/json",
  maxTimeoutSeconds: 300,
  extra: {
    name: "Example ERC20",
    version: "1",
  },
};
```

Critical rules:
- `asset` is the chosen ERC-20 token address for the Permit2 flow
- `payTo` is still the Meridian facilitator, not the seller wallet
- `maxAmountRequired` uses the chosen token's native base units
- `extra.name` and `extra.version` should describe the token you are asking the buyer to pay with
- The buyer signs with `spender = x402ExactPermit2Proxy`, not Permit2 and not the facilitator
- The Permit2 witness must bind `to` to the Meridian facilitator

USDm is one valid Permit2 asset, but not the only one.

If you expose a standard HTTP `402` challenge, return `x402Version: 1` plus `accepts: [paymentRequirements]`.

### 2. Receive the buyer `paymentPayload`

This MegaETH integration passes the raw JSON object, not a base64 blob:

```json
{
  "x402Version": 1,
  "scheme": "exact",
  "network": "megaeth",
  "payload": {
    "signature": "0x...",
    "owner": "0x...",
    "permit": {
      "permitted": {
        "token": "0xFAfDdbb3FC7688494971a79cc65DCa3EF82079E7",
        "amount": "1000000000000000000"
      },
      "nonce": "123",
      "deadline": "1741885200"
    },
    "witness": {
      "to": "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6",
      "validAfter": "0"
    }
  }
}
```

Keep bigint fields as decimal strings when you serialize to JSON.
In the current flow, only the token address and base-unit amount change from asset to asset.

If the token supports ERC-2612 and you want a first-payment flow without a prior Permit2 approval, include an additional `permit2612` object with `value`, `deadline`, `v`, `r`, and `s`. Meridian V4_2 can forward that bundle through `transferWithPermit2`, but only use this shortcut for tokens you have verified expose `permit()`.

### 3. Buyer / Agent flow

#### One-time approval to Permit2

The buyer approves Permit2, not the proxy and not the facilitator:

```typescript
import { erc20Abi, maxUint256 } from "viem";

const TOKEN = paymentRequirements.asset as `0x${string}`;
const PERMIT2 = "0x000000000022D473030F116dDEE9F6B43aC78BA3" as const;

await walletClient.writeContract({
  address: TOKEN,
  abi: erc20Abi,
  functionName: "approve",
  args: [PERMIT2, maxUint256],
});
```

This is the default buyer path. If you include a valid `permit2612` as described above, Meridian V4_2 can also settle without a standing Permit2 approval.

#### Sign the Permit2 authorization

Sign a Permit2 witness transfer that is bound to the Meridian facilitator:

```typescript
import { randomBytes } from "node:crypto";
import { SignatureTransfer } from "@uniswap/permit2-sdk";

const FACILITATOR = "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6" as const;
const EXACT_PROXY = "0x402085c248EeA27D92E8b30b2C58ed07f9E20001" as const;
const PERMIT2 = "0x000000000022D473030F116dDEE9F6B43aC78BA3" as const;
const TOKEN = paymentRequirements.asset as `0x${string}`;

const nonce = BigInt(`0x${randomBytes(32).toString("hex")}`);

const permit = {
  permitted: {
    token: TOKEN,
    amount: BigInt(paymentRequirements.maxAmountRequired),
  },
  spender: EXACT_PROXY,
  nonce,
  deadline: BigInt(Math.floor(Date.now() / 1000) + paymentRequirements.maxTimeoutSeconds),
};

const witness = {
  to: FACILITATOR,
  validAfter: 0n,
};

const witnessType = {
  Witness: [
    { name: "to", type: "address" },
    { name: "validAfter", type: "uint256" },
  ],
};

const { domain, types, values } = SignatureTransfer.getPermitData(
  permit,
  PERMIT2,
  4326,
  witness,
  witnessType,
);

const signature = await walletClient.signTypedData({
  account,
  domain,
  types,
  primaryType: "PermitWitnessTransferFrom",
  message: values,
});
```

Rules that matter:
- `permit.permitted.token` is the chosen ERC-20 token address
- `permit.permitted.amount` uses that token's native base units
- `permit.spender` is the exact Permit2 proxy
- The EIP-712 domain verifier is Permit2, not the exact proxy
- `witness.to` is the Meridian facilitator
- Use a fresh `uint256` nonce each time
- Keep the Permit2 deadline tight

If you are signing manually instead of using `@uniswap/permit2-sdk`, keep the same split:
- `permit.spender = x402ExactPermit2Proxy`
- `domain.verifyingContract = Permit2`
- `primaryType = PermitWitnessTransferFrom`

#### Send `paymentPayload` JSON to the seller

After signing, send this object to the seller:

```typescript
const paymentPayload = {
  x402Version: 1,
  scheme: "exact",
  network: "megaeth",
  payload: {
    signature,
    owner: account.address,
    permit: {
      permitted: {
        token: permit.permitted.token,
        amount: permit.permitted.amount.toString(),
      },
      nonce: permit.nonce.toString(),
      deadline: permit.deadline.toString(),
    },
    witness: {
      to: witness.to,
      validAfter: witness.validAfter.toString(),
    },
  },
};
```

If you also built an ERC-2612 permit, attach it as `payload.permit2612`. If you later wrap the payload in an `X-PAYMENT` header, the underlying fields stay the same.

## Legacy Flow: EIP-3009 / USDm Forwarder

Keep this flow only for backward compatibility with existing Meridian integrations. The seller-side API key and `/v1/settle` call stay the same; only the auth format changes.

### 1. Resolve the forwarder EIP-712 domain

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

### 2. Build legacy `paymentRequirements`

For the legacy forwarder flow, the asset is the forwarder address:

```typescript
const FACILITATOR = "0x8E7769D440b3460b92159Dd9C6D17302b036e2d6" as const;
const FORWARDER = "0x2c2d8EF0664432BA243deF0b8f60aF7aB43a60B4" as const;

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

### 3. Buyer approves the forwarder and signs `TransferWithAuthorization`

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

Sign against the forwarder domain, but set `to` to the facilitator:

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

### 4. Send the legacy `paymentPayload`

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

## Common Mistakes

- Using the seller wallet as `payTo` instead of the Meridian facilitator
- Assuming Meridian's Permit2 flow is USDm-only
- Using the forwarder address as `asset` in the Permit2 flow
- Using the USDm token address as `asset` in the legacy forwarder flow
- Approving the exact proxy or facilitator instead of Permit2 for the current flow
- Setting `witness.to` to anything other than the facilitator in the current flow
- Approving the facilitator instead of the forwarder in the legacy flow
- Reusing hardcoded 18-decimal USDm assumptions instead of the chosen token's actual decimals in the Permit2 flow
- Sending BigInts through JSON without stringifying them
- Exposing the seller API key to untrusted clients in production

## Source Material

- API keys: `https://docs.mrdn.finance/essentials/api-keys`
- Smart contracts / MegaETH forwarder flow: `https://docs.mrdn.finance/payments/smart-contracts`
- Settle endpoint: `https://docs.mrdn.finance/api-reference/endpoint/settle-payment`
- Manual integration: `https://docs.mrdn.finance/api-reference/manual-integration`
- Forwarder source: `https://github.com/TheGreatAxios/eip3009-forwarder`
