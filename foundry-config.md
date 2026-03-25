# Foundry Configuration for MegaETH

> **Source**: Foundry patterns from [getfoundry.sh/introduction/prompting](https://getfoundry.sh/introduction/prompting/), adapted for MegaETH's multidimensional gas model and chain-specific requirements.

## Project Structure

```
project/
├── foundry.toml
├── .env.example
├── src/
│   ├── interfaces/
│   └── MyContract.sol
├── test/
│   ├── unit/
│   ├── fuzz/
│   ├── invariant/
│   │   └── handlers/
│   └── fork/
├── script/
│   └── Deploy.s.sol
└── lib/
```

## foundry.toml (MegaETH-ready)

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.20"
optimizer = true
optimizer_runs = 200
dynamic_test_linking = true  # 10x+ faster compilation

# Import remappings
remappings = [
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
    "solady/=lib/solady/"
]

# Gas reporting
gas_reports = ["*"]

# Fuzz testing
[fuzz]
runs = 1000
max_test_rejects = 65536

# Invariant testing
[invariant]
runs = 256
depth = 15
fail_on_revert = false
show_metrics = true

# ⚠️ MegaETH RPC endpoints
[rpc_endpoints]
megaeth = "https://mainnet.megaeth.com/rpc"
megaeth_testnet = "https://carrot.megaeth.com/rpc"

# ⚠️ MegaETH explorer verification
[etherscan]
megaeth = { key = "${MEGAETH_ETHERSCAN_KEY}", url = "https://mega.etherscan.io/api" }
```

## Naming Conventions

> Source: [Foundry Prompting Guide](https://getfoundry.sh/introduction/prompting/) — standard Foundry conventions.

| Element | Convention | Example |
|---------|-----------|---------|
| Contract files | PascalCase | `MyVault.sol` |
| Interface files | I-prefix | `IMyVault.sol` |
| Test files | `.t.sol` suffix | `MyVault.t.sol` |
| Script files | `.s.sol` suffix | `Deploy.s.sol` |
| Functions | mixedCase | `getUserBalance()` |
| Constants | SCREAMING_SNAKE | `MAX_SUPPLY` |
| Immutables | SCREAMING_SNAKE | `DEPLOYMENT_TIME` |
| Structs/Enums | PascalCase | `UserInfo`, `Status` |

### Test Naming

```
test_FunctionName_Condition       — unit tests
test_RevertWhen_Condition         — revert tests
testFuzz_FunctionName             — fuzz tests
invariant_PropertyName            — invariant tests
testFork_Scenario                 — fork tests
```

## Deployment Script (MegaETH)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Script, console} from "forge-std/Script.sol";
import {MyContract} from "src/MyContract.sol";

contract DeployScript is Script {
    function run() public {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);

        MyContract myContract = new MyContract();

        console.log("Deployed to:", address(myContract));

        vm.stopBroadcast();
    }
}
```

### Deploy Commands

```bash
# ⚠️ CRITICAL: Always use --skip-simulation on MegaETH
# Foundry's local simulation uses standard EVM gas costs,
# which are WRONG for MegaEVM (different intrinsic gas, storage gas dimension).

# Simulate locally (will show wrong gas — use only for logic check)
forge script script/Deploy.s.sol

# Deploy to testnet (skip simulation + set gas limit)
forge script script/Deploy.s.sol \
  --rpc-url megaeth_testnet \
  --broadcast \
  --skip-simulation \
  --gas-limit 5000000 \
  -vvvv \
  --interactives 1

# Deploy to mainnet + verify on mega.etherscan.io
forge script script/Deploy.s.sol \
  --rpc-url megaeth \
  --broadcast \
  --verify \
  --skip-simulation \
  --gas-limit 5000000 \
  --interactives 1

# Resume failed deployment
forge script script/Deploy.s.sol \
  --rpc-url megaeth \
  --resume
```

## Verification

```bash
# Verify existing contract on mega.etherscan.io
forge verify-contract <address> src/MyContract.sol:MyContract \
  --chain 4326 \
  --etherscan-api-key $MEGAETH_ETHERSCAN_KEY \
  --verifier-url https://mega.etherscan.io/api
```

## Linting

```bash
# Catch security and style issues before deployment
forge lint
forge lint --severity high --severity medium

# Key lints to watch for:
# - incorrect-shift: bit shift errors
# - divide-before-multiply: precision loss
```

> Source: `forge lint` added in Foundry v1.0, from [Foundry docs](https://getfoundry.sh/forge/linting).

## ⚠️ Critical: Never Use `via_ir`

`via_ir=true` can **silently break function return values** — functions may return 0 instead of the correct value with no compiler error and no test failure on simple cases. This has been confirmed multiple times on MegaETH contracts.

```toml
# ❌ NEVER — can silently corrupt return values
# via_ir = true

# ✅ Safe default
optimizer = true
optimizer_runs = 200
```

## Large Contract Deployment (500M Gas)

Contracts with 25KB+ bytecode need **500M gas limit** on MegaETH, not 5M. The `forge script` examples above use 5M which works for small contracts but will fail with "intrinsic gas too low" for real-sized contracts.

```bash
# For large contracts, use cast send directly (more reliable than forge script)
BYTECODE=$(forge inspect MyContract bytecode)
ARGS=$(cast abi-encode "constructor(address)" 0x1234...)

cast send --rpc-url https://mainnet.megaeth.com/rpc \
  --private-key $PK \
  --gas-limit 500000000 \
  --create "0x${BYTECODE#0x}${ARGS#0x}"
```

### Why `cast send --create` over `forge script`?

`forge script --broadcast` has known issues on MegaETH:
- `--rpc-url` / `-r` flags are sometimes ignored (use `--fork-url` or `ETH_RPC_URL` env var)
- Gas estimation produces values too low for large bytecode
- `--gas-limit` flag on forge script applies to simulation, not always to broadcast

`cast send --create` with explicit gas limit is the most reliable deployment method.

## Environment Setup

```bash
# .env file
MEGAETH_RPC_URL=https://mainnet.megaeth.com/rpc
MEGAETH_TESTNET_RPC_URL=https://carrot.megaeth.com/rpc
MEGAETH_ETHERSCAN_KEY=your_key_here
PRIVATE_KEY=0x...  # Or use --interactives 1

# Install dependencies
forge install OpenZeppelin/openzeppelin-contracts
forge install vectorized/solady  # For RedBlackTreeLib, SSTORE2, ERC6909
```
