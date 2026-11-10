# Transactions

> Previous: [Consensus](04-consensus.md) · Next: [Mining and incentives](06-mining-and-incentives.md)

A transaction is the atomic unit of change in a blockchain. Doc 2 already sketched what a transaction looks like at the signature layer — a signed message from one party. This doc goes down one level and shows you the two dominant *state models* that blockchains use to represent "who has what": the **UTXO model** used by Bitcoin, and the **account model** used by Ethereum. They produce the same high-level experience ("Alice sends Bob 10 units") but they differ in some ways that will matter when we talk about smart contracts.

## The UTXO model (Bitcoin)

UTXO stands for **unspent transaction output**. In this model, there are no account balances anywhere in the system. The entire ledger is a set of unspent outputs — discrete chunks of value, each bound to a public key — and a transaction is the act of consuming some unspent outputs and producing new ones.

### A UTXO is a chunk of value

Imagine your Bitcoin wallet not as a bank account with a balance, but as a drawer full of coins of various denominations:

```
Alice's wallet:
  - UTXO #a7f3:  2.5 BTC (output from a tx 3 months ago)
  - UTXO #b2e9:  0.1 BTC (output from a tx last week)
  - UTXO #c440:  1.0 BTC (output from a tx yesterday)
```

Each UTXO is a specific, labeled chunk. Alice's "balance" is the sum of the chunks (3.6 BTC in this example), but the chain doesn't record that number anywhere — it records only the individual chunks and their ownership.

### A transaction consumes and produces UTXOs

To send Bob 2.7 BTC, Alice's wallet selects a combination of her UTXOs totaling *at least* 2.7, consumes them (they are now marked spent and cannot be reused), and produces new outputs:

```
Transaction:
  inputs:  [ UTXO #a7f3 (2.5 BTC),  UTXO #c440 (1.0 BTC) ]   # total: 3.5 BTC
  outputs:
    - 2.7 BTC → bob_public_key                                # the payment
    - 0.79 BTC → alice_public_key                             # change back to Alice
  fee: 0.01 BTC                                               # the leftover: miner takes it
```

The sum of inputs must be `≥` the sum of outputs; the difference goes to the miner as a fee. The two new outputs are now *the* 2.7 BTC and *the* 0.79 BTC. They sit in the UTXO set until spent.

### Why UTXO

This has some surprising properties:

- **Natural parallelism.** Different transactions touch different UTXOs, so they can be processed in any order as long as none of them double-spend the same UTXO. There is no single "account balance" that becomes a hot spot.
- **No ordering ambiguity.** A UTXO exists or it does not. There is no "what was Alice's balance *when* this transaction executed" question — only "is this specific UTXO currently unspent?"
- **Privacy via address reuse avoidance.** Each change output goes to a fresh address controlled by the same wallet. On-chain, this looks like many different parties, not one person's running balance. (In practice, chain analysis tools cluster these together, but the mechanism makes privacy *possible*.)

And one real downside for programmers:

- **State is fragmented.** There is no "Alice's balance" to read; you have to iterate the UTXO set for all outputs bound to Alice's key. Smart contracts, which want to say things like "if the caller has at least X, do Y," are awkward to express on UTXO.

## The account model (Ethereum)

Ethereum's answer: skip the UTXO drawer and just store an account balance, like a bank database.

```
state = {
  alice_address: { balance: 3.6 ETH, nonce: 42, code: null, storage: {} },
  bob_address:   { balance: 0.2 ETH, nonce: 7,  code: null, storage: {} },
  contract_C:    { balance: 100 ETH, nonce: 1,  code: <bytecode>, storage: {counter: 73} },
  ...
}
```

A transaction is a direct mutation:

```
Transaction:
  from:   alice_address
  to:     bob_address
  value:  2.7 ETH
  nonce:  43
  gas:    21000
  signature: <alice's signature over the above>
```

When the network processes this, it directly debits Alice's balance and credits Bob's. Simple.

### The nonce prevents replay

Notice `nonce: 43` in Alice's transaction. The state tracks a counter per account — currently 42. A transaction is only valid if its nonce is exactly one more than the account's current nonce, after which the stored nonce is incremented. If an attacker rebroadcasts Alice's old transaction, it will be rejected because the state nonce has already moved past.

In the UTXO model, replay prevention is built in: the transaction consumes specific UTXOs, which are then spent. Rebroadcasting cannot reuse them. The account model needs the explicit nonce to get the same property.

### Contracts are accounts with code

The real reason Ethereum chose the account model is in that third row of the state above. Ethereum accounts come in two flavors:

- **Externally owned accounts (EOAs)** are controlled by a private key, like a Bitcoin wallet. `nonce` increments per transaction; `code` is empty.
- **Contract accounts** have `code` — EVM bytecode — and `storage` — a key/value store. They cannot initiate transactions on their own; they execute code in response to receiving one.

A transaction to a contract account triggers the contract's code, which can read its own storage, modify it, call other contracts, transfer ETH, and so on — all as part of the transaction's atomic execution. If any step fails, the entire transaction reverts and nothing persists except the gas fee paid.

The account model exists because this kind of stateful, programmable execution is much easier to express when there is a well-defined "this contract's current storage" than when state is spread across a UTXO set.

Smart contracts get their own doc: [doc 7](07-smart-contracts.md).

## What a transaction fee is

Both models charge a fee per transaction, paid to the miner or validator that includes it. This fee serves two purposes:

1. **Rationing.** Block space is scarce. Fees sort the transaction demand: whoever is willing to pay more gets included sooner.
2. **Compensation.** The block producer did real work (hashing, or attesting) and earned real currency for it. Over time, as new-coin issuance decreases, fees are expected to become the primary compensation for block producers. Bitcoin's issuance halves every four years and will approach zero by 2140; the system's long-term security model is predicated on fees being sufficient.

In Ethereum, fees are denominated in **gas** — a unit that reflects the computational cost of the transaction. A simple transfer costs 21,000 gas; a complex contract execution can cost millions. The gas price (ETH per unit of gas) is set by the sender and influenced by current demand.

## Finality and reversibility

Once your transaction is in a block, is it done?

- **In a block, zero confirmations:** it's visible on the chain, but only one block deep. If a competing chain of one block appears, your block might lose and your transaction with it.
- **One confirmation:** your block plus one block built on top. Hard to displace, but possible.
- **Six confirmations (Bitcoin convention):** a would-be attacker would need to produce six blocks faster than the rest of the network. Extremely unlikely without majority hash power. Most exchanges treat six confirmations as "settled."
- **PoS finalization (Ethereum, ~12.8 minutes):** after two epochs of attestations, a block is **finalized**. Reversing a finalized block would require a supermajority of validators to be slashed — a defined, expensive, intentional act.

In either case, **there is no "cancel" button**. If you sent to the wrong address, if you got scammed, if you authorized a malicious contract — the chain will not help you. This asymmetry is often the biggest culture shock for newcomers.

## What to carry forward

- **UTXO** (Bitcoin) treats value as discrete chunks. A transaction consumes and produces chunks. No account balances stored anywhere.
- **Account model** (Ethereum) stores a balance (and nonce, code, storage) per account. Transactions directly mutate the state.
- The account model makes smart contracts natural; the UTXO model makes parallelism and privacy easier.
- A fee is paid to the block producer. Over time, fees are expected to replace new-coin issuance as the primary compensation.
- Finality is probabilistic in PoW (deeper is safer) and deterministic in PoS. In either case, there is no cancel.

Next: who runs the hardware, and why do they bother?

> Previous: [Consensus](04-consensus.md) · Next: [Mining and incentives](06-mining-and-incentives.md)
