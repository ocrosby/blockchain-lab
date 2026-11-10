# The EVM in depth

> Previous: [History and conceptual roots](01-history-and-conceptual-roots.md) · Next: [Solidity language tour](03-solidity-language.md)

The **Ethereum Virtual Machine** is the execution environment every full node on Ethereum runs. Every smart-contract transaction's effect is defined by what the EVM does with the contract's bytecode and current state. Getting a working mental model of the EVM is the single highest-leverage thing you can do when learning smart contracts — once you know it, Solidity stops feeling magical and starts feeling like a wrapper.

## A single-threaded stack machine

The EVM is a **stack machine**, in the same family as Forth and the JVM. Each instruction operates on a stack of 256-bit values. There are no registers and no local variables as a first-class concept — all computation is pushing values onto a stack and consuming them with operators.

A 32-bit addition on an x86 CPU is:

```
add eax, ebx   # eax = eax + ebx
```

The same thing in EVM bytecode is:

```
PUSH1 0x05     # stack: [5]
PUSH1 0x03     # stack: [5, 3]
ADD            # stack: [8]
```

`ADD` consumes the top two values and pushes their sum. All EVM operations have this flavor: pop their arguments from the stack, push their result.

The stack has a maximum depth of **1024**. Trying to push beyond that aborts the current call with an `OutOfStack` error.

## 256-bit words

Every value on the EVM stack is **256 bits wide**. Not 32, not 64 — 256. This is because every address, every hash, every numeric value in EVM semantics is designed to fit in a single 256-bit word.

The consequence is that small operations are *expensive* relative to equivalent operations on a normal CPU. Adding two 8-bit numbers still costs you a 256-bit addition. Smaller types (`uint8`, `uint32`, `address` — which is 160 bits) exist in Solidity, but at the EVM level they are stored and operated on as 256-bit values, with optional masking at the edges.

This 256-bit choice is intentional: it is wide enough to represent cryptographic hashes and elliptic-curve coordinates directly, which simplifies the bytecode for the common operations in a cryptocurrency system.

## Three places to store data

The EVM has three distinct memory regions, each with its own cost and lifetime. Confusing them is the single most common source of Solidity bugs for beginners.

| Region | Lifetime | Scope | Cost |
|---|---|---|---|
| **Stack** | During current call | Current call | Nearly free (1–3 gas per op) |
| **Memory** | During current call | Current call | Linear in size; quadratic growth at large sizes |
| **Storage** | Forever (until deleted) | Current contract | **Very** expensive; `SSTORE` is 20,000 gas for a new slot |
| **Calldata** | During current call | Current call | Read-only; cheap to access |

### Stack

Working area for the current operation. Fast, limited to 1024 entries, lost when the call returns.

### Memory

A linear, byte-addressable region that grows as needed. Think of it as "scratch RAM for this one function call." It is cleared at the start of every call and does not persist.

Memory pricing is linear up to 23 KB and **quadratic** beyond that. This design discourages contracts from allocating huge buffers — it is almost always cheaper to process data in chunks than to build a giant array in memory.

### Storage

**The actual persistent state of the contract.** A sparse mapping from 256-bit keys to 256-bit values, backed by the chain's state trie. Writes here produce real on-chain state changes that every node must record.

Storage is *the* resource the EVM gas model tries hardest to ration:

- `SLOAD` (read a storage slot): 2,100 gas for a cold slot, 100 gas for a warm one (one already touched in the current transaction).
- `SSTORE` (write a storage slot): 20,000 gas for a fresh slot, 2,900–5,000 for a modification, with partial refunds on zeroing.
- The chain charges you for the storage you *add* and (partially) refunds you for storage you *remove*. This is the mechanism behind "gas refunds for self-destructing contracts."

Compared to most computing environments, these numbers are enormous. A 100-byte write to a database file is cheap; writing one 32-byte slot on Ethereum can cost pennies to dollars depending on gas prices. **Every design decision in Solidity that reduces storage use is motivated by this.**

### Calldata

A read-only region containing the current transaction's input data. Cheap to read from but cannot be modified. When a transaction arrives, its data lives here; Solidity will sometimes copy from calldata into memory only when necessary.

## Opcode families

EVM opcodes are grouped into a few recognizable families. You do not need to memorize them, but you should recognize the groupings.

- **Stack manipulation**: `PUSH1`–`PUSH32`, `POP`, `DUP1`–`DUP16`, `SWAP1`–`SWAP16`. Pushing immediate constants, duplicating stack entries, swapping stack positions.
- **Arithmetic**: `ADD`, `SUB`, `MUL`, `DIV`, `MOD`, `EXP`, `SDIV` (signed), `SMOD`. All wrap modulo 2²⁵⁶ unless the compiler inserts overflow checks.
- **Comparison and bitwise**: `LT`, `GT`, `EQ`, `ISZERO`, `AND`, `OR`, `XOR`, `NOT`, `BYTE`, `SHL`, `SHR`, `SAR`.
- **Hashing**: `KECCAK256`. Note: Ethereum uses **Keccak-256**, which is slightly different from the finalized NIST SHA3-256. This is a historical accident with no security implications but occasional confusion.
- **Environment**: `ADDRESS`, `BALANCE`, `CALLER`, `ORIGIN`, `CALLVALUE`, `CALLDATALOAD`, `CODESIZE`, `GASPRICE`, `COINBASE`, `TIMESTAMP`, `NUMBER`, `DIFFICULTY`/`PREVRANDAO`, `GASLIMIT`, `CHAINID`, `SELFBALANCE`, `BASEFEE`, `BLOBHASH`. These read context.
- **Memory**: `MLOAD`, `MSTORE`, `MSTORE8`, `MSIZE`.
- **Storage**: `SLOAD`, `SSTORE`, `TLOAD`/`TSTORE` (transient storage, post-Cancun).
- **Control flow**: `JUMP`, `JUMPI`, `JUMPDEST`, `PC`, `STOP`, `RETURN`, `REVERT`, `INVALID`.
- **Logging/events**: `LOG0`–`LOG4`. Writes to a separate event log, cheaper than storage; indexable by off-chain tools but not readable by on-chain contracts.
- **Contract creation and interaction**: `CREATE`, `CREATE2`, `CALL`, `CALLCODE` (deprecated), `DELEGATECALL`, `STATICCALL`, `RETURNDATASIZE`, `RETURNDATACOPY`, `SELFDESTRUCT` (deprecated post-Cancun).

The last family is the interesting one.

## Call semantics: CALL, DELEGATECALL, STATICCALL

Understanding the three call variants is critical to reading real contracts.

### CALL

The normal variant. Executes the target contract's code *in the target contract's context*. The target reads and writes its own storage. Any ETH sent as `value` goes from the caller to the target.

```
A calls B.foo() with value 10 ETH:
  msg.sender inside B.foo() = A
  address(this) inside B.foo() = B
  storage touched = B's storage
  balance effects: A - 10 ETH, B + 10 ETH
```

### DELEGATECALL

The confusing one. Executes the target contract's *code* in the caller's *context*. Storage, balance, `msg.sender`, and `msg.value` are all preserved from the caller.

```
A delegatecalls B.foo():
  msg.sender inside B.foo() = A's original caller (not A itself)
  address(this) inside B.foo() = A (!)
  storage touched = A's storage (!)
  balance unchanged
```

This exists to enable **library contracts** and **upgradeable proxies**: a contract can borrow code from another contract while keeping its own state.

The reason delegatecall is dangerous: if B has a bug, or if B is upgraded to malicious code, that code runs with full write access to A's storage. The 2017 Parity multisig wallet freeze (500,000 ETH permanently locked) was caused by a delegatecall to a library contract that was accidentally self-destructed.

### STATICCALL

Like CALL, but any attempt by the callee to modify state (`SSTORE`, `LOG*`, `CREATE`, `SELFDESTRUCT`, or a nested call that modifies state) will cause the call to revert. Used to query another contract read-only; a safety net for calling into untrusted code when you only want information.

### Call failure handling

Calls do not throw exceptions. They return a boolean success flag. A call that reverts returns `success=false` but **continues execution of the caller** unless the caller explicitly checks the flag and reverts in response. Many smart-contract bugs come from ignoring a call's return value and assuming it succeeded.

Modern Solidity's high-level `address.call{value: x}("")` returns `(bool success, bytes memory data)`. If you do not check `success`, the compiler warns but does not error. It is on you to propagate failures.

## Gas accounting

Every opcode has a gas cost. The costs are published in the Ethereum Yellow Paper and updated by EIPs; the rough shape is:

| Opcode family | Typical cost |
|---|---|
| Simple arithmetic (`ADD`, `SUB`) | 3 |
| Comparison (`LT`, `EQ`) | 3 |
| `MUL`, `DIV` | 5 |
| `EXP` | 10 + 50 per byte of exponent |
| `KECCAK256` | 30 + 6 per word hashed |
| `SLOAD` (cold slot) | 2,100 |
| `SLOAD` (warm) | 100 |
| `SSTORE` (new slot, 0 → nonzero) | 20,000 + 2,100 cold-access surcharge |
| `SSTORE` (changing a nonzero) | 2,900 + 2,100 |
| `SSTORE` (setting to 0, with refund) | 2,900 + 2,100, refund of up to 4,800 |
| `LOG0`–`LOG4` | 375 base + 375 per topic + 8 per byte |
| `CALL` | 2,600 (cold target) + call-related costs |
| `CREATE` | 32,000 base + init-code cost + per-byte data |

The pattern: computation is cheap, I/O and persistent writes are expensive, and anything that permanently adds to the chain's state (new accounts, new storage slots) costs the most.

### Gas as a budget

Every transaction specifies a `gasLimit`. The EVM runs the transaction while deducting gas for each opcode; if it runs out, the transaction reverts and the sender is still charged for the gas consumed. The unused gas is refunded.

For simple ETH transfers the limit is 21,000 (and that is the entire cost). For contract calls, the limit might be 100,000 to a few million, depending on complexity. A block has a total gas limit (currently ~30 million on Ethereum mainnet), so block space is literally rationed in units of computational work.

## Logs and events

The EVM has a separate append-only log structure, written by `LOG0`–`LOG4` opcodes. Each log has a data payload plus 0–4 **topics** (indexed fields). Logs are:

- **Cheaper than storage** by roughly an order of magnitude.
- **Not readable from on-chain code** — only off-chain tools can read them.
- **Indexable** — the 4 topic slots are used by block explorers and apps to filter events ("show me all Transfer events where `to = alice`").

In Solidity, this is how `emit Transfer(from, to, amount)` compiles — the event's indexed parameters become topics, the non-indexed ones become the data payload.

The common pattern: write only what the contract needs to *compute on-chain* to storage; emit everything else as events so off-chain indexers (The Graph, Dune, Etherscan) can reconstruct history cheaply.

## Transient storage (Cancun 2024)

A new addition: `TSTORE` / `TLOAD` provide storage-like semantics but *scoped to the current transaction only*. The data is cleared at the end of the transaction, so it is cheaper (~100 gas) than persistent storage. This simplifies several patterns that previously required storage writes followed by cleanup.

Not yet available on all L2s as of this writing. A notable example of the EVM continuing to evolve.

## What to carry forward

- The EVM is a **stack machine with 256-bit words** and three distinct data regions (stack, memory, storage, plus calldata and the transient-storage region).
- **Storage is the expensive resource.** Every design decision in Solidity is shaped by the cost of `SSTORE`.
- **`CALL`, `DELEGATECALL`, `STATICCALL`** have different context semantics. `DELEGATECALL` executes callee code in the caller's storage context — powerful (libraries, proxies) and dangerous (the Parity freeze).
- **Gas accounting** rations block space by charging for every opcode, with persistent writes the most expensive.
- **Events and logs** are cheaper than storage and indexable off-chain, but not readable from on-chain code.
- The EVM continues to evolve: transient storage, EOFs, precompiled contract additions. The semantics below Solidity are a moving target.

Next: Solidity, the dominant source language for writing EVM bytecode.

> Previous: [History and conceptual roots](01-history-and-conceptual-roots.md) · Next: [Solidity language tour](03-solidity-language.md)
