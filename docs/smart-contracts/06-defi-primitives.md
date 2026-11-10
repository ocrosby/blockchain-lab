# DeFi primitives

> Previous: [Security patterns and famous exploits](05-security-patterns.md) · Next: [Non-EVM smart-contract platforms](07-non-evm-platforms.md)

**DeFi** — decentralized finance — is what most of the smart-contract activity on Ethereum ultimately is. It is not a single application but a *composable stack of primitives*: simple contracts implementing one financial operation, deployed as public infrastructure, callable by anyone and composable with each other. This doc walks the primitives in roughly the order they built on each other historically.

This is not a comprehensive catalog. It is a working knowledge of the recurring patterns, enough to read a new DeFi protocol and recognize what it is doing.

## AMMs: automated market makers

The first and most important primitive. An **automated market maker** is a smart contract that holds reserves of two assets and lets users trade between them at algorithmically-determined prices, with no central order book and no need for a counterparty.

### The constant-product formula

Uniswap v1 (2018) and v2 (2020) introduced the formula:

$$x \cdot y = k$$

Where `x` and `y` are the pool's reserves of the two assets and `k` is a constant maintained across every trade.

If someone adds `Δx` of asset X, the new reserve of Y must satisfy `(x + Δx)(y − Δy) = k`, so they receive:

$$\Delta y = y - \frac{k}{x + \Delta x} = \frac{y \cdot \Delta x}{x + \Delta x}$$

This has some wonderful properties:

- **Always a price.** The pool quotes a price for any size of trade; the price degrades smoothly as the trade size grows relative to the reserves.
- **Deep infinite liquidity at the edges.** You cannot *fully* drain a pool because draining it would require `Δx → ∞`.
- **Automatic rebalancing.** If the price of X rises on external markets, arbitrageurs buy X from the pool until its price matches, which naturally rebalances the reserves.

And some real costs:

- **Price impact.** A large trade moves the pool price significantly, so the trader gets a worse price for larger trades.
- **Impermanent loss.** A liquidity provider who deposits equal values of two assets, held through a price move, ends up with less than if they had simply held the assets separately. The difference is "impermanent loss" (misleading name — the loss is realized if you withdraw).

### The AMM implementation, roughly

```solidity
contract SimpleAMM {
    IERC20 public tokenX;
    IERC20 public tokenY;
    uint256 public reserveX;
    uint256 public reserveY;

    uint256 public constant FEE_BPS = 30; // 0.3%

    function swap(uint256 amountIn, address tokenIn) external returns (uint256 amountOut) {
        bool isX = tokenIn == address(tokenX);
        (uint256 reserveIn, uint256 reserveOut) = isX
            ? (reserveX, reserveY)
            : (reserveY, reserveX);

        uint256 amountInWithFee = amountIn * (10000 - FEE_BPS) / 10000;
        amountOut = (reserveOut * amountInWithFee) / (reserveIn + amountInWithFee);

        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(isX ? address(tokenY) : address(tokenX)).transfer(msg.sender, amountOut);

        if (isX) {
            reserveX += amountIn;
            reserveY -= amountOut;
        } else {
            reserveY += amountIn;
            reserveX -= amountOut;
        }
    }
}
```

That's the core. The fee (0.3% by convention in Uniswap v2) accrues to liquidity providers. The real Uniswap v2 contract is only a few hundred lines; the real innovation is the idea, not the complexity.

### Beyond constant product

- **Uniswap v3 (2021)** introduced **concentrated liquidity**: LPs specify price ranges, which lets capital work harder but requires active management.
- **Curve (2020)** introduced the **stableswap** curve: a hybrid of constant-product and constant-sum, optimized for assets that are supposed to trade at near-equal prices (USDC / USDT, stETH / ETH). Much lower slippage within a tight range.
- **Balancer** generalizes to **weighted pools** with multiple assets and non-50/50 weights.

The AMM is the single most influential DeFi invention. Every token on Ethereum has, somewhere, a liquidity pool that quotes a price.

## Lending markets

The second major primitive: a contract that holds deposits of various assets, pays interest to depositors, and makes over-collateralized loans to borrowers.

### The over-collateralization requirement

Lending on Ethereum is almost universally **over-collateralized**. Alice deposits $150 of ETH as collateral to borrow $100 of USDC. The contract will not lend her $100 of USDC unless she has already escrowed $150 of value.

This sounds pointless — why would anyone borrow $100 when they already have $150? Several real reasons:

- Alice wants USDC for an immediate purchase but does not want to sell her ETH (which she believes will appreciate, or on which she doesn't want to realize taxes).
- Alice wants to lever a bullish position: borrow USDC, buy more ETH, now she has $150 ETH collateral + $100 ETH position — effectively $250 of ETH exposure for $150 of capital.
- Alice wants to short USDC relative to ETH, implicitly, by borrowing it.

Over-collateralization also makes the system work *without identity*. There is no credit check; the collateral *is* the creditworthiness. If Alice's collateral falls below the required ratio (the price of ETH drops), the contract **liquidates** her position — sells her collateral to repay her loan, plus a penalty that goes to the liquidator who triggered it.

### How a lending market works

Each supported asset has:

- A pool of deposited funds (liquidity).
- A borrow book, tracking who has borrowed how much.
- An **interest rate** that varies with utilization (utilization = borrowed / deposited). High utilization → high rates (attracts more deposits, discourages more borrows).
- A **collateral factor**: for each asset, the fraction of its value that can be borrowed against. ETH might have 75% (borrow up to 75% of its value); more volatile assets have lower factors.
- A **liquidation threshold** and **liquidation bonus**.

The two largest implementations are **Aave** and **Compound**. Both are open-source, heavily audited, and have handled billions in TVL for years. Each has specific innovations (Aave's flash loans, Compound's governance model, each's isolation modes for risky assets), but the shape above describes both.

### Flash loans

Compound and Aave both offer **flash loans**: a contract can borrow any amount of any asset for the duration of a single transaction, with no collateral, as long as the loan is repaid (plus a small fee) before the transaction ends. If the loan is not repaid, the transaction reverts, undoing the borrow.

Flash loans exist because the atomicity of EVM execution makes them safe for the lender: the lender cannot lose money, regardless of what the borrower does, because non-repayment reverts the whole transaction.

The intended use case: arbitrage. If two DEXes have inconsistent prices, a trader can flash-borrow millions, buy on one, sell on the other, repay the loan, and keep the difference. In practice, flash loans have also been the weapon of choice for many DeFi exploits — if a protocol can be drained by someone with temporarily infinite capital, it will be.

## Stablecoins

A **stablecoin** is a token designed to track the price of a stable external asset (almost always the US dollar). Several designs:

### Fiat-collateralized (centralized)

**USDC** (Circle), **USDT** (Tether), **BUSD** (previously Paxos). Each token is backed by $1 of real-world assets (cash, Treasuries) held by a centralized custodian. Users can redeem tokens for real dollars through the issuer.

Tradeoff: these are the most reliable pegs but reintroduce the trusted-institution problem. The issuer can freeze specific addresses, and has — USDC has frozen Tornado-Cash-linked addresses, USDT has frozen addresses sanctioned by various jurisdictions. Your "decentralized" DeFi position in USDC is subject to Circle's compliance policies.

### Crypto-collateralized

**DAI** (MakerDAO) and its successors. Users deposit ETH or other crypto assets into a **vault** and mint DAI against them, over-collateralized (currently 150%+). If the collateral value falls below the threshold, the vault is liquidated.

Tradeoff: fully decentralized (modulo governance centralization) but vulnerable to collateral crashes. During the March 2020 crypto crash, several MakerDAO vaults were liquidated for zero DAI due to a congested network preventing timely liquidations — Maker's own shareholders had to absorb the shortfall.

### Algorithmic (mostly failed)

**Terra/LUNA** (collapsed 2022, ~$40B wiped out), **Basis Cash**, various others. Attempt to maintain the peg via on-chain mint-and-burn mechanics: when the stablecoin trades above $1, allow minting more; when below, allow burning for a sibling token that captures the arbitrage. The math works in steady state and fails under stress.

The Terra collapse was spectacular and instructive. LUNA was the sibling token that absorbed UST's downward pressure. When UST briefly depegged, LUNA was minted to defend it; this diluted LUNA's value, which further undermined UST, which triggered more LUNA minting. The feedback loop unwound both tokens to near-zero in days. Algorithmic stablecoins have not recovered as a category.

### Delta-neutral

**Ethena's USDe** and similar newer designs. Hold ETH long, short ETH perpetual futures of equal size, net zero price exposure. The peg is maintained by the perpetual-futures funding rate (positive in bull markets, negative in bear markets).

Still young as a category. Depends on perpetual-futures markets continuing to function; "it's always worked so far" is a low standard in finance.

## Oracles

Covered in the security doc, but a primitive in their own right. An oracle is a contract that reports an external value (asset price, sports score, weather) on-chain. The problem it solves: smart contracts cannot perform I/O, cannot call APIs, cannot read anything off-chain. So anyone needing off-chain data must get it through a dedicated contract that external parties push data into.

### Chainlink: the dominant oracle network

Chainlink's price oracles work by having a decentralized network of operators query multiple price sources, aggregate them, and push a median to on-chain feeds at a defined cadence. Protocols read the feed; Chainlink's operators are compensated in LINK tokens.

The trust model is important: Chainlink users are trusting (1) that the Chainlink network does not collude, (2) that the published median is a reasonable price. Both have generally held but are not mathematical certainties.

### TWAPs

A **time-weighted average price** is a rolling average of the on-chain DEX price over a window (an hour, a day). More expensive to manipulate than spot price — an attacker would have to sustain a false price over the window, which costs the arbitrage profit over the same window. Uniswap v2 and v3 both expose TWAP oracles.

### Oracle choice matters

Every lending protocol, synthetic asset, options protocol, and liquidation engine depends on an oracle. The oracle's choice is one of the most consequential design decisions a DeFi protocol makes. An under-collateralized oracle is a $100M vulnerability waiting to happen.

## Derivatives

The newer frontier. Smart contracts implementing options, futures, and other derivatives.

- **Perpetual futures** (dYdX, GMX, Hyperliquid) — leveraged long/short exposure with no expiration, stabilized by funding payments between longs and shorts.
- **Options protocols** (Lyra, Opyn) — on-chain pricing and settlement of calls and puts.
- **Yield derivatives** (Pendle) — split yield-bearing tokens into a principal token and a yield token, trade them separately.

These sit higher in the stack — they usually use AMMs for liquidity, lending markets for leverage, oracles for pricing. The composability is the point.

## Aggregators

A separate category: contracts that route a trade through multiple other protocols to find the best price.

- **1inch, Paraswap, 0x, Odos** — DEX aggregators.
- **Yearn** — yield aggregator; routes deposits to the highest-yielding opportunity across lending markets and AMMs.

An aggregator does not hold deep liquidity itself; it is a pathfinder over the ecosystem's liquidity. The aggregator business is highly competitive and gas-sensitive — every saved opcode is a saved dollar for users.

## Composability in practice

Here is a real example of the stack in use — a yield-leveraged position:

1. User deposits ETH into Lido, receives stETH (liquid staking token that earns ~3% APR).
2. User deposits stETH into Aave as collateral.
3. User borrows USDC against stETH.
4. User swaps USDC back to ETH via a DEX aggregator.
5. User deposits that ETH into Lido, repeats.

Each step is a call to an external protocol. All of it happens on-chain, trust-minimized. In practice, "looping" protocols automate this for users with a single transaction. The entire pipeline composes from primitives that were each built independently; no coordinated effort was required.

This is the property that makes DeFi novel — it is not any single primitive that is new, but the way they compose.

## What to carry forward

- **AMMs** (Uniswap, Curve, Balancer) are the single most important DeFi primitive. The constant-product formula is only three lines of math, and every token on Ethereum has some pool quoting it.
- **Lending markets** (Aave, Compound) hold deposits, make over-collateralized loans, pay variable interest based on utilization. **Flash loans** are a special case that works because of EVM atomicity.
- **Stablecoins** come in flavors: fiat-backed (reliable, centralized), crypto-collateralized (decentralized, vulnerable to collateral crashes), algorithmic (mostly failed), delta-neutral (new).
- **Oracles** are how the on-chain world learns off-chain facts. Chainlink dominates; TWAPs are a flash-loan-resistant alternative.
- **Derivatives** and **aggregators** sit higher in the stack, composing from the primitives above.
- **Composability** is the thing. No single primitive is novel; the way they compose into complex atomic transactions is.

Next: how different (non-EVM) chains approach these problems.

> Previous: [Security patterns and famous exploits](05-security-patterns.md) · Next: [Non-EVM smart-contract platforms](07-non-evm-platforms.md)
