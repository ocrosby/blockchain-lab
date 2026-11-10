# Smart contracts

> Previous: [Mining and incentives](06-mining-and-incentives.md) · Next: [Limitations and honest tradeoffs](08-limitations.md)

This doc is the main-sequence introduction to smart contracts. If after reading it you want to go deeper — the EVM in detail, Solidity, token standards, DeFi primitives, the security patterns behind the famous exploits, non-EVM platforms, and the daily workflow — there is an eight-doc deep-dive in [`docs/smart-contracts/`](smart-contracts/README.md).

Bitcoin's blockchain stores a specific kind of thing: value transfers. Ethereum's does something more general. It stores **programs** — called **smart contracts** — that run every time they are called, deterministically, on every node in the network, as part of block execution. The programs can hold currency, respond to inputs, call other programs, and keep internal state. The chain becomes less of a ledger and more of a distributed, shared computer.

The name "smart contract" is older than Ethereum — Nick Szabo coined it in the 1990s — and is slightly misleading. These programs are not necessarily smart (they can be bug-ridden and often are) and they are not contracts in the legal sense. A better name would be **shared-state programs**, but "smart contract" stuck.

## What runs where

A smart contract is deployed by sending a special transaction containing its **bytecode**. The chain assigns the new contract an address; the bytecode and initial storage are saved as part of the state. From then on, any transaction sent to that address triggers the bytecode, which:

- Can read and modify the contract's own storage.
- Can receive and send the native currency (ETH).
- Can call other contracts, synchronously, as part of the same transaction.
- Returns a value, or reverts the entire transaction (undoing every state change it made).

Every full node on Ethereum executes the bytecode independently and must arrive at the same result — otherwise consensus breaks. This is the central constraint on smart-contract design: *every operation must be deterministic*. There is no `Math.random()` without a provided seed, no network I/O, no filesystem, no clock beyond the current block's timestamp. The program's input is the transaction and the current state; the output must be a function of those alone.

The execution environment Ethereum uses is called the **Ethereum Virtual Machine (EVM)** — a stack-based bytecode interpreter with 256-bit words. Most contracts are written in a higher-level language (**Solidity** is dominant; **Vyper** is a Python-flavored alternative) that compiles down to EVM bytecode.

## The gas mechanism

If any contract can call any other contract, which can call any other contract, what stops a malicious or buggy contract from running an infinite loop and freezing the network?

**Gas.** Every EVM instruction costs a specific amount of gas. The sender of a transaction specifies a `gasLimit` — the maximum they are willing to spend — and a `gasPrice`. The transaction executes until it completes, reverts, or runs out of gas, whichever comes first. If it runs out of gas, every state change is reverted *but the sender still pays for the gas consumed*. This creates a hard upper bound on execution time per transaction and makes denial-of-service attacks expensive rather than impossible.

Gas is also why Ethereum fees spike when the network is busy: everyone bids for the same scarce resource (the gas available per block), and a congested chain produces an auction.

## The canonical example: ERC-20 tokens

The most common smart contract on Ethereum is a few hundred lines of Solidity implementing the **ERC-20** standard — a simple interface for a fungible token. It has roughly this state:

```solidity
contract Token {
    string public name;
    string public symbol;
    uint8  public decimals;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    function transfer(address to, uint256 amount) public returns (bool) {
        require(balanceOf[msg.sender] >= amount, "insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    // approve, transferFrom, allowance...
}
```

That's it. A contract like this creates a new fungible token on Ethereum. Deploy it, set an initial balance for yourself, and the token now exists — you can send it, exchange it, lend it, use it as collateral, build other contracts that interact with it. A substantial fraction of the activity on Ethereum is interaction with these kinds of tokens.

**USDC, USDT, DAI, WBTC** — most of the dollars and dollar-pegged assets on Ethereum are ERC-20 contracts holding a `balanceOf` mapping. The chain enforces the accounting; the contract's code *is* the asset's logic.

## Beyond fungible tokens

The programmable nature of the EVM means the surface area is vast. Some recognizable categories:

- **Decentralized exchanges (DEXes)** — Uniswap, Curve. Smart contracts that hold pools of two assets and let users swap between them at algorithmic prices. The infamous "x*y=k" constant-product formula is about three lines of Solidity.
- **Lending and borrowing** — Aave, Compound. Contracts that hold deposits, pay interest, make over-collateralized loans, liquidate positions when collateral falls.
- **NFTs (non-fungible tokens)** — ERC-721 and ERC-1155 standards. Each token is unique (not interchangeable like ERC-20) and typically carries a reference to off-chain media.
- **DAOs (decentralized autonomous organizations)** — governance contracts that let token holders vote on proposals to control treasury, parameters, upgrades.
- **Bridges** — contracts that lock an asset on one chain and mint a representation on another, letting value flow between chains.
- **Oracles** — Chainlink and others. Contracts that bring off-chain data (prices, sports scores, weather) on-chain in a trust-minimized way.

The composability of these pieces is a genuinely novel property. Any contract can be called by any other contract, atomically, in a single transaction. This means a sophisticated financial operation — borrow from one protocol, use the proceeds to swap on a DEX, deposit the result as collateral somewhere else — can be executed as a single atomic unit that either all succeeds or all reverts. This is sometimes called "money legos" and is the backbone of what gets called **DeFi (decentralized finance)**.

## Where smart contracts bite back

Smart contracts are software. Software has bugs. Smart contracts run with billions of dollars of real value and no undo button. This combination has produced some of the most expensive bugs in software history:

- **The DAO hack (2016)** — a reentrancy bug in a crowdfunded investment contract allowed an attacker to withdraw funds repeatedly before the contract updated its internal balance. ~3.6 million ETH (worth ~$50 million at the time, ~$10 billion today) was drained. Ethereum hard-forked to reverse the theft, splitting the chain into Ethereum and Ethereum Classic.
- **Parity multisig freeze (2017)** — a library contract was accidentally made self-destructable, and someone destroyed it, freezing ~500,000 ETH in wallets that depended on it. Not stolen — just permanently inaccessible.
- **Ronin Bridge (2022)** — $625 million drained after an attacker compromised five of nine validator keys on a cross-chain bridge.
- **Countless rug pulls** — contracts deployed by anonymous teams, token sold to the public, the deployer's own hidden backdoor used to drain the pool. Basically weekly.

The patterns here are instructive. Smart-contract bugs are not usually in the Solidity compiler or the EVM — they are in the application logic, where the developer's mental model of the contract's state machine was wrong in some subtle way. **Reentrancy** (calling back into a contract before it finishes updating state) is the most famous class. **Integer overflow** was a common issue before Solidity made overflow checks the default. **Access control** (who can call what) is a recurring source of multi-million-dollar mistakes.

The ecosystem has responded with:

- **Formal verification** — mathematically proving properties of a contract's behavior.
- **Audits** — third-party review before deployment.
- **Bug bounty programs** — paying attackers more to report bugs than exploit them.
- **Standardized, battle-tested libraries** — OpenZeppelin is the canonical set of audited ERC-20/721/governance implementations.

None of this makes smart contracts safe in the way a transaction on a conventional database is safe. The code *is* the contract, and if the code is wrong, the loss is real and typically unrecoverable.

## "Code is law" — and the limits of that slogan

An early slogan of the smart-contract world was **code is law**: whatever the contract does is what happened, because the contract's logic is the final arbiter of the outcome. If you got rug-pulled by a contract whose code was always on-chain and public, it is on you for not reading the code.

This is only partially true in practice. The DAO hard fork is the clearest counterexample — the Ethereum community collectively decided that the contract's behavior was *not* what should have happened, and rewrote history to reverse it. "Code is law" held until the social layer intervened.

More broadly: real-world legal systems, user-interface wallets, exchange operators, and off-chain reputation all affect how contracts function. A contract that lets its deployer mint infinite tokens may be technically correct but functionally a scam, and will be delisted from exchanges and labeled as such regardless of what the code says. The maximalist "code is law" position overstates the autonomy of the code from the human systems around it.

## What to carry forward

- A smart contract is a program stored on the chain. Every node executes it deterministically; every node must get the same result.
- **Gas** bounds execution cost and prevents infinite-loop denial of service. Senders pay for the gas consumed even on failure.
- **ERC-20** is the canonical fungible-token contract; most dollar-pegged assets on Ethereum are implementations of it.
- Composability — any contract calling any other atomically — is the property that makes "DeFi" an ecosystem rather than isolated apps.
- Smart contracts hold billions of dollars with no undo; bugs are expensive and common. The ecosystem has evolved audits, formal verification, and battle-tested libraries, but nothing makes contracts fully safe.
- "Code is law" is a slogan, not a complete description. The social layer around contracts remains important.

**Want to go deeper?** This is the main-sequence intro only. The eight-doc [smart-contracts deep-dive](smart-contracts/README.md) covers history and conceptual roots, the EVM in detail, Solidity, every major token standard (ERC-20/721/1155/4626/4337), security patterns with famous exploit case studies, DeFi primitives (AMMs, lending, stablecoins), non-EVM platforms (Solana, CosmWasm, Move), and the daily development workflow.

Next: an honest accounting of what blockchain is bad at.

> Previous: [Mining and incentives](06-mining-and-incentives.md) · Next: [Limitations and honest tradeoffs](08-limitations.md)
