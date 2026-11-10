# The trust problem

> Previous: [README](../README.md) · Next: [Cryptographic primitives](02-cryptographic-primitives.md)

Before you can appreciate what a blockchain *does*, you have to feel the specific discomfort of the problem it is trying to solve. This document spends some time there, because every design decision in a blockchain — every strange-looking choice about hashes, consensus, incentives — is motivated by something concrete on this list.

## What a ledger is, and who keeps it

A **ledger** is a list of entries that records who owns what, or who owes what, in some ordered way. Your bank's ledger records that your account has $1,432.17 in it. A county clerk's ledger records that the house at 414 Elm Street belongs to Alice. A hospital's ledger records that you are allergic to penicillin.

Someone has to keep the ledger. Historically, that someone has been a trusted institution:

| Ledger | Keeper | What can go wrong |
|---|---|---|
| Account balances | Commercial bank | Bank fails, freezes your account, is hacked, is subpoenaed |
| Securities ownership | Depository (DTCC in the US) | Settlement delays, broker insolvency, counterparty default |
| Property titles | County clerk's office | Title fraud, lost records, errors from paper-to-digital migrations |
| Credential verification | University registrar | Diploma mills, lost transcripts, closed institutions |
| Vote tallies | Election authority | Allegations of manipulation, unauditable electronic machines |
| Social media posts | Platform operator | Deplatforming, retroactive edits, outages |

In every case, you hold a *receipt* (a bank statement, a deed, a diploma, a tweet). The institution holds the *truth*. If the two disagree, the institution wins.

## Why this has usually been fine

For most of history, there was no technical alternative. Writing down the authoritative ledger in one trusted place was the only way to avoid chaos. The alternative — every participant keeping their own copy — immediately runs into a basic problem: if I say I have $1,000 and you say I have $5, what happens? Someone has to be right.

So the institution emerged as the answer, and the institution's own trustworthiness was propped up by regulation, auditing, reputation, and (in the case of banks) the threat of a run. This machinery is expensive but it mostly works, which is why the status quo persisted.

## The specific failures that motivated something new

The system breaks in specific ways, not in the abstract:

1. **The institution itself fails.** The 2008 financial crisis is the canonical example — the ledger-keepers were simultaneously the parties being bailed out, and millions of receipts suddenly looked like promises an insolvent party could not keep. In 2013, Cyprus imposed a "haircut" on bank deposits over €100,000 — depositors' balances were simply reduced by decree.

2. **The institution is captured by an adversary.** Governments can and do order institutions to freeze accounts, reverse transfers, or hand over records. Whether this is good or bad depends on who the adversary is to you. For a journalist whose bank account is frozen by an authoritarian state, the institution's obedience to the law is the failure.

3. **The institution loses the record.** Property-title fraud, lost transcripts from closed universities, hospital records destroyed in a fire or a ransomware attack. The institution is a single point of failure in the information-theoretic sense: if the data is only there, losing it loses it.

4. **The institution is slow or expensive.** International wire transfers settle in days and cost tens of dollars because the chain of intermediaries each take a cut and each need to check with the next. Remittance corridors (migrant workers sending money home) routinely lose 5–10% to fees.

5. **The institution excludes people.** Roughly 1.4 billion adults worldwide do not have a bank account. The reasons are mundane: no official ID, no fixed address, no credit history, the nearest branch is forty miles away. The institution is the gatekeeper, and the gate is closed to a lot of the world.

## The double-spending problem

Zoom in on one of the technical failures. Pretend you are trying to build **digital cash** — money that behaves like physical cash but lives on computers. You hit a problem almost immediately.

Physical cash has a nice property: when you hand me a $20 bill, you no longer have it. The transfer and the loss are the same physical act. Digital files do not work this way. If "money" is just a file, nothing stops you from sending me a copy, then sending the same copy to someone else, then spending it again at the coffee shop. Each recipient, looking only at their own copy, sees valid money. By the time anyone notices, the real victim is whoever shows up last.

This is the **double-spending problem**, and it is the specific problem that defeated every digital-cash scheme from the 1980s to 2008. The pre-blockchain answer was always the same: *have a trusted central server keep a ledger of who-has-what, and refuse any transaction that would overdraw*. Which works — PayPal, Visa, and your bank do exactly this — but it reintroduces the trusted institution we were trying to get rid of.

The invention that Bitcoin's 2008 whitepaper actually contributed, more than any individual cryptographic piece, was *a way to solve the double-spending problem without a central server*. Everything else in this repo is machinery in service of that one feat.

## What "trustless" really means

You will see the word **trustless** used a lot in the blockchain literature. It is a misleading word, because of course you still have to trust something — you trust the math, you trust that enough of the network is honest, you trust that your software is implemented correctly. "Trustless" does not mean "trust nothing"; it means **you do not have to trust any specific, named party**. The trust is distributed over a protocol and a large, open set of participants, rather than concentrated in one institution whose failure would ruin you.

If you prefer, a more accurate term is **trust-minimized**. The design goal is to need as little trust in any single actor as possible, not to eliminate trust altogether.

## What to carry forward

Before moving on, make sure you are comfortable with:

- There is such a thing as a *ledger*, and someone has to keep it.
- The pre-blockchain answer was always "a trusted institution keeps it," and this answer has specific failure modes.
- Digital cash in particular runs into the double-spending problem, which is the reason it did not work for thirty years before Bitcoin.
- The goal of a blockchain is not to eliminate trust but to distribute it, so that no single party's failure ruins the system.

With that grounding, the cryptographic machinery in the next doc will feel motivated rather than arbitrary.

> Previous: [README](../README.md) · Next: [Cryptographic primitives](02-cryptographic-primitives.md)
