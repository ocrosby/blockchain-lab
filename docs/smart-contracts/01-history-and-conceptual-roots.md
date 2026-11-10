# History and conceptual roots

> Previous: [Deep-dive index](README.md) · Next: [The EVM in depth](02-the-evm-in-depth.md)

The phrase "smart contract" existed before blockchain. Grasping what it originally meant, and the specific limitations of Bitcoin's scripting that motivated Ethereum, is the shortest route to understanding why the modern smart-contract stack looks the way it does.

## Szabo and the vending machine

In 1994, cryptographer and legal scholar Nick Szabo proposed **smart contracts** as "a computerized transaction protocol that executes the terms of a contract." His recurring example was the humble vending machine: a mechanism that takes an input (coins), applies rules (if the price is met, release the product), and resolves the transaction without a human intermediary. The machine is the contract.

Szabo's vision was more ambitious — he imagined derivative contracts, loan collateral, property transfers, all mechanized so that the enforcement was built into the system rather than relying on courts. But the technology of 1994 could not do most of this. There was no shared, tamper-evident state for parties to agree on. The legal system remained the only credible enforcement mechanism.

The important part of Szabo's framing, which carries forward: a smart contract does not *interpret* legal intent. It *executes* a specific state machine. If the state machine is wrong, the smart contract is wrong, and there is no judge to appeal to. "Smart" in this context means "automatic," not "intelligent."

## Bitcoin script: smart contracts, kind of

Satoshi Nakamoto's 2008 Bitcoin paper introduced the first production environment where Szabo's idea could partially work. Bitcoin transactions can, in fact, contain small programs — scripts — that control the conditions under which outputs can be spent.

Bitcoin **Script** is a stack-based language with roughly 180 opcodes (many disabled for safety). A typical "pay to public key hash" (P2PKH) script, which controls the vast majority of Bitcoin balances, looks like this in pseudocode:

```
OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
```

This says: duplicate the top stack item, hash it, verify the hash matches `<pubKeyHash>`, then verify a signature. If all checks pass, the output can be spent. Otherwise it cannot.

More complex scripts are possible. Multi-signature wallets ("3 of 5 of these keys must sign"), hash-locked contracts ("reveal a preimage and the output is spendable"), and time-locked contracts ("not spendable until block N") are all expressible. These are smart contracts in Szabo's original sense: the rules are executed by the protocol, not the legal system.

### What Bitcoin Script cannot do

Three limitations, by design:

1. **Not Turing-complete.** No loops, no recursion. Every script terminates in bounded time. This was a deliberate choice to prevent denial-of-service attacks — you cannot write a Bitcoin script that runs forever.
2. **No persistent state.** A script runs at the moment an output is spent, in isolation. It cannot read from storage, update counters, or coordinate with other scripts in later transactions.
3. **No inter-script calls.** Scripts cannot call each other. Each is a standalone validator for a single spending event.

These limitations keep Bitcoin simple and safe but exclude vast categories of applications — anything that needs mutable state over time, anything that needs multi-party coordination through code rather than just signatures, anything that needs to compose with other on-chain logic.

Bitcoin's designers knew this. The limitations were not accidents; they were the point. Bitcoin set out to be digital cash with scripting rich enough for the common cases and nothing more. The ecosystem has extended this over time — **Taproot** (2021) added more scripting flexibility, and **BitVM** (2023) proposed ways to simulate richer computation using challenge-response protocols. But the fundamental model has stayed close to Satoshi's design.

## Ethereum: generalizing to the state machine

In 2013, a 19-year-old Vitalik Buterin proposed a different framing in his Ethereum whitepaper. Rather than Bitcoin's "ledger plus restricted scripting," Ethereum would be "a world computer": a global shared state, modified by transactions, where each transaction can execute arbitrary code.

Buterin's framing explicitly accepted the costs that Bitcoin's designers had refused:

- **Turing completeness.** Loops and recursion allowed. The DoS problem is solved by metering execution — every operation costs gas, and transactions run out of gas before they can halt the network.
- **Persistent per-contract state.** Every contract has its own storage, modifiable by its own code.
- **Composability.** Contracts can call other contracts synchronously, atomically, as part of a single transaction.

The claim was that these three capabilities, together, enabled a *qualitatively different* set of applications. A system with all three can represent any state machine you can write; a system without any of them is restricted to what you can express as a one-shot predicate.

Ten years later, the claim seems substantially correct. Decentralized exchanges, lending protocols, NFT markets, DAOs, bridges, prediction markets, options protocols, and essentially every other on-chain application depends on one or more of these three capabilities.

## The EVM: Buterin's commitment to language-independence

A detail that often goes unappreciated: Ethereum did not bet on a specific *programming language*. It defined the **Ethereum Virtual Machine**, a bytecode interpreter with well-specified semantics, and let compilers target it.

The first widely-used source language was **Solidity**, which Gavin Wood designed around 2014 with syntax inspired by JavaScript and C++. Others followed: **Vyper** (Python-inspired, security-focused, smaller feature surface), **Fe** (newer, Rust-inspired), **Huff** (low-level, direct EVM access), **Yul** (intermediate language used by the Solidity compiler itself).

This matters because the EVM is the stable contract between language ecosystems and the chain. The language you write in is a choice; the EVM semantics are not. Any audit, formal verification, or tooling at the EVM level is applicable regardless of source language.

The non-EVM chains covered in [doc 7](07-non-evm-platforms.md) made different choices here. Solana runs programs compiled to eBPF. Aptos and Sui run **Move**, a language designed from scratch for resource-safe asset modeling. CosmWasm runs WebAssembly. Each design has its own tradeoffs, and the choice of execution environment is one of the larger decisions a chain makes.

## Why "contract" is a weak metaphor

A frequent critique, and worth addressing explicitly: smart contracts are not contracts.

A legal contract has:

- An agreement between parties to specific terms.
- Mechanisms to interpret ambiguous clauses in light of intent.
- Appeal to a legal system when things go wrong.
- Mechanisms to adapt when the world changes (force majeure, impossibility, reformation).

A smart contract has:

- A compiled bytecode program.
- Deterministic execution of whatever the code happens to say.
- No appeal, no interpretation, no adaptation.

The DAO hack in 2016 is the clearest demonstration. The attacker exploited a bug in a crowdfunded investment contract, draining 3.6 million ETH. From a *legal* perspective, the money had been stolen — the attacker's actions were clearly against the intent of the DAO's organizers and participants. From a *code* perspective, the attacker had done nothing but call a function that the contract allowed to be called. The Ethereum community eventually hard-forked the chain to reverse the theft, implicitly admitting that "the contract's code is the contract" is not a sustainable position when the outcome is sufficiently wrong.

So "smart contract" is best read as **"self-executing state machine with on-chain effects."** The word "contract" is baggage from Szabo's original framing, and leads to real confusion when people treat the mechanism as having the properties of legal contracts. It does not.

## The layered mental model

Putting the layers together, here is a map of where things sit:

```
┌──────────────────────────────────────────────┐
│  Applications (DEXes, lending, NFT markets)  │  ← what users interact with
├──────────────────────────────────────────────┤
│  Smart contracts (Solidity, Vyper source)    │  ← what developers write
├──────────────────────────────────────────────┤
│  EVM bytecode                                │  ← what the chain executes
├──────────────────────────────────────────────┤
│  Transactions (signed state transitions)     │  ← how things get in
├──────────────────────────────────────────────┤
│  Blocks, consensus, cryptography             │  ← the base chain
└──────────────────────────────────────────────┘
```

Everything in the main sequence of this repo was about the bottom layer. Everything in this subdirectory is about the layers above it.

## What to carry forward

- Szabo's "smart contract" concept predates blockchain by about 15 years. The original meaning is *automatic, not intelligent*.
- Bitcoin Script enables a restricted form of smart contracts — enough for multi-sig, hash locks, and time locks. It is not Turing-complete by design.
- Ethereum generalized the model by introducing the EVM: Turing-complete, with per-contract persistent state, and inter-contract composability.
- The EVM is a *bytecode interpreter*, not a language. Multiple source languages (Solidity, Vyper, Fe, Yul, Huff) compile to it.
- "Smart contract" is a weak metaphor. A smart contract is a self-executing state machine, not a contract in the legal sense. This metaphor confusion has produced real-world disputes with large financial consequences.

Next: a detailed look at the EVM itself.

> Previous: [Deep-dive index](README.md) · Next: [The EVM in depth](02-the-evm-in-depth.md)
