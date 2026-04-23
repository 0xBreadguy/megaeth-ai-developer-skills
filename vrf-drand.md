# Verifiable Randomness (drand VRF)

MegaETH provides a predeployed drand verifier contract, `DrandOracleQuicknet`, that lets contracts verify public randomness beacons onchain without trusting an app backend.

Use this file when the task involves:
- lotteries, raffles, winner selection
- NFT trait reveals / randomized mint ordering
- game mechanics needing public randomness
- commit/reveal design on MegaETH
- integrating drand quicknet beacons into contracts or backend relayers

## What it is

MegaETH ships a verifier for **drand quicknet** randomness:
- public randomness beacon
- new round every **3 seconds**
- any relayer/user/keeper can fetch the beacon from `api.drand.sh`
- onchain verifier checks the BLS proof and returns canonical randomness

This is **not same-transaction randomness**. It is an async commit/reveal pattern.

## Contract addresses

| Network | Chain ID | Address |
|---------|----------|---------|
| Mainnet | 4326 | `0x7a53a6eFA81c426838fcf4824E6e207923969b36` |
| Testnet | 6343 | `0x4e1673dcAA38136b5032F27ef93423162aF977Cc` |

## Recommended API

Use:
- `verifyNormalized(uint64 round, bytes sig)`

Prefer it over raw verification helpers because it gives canonical, encoding-invariant randomness suitable for consumption.

## Mental model

1. App commits to a **future** drand round.
2. App locks every input that could affect the outcome.
3. After that round is published, anyone can fetch the signature from drand.
4. A reveal transaction submits the signature.
5. Contract verifies it and settles using the returned random value.

## Best practices (critical)

### 1. Always commit to a future round
Never use the current or already-published round at commit time.

If the round is already known, users can selectively proceed only when the outcome is favorable.

Rule:
- compute a round whose publish time is strictly in the future
- reject reveals for any round other than the committed round

### 2. Lock every outcome-relevant input at commit time
When randomness is committed, also freeze:
- participant set
- stake amounts
- pool contents
- chosen tier / options
- token IDs under consideration
- any app state that influences the result

If state remains mutable after commit, users can manipulate inputs after seeing the beacon.

### 3. Design for async reveal
This primitive gives **fresh randomness every ~3 seconds**, but it is still async.

Do not model it as instant entropy in the same tx.

Good fits:
- raffles
- delayed reveals
- async game turns
- randomized mint finalization

Less ideal fits:
- same-tx randomness UX
- highly synchronous game loops

### 4. Plan reveal liveness
The verifier is stateless; it does not pull beacons for you.

Your app still needs a reveal strategy:
- user-submitted reveal
- keeper / relayer
- bot-triggered finalization
- incentive for third-party settlement

If no one reveals, the app can stall unless you design for it.

### 5. Treat the verifier as verification only
`DrandOracleQuicknet` only answers:
- “is this a valid drand beacon for this round?”

It does **not** protect you from:
- selective execution
- mutable app inputs
- bad round selection
- liveness failures
- weak payout / settlement logic

Application safety is still on you.

### 6. Be precise in product claims
Do not describe this internally as “instant randomness” or “same-tx VRF”.

More accurate framing:
- public, verifiable randomness
- low-latency async randomness
- ~3 second cadence via drand quicknet

## Bull case for builders

Why this matters on MegaETH:
- no need to operate your own randomness backend
- no dependency on a proprietary oracle integration for basic use cases
- 3-second cadence is good enough for many consumer/game mechanics
- trust story is better than backend-generated randomness
- easy primitive for games, NFT reveals, raffles, social apps

## Bear case / caveats

- easy to misuse if builders do not lock state correctly
- not same-tx randomness
- still needs relayer/keeper UX for smooth production use
- inherits drand’s external security/liveness assumptions
- some teams will hear “VRF” and underestimate the app-layer design work

## Backend / relayer pattern

Typical offchain flow:
1. app stores committed round
2. worker polls drand API for that round
3. once available, worker sends reveal tx with signature
4. contract verifies and settles

Example fetch:

```bash
curl -fsSL "https://api.drand.sh/v2/chains/52db9ba70e0cc0f6eaf7803dd07447a1f5477735fd3f661792ba94600c84e971/rounds/$ROUND"
```

The quicknet chain hash above is fixed.

## Guidance for AI agents

When asked to build with MegaETH randomness:
- recommend commit/reveal, not same-tx randomness
- insist on locking all outcome-relevant inputs at commit time
- prefer `verifyNormalized`
- call out keeper/relayer requirements
- warn against using already-published rounds
- explain that 3 seconds is the drand cadence floor, not something MegaETH can trivially reduce

## Pointers

- Docs: `https://docs.megaeth.com/developer-docs/vrf`
- Example: drand VRF lottery in MegaETH docs/examples
- Verifier source: `https://github.com/Zodomo/DrandVerifier`
