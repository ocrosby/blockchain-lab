# Glossary

> Previous: [Limitations and honest tradeoffs](08-limitations.md) · Next: [Further reading](10-further-reading.md)

One-line definitions of every term the rest of this repo uses, so you can look something up without re-reading the doc it first appeared in. When a term has a longer treatment in a specific doc, that doc is linked.

## A

**Account model** — a state representation where each address has a balance, nonce, code, and storage. Used by Ethereum. See [transactions](05-transactions.md).

**Address** — a public identifier derived from a public key, used as the destination of transactions. Not an account, in the UTXO sense; it is a key under which value lives.

**Append-only** — the property that new entries can be added but existing entries cannot be removed or modified. See [blocks and chains](03-blocks-and-chains.md).

**Archive node** — a full node that retains the complete historical state at every block, not just the current state. Multi-terabyte storage.

**ASIC** — application-specific integrated circuit. Specialized Bitcoin-mining hardware; thousands of times faster at SHA-256 than general-purpose CPUs or GPUs.

## B

**Base layer / L1** — the blockchain itself (Ethereum mainnet, Bitcoin mainnet), as distinguished from Layer 2 systems built on top.

**Block** — a batch of transactions, bundled with a header containing the parent block's hash and other metadata. See [blocks and chains](03-blocks-and-chains.md).

**Block height** — the number of blocks between a given block and the genesis block. The current "top" of the chain's block height increases by one with each new block.

**Block reward** — the newly issued currency awarded to the producer of a block, separate from transaction fees.

**Bridge** — a contract that locks an asset on one chain and mints a representation on another, allowing value to flow between chains. Historically, a frequent source of large exploits.

**Burn** — destroying currency by sending it to an address nobody can spend from, or by a protocol-level mechanism (Ethereum's EIP-1559 burns the base fee of every transaction).

**Byzantine fault tolerance (BFT)** — the property of a distributed system that remains correct in the presence of participants who behave arbitrarily, including maliciously.

## C

**Chain** — the full ordered sequence of blocks from genesis to the current tip.

**Chain analysis** — tools and techniques for de-anonymizing addresses and tracing fund flows across public blockchains.

**Composability** — the property that any contract on a blockchain can be called by any other contract, atomically, in a single transaction. The foundation of DeFi.

**Confirmation** — one block built on top of the block containing a given transaction. More confirmations = exponentially less likely the transaction gets reorganized out.

**Consensus** — the problem of getting a distributed set of participants to agree on a single ordered history. See [consensus](04-consensus.md).

**Contract account** — in Ethereum, an account with code and storage. Executes bytecode when called.

## D

**DAO (Decentralized Autonomous Organization)** — a governance structure encoded in smart contracts, where token holders vote on proposals.

**DApp (Decentralized Application)** — a user-facing application whose backend logic runs on a blockchain via smart contracts.

**DeFi (Decentralized Finance)** — the ecosystem of financial applications (exchanges, lending, derivatives) built as smart contracts on general-purpose chains.

**Deterministic execution** — the requirement that every full node executing the same bytecode on the same state reaches the same result. Prevents any operation that depends on local time, randomness, I/O, or other non-shared inputs.

**Difficulty** — in proof of work, a parameter that adjusts how hard it is to find a valid block. Increases when blocks are arriving too fast; decreases when too slow. On Bitcoin, retargets every 2016 blocks.

**Digital signature** — an unforgeable statement, produced with a private key, that attests the holder endorsed a specific message. See [cryptographic primitives](02-cryptographic-primitives.md).

**Double-spending** — the attack of trying to spend the same unit of digital currency twice. The core problem a blockchain solves. See [the trust problem](01-the-trust-problem.md).

**DPoS (Delegated Proof of Stake)** — a consensus variant where coin holders vote for a small set of block-producing validators.

## E

**ECDSA** — Elliptic Curve Digital Signature Algorithm. The signature scheme used by Bitcoin and (originally) Ethereum.

**EIP** — Ethereum Improvement Proposal. The formal mechanism for proposing changes to Ethereum.

**EOA (Externally Owned Account)** — an Ethereum account controlled by a private key, as opposed to a contract account.

**ERC-20** — the standard interface for a fungible token on Ethereum. The vast majority of tokens, including dollar-pegged stablecoins (USDC, USDT, DAI), implement it.

**ERC-721** — the standard interface for a non-fungible token (NFT) on Ethereum.

**EVM (Ethereum Virtual Machine)** — the stack-based bytecode interpreter in which Ethereum smart contracts execute. See [smart contracts](07-smart-contracts.md).

## F

**Finality** — the point at which a transaction is considered irreversible. Probabilistic on PoW chains (deeper = safer); deterministic on PoS chains (after a fixed number of attestations).

**Fork** — a divergence in the chain. **Soft fork**: new rules are a strict subset of old rules; old nodes still accept new blocks. **Hard fork**: new rules are incompatible; the chain splits unless everyone upgrades.

**Full node** — a node that downloads and verifies every transaction and block against the protocol rules. Does not necessarily produce blocks.

## G

**Gas** — the unit measuring the computational cost of an Ethereum transaction. Capped per block; priced by market.

**Genesis block** — the first block in a chain, hardcoded into the client software. Has no parent.

## H

**Hard fork** — a protocol change that is not backward compatible; old nodes will reject new blocks. See *fork*.

**Hash function** — a function that maps arbitrary-length input to fixed-length output. **Cryptographic** hash functions are additionally deterministic, pre-image resistant, collision resistant, and avalanche-prone. See [cryptographic primitives](02-cryptographic-primitives.md).

**Hash rate** — the number of hash operations per second a miner or network can perform. Bitcoin's total is measured in exahashes per second (10¹⁸ hashes/sec).

## K

**Key pair** — a private key (kept secret, used for signing) and a public key (shared, used for verification). See [cryptographic primitives](02-cryptographic-primitives.md).

**KYC (Know Your Customer)** — identity-verification requirements, usually imposed by exchanges on their users as a regulatory obligation.

## L

**Layer 2 (L2)** — a scaling system built *on top of* a base chain. Transactions happen off-chain and are settled back to L1 in batches. **Rollups** (Arbitrum, Optimism) are the dominant form.

**Light client** — a node that holds only block headers and requests Merkle proofs for transactions it cares about. Trades some trust for small storage.

**Longest-chain rule** — the rule that nodes adopt the chain with the most accumulated work (PoW) or most attestations (PoS). Used to resolve competing tips.

## M

**Mainnet** — the production, real-value blockchain, as distinguished from **testnet** (an identical chain with no economic value, used for development).

**Merkle root** — a single hash at the top of a Merkle tree that commits to all the transactions in a block. See [blocks and chains](03-blocks-and-chains.md).

**Merkle tree** — a binary tree of hashes used to efficiently prove that a specific transaction is in a block without transmitting the whole block.

**MEV (Maximal/Miner Extractable Value)** — the profit a block producer can extract by choosing the order and inclusion of transactions. See [mining and incentives](06-mining-and-incentives.md).

**Mining** — the process of producing blocks in a proof-of-work system. Requires expending computational work to find a valid nonce.

**Mining pool** — a group of miners that combine their hashing power and share rewards proportionally, reducing payout variance.

## N

**Nakamoto consensus** — the specific combination of proof-of-work and longest-chain rule introduced by Bitcoin.

**NFT (non-fungible token)** — a token where each unit is unique and not interchangeable. ERC-721 is the Ethereum standard.

**Nonce** — (1) in proof of work, the arbitrary number a miner adjusts to find a hash below the difficulty target. (2) In the account model, a per-account counter that prevents transaction replay.

## O

**Oracle** — a mechanism for bringing off-chain data (prices, event outcomes) on-chain in a trust-minimized way. Chainlink is the canonical example.

## P

**Permissioned chain** — a chain whose participants are a known, authorized set. Faster and cheaper than open chains; rebuilds a partial trusted-intermediary model.

**Permissionless chain** — a chain any participant can join or leave at will without authorization. Bitcoin, Ethereum.

**PoA (Proof of Authority)** — a consensus mechanism in which a known set of validators sign blocks. Fast; requires trust in the named validators.

**PoS (Proof of Stake)** — a consensus mechanism in which validators lock capital, and the right to propose blocks is assigned pseudo-randomly in proportion to stake. See [consensus](04-consensus.md).

**PoW (Proof of Work)** — a consensus mechanism in which producing a block requires solving a computational puzzle. See [consensus](04-consensus.md).

**Private key** — the secret half of a key pair, used to sign transactions. Losing it is unrecoverable; stealing it is indistinguishable from owning it.

**Public key** — the shareable half of a key pair, used to verify signatures.

**Pseudonymity** — the property of being identified by an address that is not directly linked to real-world identity, but which can be *linked* with sufficient analysis. Contrast with true anonymity.

## R

**Reentrancy** — a class of smart-contract bug where an attacker calls back into a contract before it has finished updating its state. The DAO hack (2016) was a reentrancy exploit.

**Reorg (reorganization)** — the event where a node abandons a chain tip in favor of a longer competing chain, undoing the blocks on the abandoned side.

**Rollup** — a Layer 2 scaling system that batches many L2 transactions and posts compact proofs (**optimistic rollups**: fraud proofs after the fact; **zk-rollups**: validity proofs upfront) to the base chain.

## S

**Settlement** — the point at which a transaction is considered final and irreversible.

**SHA-256** — the specific 256-bit cryptographic hash function used by Bitcoin.

**Signature** — see *digital signature*.

**Slashing** — in proof of stake, the penalty for a validator's misbehavior (double-signing, going offline), in which some of their staked capital is destroyed.

**Smart contract** — a program stored on a blockchain that executes as part of transaction processing. See [smart contracts](07-smart-contracts.md).

**Soft fork** — a protocol change where new rules are a strict subset of old rules; old nodes still accept new blocks. See *fork*.

**Stablecoin** — a token designed to maintain a stable value, typically pegged to a fiat currency. USDC and USDT are the largest; DAI is a decentralized alternative.

**Staking** — locking the chain's native currency into a smart contract in exchange for eligibility to validate blocks and earn rewards.

**State** — the current derived view of all accounts and their balances/code/storage, computed by replaying transactions from genesis.

**Sybil attack** — attacking a system by creating many fake identities to gain disproportionate influence.

## T

**Testnet** — a separate chain with the same software as mainnet but no economic value, used for development and staging.

**Token** — an on-chain asset implemented as a smart contract (ERC-20 for fungible tokens, ERC-721 for NFTs), as distinguished from the chain's native currency.

**Transaction (tx)** — a signed instruction to modify the chain's state (transfer value, call a contract, deploy a contract).

**Trustless** — the property that participants do not need to trust any specific named party. Trust is distributed over the protocol rather than concentrated. "Trust-minimized" is a more accurate term.

## U

**UTXO (Unspent Transaction Output)** — a discrete chunk of value bound to a public key, waiting to be spent as an input to a future transaction. The state model used by Bitcoin. See [transactions](05-transactions.md).

## V

**Validator** — in proof of stake, a node that produces and attests to blocks, having staked currency to earn the right.

## W

**Wallet** — software that manages a user's private keys and constructs transactions. Can be custodial (keys held by a third party) or non-custodial (user holds the keys).

**Whitepaper** — the founding document of a chain or project. Bitcoin's whitepaper (Nakamoto, 2008) and Ethereum's (Buterin, 2013) are the two canonical ones.

## Z

**Zero-knowledge proof** — a cryptographic proof that a statement is true without revealing the underlying data. Used in zk-rollups, privacy coins (Zcash), and various application-specific protocols.

**51% attack** — an attack in which a single actor controls a majority of the chain's hashing power (PoW) or stake (PoS), enabling them to rewrite recent history. Economically irrational on large chains but has happened on smaller ones. See [consensus](04-consensus.md).

> Previous: [Limitations and honest tradeoffs](08-limitations.md) · Next: [Further reading](10-further-reading.md)
