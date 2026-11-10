# Smart contracts in depth

> Up to [main sequence doc 07](../07-smart-contracts.md) · Next: [History and conceptual roots](01-history-and-conceptual-roots.md)

The main sequence doc on [smart contracts](../07-smart-contracts.md) introduced what smart contracts are and why they exist. This subdirectory is the long version — everything you would need to go from "I understand the idea" to "I could read a real contract, assess its risks, and reason about how to write one."

## Who this deep-dive is for

Someone who has finished `docs/01-` through `docs/10-` and wants to work with smart contracts specifically. If you plan to deploy real contracts, audit them, or evaluate the smart-contract portion of a system's design, this material is prerequisite.

If you are only interested in blockchain as a payment rail or a historical curiosity, you can skip this entire subdirectory without loss.

## The problem this material addresses

The intro doc had to compress a huge surface into a few pages. It said "the EVM is a stack-based VM" and moved on. It said "reentrancy is a famous bug class" and listed three examples. It said "DeFi" and gestured. Each of those compressions buried the thing that would actually let you reason about new situations:

- You cannot audit an unfamiliar contract without a working model of the EVM's gas, storage, and call semantics.
- You cannot avoid a new class of smart-contract bug without knowing the *patterns* that connect reentrancy, access-control errors, and oracle manipulation.
- You cannot evaluate a "DeFi protocol" without knowing the primitives (AMMs, lending markets, stablecoins) it is composed from.
- You cannot pick between Ethereum, Solana, or a Cosmos-SDK chain for your application without knowing how their execution models differ.

This deep-dive fills those in.

## Reading order

Each doc builds on the previous; take them in sequence the first time.

1. [History and conceptual roots](01-history-and-conceptual-roots.md) — where the idea came from, what Bitcoin's scripting could and could not do, and why a Turing-complete blockchain changed the design space.
2. [The EVM in depth](02-the-evm-in-depth.md) — stack machine, opcode families, call/delegatecall/staticcall semantics, storage vs memory vs calldata, the gas table.
3. [Solidity language tour](03-solidity-language.md) — the pragmatic subset you need to read and write real contracts. Not a replacement for the official docs; a guide to the parts that matter.
4. [Token standards](04-token-standards.md) — ERC-20, ERC-721, ERC-1155, ERC-4626, and ERC-4337 (account abstraction). What each solves and the mistakes in each.
5. [Security patterns and famous exploits](05-security-patterns.md) — reentrancy, access control, integer issues, oracle manipulation, flash-loan attacks, front-running. The defensive patterns and the failures that motivated them.
6. [DeFi primitives](06-defi-primitives.md) — AMMs, lending markets, stablecoins, derivatives, oracles. How they compose.
7. [Non-EVM smart-contract platforms](07-non-evm-platforms.md) — Solana (Rust / Anchor), Cosmos (CosmWasm), Aptos and Sui (Move). What is different and why it matters.
8. [Tooling and lifecycle](08-tooling-and-lifecycle.md) — Foundry, Hardhat, testing, deployment, verification, upgradability, formal verification. The daily-use workflow.

## What is out of scope

Specific project post-mortems are not re-litigated here beyond their role as examples; deep legal / regulatory analysis is out of scope; and this is not a Solidity *tutorial* — the Solidity doc here assumes you can pick up unfamiliar language syntax from context and the official docs.

## When you're done

The honest marker for being "done" with this material: you can walk into a verified contract on Etherscan — any one — read its source, identify its state variables, trace what happens on each function call, identify any inherited behavior, and flag at least three things that could go wrong. If you can do that for a random contract in under 20 minutes, you have what this subdirectory is trying to give you.

> Up to [main sequence doc 07](../07-smart-contracts.md) · Next: [History and conceptual roots](01-history-and-conceptual-roots.md)
