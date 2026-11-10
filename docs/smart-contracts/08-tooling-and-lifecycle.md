# Tooling and lifecycle

> Previous: [Non-EVM smart-contract platforms](07-non-evm-platforms.md) · Up to: [Deep-dive index](README.md)

Writing smart contracts in production involves a specific toolchain and workflow: local development against a forked mainnet, unit and integration tests, formal verification for high-value contracts, a careful deployment process that might include multisig transactions and timelocks, source verification on block explorers, and ongoing monitoring. This doc covers the day-to-day: what the tools are, how they fit together, and the lifecycle from first write to long-term operation.

Everything below is EVM-focused; the non-EVM equivalents from the previous doc have analogous (but distinct) toolchains.

## The two dominant frameworks

### Foundry

The current default for serious new projects. Written in Rust by the Paradigm team, deliberately fast.

- **`forge`** — build, test, format.
- **`cast`** — command-line tool for chain interactions (call a function, send a transaction, decode calldata).
- **`anvil`** — a local Ethereum node, replacement for Ganache. Supports forking mainnet.
- **`chisel`** — Solidity REPL.

Test files are written **in Solidity itself**, which keeps the tests next to the code they test with no language context-switch:

```solidity
// test/Counter.t.sol
pragma solidity ^0.8.24;
import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter counter;

    function setUp() public {
        counter = new Counter();
    }

    function test_Increment() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    function test_SetNumber(uint256 x) public {        // fuzz test
        counter.setNumber(x);
        assertEq(counter.number(), x);
    }
}
```

`forge test` runs these; fuzz parameters get random inputs each run. **Invariant testing** (`function invariant_Foo()`) runs sequences of random calls to verify a property always holds — useful for finding subtle bugs that single-function tests miss.

### Hardhat

The older dominant framework. JavaScript / TypeScript based. Tests written in JS using Chai and ethers.js. Still widely used, especially in projects integrating with web frontends or complex deployment pipelines.

```javascript
// test/Counter.js
const { expect } = require("chai");

describe("Counter", function () {
  it("increments correctly", async function () {
    const Counter = await ethers.getContractFactory("Counter");
    const counter = await Counter.deploy();
    await counter.increment();
    expect(await counter.number()).to.equal(1);
  });
});
```

Hardhat's strength: rich JavaScript ecosystem, mature plugin system, good for projects where the deployment and interaction scripts need to be in JS/TS anyway.

**Which to pick for a new project?** Foundry if you want speed and Solidity-native tests. Hardhat if you need extensive scripting, want to share logic with a frontend, or are working in a team that already knows it. Many projects use both.

## The development loop

### Local chain for iteration

A local Ethereum node runs instantly, has no fees, and can be reset to any state. For development:

```
anvil --fork-url https://mainnet.infura.io/v3/<KEY> --fork-block-number 19000000
```

This spins up a local node that *forks mainnet* at block 19,000,000. You now have a local chain where every contract on mainnet exists at the state it had at that block — USDC, Uniswap, Aave, everything. You can send transactions against them as if you were on mainnet, but for free and in isolation.

Forked testing is the gold standard for integration testing. You develop against the actual contracts your protocol will interact with, in their actual production state.

### Testing patterns

- **Unit tests** — isolate a contract, test its functions directly.
- **Fuzz tests** — give the test random inputs, verify properties hold.
- **Invariant tests** — define a property, let the framework explore state space trying to break it.
- **Integration tests on a fork** — test your contract interacting with real mainnet contracts.
- **Mainnet shadow tests** — deploy your contract to mainnet, replay real transactions through it (for migrations or forks of existing protocols).

Code coverage via `forge coverage` tells you which lines were hit. 100% line coverage is not enough — you need to think about what inputs you *didn't* test, which is why fuzzing and invariant testing matter.

## Static analysis

Several tools scan Solidity for known bug patterns:

- **Slither** (by Trail of Bits) — the standard. Detects reentrancy, uninitialized state variables, suspicious patterns, outdated Solidity version, incorrect access modifiers, and more. Fast and easy to run.
- **Mythril** — symbolic execution to find vulnerabilities by exploring code paths.
- **Semgrep** — pattern-based rules, including crypto-specific rule sets.

A slither run is cheap enough that it should be in every project's CI. The false-positive rate is modest; serious findings are rare but valuable.

## Formal verification

For the highest-stakes contracts (stablecoins, bridges, foundational infrastructure), **formal verification** proves specific properties mathematically rather than testing them.

- **Certora Prover** — commercial tool, writes specifications in CVL, machine-checks contracts against them.
- **Halmos** — open-source symbolic execution, works with Foundry.
- **Kontrol** — K Framework-based, used for the deepest proofs.

Writing specifications is *itself* valuable. The exercise of formalizing "what should this contract never do" catches design errors before any code. Several famous bugs (including some in audited, reviewed protocols) were caught by formal verification *after* the contracts had been audited.

Formal verification is not cheap — it requires specialists and takes time. For a $10M TVL protocol it is overkill; for a $1B TVL protocol it is often economic.

## Audits

Before serious deployment, serious projects get an **audit**: a third party reviews the code against known bug classes and the project's own specifications.

- **Reputable firms**: Trail of Bits, OpenZeppelin, Consensys Diligence, Spearbit, Zellic, Code4rena (contest-based), Sherlock (contest-based).
- **Timing**: weeks to months, depending on code volume.
- **Cost**: $50k to $500k+ for a serious audit.

An audit does not guarantee safety. Many exploited protocols had audits. An audit does mean *this specific set of bugs, known to this reviewer, is not present*. New bug classes can be discovered later; the audited version may not match the deployed version; audited code can interact with unaudited code.

Multiple audits from different firms are better than one audit. **Immunefi** bug-bounty programs (paying attackers more to report bugs than exploit them) complement audits well.

## Deployment

### Scripts

Foundry uses Solidity deployment scripts:

```solidity
// script/Deploy.s.sol
import "forge-std/Script.sol";
import "../src/MyContract.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast();
        MyContract c = new MyContract();
        c.setParameter(42);
        vm.stopBroadcast();
    }
}
```

Run with `forge script script/Deploy.s.sol --rpc-url $RPC --private-key $KEY --broadcast`. The script records every transaction it broadcasts, which is useful for reproducibility.

### Deployment addresses

A contract's address is determined by `keccak256(sender, nonce)` for normal `CREATE`, or `keccak256(0xff, sender, salt, keccak256(init_code))` for `CREATE2`.

**CREATE2** lets you compute the deployment address in advance, which is critical for protocols that need deterministic addresses across chains (bridges, multichain tokens). Many projects use CREATE2 for everything via a "deterministic deployer" — a well-known contract at a fixed address that proxies `CREATE2` calls.

### Source verification

After deployment, upload your source code to Etherscan (and chain-specific explorers). The explorer recompiles your source and checks that the resulting bytecode matches what was deployed. If it matches, the contract is **verified** — users can read the source on Etherscan.

**Unverified contracts should be treated as hostile.** An unverified contract reveals nothing about what it does; interacting with one is always a risk. Any legitimate project verifies within hours of deployment.

### Multisig and timelocks

Admin actions (parameter changes, upgrades, pausing) should route through a **multisig** (typically Safe, previously Gnosis Safe) requiring M of N signatures. 3 of 5 or 4 of 7 are common.

For high-stakes actions (contract upgrades, treasury movements), route through a **timelock**: the multisig initiates the action; the timelock queues it for N days; after the delay, anyone can execute it. The delay gives users time to react — if an upgrade is malicious, users can withdraw before it takes effect.

OpenZeppelin ships `TimelockController` for this; it is the standard.

## Upgrade patterns

Smart contracts cannot have their bytecode modified in place. To make a contract "upgradeable," you deploy a **proxy** contract that `DELEGATECALL`s into an **implementation** contract whose address can be changed.

### Transparent proxy pattern

- Proxy contract: stores the implementation address and forwards all calls via delegatecall.
- Implementation contract: holds the actual logic. Has no state; the state lives in the proxy's storage.
- Admin address: can upgrade the implementation via a specific function.

The proxy must carefully manage storage layout — because the implementation delegatecalls, it operates on the proxy's storage, so storage slots must line up across versions. OpenZeppelin's `TransparentUpgradeableProxy` handles the common cases.

### UUPS (Universal Upgradeable Proxy Standard)

Slight variation: the upgrade logic lives in the *implementation* rather than the *proxy*. This makes the proxy smaller (cheaper to deploy and run) but puts the responsibility of not breaking upgradeability on the implementation author. OpenZeppelin ships this too.

### Beacon pattern

For many contracts that should all upgrade together (e.g., many instances of the same vault contract), each proxy points to a **beacon** contract that holds the current implementation address. Upgrading the beacon upgrades every proxy in one transaction.

### The diamond pattern (ERC-2535)

A proxy that can route calls to multiple implementations ("facets") based on the function selector. Enables large contracts to split across multiple implementation contracts and selectively upgrade pieces. Complex but useful for large protocols.

### The honest tradeoffs

Upgradability is useful but weakens the "code is law" property. Every upgradeable contract has an admin who could, in principle, replace the implementation with something malicious. Users are trusting the admin. This is often an acceptable tradeoff for the ability to fix bugs, but it should be disclosed and ideally mitigated (timelock, multisig, eventual renunciation of upgrade rights).

Some projects deploy immutable contracts from the start, accepting that bugs cannot be fixed. Uniswap v2 and v3 are examples — the core contracts cannot be upgraded. This is a strong trust-minimization stance but means the only way to "fix" a bug is to deploy a new version and migrate.

## Monitoring and incident response

After deployment, running a protocol requires ongoing monitoring:

- **Watchtower services** (Tenderly, OpenZeppelin Defender, Forta) — monitor transactions, alert on anomalies.
- **Pause switches** — a privileged role that can pause the contract in an emergency. Must be used sparingly; routine pausing undermines user trust.
- **Oncall rotations** — someone monitoring 24/7, with authority to pause and the keys to do so.
- **War-game exercises** — practice responding to a hypothetical exploit. Several protocols have regular drills.

The first hour after an exploit is critical. Protocols that detect, pause, and communicate within that window often recover; protocols that lose it typically lose everything.

### Incident post-mortems

A culture that takes smart-contract security seriously publishes **post-mortems** after incidents: what happened, what was attempted, what was recovered, what the fix will be. Reading these is one of the best ways to improve your own practice — every post-mortem is a case study in a real failure mode.

**rekt.news** maintains an archive of incidents and (sometimes) post-mortems. The DeFi Hacks GitHub repository (by @SunWeb3Sec and others) collects proof-of-concept exploits for educational study. Reading a dozen post-mortems and trying to predict the next common exploit class is a surprisingly effective exercise.

## The daily shape of working

Pulling it together, here is what a normal day on a serious smart-contract project looks like:

1. Pull latest. `forge build` or `yarn hardhat compile` to make sure everything still compiles.
2. Write new code with tests in the same commit.
3. Run `forge test` and `forge coverage` locally.
4. Run `slither` (or equivalent) as a quick sanity check.
5. Open a PR. CI runs the full test suite, coverage check, slither, fuzz tests, maybe a mainnet-fork integration test.
6. Review from at least one other engineer. For sensitive changes, more.
7. If the PR touches deployed contracts (via upgrade or new deployment), it also needs an audit sign-off or an acknowledgment that the change is scoped small enough to ship without one.
8. Once merged, a deployment script runs in a staging environment; after validation, it's queued to the multisig; after approval and timelock, it ships to mainnet.

Projects that follow this discipline have proportionally fewer catastrophic bugs. Projects that short-circuit it — rushing to ship without audits, admin-keying things without multisig, deploying without verification — show up in the exploit archives.

## What to carry forward

- **Foundry** and **Hardhat** are the two dominant frameworks. Foundry is faster and Solidity-native; Hardhat has a mature JS/TS ecosystem.
- **Fork a mainnet state** for integration tests against real production contracts.
- **Slither** should be in every CI pipeline. **Formal verification** matters for high-value contracts.
- **Audits** reduce risk but do not eliminate it. Multiple audits + bug bounty programs are the standard for serious protocols.
- **Verify source** on block explorers. Unverified contracts are hostile.
- **Multisig + timelock** for admin actions. The pattern exists to mitigate admin-key compromise and malicious upgrades.
- **Upgrade patterns** (Transparent, UUPS, Beacon, Diamond) exist because contracts are immutable; each has tradeoffs. Immutable-from-deployment is a strong stance with its own costs.
- **Monitoring and incident response** matter. The first hour after an exploit is often the whole game.
- Reading **post-mortems** is one of the most effective ways to improve your practice.

## Where to go from here

You now have the full picture: the problem blockchain solves (main sequence), what smart contracts are and why they exist (main doc 07), the EVM and Solidity in depth, the token standards and primitives, the security patterns and famous exploits, the non-EVM alternatives, and the daily workflow of production smart-contract development.

The next step is practice. Deploy a toy contract on a testnet. Read a real deployed protocol's source on Etherscan. Write a proof-of-concept exploit for a historical bug. Build a small AMM, lending market, or governance contract from scratch and compare it to the audited reference implementations. There is no substitute for touching this material directly.

Return to the [deep-dive index](README.md) for the full map, or to the [main repo README](../../README.md) for the top-level reading plan.

> Previous: [Non-EVM smart-contract platforms](07-non-evm-platforms.md) · Up to: [Deep-dive index](README.md)
