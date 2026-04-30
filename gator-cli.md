# Gator CLI

## Overview

`gator-cli` is MetaMask's command-line interface for the Delegation Framework. Use it when you want an agent or operator workflow to:

- inspect the delegate account address used for redemptions
- create or import delegation payloads
- store delegations for later redemption
- redeem scoped permissions from a terminal or automation runner
- inspect local delegation state during demos or dev workflows

**Repo:** https://github.com/MetaMask/gator-cli

## Why it matters on MegaETH

MegaETH is optimized for fast execution, but agent safety still comes from the permission layer. `gator-cli` is useful when you want to show or automate the flow where:

1. a user grants scoped authority off-chain or through a wallet UX
2. the delegation is stored for an agent or operator
3. the delegated action is redeemed on-chain with explicit bounds

That makes it a good companion to:
- `erc7710-delegations.md` for the permission model
- `smart-accounts.md` for wallet / signer setup
- MegaETH's fast transaction path when sending the eventual redemption

## Typical workflow

### 1. Check the agent/delegate identity

Use the CLI account command first so you know which address should receive the delegation.

```bash
gator-cli account
```

This is the address you typically plug into any app or wallet flow that asks for the **delegate**.

### 2. Create or receive a delegation

In practice, the delegation may come from:
- a wallet UX using ERC-7715 permission requests
- a custom app saving delegation JSON for later use
- another operator handing off a signed delegation chain

The important part is that the delegation is scoped and reviewable before redemption.

### 3. Store the delegation for the CLI

Many local demos and agent flows persist delegations under the CLI's local state directory so the terminal flow can inspect or redeem them later.

Common local paths used in MetaMask-adjacent demos:

```bash
~/.gator-cli/config.json
~/.gator-cli/delegations/default.json
```

Treat these as sensitive local capability artifacts — whoever controls them may be able to redeem the delegated authority within the granted bounds.

### 4. Redeem the delegation

After the delegation is available locally, use the CLI's redeem flow to execute the bounded action.

The exact command shape can evolve with the CLI version, so check the current README in the repo when scripting around it:

- https://github.com/MetaMask/gator-cli

### 5. Verify the on-chain result

On MegaETH, pair the CLI flow with fast receipt verification and standard explorer checks.

## Demo pattern that works well

A strong demo flow is:

1. show the delegate address from `gator-cli account`
2. grant permission from wallet/app UX to that delegate
3. save or sync the resulting delegation into the CLI state
4. switch to terminal and redeem it with `gator-cli`
5. show the bounded on-chain result

This makes the permission boundary tangible: the agent is not acting with a blank-check key — it is acting through explicit delegated authority.

## Safety notes

- Prefer narrow scopes: target, method, value, token amount, time window, call count
- Keep delegation files out of logs and screenshots when they contain redeemable data
- Revoke or rotate delegations after demos
- Treat CLI state as capability material, not harmless cache
- If a flow can be done either with raw contract calls or `gator-cli`, prefer the CLI when the goal is operator clarity and repeatable demos

## Related resources

- Delegation model: `erc7710-delegations.md`
- Smart account setup: `smart-accounts.md`
- MetaMask Delegation Framework contracts: https://github.com/MetaMask/delegation-framework
- Gator CLI repo: https://github.com/MetaMask/gator-cli
