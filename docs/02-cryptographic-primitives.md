# Cryptographic primitives

> Previous: [The trust problem](01-the-trust-problem.md) · Next: [Blocks and chains](03-blocks-and-chains.md)

Two cryptographic tools do almost all the heavy lifting in a blockchain: **cryptographic hash functions** and **digital signatures**. Neither was invented for blockchain — both are decades old, well understood, and widely used in TLS, SSH, package managers, and version-control systems. Blockchains combine them in a particular way, but the primitives themselves are standard.

If you understand these two tools, the data structure in the next doc will look obvious.

## Cryptographic hash functions

A **hash function** takes an input of any length and produces a fixed-length output. SHA-256, the one Bitcoin uses, always produces 256 bits (32 bytes, usually displayed as 64 hex characters):

```
sha256("hello")                → 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
sha256("hello.")               → 6ad78a2a5f6d6d3ff50fd4bd66a2bf5d81e8e9b68a75b7a0d7a8d1e6a49a5e57 (different)
sha256("the quick brown fox…") → c03905fcdab297513a620ec81ed46ca44ddb62d41cadfe5e3c3b0fe4a5a9d1d8
```

**Cryptographic** hash functions have three additional properties that an ordinary hash function (like one you'd use in a hash table) does not need:

1. **Deterministic.** Same input, same output. Forever.
2. **Fast to compute, forward.** Hashing a megabyte of data takes milliseconds.
3. **Pre-image resistant.** Given the output, you cannot feasibly find *any* input that produces it. "Feasibly" here means "even with all the computers currently on Earth, you would finish long after the heat death of the universe."
4. **Collision resistant.** You cannot feasibly find two different inputs that produce the same output. (A collision certainly exists — the output space is finite and the input space is infinite — but finding one is computationally out of reach.)
5. **Avalanche.** Changing a single bit of the input produces an entirely unrecognizable output. The second line above differs from the first only by a period, and the outputs share no structure.

A useful mental model: a cryptographic hash is a **fingerprint for data**. Two files with the same fingerprint are the same file, in practice. Two files that differ anywhere — even in one bit — have fingerprints that look completely unrelated.

### Why hashes matter for a ledger

Three properties fall out of this immediately:

- **Tamper-evidence.** If you publish a hash of a document today, anyone can check tomorrow whether the document has been modified. Any change, anywhere in the document, produces a different hash.
- **Compact commitment.** You can commit to a terabyte of data by publishing 32 bytes. "I'll stand behind whatever was in this file, and here's the fingerprint" is a 256-bit statement.
- **Linking.** If block N contains the hash of block N−1, then block N is committed to the exact content of block N−1. If you change block N−1, its hash changes, and block N no longer links to it correctly.

That last property is the "chain" in blockchain. We'll get to it in the next doc.

### Hashes are not encryption

Common confusion: hashes are one-way. There is no "un-hashing." A hash is not a reversible transformation of the input — it is a fingerprint derived from it. You cannot recover the original data from a hash the way you can recover plaintext from a ciphertext.

## Digital signatures

A hash lets you commit to data. A **digital signature** lets you commit to *having authored* data.

The setup is a **public-key cryptosystem**: every user generates a **key pair** consisting of a **private key** (kept secret) and a **public key** (shared freely). The math has the following property:

- The holder of the private key can produce a signature over a message.
- Anyone holding the public key can verify that the signature is valid for that exact message.
- No one without the private key can produce a valid signature.

```
alice_private_key, alice_public_key = generate_keypair()

message   = "pay bob 10 units"
signature = sign(alice_private_key, message)

verify(alice_public_key, message, signature)         → True
verify(alice_public_key, "pay bob 20 units", signature) → False  (message changed)
verify(bob_public_key,   message, signature)         → False  (wrong public key)
```

Conceptually, a signature is an unforgeable statement that says "the holder of the private key corresponding to this public key endorsed this exact message." Change the message by one character, the signature no longer verifies. Try to sign a message without the private key, you cannot.

Bitcoin and Ethereum use the ECDSA algorithm over the secp256k1 curve for signing; the math is different from RSA (which you may have seen in TLS), but the interface is the same.

### Addresses are public keys (sort of)

In a blockchain, your "account" or "address" is derived from your public key — typically by hashing the public key and encoding the hash in some human-friendly format. There is no sign-up process, no username, no registrar. You generate a key pair locally, and the public half (or a hash of it) is your identity.

This has two sharp consequences you should internalize now:

1. **No one can lock you out.** There is no password to reset because there is no account to recover. You either have the private key or you do not.
2. **No one can bail you out.** If you lose the private key, the funds or data controlled by it are gone. Permanently. There is no "contact support."

This is the most unforgiving aspect of blockchain systems and the source of a significant fraction of the actual value ever locked into them. Estimates suggest 3–4 million bitcoins — roughly 20% of the total supply that will ever exist — are in wallets whose keys have been lost.

### Signing a transaction

Putting hashes and signatures together, the structure of a transaction is now visible:

```
transaction = {
  from:   alice_public_key,
  to:     bob_public_key,
  amount: 10,
  nonce:  7,                       # a counter to prevent replay
}
tx_hash   = sha256(serialize(transaction))
signature = sign(alice_private_key, tx_hash)

broadcast(transaction, signature)
```

Any node receiving this can:

- Recompute `sha256(serialize(transaction))` and confirm the hash matches what was signed.
- Call `verify(alice_public_key, tx_hash, signature)` to confirm the signature is valid.
- Check its local copy of the ledger to confirm Alice has 10 units to spend.

If all three pass, the transaction is valid *in isolation*. Validity *in sequence* — making sure Alice didn't already spend those same 10 units in another transaction thirty seconds ago — is the job of consensus, in [doc 4](04-consensus.md).

## What to carry forward

- A cryptographic hash is a fingerprint. Same input → same output; any change → an unrecognizable output; the output tells you nothing about the input.
- A digital signature is an unforgeable statement that the holder of a private key endorsed a specific message. The public key verifies it.
- Blockchain identity is a key pair you generate yourself. No account to recover; no one to bail you out.
- A transaction is a signed message. Verifying one in isolation is pure cryptography and requires no trust.

Next we assemble these primitives into the "chain of blocks" data structure that makes a tamper-evident history possible.

> Previous: [The trust problem](01-the-trust-problem.md) · Next: [Blocks and chains](03-blocks-and-chains.md)
