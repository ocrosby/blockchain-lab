# Security patterns and famous exploits

> Previous: [Token standards](04-token-standards.md) · Next: [DeFi primitives](06-defi-primitives.md)

Smart contracts control real money, run forever, and cannot be fixed after deployment without careful upgrade mechanisms. The combination guarantees that any bug class with real consequences will eventually produce a real exploit with real losses. This doc is a tour of the most important ones, the patterns that have evolved to prevent them, and the famous incidents that made each pattern necessary.

Learning this material by reading alone is not enough — you must read actual contracts and actual post-mortems alongside it. Treat this doc as an index.

## The threat model

Before cataloging specific bugs, name the attackers:

- **The MEV searcher**, watching every pending transaction and willing to front-run, sandwich, or back-run yours for profit. Assume sophisticated, well-funded, operating in real time.
- **The flash-loan attacker**, who can borrow millions of dollars of any asset for the duration of a single transaction with no collateral, as long as they return it before the transaction ends. Any economic model that could be gamed with "what if you temporarily had infinite capital" must consider them.
- **The slow attacker**, running static analyzers and fuzzing tools over verified contracts looking for vulnerabilities to exploit.
- **The social engineer**, phishing users into signing malicious transactions, airdrop scams, fake interfaces. The contract itself might be perfectly safe; the user is the attack surface.
- **The insider**, including the project's own team if they have unrestricted admin keys or back-door upgradability.

Any production contract must survive all five. That is a very high bar.

## Reentrancy

The famous one. Named by the DAO hack in 2016 but still showing up every year.

### The pattern

A contract calls out to an external address (sending ETH, calling a token's transfer, calling an unknown contract) **before** finishing its own state updates. The external address calls back into the original contract, which — because its state is still mid-update — allows the attacker to repeat whatever action they were doing.

```solidity
// Vulnerable pattern
function withdraw() external {
    uint256 balance = balances[msg.sender];
    (bool success,) = msg.sender.call{value: balance}("");
    require(success);
    balances[msg.sender] = 0;  // Updated AFTER the call
}
```

If `msg.sender` is a contract with a `receive()` that calls back into `withdraw()`, the second invocation sees the still-full balance, withdraws it again, calls back once more. Repeat until the contract is drained.

### The fix

**Checks-Effects-Interactions.** The sequence inside every function that calls external code:

1. **Checks** — validate all preconditions.
2. **Effects** — update all state.
3. **Interactions** — call external contracts.

```solidity
// Safe pattern
function withdraw() external {
    uint256 balance = balances[msg.sender];
    require(balance > 0, "no balance");
    balances[msg.sender] = 0;                             // Effects
    (bool success,) = msg.sender.call{value: balance}(""); // Interactions
    require(success);
}
```

Belt-and-suspenders: a **reentrancy guard**, a mutex that prevents the function from being called while it is still executing.

```solidity
uint256 private _locked = 1;

modifier nonReentrant() {
    require(_locked == 1, "reentrant");
    _locked = 2;
    _;
    _locked = 1;
}

function withdraw() external nonReentrant { ... }
```

OpenZeppelin ships `ReentrancyGuard` for exactly this; use it.

### Read-only reentrancy

A subtler variant: a getter view function is called *during* another contract's reentrancy window and returns stale, inconsistent state. If a lending protocol's price oracle queries a DEX while that DEX is mid-update, it can read an inconsistent price. Several exploits have used this pattern.

The mitigation: design view functions to either be free of intermediate states or use reentrancy guards that also protect reads.

### Famous cases

- **The DAO** (2016) — the original. 3.6M ETH drained. Led to the Ethereum/Ethereum Classic fork.
- **Cream Finance** (2021) — $130M via read-only reentrancy in integration with a flash-mint token.
- **Rari Capital** (2022) — $80M via reentrancy in a `Fuse` lending pool.
- **Curve Finance** (2023) — $70M across several pools via a compiler bug in Vyper that broke reentrancy guards.

## Access control

The second-most-common vulnerability category: functions that should only be callable by the owner or admin are callable by anyone.

### The patterns

Standard access-control implementations:

- **`Ownable`** — a single `owner` address with an `onlyOwner` modifier.
- **`AccessControl`** — role-based (OpenZeppelin), with `DEFAULT_ADMIN_ROLE` and arbitrary user-defined roles.
- **Timelock** — a contract that executes privileged operations only after a delay, giving users a chance to withdraw if the action is malicious.
- **Multisig** — the admin "address" is itself a contract that requires M of N signatures to execute.

### Common failures

- **Initialization race.** An initializer function (`function initialize() external` on a proxy contract) that anyone can call. Whoever calls it first becomes the owner. The 2017 Parity Multisig incidents (both of them) were caused by this.
- **Public internal functions.** Marking a function `public` when it should be `internal`, exposing a privileged helper.
- **Default visibility trap** (pre-0.5). In old Solidity, functions without a visibility specifier defaulted to `public`. Countless contracts leaked privileged functions this way before the default was removed.
- **tx.origin instead of msg.sender.** `tx.origin` is the originating EOA of a transaction; `msg.sender` is the immediate caller. Using `require(tx.origin == owner)` lets any contract that the owner calls act as the owner — devastating because phishing attacks can induce the owner to call a malicious contract.

### Famous cases

- **Parity Multisig 1** (2017) — $30M stolen when an `initWallet` function was left callable.
- **Parity Multisig 2** (2017) — $280M+ frozen (not stolen) when a library contract's `kill` function was callable by anyone, destroying every wallet that depended on it.
- **Audius** (2022) — $6M via an initialization vulnerability on a governance contract.
- **Innumerable "rug pulls"** — developers retain backdoor admin keys, dump the pool, disappear.

## Integer overflow and underflow

Pre-Solidity-0.8, all arithmetic was modular. `uint256 x = 0; x -= 1;` set `x` to `2^256 - 1`. Likewise `x + y` could wrap around if the sum exceeded `2^256 - 1`.

This was the source of massive early exploits. The **BatchOverflow** vulnerability (2018) allowed attackers to mint trillions of tokens from vulnerable ERC-20 contracts by crafting inputs whose product overflowed a check.

**Solidity 0.8 made overflow revert by default.** For contracts written after late 2020, this is no longer a concern unless they are using `unchecked { ... }` blocks for gas optimization. If you use `unchecked`, you are explicitly promising that the math inside cannot overflow; a review should scrutinize that promise.

Pre-0.8 contracts commonly used OpenZeppelin's `SafeMath` library. If you are reading older code, `a.add(b)` is their explicit overflow check.

### Rounding and precision

Even without overflow, division is tricky. Solidity has no floating point. Division truncates toward zero. So `(a * b) / c` and `(a / c) * b` give different answers when `a / c` is not exact. Standard protocol math multiplies first and divides last, to retain precision — but the order matters, and getting it wrong can produce bugs where fractions of a cent (or fractions of a thousand dollars, in badly-written contracts) silently leak.

**Inflation attack** on ERC-4626 (covered in the token-standards doc) is a rounding attack.

## Oracle manipulation and flash-loan attacks

Many DeFi protocols need to know an asset's price. The cheapest source is to read the spot price from a decentralized exchange — but that price is manipulable within a single transaction. Combined with **flash loans**, this creates a powerful attack:

1. Attacker takes a $100M flash loan of asset X.
2. Attacker dumps X into a DEX pool used as the price oracle for lending protocol P.
3. X's price on that DEX crashes temporarily.
4. P's price oracle reads the manipulated price. Attacker can now borrow against an asset at a wildly favorable (incorrect) collateral ratio, drain P.
5. Attacker buys X back from the DEX cheaply, pays off the flash loan.

Entire transaction, start to finish, atomic. If any step fails, everything reverts. If every step succeeds, the attacker walks away with millions.

### The fix

- **Never use spot DEX prices as the sole oracle.** Use time-weighted average prices (TWAPs), which cost real capital to manipulate over time.
- **Use dedicated oracle services.** Chainlink aggregates prices from multiple sources with its own update cadence. Not manipulable within a single transaction.
- **Reality checks.** If the price your oracle is telling you is 50% off the external market, something is wrong — reject the price or at least pause the protocol.

### Famous cases

- **bZx** (2020) — multiple price-oracle exploits totaling $8M, the first widely-covered flash-loan attacks.
- **Mango Markets** (2022) — $114M by manipulating MNGO's price on-chain via large trades, then taking out loans against the inflated collateral.
- **Beanstalk** (2022) — $182M via a governance attack combined with a flash loan to acquire voting power for a single block.
- **Many smaller protocols** — a steady stream throughout 2020–2024.

## Front-running and MEV

Every pending transaction is visible in the public mempool before inclusion. Anyone can:

- **Front-run.** See a profitable trade coming, submit the same trade first with higher gas, capture the price movement.
- **Sandwich.** Place a buy before a large pending buy, and a sell after, capturing the price impact.
- **Back-run.** See a transaction that creates an arbitrage opportunity (an oracle update, a large trade), race to capture it.

### Mitigations

- **Commit-reveal schemes** for actions that must be private before execution.
- **Private mempools** (Flashbots, now MEV-Share) — submit transactions directly to block builders without appearing publicly.
- **Slippage protection** — every DEX trade accepts a `minOutput` parameter; if the actual output is below it, the trade reverts. This prevents sandwich attacks from being arbitrarily bad, but does not eliminate them.
- **Batch auctions** — some DEXes (CoW Swap) aggregate orders into batches cleared at a single price, removing ordering MEV at the protocol level.

MEV is not a bug — it is an emergent consequence of block producers having both ordering power and transaction visibility. The ecosystem has evolved layers (proposer-builder separation, private relays) that mitigate the worst forms but cannot eliminate it.

## Signature replay

A signed message authorizes something. If that message does not include a nonce, a chain ID, or a context identifier, the same signature can be replayed later or on a different chain.

- **Replay on same chain**: Attacker captures a signed message authorizing a one-time action. Submits it again. The protocol accepts the same signature again.
- **Cross-chain replay**: Alice signs a "withdraw 100 USDC" message intended for Ethereum. Attacker submits the same signature on Polygon, where Alice also has funds.

### The fix

EIP-712 structured signing: every signed message includes a domain separator containing the chain ID, the contract address, and the contract name. Signatures are only valid in their intended context. Every production meta-transaction protocol uses EIP-712.

### Famous cases

- **Wormhole** (2022) — $325M via a signature-verification bug in the cross-chain bridge.
- **Ronin Bridge** (2022) — $625M via compromised validator keys (not exactly signature replay, but same category — unauthorized "valid" signatures).

## Delegatecall to untrusted code

Covered in the EVM doc, worth restating: a `DELEGATECALL` executes the target's code with the caller's storage. If the target is untrusted or upgradable, the caller's state is fully at the target's mercy.

The Parity Multisig freeze happened because a library contract was `selfdestruct`ed, and every wallet that delegatecalled into it suddenly had no code to borrow — any ETH held by those wallets became permanently unreachable.

### The fix

- Only delegatecall into contracts whose code you fully control and whose upgrade path is auditable.
- Proxy patterns (Transparent, UUPS) carefully isolate the delegatecall to an implementation contract managed by well-reviewed upgrade logic.

## Governance and upgrade attacks

Many protocols have a governance mechanism (token-weighted voting) or an admin role (multisig, timelock, team-held) that can change parameters, upgrade logic, or migrate funds. Each of these is an attack surface.

- **Vote-buying.** Buy enough governance tokens to pass a malicious proposal.
- **Flash-loan governance.** Borrow enough tokens for a single block to pass a proposal immediately, if the governance system allows single-block proposals.
- **Compromised admin keys.** Phishing or social engineering the team.
- **Malicious upgrades.** The admin has the power to replace the implementation contract. If the admin is compromised, every user's funds are at risk.

### The mitigations

- **Timelocks on upgrades.** Any upgrade proposal must wait N days before execution. Users can withdraw in the meantime.
- **Multisig admin, not single-key.** M of N signatures required.
- **Transparent governance processes.** Public proposals, public discussion, monitored by watchdogs.
- **Snapshot-based voting** that uses historical block state to prevent flash-loan governance.

### Famous cases

- **Beanstalk** (above) — single-block governance proposal with flash-loaned voting power.
- **Ronin Bridge** (above) — admin key compromise.
- **Compound** (2021) — a buggy governance upgrade misallocated $90M. Not an attack; a self-inflicted error. The team recovered most of it by publicly asking recipients to return the funds.

## The patterns, summarized

| Pattern | Defends against |
|---|---|
| Checks-Effects-Interactions | Reentrancy |
| `ReentrancyGuard` modifier | Reentrancy (belt-and-suspenders) |
| OpenZeppelin `AccessControl` / `Ownable` | Access-control bugs |
| Timelocks | Malicious admin upgrades |
| Multisig admin | Single-key compromise |
| `SafeERC20` library | Non-standard ERC-20 return conventions |
| EIP-712 typed signing | Signature replay |
| TWAPs and dedicated oracles | Flash-loan price manipulation |
| Slippage protection | Sandwich attacks |
| OpenZeppelin audited contracts | Rolling your own buggy implementation |
| Formal verification | Entire bug classes at the specification level |

## The painful meta-lesson

Looking at the exploit history, a pattern emerges: nearly every major exploit was *preventable with known patterns*. Reentrancy has been the textbook example since the DAO, and reentrancy exploits still happen every year. Admin-key centralization has been criticized for a decade, and bridge hacks via admin-key compromise still drain hundreds of millions.

The problem is not that the patterns are unknown. The problem is that deploying a smart contract is *easy*, security review is *expensive*, and the incentives often favor ship-fast over ship-safe. Combined with the irreversibility of deployed contracts, the industry has an accumulated backlog of known-bad patterns that keep showing up.

The strongest single move for any new project: **use OpenZeppelin's audited contracts for every standard pattern you can, get a credible third-party audit before any serious TVL, and use a timelock for all admin actions.** This alone would have prevented a large majority of historical exploits.

## What to carry forward

- **Reentrancy** is the most famous bug class. Checks-Effects-Interactions plus `ReentrancyGuard` prevents the classic form. Read-only reentrancy is subtler and still common.
- **Access control** errors are the second-most-common category. Single-key admins and uncalled initializers are both recipes for disaster.
- **Oracle manipulation** combined with **flash loans** is the signature DeFi attack of the 2020s. Use TWAPs or dedicated oracles; never spot DEX prices.
- **MEV** is not a bug but an adversarial dynamic you must design around.
- **Signature replay** is prevented by EIP-712 typed signing with a domain separator.
- **Delegatecall** is powerful and dangerous. Only delegatecall into code you fully control.
- **Admin keys and governance** are attack surfaces. Timelocks, multisigs, and watchdog processes mitigate.
- Nearly every major exploit was preventable with known patterns. Use OpenZeppelin, get audits, use timelocks.

Next: the DeFi primitives that these security considerations exist to protect.

> Previous: [Token standards](04-token-standards.md) · Next: [DeFi primitives](06-defi-primitives.md)
