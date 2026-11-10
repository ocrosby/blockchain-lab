# Limitations and honest tradeoffs

> Previous: [Smart contracts](07-smart-contracts.md) · Next: [Glossary](09-glossary.md)

A reference that is only enthusiastic is a brochure. This doc catalogs what blockchains are genuinely bad at, what problems they do not solve, and the categories of use case where a conventional database is the right answer. Reading the rest of the repo and this doc together is what moves you from "I understand how it works" to "I understand when it matters."

## The scalability trilemma

A well-known informal claim: a blockchain can offer at most two of three properties simultaneously.

- **Decentralization** — the chain is maintained by many independent participants with no privileged actors.
- **Security** — attacking the chain requires a large and growing amount of resources.
- **Scalability** — the chain can process a large number of transactions per second with low fees.

Bitcoin and Ethereum pick decentralization and security, and are slow. Solana and similar high-throughput chains pick scalability and security, and concede decentralization (fewer validators, higher hardware requirements, occasional outages). Private/permissioned chains pick scalability and decentralization within a known set, and concede the "anyone can join" form of security.

This is not a mathematical theorem — researchers have proposed many designs aimed at breaking the trilemma — but it is a durable observation. Every design that claims to solve all three is worth scrutinizing for which property it actually sacrifices under load.

### Concrete throughput numbers

| Chain | Approximate transactions per second | Finality time |
|---|---|---|
| Bitcoin | 3–7 | 60 minutes (6 confirmations) |
| Ethereum L1 | 15–30 | 12.8 minutes (two epochs) |
| Solana | 1,000s (sustained), 50,000+ (peak claims) | ~13 seconds |
| Visa (centralized) | 24,000 average, 65,000+ peak | Milliseconds (reversible) |

Bitcoin's throughput is an *intentional* limit — block size is capped at roughly 1 MB (effectively more with SegWit), and block time at ten minutes, to keep the full node storage and bandwidth low enough that ordinary people can still run one. Raising these limits improves throughput at the cost of centralizing full nodes to those with large hardware budgets.

### Layer 2: the pragmatic answer

Rather than scale the base chain, modern designs scale *on top of* it. **Layer 2 (L2)** systems — **rollups** (Arbitrum, Optimism, Base on Ethereum; Stacks, Lightning on Bitcoin) — batch many transactions off-chain and post compact proofs of them to the base chain. The base chain (**L1**) remains the settlement and security layer; the L2 handles the throughput.

Modern Ethereum's working throughput, counted across L2s, is in the thousands of transactions per second with fees of fractions of a cent. The underlying L1 still processes only ~15 per second, but what users actually transact on is the L2.

This is a *substantial* change in the practical story of Ethereum from a few years ago. If you learned the "15 TPS, dollar-plus fees" framing, it is now outdated for most real user activity.

## Irreversibility is a bug *and* a feature

Blockchain transactions are final. For legitimate transactions this is wonderful — the counterparty cannot reverse your payment, nothing can be clawed back after settlement.

For illegitimate transactions this is a disaster.

- **Wrong address** — your money is gone to an address that may be nobody's.
- **Phishing** — you signed a transaction authorizing a malicious contract to drain your wallet, and it did.
- **Private key theft** — a hacker with your key *is* you, and anything they do is as valid as anything you would do.
- **Contract bug** — the contract you trusted had a reentrancy vulnerability, your funds are drained, there is no appeal.

A credit card company will reverse a fraudulent charge in hours. A blockchain will not. The entire support apparatus that consumer finance has built over decades is absent, by design. For many users, this cost outweighs the benefit of self-custody — which is why most crypto users still hold their coins on a centralized exchange that *does* have a support team, at the cost of re-introducing the trusted institution the system was invented to avoid.

## Privacy is weaker than people think

A common first assumption is that Bitcoin is anonymous. It isn't. The chain is **pseudonymous** — your address is not linked to your identity on the chain, but once anyone links it to you (via an exchange KYC record, a public donation address, a careless social media post), they can see every transaction you have ever made and every transaction anyone you have ever paid has made. Chain analysis firms (Chainalysis, TRM Labs) specialize in de-anonymizing addresses and sell this capability to governments and exchanges.

Privacy-focused chains (**Monero**, **Zcash**) and privacy tools on general chains (**Tornado Cash**, since sanctioned) attempt to break this linkability using cryptography (ring signatures, zero-knowledge proofs). They work, to varying degrees, but adoption is small and regulatory pressure on privacy tools has been severe.

For any sensitive use — medical records, voting, whistleblowing — the pseudonymity of a public chain is usually not enough on its own.

## Throughput of what, exactly: storage costs

Every full node holds a complete copy of the chain. Bitcoin's chain is ~550 GB. Ethereum's is ~1 TB for a standard full node, and multi-TB for an archive node that keeps all historical state. Growth is continuous.

This grows even faster on high-throughput chains. Solana produces much more data per unit time, and the hardware required to run a Solana validator is accordingly steep (hundreds of gigabytes of RAM; multi-terabyte NVMe). The practical consequence is that *fewer people can afford to run a full node*, which is the exact centralization pressure that Bitcoin's throttled throughput was designed to avoid.

## Blockchain is not a good database

For many problems people reach for blockchain, a boring database is the right answer:

- **Internal enterprise record-keeping** — you control the database. You do not need Byzantine fault tolerance against your own colleagues. A Postgres instance with audit logging solves this better and cheaper.
- **Medical records between known hospitals** — the hospitals know each other and are regulated. HL7 FHIR over TLS does what you want. The ceremony of blockchain adds cost without adding trust-minimization because there was no trust problem to begin with.
- **Supply chain tracking between known partners** — same. If you trust your suppliers, a shared database works. If you don't, blockchain does not magically make the physical items match their digital records — someone still has to scan them, and that someone can lie.
- **Any system where the real trust problem is at the input layer, not the storage layer** — this is huge. "Put real estate records on blockchain" does not solve the problem that a county clerk might still enter fraudulent records. It just makes those fraudulent records tamper-evident after the fact.

The question to ask, before reaching for blockchain, is: *is there a specific moment in this workflow where a mutually distrustful set of parties need to agree on something without a trusted intermediary?* If yes, blockchain might help. If no, there is no problem for blockchain to solve, and a database will do.

## Environmental and governance critiques

- **Energy use** (PoW) — real and large. Bitcoin's annual electricity consumption is comparable to small countries. Defenders point out that a non-trivial fraction is from otherwise-stranded renewable energy (hydro, flared gas); critics point out that marginal demand still raises emissions. The honest answer is that it depends on the local energy mix of a given mining operation.

- **Wealth concentration** — the distribution of ownership on major chains is highly unequal. A small fraction of addresses holds a large majority of the supply. This is partly early-adopter advantage, partly natural wealth dynamics; on PoS chains, it also means voting-power concentration.

- **Regulatory environment** — rapidly shifting, jurisdiction-dependent, and often adversarial. Tools and designs that are legal in one jurisdiction are unambiguous crimes in another. Participants should not assume the regulatory environment they built under will persist.

- **Social coordination failures** — governance disputes have repeatedly produced hard forks that split chains in half (Bitcoin / Bitcoin Cash, Ethereum / Ethereum Classic, Steem / Hive). The "no central authority" property means these disputes have no resolution mechanism other than fork.

## When blockchain *is* the right answer

To close on a positive note. The use cases where blockchain is materially better than the alternative share a few properties:

- **Open participation needed.** Permissionless systems where "the set of users" cannot be known in advance.
- **No trusted intermediary available or acceptable.** Cross-border value transfer without correspondent banking; financial access in authoritarian contexts; novel coordination games.
- **Settlement finality valuable enough to outweigh irreversibility risk.** Settlement between institutions that do not trust each other.
- **Programmable value with composability.** DeFi, where the composition of financial primitives produces functionality no single provider offers.
- **Censorship resistance is itself a feature.** Where "no one can stop me from transacting" is worth the other tradeoffs.

Not all of these are universally good — censorship resistance helps dissidents and ransomware operators alike. The point is that there are cases where the specific properties of a public blockchain are the right fit, and cases where they are not. A thoughtful reader can tell the difference.

## What to carry forward

- The scalability trilemma is a durable informal constraint. Modern practice scales *around* the base chain via **L2 rollups**, not by raising the base chain's throughput.
- Transactions are final. This is wonderful for legitimate activity and brutal for mistakes, theft, and bugs.
- Public chains are **pseudonymous**, not anonymous. Privacy is weaker than first-time users assume.
- Running a full node is expensive on high-throughput chains, which pushes toward centralization.
- Most problems people reach for blockchain for are better solved by a boring database. The useful question is: *is there a specific moment where mutually distrustful parties need to agree without an intermediary?*
- Blockchain-specific advantages (permissionless access, censorship resistance, composability of programmable value) are real but narrow. Treat them as a specific toolkit, not a universal solvent.

Next up: a glossary of all the terms this repo has thrown at you, for quick reference when something earlier stops making sense.

> Previous: [Smart contracts](07-smart-contracts.md) · Next: [Glossary](09-glossary.md)
