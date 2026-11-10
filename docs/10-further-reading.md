# Further reading

> Previous: [Glossary](09-glossary.md) · Next: [Smart contracts in depth](smart-contracts/README.md)

A curated list. Every entry has a short note on *why* it is worth your time — anyone can hand you a reading list; the value is in the filter.

## The canonical whitepapers

- **Nakamoto, Satoshi. *Bitcoin: A Peer-to-Peer Electronic Cash System* (2008).** 9 pages. The original. If you have read any of this repo, you can read this paper. Section 8 ("Simplified Payment Verification") and Section 11 ("Calculations") are the ones to pay attention to — they contain the probabilistic argument for why 51% attacks are hard and the design for light clients.

- **Buterin, Vitalik. *Ethereum: A Next-Generation Smart Contract and Decentralized Application Platform* (2013).** The whitepaper that generalized Bitcoin's scripting into a Turing-complete execution environment. Long, discursive, and dated in parts; worth reading alongside the Yellow Paper below.

- **Wood, Gavin. *Ethereum: A Secure Decentralised Generalised Transaction Ledger* (The "Yellow Paper", 2014+).** The formal specification of the EVM. Mathematically dense. Read this when you are debugging an EVM-level question and the normal Solidity docs are not specific enough.

## Books

- **Antonopoulos, Andreas. *Mastering Bitcoin* (2nd edition, O'Reilly, 2017).** The best book-length introduction to Bitcoin at the protocol level. Covers the wire format, the scripting language, keys, addresses, mining — with enough code to actually implement things.

- **Antonopoulos, Andreas, and Gavin Wood. *Mastering Ethereum* (O'Reilly, 2018).** The Ethereum equivalent. Dated on anything post-Merge, but the EVM semantics, Solidity, and smart-contract fundamentals are still accurate.

- **Narayanan, Bonneau, Felten, Miller, Goldfeder. *Bitcoin and Cryptocurrency Technologies* (Princeton, 2016).** More academic in flavor. The companion Coursera course is free and gives you the pseudocode-level intuition for what a Merkle tree, a hash commitment, and a signature scheme actually are. Start here if the cryptography in doc 2 felt hand-wavy.

- **Werbach, Kevin. *The Blockchain and the New Architecture of Trust* (MIT Press, 2018).** A non-technical treatment of the social and legal implications. Useful for conversations with lawyers and regulators.

## Papers worth tracking down

- **Lamport, Shostak, Pease. *The Byzantine Generals Problem* (1982).** The classical distributed-systems paper that defined the problem Bitcoin solves in the permissionless setting. Short, famous, worth reading for context.

- **Dwork, Naor. *Pricing via Processing or Combatting Junk Mail* (1992).** The paper that introduced the proof-of-work concept, originally as an anti-spam mechanism. Pre-dates Bitcoin by 16 years.

- **Back, Adam. *Hashcash: A Denial of Service Counter-Measure* (2002).** Direct ancestor of Bitcoin's proof-of-work. The whitepaper cites it.

- **Chaum, David. *Blind Signatures for Untraceable Payments* (1982).** The first serious attempt at digital cash, predating blockchain by decades. Shows what the pre-blockchain state of the art looked like.

- **Daian et al. *Flash Boys 2.0* (2019).** The paper that named and quantified MEV on Ethereum. Introduces the concepts and the empirical measurements.

## Blogs and long-form online

- **Vitalik Buterin's blog, vitalik.eth.limo.** The most important single source of long-form thinking about Ethereum's evolution. His posts on PoS, rollups, and the "trilemma" are foundational.

- **a16z crypto research, a16zcrypto.com/posts.** Despite the venture-capital source, the technical write-ups (on zero-knowledge proofs, consensus, data availability) are high quality.

- **Paradigm research, paradigm.xyz/writing.** Similar caveat, similar quality. Georgios Konstantopoulos and Dan Robinson in particular.

- **The Ethereum Foundation blog, blog.ethereum.org.** Official announcements and technical deep-dives.

## Reference documentation

- **The Bitcoin wiki, en.bitcoin.it.** Technical reference maintained by the Bitcoin community. Useful for specific protocol details.

- **The Ethereum documentation, ethereum.org/en/developers/docs.** The official developer docs, well-maintained, covers everything from account model to gas to deployment.

- **The Solidity documentation, docs.soliditylang.org.** If you are going to write smart contracts, this is where you live.

- **ethereum.stackexchange.com.** Where you go when a specific Solidity or EVM question has you stuck.

## Tools for actually playing with this

- **A hardware wallet (Ledger, Trezor, Coldcard).** The right way to hold any amount of currency you care about. Everything else is a step down in security.

- **A testnet wallet (MetaMask, Rabby) on Sepolia or Holesky.** Get free test ETH from a faucet, deploy contracts, interact with testnets. Zero real money at risk.

- **Foundry (`forge`, `cast`, `anvil`), getfoundry.sh.** The modern Ethereum development toolchain. Fast, Rust-based, replaces Hardhat for most new projects. See the smart-contracts deep-dive's [tooling and lifecycle](smart-contracts/08-tooling-and-lifecycle.md) doc.

- **Etherscan, etherscan.io.** Block explorer for Ethereum. Lets you read any contract's source (if verified), call any view function, and watch transactions execute in real time. Learn to read Etherscan.

- **bitcoin.org's full node instructions.** Running a Bitcoin full node on a spare machine is a $50–200 one-time cost (an SSD and a Raspberry Pi) and gives you hands-on intuition for what "verifying for yourself" actually means.

## Critical perspectives

A reader should seek out criticism as deliberately as advocacy.

- **Diehl, Stephen, et al. "Crypto Critics Corner" podcast.** Technical critique from engineers rather than cultural critique from commentators.

- **Amy Castor and David Gerard, amycastor.com and davidgerard.co.uk/blockchain.** Journalism tracking scams, rug-pulls, and unmet promises. Corrective to the industry's own narrative.

- **Molly White, web3isgoinggreat.com.** An ongoing catalog of crypto-world failures, scams, and losses.

- **Tim Bray. "On Talking to Crypto Enthusiasts" (2022).** A short, well-argued piece on the asymmetry between what blockchain promises and what it has delivered.

Read these alongside the whitepapers. A field that cannot withstand criticism is not a field worth learning.

## A reading plan if you want to go deep

If you want a serious sequence rather than a reading list:

1. This repo's `docs/01-...` through `docs/08-...` — you're doing it now.
2. This repo's `docs/smart-contracts/` deep-dive — covers everything above Solidity-level in detail.
3. Nakamoto 2008 — read the original.
4. Narayanan et al. textbook — fills in the academic gaps.
5. Buterin 2013 whitepaper + ethereum.org developer docs — move from Bitcoin to Ethereum.
6. Antonopoulos, *Mastering Bitcoin* and *Mastering Ethereum* — book-length depth.
7. Build something small on a testnet. A toy ERC-20, a simple vault contract, anything that forces you to touch gas, signatures, and the deployment loop.
8. Read a post-mortem of a recent major hack. Then re-read it after you have deployed your own contract. The second reading is the one where the lessons stick.

Everything after step 8 is specialization — zero-knowledge proofs, consensus research, MEV, account abstraction, specific chain ecosystems. By that point you can pick your own direction.

> Previous: [Glossary](09-glossary.md) · Next: [Smart contracts in depth](smart-contracts/README.md)
