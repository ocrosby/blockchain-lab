# Non-EVM smart-contract platforms

> Previous: [DeFi primitives](06-defi-primitives.md) · Next: [Tooling and lifecycle](08-tooling-and-lifecycle.md)

Everything so far has been EVM-centric: Ethereum mainnet, L2 rollups, EVM-compatible sidechains (Polygon PoS, BSC), and other chains that explicitly share the EVM bytecode. The EVM dominates by activity and value, but it is not the only smart-contract execution environment. Several chains have made meaningfully different design choices, and understanding them is useful both for comparison (what are the EVM's warts?) and for practical cross-chain work.

This doc covers the three non-EVM platforms a working engineer most often encounters: **Solana** (eBPF-based parallel execution), **Cosmos ecosystem with CosmWasm** (WebAssembly-based, app-chain model), and **Move-based chains** (Aptos and Sui, resource-oriented).

## The dimensions on which platforms differ

Before each platform, name the axes:

- **Execution model** — single-threaded (EVM), parallel (Solana, Sui), deterministic single-threaded with explicit conflict declaration (Aptos).
- **Account model** — accounts have code or don't; are state objects first-class, or just stored under an owning account?
- **Source language** — Solidity/Vyper (EVM), Rust (Solana, CosmWasm), Move (Aptos, Sui), AssemblyScript / others (CosmWasm).
- **Programming model** — contracts call each other synchronously (EVM), programs operate on explicitly declared accounts (Solana), objects have owners (Sui), resources are moved not copied (Move).
- **Composability** — any contract can call any other within a transaction (EVM), each transaction declares its full access set upfront (Solana), object ownership restricts what can interact (Sui).

The honest summary: each choice has tradeoffs. EVM maximizes developer familiarity and composability at the cost of throughput and programming-error surface. The alternatives trade some of the composability for other properties (parallelism, safer resource semantics, app-chain specialization).

## Solana

A monolithic, high-throughput chain launched in 2020. Different enough from Ethereum that it deserves its own mental model, not just "Ethereum but faster."

### Execution model

Programs on Solana are compiled to **eBPF** bytecode (the same format used for Linux kernel extensions, with Solana-specific modifications). An eBPF VM in each validator executes them.

The critical design decision: **transactions declare the full list of accounts they will touch upfront.** This lets the runtime parallelize non-conflicting transactions across cores. In theory, if ten transactions each touch ten disjoint accounts, they can all execute simultaneously.

```
Transaction:
  program_id: TokenProgram
  accounts: [alice_token_account, bob_token_account, token_program_id, ...]
  instruction_data: <bytes>
  signatures: [alice_signature]
```

This is *very* different from the EVM, where a transaction says "call `transfer(bob, 100)`" and the called contract decides which storage slots to touch dynamically. Solana trades flexibility for parallelism.

### Accounts and programs

- **Accounts** hold data (any bytes) and have an **owner** (another account's public key, typically a program's).
- **Programs** are stateless accounts holding executable code, flagged as "executable."
- Programs can be invoked by transactions. They read/write accounts whose owner they are (or that are passed with write permission).

The account/program separation means "the contract" and "its state" are not the same account on Solana — the contract is one account (the program), and its state is stored in many accounts owned by that program. If you deployed a token contract on Solana, the SPL Token program handles the logic, and every user's balance is a separate account owned by the Token program.

### Anchor: the Solidity-equivalent framework

Writing raw Solana programs in Rust is painful — lots of boilerplate for account validation, instruction dispatch, and PDA (program-derived address) derivation. **Anchor** is a Rust framework that provides macros and conventions for the common patterns:

```rust
#[program]
pub mod my_program {
    use super::*;
    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        ctx.accounts.counter.count += 1;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
}

#[account]
pub struct Counter {
    pub count: u64,
}
```

The `#[derive(Accounts)]` macro declares which accounts this instruction expects, which the framework validates before the handler runs. Anchor is roughly to Solana what Hardhat/Foundry is to Ethereum — the dominant developer-experience layer.

### Why the compute budget is different

Solana has no "gas" in the Ethereum sense. Each transaction has a **compute budget** (currently 1.4M "compute units" by default, raisable to 200M in a single transaction). Programs consume compute units as they execute; running out aborts the transaction. Fees are a small flat cost per transaction plus a priority fee for faster inclusion.

The practical difference: Solana transactions are much cheaper than Ethereum L1 transactions (fractions of a cent typically), but the throughput advantage comes with validator hardware requirements that are much higher.

### Where Solana differs painfully from Ethereum

- **Account resizing is explicit.** If your account needs to grow, you must re-allocate it with a specific instruction.
- **Rent.** An account that cannot pay its "storage rent" (based on size and current rates) is deleted. In practice, most accounts are "rent-exempt" by maintaining a minimum balance, but the mental model is different.
- **Program upgrades.** Programs can be upgraded by whoever holds the upgrade authority — equivalent to the Ethereum proxy pattern, but simpler (and with the same concentration-of-authority concerns).
- **Cross-program invocations (CPIs)** are the Solana equivalent of inter-contract calls. More ceremony than EVM call: you must pass through all accounts the invoked program will touch.

### The Solana outage history

Solana has had multiple multi-hour outages (2021, 2022, 2023) where the network simply stopped producing blocks. Each was traced to specific implementation bugs or resource-exhaustion attacks; each was fixed, and the frequency has decreased. Still, this is a real reliability tradeoff compared to Ethereum's uninterrupted record.

## Cosmos and CosmWasm

Cosmos is less a single chain and more a **framework** (Cosmos SDK) for building application-specific chains that interoperate via IBC (Inter-Blockchain Communication).

### The app-chain thesis

The Cosmos bet: rather than deploy your app on a general-purpose chain where you share block space with everyone else, deploy *your app's own chain*. You get the full block space, you can tune parameters for your use case, you can upgrade freely. Interoperability with other Cosmos chains is handled by IBC.

The downside: operating your own chain requires acquiring validator buy-in, bootstrapping security, maintaining the client software. Not every app is big enough to justify this. The Cosmos ecosystem has a long tail of smaller app-chains and a shorter list of large ones (dYdX v4, Celestia, Osmosis, Injective).

### CosmWasm

On any Cosmos chain that enables it, **CosmWasm** provides a general-purpose smart-contract environment. Contracts are compiled to **WebAssembly** (typically from Rust), and the CosmWasm runtime provides sandboxed execution.

```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::Transfer { recipient, amount } => try_transfer(deps, info, recipient, amount),
    }
}
```

The programming model is **actor-based**: contracts receive messages, process them, and return a Response that can include further messages to send to other contracts. There is no synchronous call stack in the EVM sense — contract-to-contract calls are explicit messages processed in sequence.

This has interesting consequences. There is no reentrancy in the EVM sense (messages are processed one at a time), but there is also less composability-in-a-single-transaction. You can still chain operations, but the semantics are different.

### Where CosmWasm wins

- **Rust ecosystem.** You get the full Rust tooling, testing, error handling.
- **WebAssembly as a stable compilation target.** Multiple languages can target it (though Rust dominates in practice).
- **Explicit message passing.** Avoids reentrancy by construction.

### Where CosmWasm loses

- **Smaller TVL and liquidity** compared to EVM chains. If your app needs deep liquidity for its token, EVM is still the default.
- **Steeper learning curve** for developers unfamiliar with Rust.
- **Less tooling maturity** compared to the EVM ecosystem (no Etherscan equivalent per chain; fewer standardized indexers).

## Move: Aptos and Sui

Move is a language designed from scratch at Facebook/Meta for Diem (the failed stablecoin project). After Diem's cancellation, its engineers forked into two chains using Move: **Aptos** (monolithic, parallel execution via conflict detection) and **Sui** (object-oriented model).

### The resource type system

Move's central idea: assets are **resources**, which have special semantics the compiler enforces. A resource cannot be copied; it can only be **moved** from one place to another. It cannot be implicitly discarded; it must be explicitly consumed.

```move
struct Coin has key, store {
    value: u64
}

// Compile error - cannot duplicate a resource:
let coin = Coin { value: 100 };
let copy_attempt = coin;        // moves ownership
let use_original = coin.value;  // ERROR: coin was moved

// Compile error - cannot drop a resource silently:
fn make_and_drop() {
    let _ = Coin { value: 100 }; // ERROR: must explicitly consume or store
}
```

This is a strong type-system property applied to the thing smart contracts most want to be careful about: assets. The Move type system makes a large class of asset-handling bugs (double-spending within a transaction, silent loss, duplication) *compile errors* rather than runtime errors.

### Aptos

Aptos uses Move with an account-based storage model (similar to Ethereum — accounts have resources stored under them) and parallel execution via **Block-STM** (optimistic parallel execution with conflict detection and re-execution on conflict).

### Sui

Sui uses a variant called **Sui Move**, with a dramatically different model: **everything is an object**. Each object has an owner (an address or another object) and a unique ID. Objects are passed to functions explicitly as arguments, not fetched from global storage.

```move
public fun transfer(coin: Coin, recipient: address) {
    transfer::public_transfer(coin, recipient)
}
```

The transaction specifies the objects it will touch (similar to Solana), which enables parallel execution. The ownership model also makes some patterns ergonomic that are awkward in other systems — NFTs with nested objects, gaming items with dynamic inventories.

### Where Move wins

- **Resource safety by construction.** The type system prevents whole bug classes.
- **Explicit ownership.** No ambient "balances" mapping; assets live in specific places.
- **Formal verification baked in.** The Move Prover tooling is part of the ecosystem, not an afterthought.

### Where Move loses

- **Smaller ecosystem** — much smaller than EVM.
- **Developer unfamiliarity** — hiring Move engineers is harder than hiring Solidity engineers.
- **Fewer deployed primitives** — the DeFi primitives from doc 6 mostly exist on Move chains, but with less liquidity, less battle-testing, and fewer forks to learn from.

## Comparing to EVM: a signal table

| Property | EVM | Solana | CosmWasm | Move (Aptos/Sui) |
|---|---|---|---|---|
| Parallel execution | No | Yes (declared accounts) | Partial (app-chain level) | Yes (block-STM or object-based) |
| Max observed throughput | ~15 TPS L1, 1000s on L2 | 1000s–10,000s | Varies per app-chain | 1000s |
| Source language | Solidity (dominant) | Rust (Anchor) | Rust | Move |
| Composability | Synchronous calls, high | Explicit CPIs, lower | Message passing | Varies |
| Reentrancy possible | Yes | Mostly no (parallel model) | No (message model) | Mitigated by type system |
| Asset-handling bugs | Common | Common | Common | Many caught by compiler |
| Tooling maturity | Highest | High (Anchor) | Moderate | Growing |
| Real-world adoption | Highest | High | Moderate | Growing |

## Which to pick?

Honest guidance for someone starting a new project:

- **If your project depends on deep liquidity or composability with existing DeFi**, go EVM. Ethereum mainnet or an L2. The network effects are real and hard to replicate elsewhere.
- **If your project is computation-heavy or needs high throughput** (games, high-frequency trading, consumer social), Solana or Sui are worth serious consideration.
- **If your project is primarily a self-contained app that doesn't need to integrate with existing DeFi**, a Cosmos app-chain gives you the most control.
- **If your project is handling complex asset types where resource safety matters**, Move's type system is a real advantage.

None of these are unconditional. The right answer depends on what your project actually does.

## What to carry forward

- The EVM is dominant but not universal. Three major alternatives: Solana (parallel eBPF), CosmWasm (WebAssembly in app-chains), Move (Aptos/Sui, resource type system).
- **Solana** trades flexibility for parallelism. Transactions declare all accounts they touch upfront. Anchor is the dominant framework.
- **Cosmos app-chains** bet on each app running its own chain, with IBC for interop. CosmWasm provides a general contract environment on top.
- **Move** was designed from scratch for safe asset modeling. Resources cannot be copied or silently dropped — bugs that are runtime failures elsewhere are compile errors here.
- Platform choice is a real decision with tradeoffs. Deep DeFi composability → EVM. High throughput → Solana/Sui. App-specific chain → Cosmos. Resource safety → Move.

Last doc: the tooling and daily workflow a working smart-contract developer uses.

> Previous: [DeFi primitives](06-defi-primitives.md) · Next: [Tooling and lifecycle](08-tooling-and-lifecycle.md)
