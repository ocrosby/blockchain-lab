# Consensus

> Previous: [Blocks and chains](03-blocks-and-chains.md) · Next: [Transactions](05-transactions.md)

The data structure from the last doc works beautifully if there is exactly one person deciding what the next block contains. But there is no such person — the whole point of this exercise was to eliminate the central operator. So: thousands of nodes, all over the world, run by strangers who have never met, each with their own copy of the chain and their own pending pile of transactions. How do they agree, continuously and forever, on which block is the next one?

This is the **consensus problem**, and solving it in an open network is the actual innovation of Bitcoin.

## The constraints

Make the problem explicit. A consensus mechanism for an open blockchain needs to:

1. **Work with anonymous, permissionless participants.** No membership list; anyone can join or leave at any time.
2. **Tolerate dishonest participants.** Some nodes will lie — propose invalid blocks, withhold blocks, propose two conflicting blocks at once. The protocol must give the honest majority a way to prevail.
3. **Produce a single, globally agreed-upon history.** Eventually, every honest node must converge on the same chain. Short disagreements are fine; permanent forks are not.
4. **Resist Sybil attacks.** If joining is free, a single adversary can spin up a million fake nodes. Majority voting does not work when the adversary controls the majority of (cheap) identities.

Classical distributed-systems consensus algorithms (Paxos, Raft, PBFT) solve similar problems, but they all assume a known set of participants. The open-network constraint rules them out. Blockchain's answer is to tie the right to propose a block to **a scarce resource** — something an adversary cannot cheaply manufacture.

## Proof of work (PoW)

Bitcoin's answer: the scarce resource is **computation**.

### The rule

To propose a valid block, a node must find a `nonce` (an arbitrary number) such that:

```
sha256(block_header_including_nonce) < target
```

Where `target` is a very small number — the chain's **difficulty**. Because SHA-256 is pseudorandom, the only way to find a qualifying nonce is to try nonces one at a time until one works. On today's Bitcoin network, this requires performing on the order of 10²² hash attempts per block, adjusted every 2016 blocks (≈two weeks) so that blocks arrive on average every ten minutes.

This activity is called **mining**. The node that finds a valid nonce **publishes the block** to the network. Other nodes verify in a single hash operation that `sha256(header) < target`, see that the included transactions are valid, and append the block to their own copy of the chain.

### Why this works as consensus

Three properties combine:

1. **Proposing a block is expensive.** You spent real electricity on real hardware to find that nonce.
2. **Verifying a block is cheap.** One hash. Anyone, with any hardware, can check whether a proposed block is valid.
3. **The longest chain wins.** If two miners publish competing blocks at nearly the same time, the network temporarily splits. Each half keeps mining on its preferred tip. Whichever tip gets the next block *first* becomes the longer chain, and the other half of the network abandons their fork and switches over.

"Longest chain" is slight shorthand; the precise rule is "the chain with the most accumulated work." Since difficulty changes over time, a chain with more blocks could technically have less total work than a shorter chain mined under higher difficulty. In practice, these are nearly equivalent.

### The 51% attack, concretely

Here is why the "attacker must redo the work" claim from doc 3 has teeth. Suppose an attacker wants to double-spend: pay Alice in a transaction, wait for the merchandise to ship, then rewrite history so that payment goes to themselves instead.

To do this, the attacker starts from the block containing their payment to Alice and builds a competing chain in secret. For this competing chain to "win" and be adopted by the network, it must become longer than the honest chain. But the honest chain is being extended by every other miner on Earth. If the attacker controls less than 50% of the total hash power, the probability that they overtake the honest chain drops exponentially with every honest block mined. After 6 blocks (one hour on Bitcoin), the probability is vanishingly small, which is why "6 confirmations" is Bitcoin-wallet folklore for a settled payment.

At 50%+ of hash power, the attacker catches up on expectation and the attack becomes economically plausible. Hence "51% attack." Bitcoin's defense is that acquiring 51% of the hash power on a network currently backed by billions of dollars of specialized hardware is itself prohibitively expensive, and *would destroy the value of the chain the attacker just captured*. So the attacker's incentive is weirdly aligned with the network's: own the hardware, play honestly.

### What proof of work actually costs

This is where honest accounting gets uncomfortable. Bitcoin's hash power is produced by specialized hardware (ASICs) running continuously. The total energy consumption, as of this writing, is estimated at **150–200 TWh per year** — comparable to the entire electricity consumption of a small country (Poland, Argentina).

This is a design feature, not a bug. The whole protection against attack is that mining is **genuinely costly**. If it got cheap, the chain would be cheap to attack. But it does mean the security model of proof-of-work blockchains is built on real-world energy use.

## Proof of stake (PoS)

Ethereum's current answer (since the 2022 "Merge"): the scarce resource is **capital at risk**.

### The rule

Instead of competing to solve a hashing puzzle, validators **stake** a quantity of the chain's native currency (32 ETH on Ethereum) by locking it in a special contract. The protocol then pseudo-randomly selects one validator at a time to propose the next block. Other validators attest to the proposed block's validity. Blocks that receive enough attestations are finalized; blocks that are invalid cause the proposing validator to be **slashed** — a portion of their staked currency is destroyed.

### Why this also works

- **Proposing a block requires capital at risk.** You can only be a validator if you have locked up ETH. You cannot spin up a million validators without locking up a million × 32 ETH.
- **Verifying a block is still cheap.** Validators check signatures and transaction validity; no mining puzzle.
- **Misbehavior is punishable by slashing.** A validator who signs conflicting blocks, goes offline, or proposes invalid blocks loses part of their stake. Attacking the chain means losing your own money.

The 51%-style attack on PoS requires controlling a majority of the staked currency. The attacker would need to acquire, lock up, and then burn (via slashing) more than half of the total ETH staked — currently tens of billions of dollars. And again, a successful attack would destroy the value of the asset they just captured.

### The honest tradeoff between PoW and PoS

| | Proof of Work | Proof of Stake |
|---|---|---|
| Scarce resource | Computation (energy) | Capital |
| Entry cost | Hardware + electricity | Must acquire native currency to stake |
| Energy use | High (feature of the security model) | Low |
| Attack cost | Buy enough hardware + electricity | Buy enough of the native currency |
| Centralization pressure | Economies of scale in mining hardware; mining pools | Wealth concentration; staking pools |
| Punishment for cheating | Wasted electricity | Slashing (direct loss of staked capital) |
| Entry barrier for small participants | Hardware cost, but mining pools lower the floor | 32 ETH outright, or participation via staking services |
| Finality | Probabilistic (deeper = safer) | Deterministic after two epochs (~12.8 min on Ethereum) |

Neither mechanism is "better" in the abstract. PoW spends energy to buy security; PoS forgoes the energy cost but concentrates influence in the hands of whoever already owns the currency. Both have been running real networks with billions of dollars at stake for years, and both have specific failure modes that have been studied, modeled, and occasionally exploited.

## Other mechanisms worth knowing exist

- **Delegated proof of stake (DPoS)** — coin holders vote for a small set of validators (Cosmos, EOS). Faster, more centralized.
- **Proof of authority (PoA)** — a known set of permissioned validators sign blocks (used in some enterprise chains). Fast, but requires trust in the named validators, which rebuilds the trusted institution we were trying to avoid.
- **Proof of history (Solana)** — validators produce a verifiable sequence of time-stamps as a pre-step to PoS; aims to increase throughput.
- **Practical Byzantine Fault Tolerance (PBFT) and variants** — classical algorithms used in permissioned chains where the participant set is known.

The ecosystem is a design space, not a single answer. The two mechanisms above (PoW and PoS) dominate open, permissionless networks and are the ones you will encounter most often.

## What to carry forward

- Consensus is the problem of getting an open set of mutually distrustful nodes to agree on a single ledger history.
- The trick is to tie the right to propose blocks to a **scarce resource** that cannot be cheaply faked.
- **Proof of work** uses computation (and therefore energy). The 51% attack is theoretically possible but economically irrational on a large network.
- **Proof of stake** uses locked capital. The equivalent attack requires acquiring and then destroying a majority of the chain's own currency.
- "Finality" is probabilistic in PoW (deeper is safer) and deterministic in PoS (after a fixed number of attestation rounds).
- Both mechanisms have been running production networks with billions of dollars at stake for years.

Next: now that we can agree on the order of transactions, let's actually look at what a transaction *is*.

> Previous: [Blocks and chains](03-blocks-and-chains.md) · Next: [Transactions](05-transactions.md)
