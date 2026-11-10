# Mining and incentives

> Previous: [Transactions](05-transactions.md) · Next: [Smart contracts](07-smart-contracts.md)

A blockchain survives because thousands of strangers run the software that keeps it alive. They produce blocks, verify them, relay them, and store the history. They do this without anyone ordering them to and without a formal employment relationship with anyone. The only reason they do it is that the protocol pays them to, and the only reason they are paid in a currency anyone wants is because enough people have decided that currency is worth something. This doc looks at that incentive loop and why it holds together.

## Who the participants are

Several roles exist. You can be more than one.

- **Full nodes** download every block, verify every transaction and signature against the protocol rules, and maintain a complete copy of the ledger state. They do not produce blocks. They exist because *verifying the chain yourself* is the only way to not depend on anyone else. A full node running on your laptop will reject any block that violates the rules, even if 99% of the network accepts it.
- **Miners** (PoW) or **validators** (PoS) are a subset of full nodes that also produce blocks. They are the ones compensated by the protocol.
- **Light clients** hold only block headers and request Merkle proofs when needed. They trade some trust for drastically smaller storage.
- **Users** send transactions. They may or may not run any node of their own — most use a wallet that talks to someone else's node.

The health of the network depends on there being many full nodes, held by many distinct parties, in many jurisdictions. Full nodes are what makes "decentralization" real, because they are what *enforce the rules*. Miners produce blocks, but the full nodes decide which blocks are legitimate. If every miner conspired to issue themselves a trillion coins in violation of the protocol, full nodes would reject those blocks and the miners would be mining on a chain no one else recognized.

## How block producers get paid

Two revenue streams, in both PoW and PoS:

1. **Block subsidy (new issuance).** The protocol creates new currency and awards it to the block producer. Bitcoin started at 50 BTC per block, halves every 210,000 blocks (~four years), and is 3.125 BTC per block as of 2024. It will approach zero by 2140, with the total supply capped at 21 million.
2. **Transaction fees.** Everything the senders in the block were willing to pay.

Early in a blockchain's life, the subsidy dominates. Over time it decays, and fees are expected to take over. Whether fees can sustainably cover the cost of running the network at today's security level is an open and actively debated question for Bitcoin specifically.

Ethereum's post-Merge issuance is small compared to Bitcoin's and is partially *offset* by EIP-1559's burn mechanism, which destroys the "base fee" portion of every transaction. When network activity is high enough, Ethereum's net issuance can go **negative** — more ETH is burned than created — which is a distinct economic dynamic from Bitcoin's fixed-supply model.

## Why mining pools exist

A miner on their own, mining Bitcoin from a garage with a few ASICs, has a real but *very small* probability of finding a block in any given ten-minute interval. The expected revenue per unit time is the same whether they mine alone or in a pool, but the variance is huge — they might go a year without finding a block.

Mining **pools** fix this by having many miners contribute their hashing power to a shared effort. Whichever pool member finds a block, the pool distributes the reward across all members in proportion to how much work each contributed. The *expected* payout per miner is roughly the same, but the *variance* collapses — payouts become steady weekly income instead of jackpot lottery.

This is good for individual miners and bad for decentralization. Four large Bitcoin pools currently control more than two-thirds of the network's hash power. Each pool is a single actor making decisions on behalf of the miners pointed at it — which blocks to build on, which transactions to include. The miners can switch pools at will, which acts as a check, but the concentration is real.

Staking pools play an analogous role in proof-of-stake. Individual holders without 32 ETH to run a solo validator delegate their stake to a pool, which runs validators on their behalf and shares rewards. Lido Finance, the largest, controls roughly 30% of all staked ETH. The same concern applies, with the same mitigation (delegators can withdraw and re-stake elsewhere).

## Why honest mining is the rational strategy

The incentives seem brittle. A miner who finds a block could, in principle, include fraudulent transactions. Why don't they?

Because every full node will reject invalid blocks. A miner who includes a transaction that double-spends, or that creates coins out of thin air, produces a block that no one else will accept. The mining reward exists in a block that is not part of the chain anyone else is extending. The miner has spent real electricity to produce a worthless block.

More subtly: a miner who finds themselves able to 51%-attack the chain has a choice. They can:

- **Attack.** Rewrite recent history, double-spend, destroy the market's confidence in the currency, watch the price crash.
- **Keep mining honestly.** Continue earning block rewards in a currency that remains valuable.

The attacking miner sabotages the value of the exact asset they are paid in. The only situation in which attacking is rational is if the attacker is paid *from outside* the system to destroy it — a state actor, say, or a short seller. This is a real threat model, especially for smaller chains. For the largest chains, the required capital expenditure is high enough to make it uneconomical for most actors.

## MEV: a nuance you should know exists

**Miner Extractable Value** (or **Maximal Extractable Value** in PoS) is the profit a block producer can extract by choosing *which transactions to include in a block, and in what order*. Because the producer can see all the pending transactions before deciding, they can:

- **Front-run.** See a user's pending trade on a decentralized exchange, insert their own transaction first to buy the same asset, let the user's trade push the price up, sell at a profit.
- **Sandwich.** Front-run and back-run around the user's trade, capturing the price movement from both sides.
- **Arbitrage.** Notice that two decentralized exchanges have different prices for the same asset, buy on one and sell on the other in the same block.

MEV is a real, large, ongoing phenomenon on Ethereum (billions of dollars extracted since 2020) and is the subject of significant protocol-level research. The core protocol does not prevent it; it is a consequence of block producers having both monopoly ordering power and visibility into pending transactions.

This is one of the places where the honest story about blockchain incentives gets uncomfortable. The incentive structure that keeps the network secure also creates adversarial opportunities between block producers and users. Work like **proposer-builder separation** aims to mitigate this, with partial success.

## Why the whole thing is self-reinforcing

Pull the loop together:

1. The block producer is paid in the chain's native currency.
2. The currency has value because users want to transact, hold, or build on the chain.
3. Users do those things because the chain is secure.
4. The chain is secure because enough honest producers find it profitable to produce blocks.
5. Producers find it profitable because the currency has value.

Remove any one of these and the loop collapses. This is why the question "what is Bitcoin worth?" or "what is Ethereum worth?" is not a trivial one — the answer reflexively affects the security of the system itself. A chain whose currency goes to zero has no one willing to mine or validate, and the chain goes with it.

Many smaller chains have discovered this the hard way. A chain with a healthy market cap today may have a dead zone in its history when the currency briefly bottomed out, hash power vanished, and attacks became cheap. Ethereum Classic has been 51%-attacked three times for this reason.

## What to carry forward

- **Full nodes** verify the chain and reject invalid blocks. They are what makes decentralization real.
- Block producers are paid via **new issuance + transaction fees**. The subsidy decays; fees are expected to take over.
- **Mining and staking pools** reduce reward variance and enable small participants, at the cost of concentrating influence in a few pool operators.
- Honest mining is rational because an attacker destroys the value of the asset they are paid in.
- **MEV** is a real, ongoing adversarial dynamic between block producers and users that the base protocol does not prevent.
- The whole system is a self-reinforcing loop: security requires value; value requires security.

Next: what happens when block contents are not just transfers, but programs?

> Previous: [Transactions](05-transactions.md) · Next: [Smart contracts](07-smart-contracts.md)
