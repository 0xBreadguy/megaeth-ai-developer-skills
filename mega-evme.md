# mega-evme Debugging Skill

Use this guide when the task is specifically about **local transaction replay, MegaEVM debugging, gas profiling, or spec-aware execution analysis**.

This is more specific than general testing guidance. Reach for `mega-evme` when you need to answer questions like:
- why did this MegaETH transaction revert?
- what exact MegaEVM rule/spec caused this behavior?
- how do I replay a tx locally and inspect traces?
- how do I debug gas detention, storage gas, or system-contract interactions?
- how do I capture RPC/state locally for reproducible replay?

## What mega-evme is best for

- replaying historical MegaETH transactions
- inspecting execution traces and call structure
- understanding MegaEVM-specific gas / resource accounting
- debugging system-contract interactions
- reproducing failures locally with cached RPC/state inputs
- comparing behavior across specs / hardfork assumptions

## Mental model

`mega-evme` is not just a trace dumper.
It is the **reference local execution/debugging harness** for MegaEVM behavior.

Use it when generic EVM tooling is not enough because the issue depends on:
- MegaEVM resource accounting
- volatile data / oracle detention
- system contracts
- spec-specific behavior
- MegaETH RPC/state assumptions

## Installation

```bash
git clone https://github.com/megaeth-labs/mega-evm
cd mega-evm/bin/mega-evme
cargo build --release
```

## Core commands

### Replay a transaction

```bash
mega-evme replay <txhash> --rpc https://mainnet.megaeth.com/rpc
```

### Replay with trace output

```bash
mega-evme replay <txhash> --trace --trace.output trace.json --rpc <endpoint>
```

### Replay with call tracer

```bash
mega-evme replay <txhash> --trace --tracer call --trace.output calls.json --rpc <endpoint>
```

## Newer workflows worth using

Recent mega-evme updates added better support for local reproducibility via cached RPC/state inputs.
When debugging tricky issues, prefer workflows that preserve inputs instead of relying on a fresh remote replay every time.

### RPC capture / replay
Use RPC capture/replay flows when you want deterministic local debugging or to share a reproducible artifact.

Look at the current mega-evme docs for:
- `--rpc.capture-file`
- `--rpc.replay-file`

These are useful when:
- the live RPC is flaky
- state may change between debugging attempts
- you want to hand a reproducible case to another engineer or agent

### Offline / cached replay mindset
If you hit a bug that is hard to reproduce, preserve:
- tx hash
- block number
- trace output
- rpc capture file / replay file if available
- exact spec/hardfork assumptions

Do not rely only on “it reverted once on mainnet/testnet.”

## Spec-aware debugging rules

### 1. Always ask which spec behavior matters
When replaying or diagnosing a tx, check whether the behavior depends on:
- EQUIVALENCE / MINI_REX / REX / REX1 / REX2 / REX3 / REX4 / REX5

This matters especially for:
- Oracle behavior
- KeylessDeploy
- SELFDESTRUCT
- gas detention
- per-frame resource accounting
- system-contract semantics

### 2. Do not assume unstable REX5 behavior is live everywhere
REX5 introduces:
- `SequencerRegistry`
- dynamic system address resolution
- Oracle v2.0.0 authority changes
- caller-account update deduplication
- stricter keyless trailing-bytes rejection

Treat those as **spec-conditional**, not universal.

### 3. Distinguish execution bugs from accounting differences
Some changes only affect:
- data-size accounting
- KV update accounting
- gas/resource usage reporting

They may not change final state transitions.

## What to check first when a tx fails

### Contract-level / generic
- revert reason
- bad calldata / bad approvals
- insufficient balance / allowance
- owner/minter/role checks

### MegaEVM-specific
- storage gas blow-up from new slot writes
- volatile data access / detention behavior
- per-frame resource budget exhaustion
- state-growth / KV update / data-size limits
- system-contract access assumptions
- keyless deploy encoding rules

## Resource-accounting issues

Use mega-evme when the user says things like:
- "works on standard EVM sim, fails on MegaETH"
- "gas estimate looks wrong"
- "it only fails when block.timestamp is touched"
- "nested calls revert unexpectedly"

Common MegaEVM-specific suspects:
- expensive first-time storage writes
- volatile data compute cap
- frame-level budget depletion
- spec-version differences in accounting

## Oracle / system-contract debugging

Be careful with statements about Oracle/system behavior.
There are meaningful differences between pre-REX5 and REX5-era semantics.

For debugging, verify:
- whether Oracle storage access actually occurred
- whether gas detention was triggered by oracle reads vs block env access
- whether system-address assumptions are hardcoded or dynamic for the target spec

## When to prefer mega-evme over Foundry alone

Prefer mega-evme when:
- the issue depends on live chain context
- standard local simulation disagrees with MegaETH behavior
- you need spec-aware replay
- you need MegaEVM-specific accounting details

Foundry is still great for:
- writing tests
- isolated contract debugging
- rapid iteration

But mega-evme is the right tool for **"what exactly happened on MegaETH?"**

## Relationship to Foundry

A good debugging workflow is often:
1. identify failing tx / contract path
2. replay in mega-evme
3. extract the specific failure mode
4. write a focused Foundry regression test
5. use mega-evme again if needed to compare against chain behavior

## Practical artifacts to save

When debugging a serious issue, save:
- tx hash
- block number
- contract addresses involved
- trace output
- replay/capture files
- notes on the suspected spec-dependent behavior

This makes the problem transferable between agents/humans.

## Anti-patterns

Do not:
- treat mega-evme as just another generic trace tool
- ignore spec version when analyzing system-contract behavior
- assume REX5 behavior is active by default
- rely only on live RPC replay if reproducibility matters
- conflate synchronous receipt return with full execution/debug determinism

## Pointer docs

For up-to-date command flags and workflows, consult:
- local repo: `knowledge/github/mega-evm/docs/mega-evme/**`
- upstream repo: `https://github.com/megaeth-labs/mega-evm/tree/main/docs/mega-evme`
