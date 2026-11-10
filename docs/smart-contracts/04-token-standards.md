# Token standards

> Previous: [Solidity language tour](03-solidity-language.md) · Next: [Security patterns and famous exploits](05-security-patterns.md)

Most activity on Ethereum — by transaction count, by value, by developer attention — involves tokens of some kind. A **token** in this context is a smart contract that tracks ownership of some asset (fungible currency, unique collectible, fractional share, position in another protocol). The Ethereum ecosystem has converged on a small set of **standards** — well-specified interfaces that any contract can implement so that wallets, exchanges, and other contracts can interact with them uniformly.

This doc covers the five standards you need to know: **ERC-20**, **ERC-721**, **ERC-1155**, **ERC-4626**, and **ERC-4337** (account abstraction, strictly not a token standard but important enough to include).

## ERC-20: fungible tokens

The original and most important. ERC-20 defines a minimal interface for a fungible token — a token where one unit is interchangeable with any other unit (like dollars, grams of gold, or Bitcoin).

### The interface

```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

Six functions, two events. That is the entire specification.

### The allowance mechanism

The interesting pattern is `approve` / `transferFrom`. Direct `transfer` lets Alice send tokens to Bob. But what if Alice wants to *authorize* Bob (or a contract) to pull tokens from her, on her terms?

The flow:

1. Alice calls `approve(exchangeContract, 1000)`. The ERC-20 records `allowance[alice][exchangeContract] = 1000`.
2. Alice interacts with the exchange contract, which internally calls `token.transferFrom(alice, exchange, 300)`.
3. The ERC-20 checks the allowance (1000 >= 300), decrements it to 700, and moves 300 tokens.

This two-step pattern is what makes DEXes, lending markets, and nearly every interactive DeFi protocol possible. It lets a contract move your tokens *only up to a limit you set*, without you needing to send the tokens into the contract first.

### The ERC-20 footguns

Three well-known traps:

1. **Return-value inconsistency.** The spec says `transfer` returns `bool`. Many early tokens — notably **USDT** (Tether) — do not return anything. A contract written to assume a return value will fail to compile a call against them, or will succeed but then fail at runtime when the call "returns" garbage. The idiomatic solution is OpenZeppelin's `SafeERC20` library, which uses low-level calls and handles both conventions.

2. **The approval race condition.** The original `approve` function allows setting a non-zero allowance to a new non-zero value. This creates a race: if Alice approves 100, and Bob (a malicious spender) notices, Bob can transferFrom 100 before Alice's new "approve 50" transaction lands, then transferFrom another 50 after — total 150, more than Alice ever intended to allow. The workaround is to set approval to zero first, then to the new amount; or use `increaseAllowance` / `decreaseAllowance` where available.

3. **Fee-on-transfer and rebasing tokens.** Some tokens take a fee on every transfer (reducing the amount received below the amount sent), and some rebase balances based on external triggers. Code that assumes `transfer(to, 100)` results in `+100` to the recipient is wrong for these tokens. Protocols that integrate ERC-20s defensively measure the recipient balance before and after, and use the difference.

### The decimals convention

ERC-20 has an optional `decimals()` function. By convention, a balance of `1000000` for a token with 6 decimals represents 1.0 units of the token. **USDC uses 6 decimals; most tokens use 18** (matching ETH's own smallest-unit convention of "wei").

Nothing about this is enforced on-chain. Decimals exist entirely for front-ends to format numbers. A contract comparing balances across tokens with different decimals must normalize — forgetting this has been a source of bugs.

## ERC-721: non-fungible tokens (NFTs)

ERC-721 defines a token where each unit is *unique* and not interchangeable with any other unit. Each token is identified by a `tokenId` (a 256-bit integer), and each `tokenId` has at most one owner.

### The interface

```solidity
interface IERC721 {
    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);

    function transferFrom(address from, address to, uint256 tokenId) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function isApprovedForAll(address owner, address operator) external view returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
}
```

The shape mirrors ERC-20 but tokens are identified by `tokenId`, not by amount. `balanceOf` returns the count of tokens owned by an address, which is less useful than ERC-20's balance because you still need to enumerate `tokenId`s separately.

### `safeTransferFrom` and ERC-721 receiver

`safeTransferFrom` checks that the recipient is either an externally-owned account (an address with no code) or a contract that implements `onERC721Received`. This prevents NFTs from being locked by sending them to contracts that have no idea what to do with them.

Any contract designed to hold NFTs should implement:

```solidity
function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data)
    external returns (bytes4)
{
    return this.onERC721Received.selector;
}
```

Returning the magic selector signals "yes, I know about NFTs, please send it to me."

### Metadata and off-chain assets

ERC-721 includes an extension interface with `tokenURI(uint256 tokenId)` returning a URI. By convention, this URI points to a JSON document describing the token: name, description, image URL, attributes.

**The actual media (image, video, 3D model) is almost always off-chain.** Common storage: IPFS (content-addressed, decentralized-ish), Arweave (pay-once, permanent), or regular HTTP servers (vulnerable to the operator's whim). The NFT on-chain is a *pointer* to the media; whether the media persists is a separate question.

This has produced the uncomfortable reality that many NFTs are pointers to media on servers that eventually go offline. "Owning the NFT" and "being guaranteed access to the media" are not the same thing, and the gap is a real one.

### ERC-721 enumerable extension

An optional `IERC721Enumerable` extension adds functions to list all tokens and all tokens owned by an address. It is not required and has gas-cost tradeoffs — maintaining the enumeration requires extra storage writes on every mint/transfer. Most large NFT collections omit it and rely on off-chain indexing for enumeration.

## ERC-1155: multi-token

ERC-1155 (originally proposed by Enjin for in-game items) is a single contract that tracks multiple distinct tokens, each of which can be fungible, non-fungible, or semi-fungible. A game might want 10,000 identical healing potions (fungible), one unique legendary sword (non-fungible), and 50 identical crafting materials (fungible) — ERC-1155 lets all of these live in a single contract.

### The interface highlights

```solidity
function balanceOf(address account, uint256 id) external view returns (uint256);
function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids) external view returns (uint256[] memory);
function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data) external;
function safeBatchTransferFrom(address from, address to, uint256[] calldata ids, uint256[] calldata amounts, bytes calldata data) external;
```

The key innovation is **batch operations** — transfer or query many token types in a single call, which is much cheaper than N separate calls.

Receiver contracts implement `onERC1155Received` and `onERC1155BatchReceived`, analogous to ERC-721's `onERC721Received`.

### When to use which

| Use case | Standard |
|---|---|
| Fungible token with a single supply (USDC, UNI) | ERC-20 |
| Unique collectibles with per-item metadata | ERC-721 |
| Game items, concert tickets, credential batches | ERC-1155 |
| Shares in a yield-bearing vault | ERC-20 + ERC-4626 (below) |

## ERC-4626: tokenized vaults

A newer standard (finalized 2022) for the specific case of a smart contract that accepts deposits of some underlying ERC-20, invests those deposits, and issues share tokens representing ownership of the pool.

The problem this solves: before ERC-4626, every yield-bearing vault protocol (Yearn, Compound cTokens, Aave aTokens, countless forks) had its own interface for "deposit X, get shares back, redeem shares for X plus accrued yield." Integrating with multiple vaults required bespoke code for each.

### The interface

```solidity
interface IERC4626 is IERC20 {
    function asset() external view returns (address);        // underlying
    function totalAssets() external view returns (uint256);  // total underlying held

    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function mint(uint256 shares, address receiver) external returns (uint256 assets);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);

    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    // ... plus previewDeposit, previewMint, previewWithdraw, previewRedeem, maxDeposit, maxMint, maxWithdraw, maxRedeem
}
```

The vault itself is an ERC-20 (shares are fungible). The underlying asset is a different ERC-20 accessed via `asset()`. The share-to-asset exchange rate rises over time as the vault accrues yield.

### Inflation attack (a famous ERC-4626 trap)

A newly-deployed empty ERC-4626 vault is vulnerable to a "donation attack." An attacker:

1. Deposits 1 wei of the underlying, receiving 1 share.
2. *Directly transfers* a large amount of the underlying to the vault's address (bypassing `deposit`). The vault's `totalAssets` increases but `totalSupply` of shares does not.
3. A victim then calls `deposit(amount)`. The share price has been inflated: their deposit mints *zero* shares due to integer division rounding down. The attacker now owns 100% of the pool.

Fixes exist (minimum initial deposit by the deployer, virtual shares offsets, deadshares burn). OpenZeppelin's reference implementation includes mitigations as of v5.0. Any ERC-4626 vault deployed without these is actively vulnerable.

## ERC-4337: account abstraction

Not a token standard, but important enough to cover here. ERC-4337 (finalized 2023) is a protocol-level change that enables **account abstraction**: smart-contract wallets that behave as first-class accounts, with features that externally-owned accounts cannot have.

### What problem it solves

An externally-owned account (EOA) on Ethereum has fixed, built-in rules:

- Transactions must be signed by its single private key.
- Transactions must pay gas in ETH, from the account itself.
- Nonces must increment by exactly one.

This excludes a lot of ergonomic features users expect: multi-factor authentication, social recovery (friends can help you recover a lost key), gas abstraction (pay fees in a stablecoin, or have someone else pay), batched transactions (combine multiple actions into one "tap").

### How it works (roughly)

ERC-4337 defines a parallel transaction type — a **UserOperation** — that goes through a separate **mempool**, is validated by special **bundler** actors, and ultimately executes against an **EntryPoint** contract that calls into a user's smart-contract wallet.

The user's wallet is itself a smart contract that implements a `validateUserOp` function. This function can enforce *any* authorization logic — require two signatures, check a timelock, allow guardians to replace a key. From the user's perspective, they interact with their wallet much like a normal wallet, but the wallet is itself programmable.

### Current state

ERC-4337 is live on Ethereum and most L2s. Adoption is significant on L2s (where fees are low enough for the overhead to be acceptable) and lighter on mainnet. Projects like Safe, Argent, and Biconomy are prominent implementations.

A pending EIP (EIP-7702) would let EOAs *temporarily* delegate to smart-contract code, bridging the gap more smoothly. As of this writing it is on the Ethereum roadmap for 2026.

## Interface detection: ERC-165

Several of the standards above include a note about implementing `supportsInterface(bytes4 interfaceId)`. This is **ERC-165**, the interface-detection standard. A contract reports which standardized interfaces it implements, and other contracts can query before assuming.

```solidity
if (target.supportsInterface(type(IERC721).interfaceId)) {
    // safe to treat as ERC-721
}
```

ERC-165 is not universally implemented (ERC-20 predates it, so many ERC-20s do not support it). But for ERC-721, ERC-1155, and most newer standards, it is expected.

## What to carry forward

- **ERC-20** is the fungible-token standard. Watch for return-value inconsistency (use `SafeERC20`), the approval race, and fee-on-transfer tokens.
- **ERC-721** is non-fungible. Prefer `safeTransferFrom` and implement `onERC721Received` in contracts that should hold NFTs. Media is typically off-chain.
- **ERC-1155** unifies fungible and non-fungible tokens in a single contract with batch operations. Useful for game-item-like use cases.
- **ERC-4626** standardizes yield-bearing vaults. Watch for the inflation attack; use OpenZeppelin's mitigations.
- **ERC-4337** enables account abstraction — smart-contract wallets with programmable authorization, gas abstraction, and batched transactions.
- **ERC-165** is the interface-detection standard that lets contracts verify what another contract implements.

Next: the security patterns and famous exploits that shaped how these standards are actually implemented.

> Previous: [Solidity language tour](03-solidity-language.md) · Next: [Security patterns and famous exploits](05-security-patterns.md)
