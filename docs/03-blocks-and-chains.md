# Blocks and chains

> Previous: [Cryptographic primitives](02-cryptographic-primitives.md) · Next: [Consensus](04-consensus.md)

Now we build the data structure. The word *blockchain* names a specific shape: a sequence of **blocks**, each of which contains a batch of transactions plus a pointer to the previous block. The pointer is not a memory address or a database foreign key — it is a **cryptographic hash** of the previous block's contents. That one detail is what makes the whole thing tamper-evident.

## A block

Strip away the jargon and a block is a plain record with two parts:

```
block N:
  header:
    previous_block_hash:  <sha256 of block N-1's header>
    merkle_root:          <sha256-derived fingerprint of all transactions in this block>
    timestamp:            <when this block was proposed>
    nonce:                <a number the miner chose; explained in doc 4>
  body:
    transactions:         [tx_0, tx_1, tx_2, ...]
```

The **hash of the block** is the hash of its header. Because the header includes the Merkle root (which depends on every transaction) and the previous block's hash (which depends on every transaction before it), the hash of block N is effectively a fingerprint of every transaction in the entire chain up to and including block N.

## The chain

Now chain blocks together. Each block's header contains the previous block's hash:

```
[ Block 0 ]            [ Block 1 ]            [ Block 2 ]            [ Block 3 ]
 hash = H0   ◄───────── prev = H0              prev = H1              prev = H2
 (genesis)              hash = H1    ◄───────── hash = H2    ◄───────── hash = H3
```

Block 0 is the **genesis block**, hardcoded into the software. Every subsequent block references its parent by hash. The genesis block has no parent.

### Why tampering is detectable

Here is the key property. Suppose an attacker wants to change a transaction inside block 1 — say, rewriting Alice's payment to Bob into a payment to Eve.

1. Changing a transaction changes the Merkle root of block 1.
2. Changing the Merkle root changes block 1's header, which changes `H1`.
3. Block 2's header still records the *old* `H1` as its `prev`. Block 2 no longer links to the modified block 1. The chain is broken at that joint.
4. To hide the tampering, the attacker must also rewrite block 2's `prev` to the new `H1`. But that changes block 2's hash, breaking its link with block 3. And so on, for every block after the tampered one.

**An attacker who modifies any past transaction must redo the hash of that block and every subsequent block.** In a system where producing a valid block is deliberately expensive (proof of work) or requires authorization you do not have (proof of stake), this rewrite is more expensive than the honest network's cumulative work. We'll quantify that in the next doc.

For now, the data structure gives you: *any modification to any past block is immediately detectable by anyone holding a copy of any later block's hash.*

## The Merkle root

A block holds potentially thousands of transactions. Hashing the entire block as one blob would work, but it would force any node wanting to prove "this transaction is in this block" to download the whole block. The **Merkle tree** is a tidier arrangement.

Take the transactions in a block, pair them up, hash each pair:

```
         [       root = H(H12 || H34)        ]
         /                                    \
   [ H12 = H(H1 || H2) ]                [ H34 = H(H3 || H4) ]
    /              \                     /              \
[ H1=H(tx1) ] [ H2=H(tx2) ]        [ H3=H(tx3) ] [ H4=H(tx4) ]
```

The root is a single hash that commits to every transaction below it. Two useful properties:

1. **Compactness.** One 32-byte root in the block header stands in for the entire body.
2. **Efficient inclusion proofs.** To prove that `tx3` is in the block, you need to reveal only `H4` (the sibling) and `H12` (the sibling of `H34`). The verifier recomputes the root from `tx3` + `H4` → `H34`, then `H12` + `H34` → `root`, and checks the result against the block header. For a block with N transactions, the proof is `log₂(N)` hashes, not N.

This is what makes **light clients** possible: wallets and mobile apps that can prove a transaction landed in a block without downloading every transaction in the world. They hold block headers only (small) and request Merkle proofs on demand.

## Append-only, not truly immutable

The chain is **append-only**: new blocks are added at the tip, and the structure makes modifying old blocks prohibitively expensive. But "immutable" is a slight overstatement. The chain is immutable *in practice, under the assumption that honest participants control more resources than any single attacker*. If that assumption fails — a "51% attack," where one actor controls a majority of the network's hashing power or stake — the attacker can rewrite recent history.

This has happened. Smaller blockchains with less total hashing power behind them (Ethereum Classic in 2019 and 2020, Bitcoin Gold in 2018) have had their recent history rewritten by well-resourced attackers. Bitcoin and Ethereum are considered safe against this because their honest hash power / stake is now too large and expensive to out-muscle, but the *mechanism* that protects them is economic, not mathematical.

The honest framing, then: the chain is immutable *to the extent that attacking it costs more than it is worth*. That cost grows as the chain grows — every honest block added to the tip is one more block an attacker would have to redo. Transactions buried deep in the chain are vastly more secure than transactions in the most recent block. "Wait for N confirmations" is Bitcoin-wallet-speak for "wait N blocks after yours before treating the payment as final."

## State: how the current balances are computed

A blockchain stores **transactions**, not **balances**. If you want to know "how much does Alice have right now," you start from the genesis block and replay every transaction in order, applying each one to a running tally. Alice's balance today is the sum of every payment she ever received minus every payment she ever sent (minus fees).

Full **nodes** do exactly this — they maintain the current state as a derived view of the full transaction history. Different blockchains make different choices about *what shape* that state takes, which is the topic of [doc 5](05-transactions.md).

## What to carry forward

- A **block** bundles transactions and names its parent by hash.
- A **chain** of such blocks has the property that modifying any past block invalidates every subsequent block, so tampering is detectable.
- The **Merkle root** inside each block header is a compact fingerprint of every transaction in the block, and enables efficient inclusion proofs.
- The chain is **append-only** and **immutable in practice**, where "in practice" means "to the extent that attacking it costs more than it is worth."
- The current ledger state is derived by replaying transactions from genesis; the chain does not store balances directly.

Next we tackle the social problem: how do thousands of nodes, run by strangers, agree on which block is the next one?

> Previous: [Cryptographic primitives](02-cryptographic-primitives.md) · Next: [Consensus](04-consensus.md)
