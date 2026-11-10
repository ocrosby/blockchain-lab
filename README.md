# blockchain-lab

A reader-first reference for people who have heard of blockchain but have never understood how any of it actually works.

## Who this is for

Someone technically literate — a programmer, engineer, student, or curious generalist — who can read pseudocode and knows what a hash function is at the "it turns input into a fixed-size string" level, but has never worked through the mechanics of Bitcoin, Ethereum, or a distributed ledger. If you have heard the words *nonce*, *Merkle root*, and *proof of work* and nodded politely without really knowing what they do, you are the target reader.

If you already run a validator node, this is not for you.

## The problem blockchain solves

**Anchor.** Every time you send money, verify a diploma, transfer a property title, or prove you own something digital, somebody in the middle keeps the real record. Your bank knows your balance. Visa knows your charges. A university registrar knows your degree. The county clerk knows who owns the house. You do not hold the authoritative copy of any of these facts — you hold a receipt, and the institution holds the truth.

**Gap.** This works, but only because we trust the institution. When the institution is unavailable, dishonest, hacked, censored, or operating under a government you do not trust, the receipt in your pocket is worth whatever the institution says it is worth on that day. The 2008 financial crisis, the Cypriot bank deposit haircuts in 2013, every frozen remittance corridor, every "we lost your records" letter — all of these are failures of the same structure: *one party keeps the ledger, and everyone else has to hope*.

**Cost.** If you wanted to remove the trusted middleman and let strangers transact directly, you would have to solve a handful of problems that look easy and are not:

- How do two strangers agree on *who owns what* without a referee?
- How do you prevent me from spending the same dollar twice, in two different places, before anyone notices? (This is the **double-spending problem**, and it is the reason digital cash did not work for thirty years.)
- How do you let anyone join or leave the network at any time without a central operator maintaining a member list?
- How do you make the history of transactions *tamper-evident* — so that if someone rewrites yesterday's ledger, it is obvious?
- How do you incentivize the strangers doing the bookkeeping to be honest, when they have never met and will never meet?
- How do you keep the system running when some fraction of participants are actively trying to break it?

Any one of these is a hard distributed-systems problem. All six at once, in an open network with no admin, is the problem blockchain was invented to solve.

**Payoff.** A blockchain is a shared, append-only ledger that a group of mutually distrustful strangers can agree on without a central operator. It achieves this by combining three primitives you probably already know in isolation — cryptographic hashes, digital signatures, and a consensus mechanism — into a data structure where tampering with old entries would require redoing more work than the honest majority of the network has done since. The result is that any participant can read the full history, verify it independently, and be confident that no single actor can rewrite it.

Whether that payoff is *worth* the costs it introduces (slow throughput, energy use, irreversibility of mistakes, and a significant set of things blockchains are bad at) is a separate question, and one the later docs address honestly.

## How to read this repo

Read the docs in order the first time through — each one builds on the previous. After that, treat them as reference.

1. [The trust problem](docs/01-the-trust-problem.md) — a longer look at *why* a central ledger-keeper has historically been necessary, and the specific failures that motivated a replacement.
2. [Cryptographic primitives](docs/02-cryptographic-primitives.md) — hash functions and digital signatures, the two building blocks everything else rests on.
3. [Blocks and chains](docs/03-blocks-and-chains.md) — the data structure itself, and why tampering with an old block invalidates every block after it.
4. [Consensus](docs/04-consensus.md) — how a network of strangers agrees on a single history. Proof of work and proof of stake.
5. [Transactions](docs/05-transactions.md) — how value actually moves. UTXO (Bitcoin) versus account (Ethereum) models.
6. [Mining and incentives](docs/06-mining-and-incentives.md) — why anyone spends real electricity to maintain the ledger, and what keeps them honest.
7. [Smart contracts](docs/07-smart-contracts.md) — what happens when the ledger entries are programs, not just transfers.
8. [Limitations and honest tradeoffs](docs/08-limitations.md) — what blockchains are genuinely bad at, and when a boring database is the right answer.
9. [Glossary](docs/09-glossary.md) — one-line definitions, for when a word in an earlier doc stopped making sense.
10. [Further reading](docs/10-further-reading.md) — the canonical papers, books, and resources, with notes on why each one is worth your time.

### Going deeper: smart contracts

The main sequence covers blockchain as a general technology. If you want to work specifically with smart contracts — read deployed contracts, audit them, or write your own — the [`docs/smart-contracts/`](docs/smart-contracts/README.md) subdirectory is an eight-doc deep-dive that assumes you have finished the main sequence.

It covers:

1. [History and conceptual roots](docs/smart-contracts/01-history-and-conceptual-roots.md) — Szabo, Bitcoin Script, why Ethereum generalized it.
2. [The EVM in depth](docs/smart-contracts/02-the-evm-in-depth.md) — stack machine, call semantics, storage economics.
3. [Solidity language tour](docs/smart-contracts/03-solidity-language.md) — the pragmatic subset for reading and writing production code.
4. [Token standards](docs/smart-contracts/04-token-standards.md) — ERC-20, ERC-721, ERC-1155, ERC-4626, ERC-4337.
5. [Security patterns and famous exploits](docs/smart-contracts/05-security-patterns.md) — reentrancy, access control, oracle manipulation, and the incidents that defined each.
6. [DeFi primitives](docs/smart-contracts/06-defi-primitives.md) — AMMs, lending, stablecoins, oracles, composability.
7. [Non-EVM platforms](docs/smart-contracts/07-non-evm-platforms.md) — Solana, CosmWasm, Move (Aptos/Sui).
8. [Tooling and lifecycle](docs/smart-contracts/08-tooling-and-lifecycle.md) — Foundry/Hardhat, audits, deployment, upgradability, monitoring.

Treat it as a reference you work through after the main sequence rather than as part of a first pass.

## Repo layout

- `docs/` — the reference material above; the main product of this repo
- `notes/` — working notes, summaries, open questions as the author studies
- `papers/` — reference papers (Nakamoto, Buterin, academic follow-ups)
- `code/` — small experiments and prototypes that illustrate specific concepts

## License

See `LICENSE` if present; otherwise assume all original prose in this repo is released under CC BY 4.0 and all code under MIT, matching the author's defaults.
