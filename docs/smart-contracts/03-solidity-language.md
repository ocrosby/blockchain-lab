# Solidity language tour

> Previous: [The EVM in depth](02-the-evm-in-depth.md) · Next: [Token standards](04-token-standards.md)

This is not a replacement for the official Solidity documentation. It is a guide to the parts of Solidity that matter for reading and writing production smart contracts, structured so that a reader coming from a general-purpose language (JavaScript, Python, Go, Rust) can map features back to the EVM semantics in the previous doc.

Solidity is C/JavaScript-shaped syntax wrapped around EVM semantics. Every "surprising" thing it does is a direct consequence of something in the EVM model. If a Solidity feature seems weird, ask what the underlying bytecode looks like.

## File structure

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    uint256 public totalMinted;

    constructor() ERC20("MyToken", "MTK") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
        totalMinted += amount;
    }
}
```

The top of every Solidity file: an SPDX license identifier (required since 0.6, warning otherwise) and a `pragma` specifying the compiler version. The `^0.8.24` syntax means "0.8.24 or any later 0.8.x" — patch and minor updates within the same major-feature band.

A `contract` is the closest analog to a class in OOP languages, but it compiles to a distinct bytecode blob and lives at a distinct address on-chain. One source file can contain multiple contracts; usually one per file is the convention.

## Types

### Value types

Stored in 256-bit words at the EVM level, even when the Solidity declaration is smaller.

- `bool` — true or false.
- `uint8`, `uint16`, ..., `uint256` — unsigned integer of n bits (n is a multiple of 8). `uint` is an alias for `uint256`.
- `int8`, ..., `int256` — signed. `int` is `int256`. Two's-complement.
- `address` — a 20-byte Ethereum address. `address payable` is a subtype that can receive ETH via `.transfer()` or `.send()`.
- `bytes1`, ..., `bytes32` — fixed-size byte arrays. `bytes32` is the natural type for a hash.
- `enum` — a named set of integer constants.

Important: since Solidity 0.8.0, arithmetic on integers **reverts on overflow and underflow by default**. Previously, overflow wrapped (which was the source of countless bugs). Explicit wrapping arithmetic is available via `unchecked { ... }` blocks for cases where you know wrapping is safe and want the gas savings.

### Reference types

Stored in memory, storage, or calldata as specified; the location is part of the type.

- `bytes` — dynamically-sized byte array.
- `string` — UTF-8 dynamically-sized string. Internally equivalent to `bytes` with no character-boundary enforcement.
- `T[]` — dynamically-sized array of type T.
- `T[N]` — fixed-size array of N elements of type T.
- `mapping(K => V)` — a hash table from K to V. Cannot be iterated. No way to get a list of keys. Only usable as a storage variable.
- `struct` — a user-defined aggregate.

Example struct:

```solidity
struct Position {
    uint256 collateral;
    uint256 debt;
    uint64 lastUpdate;
    bool active;
}

mapping(address => Position) public positions;
```

### Memory, storage, calldata as type modifiers

Reference types in function parameters and local variables must specify where they live:

```solidity
function doThing(bytes calldata input, Position memory pos) external { ... }
```

`calldata` is the input region (read-only, cheap). `memory` is scratch RAM (modifiable, released after the call). `storage` is the contract's persistent state.

Assigning between regions is not a no-op — it is a **copy**. A `Position memory pos = positions[alice]` copies Alice's position from storage into memory; modifying `pos` does not affect storage.

To modify storage through a local variable, declare it `storage`:

```solidity
Position storage pos = positions[alice];  // reference, not copy
pos.collateral += 100;                     // modifies storage
```

This distinction is critical and the source of many bugs. A beginner expects `pos.collateral += 100` to persist and it doesn't, because `pos` is a `memory` copy. Modern Solidity warns about this but the warning is not always shown.

### Mappings have an infinite domain

`mapping(address => uint256) balances` has an entry for every possible address. Addresses with no explicit assignment return the zero value for their type. There is no way to ask "does this key exist?" or "give me all keys." Both of these facts shape how you design contracts around mappings.

The usual workarounds:

- Store an explicit `bool active` field alongside, or use a sentinel value like `type(uint256).max` to represent "not set."
- Maintain a separate `address[]` array of keys if you need to iterate. Writes to both must stay in sync.

## Visibility and function modifiers

Every function has a visibility:

- `public` — callable from anywhere. For state variables, creates an auto-generated getter.
- `external` — callable only from outside the contract (including via `this.foo()`). Slightly cheaper than public for large argument types because arguments stay in calldata.
- `internal` — callable from this contract and inherited contracts only. Default for state variables.
- `private` — callable from this contract only. Does *not* hide the data from the chain; it only restricts which code can read it. Private state is still fully readable off-chain via `eth_getStorageAt` or by anyone computing the storage layout.

**Important**: "private" in Solidity is a *language-level* restriction, not a secrecy guarantee. There is no such thing as a private value on a public blockchain. If you need secrecy, use off-chain computation or zero-knowledge proofs — not the `private` keyword.

Function state mutability is also declared:

- `pure` — reads nothing from state, writes nothing.
- `view` — reads state, writes nothing. Can be called via `eth_call` without a transaction.
- (default) — may read and write state. Requires a transaction.
- `payable` — may receive ETH. Transactions to non-payable functions with non-zero `msg.value` revert.

## Modifiers

Modifiers are reusable pre- and post-conditions wrapped around functions.

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "not owner");
    _;
}

function withdraw() external onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```

The `_` is where the modified function's body gets substituted. Modifiers can run code before, after, or on both sides of the function body.

Beware: modifiers that *skip* the `_` based on a condition can create contracts whose function sometimes executes and sometimes silently does nothing. This is almost always a bug in disguise. Prefer `require(condition)` inside the modifier, which reverts rather than skipping.

## Inheritance

Solidity supports multiple inheritance with C3 linearization (same as Python). A contract inheriting from multiple parents must specify them in order of "most base" to "most derived":

```solidity
contract Token is Context, IERC20, Ownable { ... }
```

The function resolution order walks the linearization right-to-left. Calling `super.foo()` dispatches to the next parent in the linearization. The OpenZeppelin contracts lean on this heavily — `ERC20Capped` extends `ERC20` and overrides `_mint` to add a supply cap.

Interfaces (`interface IFoo`) are pure declarations with no implementation. A contract `is IFoo` must implement every function with matching signatures. Interfaces are the Solidity convention for ABI-level typing: when your contract needs to call "any contract that behaves like an ERC-20," you take an `IERC20` parameter.

## Error handling

Three ways a Solidity function can fail:

1. **`require(condition, "message")`** — the dominant pattern. Checks a condition and reverts with a message if false. Refunds remaining gas. The standard way to validate inputs and preconditions.

2. **`revert CustomError(args...)`** — a newer pattern using **custom errors** (added in 0.8.4). More gas-efficient than string messages because the error is identified by a 4-byte selector rather than a full string. Preferred for production.

3. **`assert(condition)`** — for internal invariants that should never fail. Uses an `INVALID` opcode that consumes all remaining gas. Reserve for "this is a bug if it happens" situations.

A reverted transaction undoes *all* state changes it made, including changes in nested calls. This is the atomicity that makes DeFi composition safe: if any step in a multi-step operation fails, the whole thing reverts.

The caller of a failing call sees `success = false` from the low-level `.call(...)`. For higher-level typed calls (`token.transfer(bob, 100)`), Solidity auto-propagates the revert.

## Events

Events are the EVM log mechanism from the previous doc, with Solidity syntactic sugar.

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);

function _transfer(address from, address to, uint256 amount) internal {
    balances[from] -= amount;
    balances[to] += amount;
    emit Transfer(from, to, amount);
}
```

`indexed` parameters become topics (up to 3 per event); non-indexed parameters become the data payload. Off-chain indexers filter by indexed fields — this is how Etherscan shows you "all Transfer events touching address X."

Events are **not** readable from on-chain code. Their purpose is to communicate state changes to the outside world cheaply.

## Low-level calls

Besides calling contracts via typed interfaces, Solidity exposes the raw EVM call opcodes:

```solidity
(bool success, bytes memory data) = target.call{value: 1 ether, gas: 50000}(
    abi.encodeWithSignature("doThing(uint256)", 42)
);
require(success, "call failed");
```

Use these when:

- You need to call a function whose signature you only know at runtime.
- You need to forward arbitrary data (relayed meta-transactions).
- The target contract's interface is unknown or variable.

Avoid these when a typed interface would work. Low-level calls bypass type-checking, bypass the compiler's automatic revert propagation, and are a common source of bugs (forgetting to check `success` is a classic).

## Receiving ETH

A contract can receive ETH in three ways:

1. **Via a function marked `payable`**. The caller specifies `value` in their transaction.
2. **Via the `receive()` function**, which is a special payable function called when ETH is sent with empty calldata.
3. **Via the `fallback()` function**, called when calldata is provided but doesn't match any function. May or may not be payable.

A contract without a `receive()` or payable `fallback()` cannot accept plain ETH transfers. Trying to send ETH to such a contract reverts.

## Imports and the OpenZeppelin convention

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
```

OpenZeppelin publishes the canonical audited implementations of ERC-20, ERC-721, access-control patterns, upgradeable proxies, and much more. For production code, the strong default is to use OpenZeppelin's contracts rather than rolling your own. Writing your own ERC-20 from scratch is an exercise in finding ways to be subtly wrong.

Package management is via `npm` or `foundry`'s `forge install` — these pull OpenZeppelin into your project, and the import paths resolve to the installed location.

## Compiler, optimizer, metadata

The Solidity compiler (`solc`) takes source code and produces:

- **Bytecode** (deployed to the chain).
- **ABI** (the JSON description of how to call the contract's functions).
- **Metadata** (a JSON object with compiler version, source hashes, optimization settings).

The optimizer is on by default and takes a `runs` parameter: higher `runs` optimizes for cheaper runtime (at the cost of larger bytecode and more expensive deployment). The conventional default is `200`.

The metadata hash is included in the deployed bytecode's tail. Etherscan uses this to verify that a submitted source file produces the exact same bytecode as what is deployed — this is how "verified contract" status works.

## The parts not covered

Solidity is a large language. Things this doc skips, which you should learn as they come up:

- **Assembly blocks** (`assembly { ... }`) — inline Yul for direct EVM access. Needed for gas-critical code and some patterns (meta-transactions, assembly-level proxies).
- **ABI encoding/decoding** (`abi.encode`, `abi.decodeWithSelector`, `keccak256` of a function signature) — needed for low-level calls and meta-transactions.
- **Function types** (`function (uint) external returns (uint)`) — for callbacks.
- **Try/catch** — a limited form of exception handling for external calls, where you can catch a revert from a call without propagating it.
- **User-defined value types** (`type UFixed is uint256`) — zero-cost newtypes.

The official Solidity documentation at `docs.soliditylang.org` is well-maintained and covers all of these.

## What to carry forward

- Solidity is a wrapper around EVM semantics. Every "surprising" feature traces to something in the EVM.
- **Storage vs memory vs calldata** is a type-level distinction that affects correctness, not just performance.
- Arithmetic has **reverted on overflow since 0.8.0**. Older contracts may use SafeMath; newer ones don't need to.
- `private` state is private at the language level only. It is fully visible on the chain.
- **OpenZeppelin** is the canonical set of audited contracts for standard patterns. Use them.
- **Events are for off-chain indexers**; they cannot be read on-chain.
- Reverts undo all state changes in the entire call stack. This is the property that makes DeFi composition safe.

Next: the token standards that most Ethereum activity ultimately comes down to.

> Previous: [The EVM in depth](02-the-evm-in-depth.md) · Next: [Token standards](04-token-standards.md)
