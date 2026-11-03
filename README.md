# Checkpoint — Mini Banking System: Transaction Flow and Distribution Planning

GoMyCode checkpoint — *Computer Systems and Their Fundamentals: Database Management systems*. Written analysis (no code), covering concurrency control and distributed database design for a small banking system.

## Part 1 — Transaction Management (Conceptual)

**Scenario:** two users simultaneously initiate a transfer that touches the same account, `A`.

### Concurrency issue

This is the textbook **lost update** scenario:

1. T1 (transfer A → B) reads `balance(A) = 1000`.
2. T2 (transfer A → C) also reads `balance(A) = 1000` before T1 has written anything back.
3. T1 computes `1000 - 100 = 900` and writes it.
4. T2 computes `1000 - 50 = 950` (based on the *stale* value it read) and writes it, overwriting T1's debit.

Result: the account should have lost `150` total, but it only shows a `50` debit — T1's `-100` was silently lost. If the two transactions also read each other's *uncommitted* intermediate values (rather than the original balance), the same scenario can additionally produce a **dirty read**.

### Locking mechanism

Use an **exclusive (X) lock** on the account row for the whole "read balance → compute new balance → write balance" sequence, acquired with `SELECT balance FROM accounts WHERE id = 'A' FOR UPDATE`. An exclusive lock blocks both other readers-for-update and writers until it's released, so the second transaction can't read a balance that's about to be invalidated by the first one's commit.

A shared (S) lock would not be enough here: shared locks only block writers, not other readers, and both T1 and T2 need to *write*, so they'd both still get in and recreate the lost update.

### Pessimistic or optimistic locking?

**Pessimistic locking.** Reasoning:

- The two transactions are **very likely to conflict** (same account), not rarely — optimistic locking's core assumption ("conflicts are rare") doesn't hold here.
- The cost of letting a conflict through is **real money being lost**, not just a UI state going briefly stale. An optimistic approach would detect the conflict only at commit time and abort one transaction, which under contention could mean repeated retries — worse, if the retry logic is wrong, a silent lost update.
- Pessimistic locking's downside (transactions wait instead of running freely) is an acceptable trade-off for a money transfer: better to make T2 wait a few milliseconds for T1 to finish than to risk an inconsistent balance.

Optimistic locking is still reasonable for the "view balance" read path (no write involved, no conflict possible), just not for the transfer write path.

### Schedule example

**With the exclusive lock (safe / serializable):**

| Time | T1 — Transfer A → B ($100) | T2 — Transfer A → C ($50) | Lock on account A |
|---|---|---|---|
| t1 | `BEGIN; SELECT balance FROM accounts WHERE id='A' FOR UPDATE;` → reads 1000 | — | T1 holds X-lock |
| t2 | — | `BEGIN; SELECT balance ... FOR UPDATE;` → **blocks**, waits for T1 | T2 waiting |
| t3 | `UPDATE accounts SET balance=900 WHERE id='A'; COMMIT;` | — | lock released |
| t4 | — | lock granted → reads 900 → `UPDATE balance=850; COMMIT;` | T2 holds, then releases |

Final balance: `850`. Correct — both the `-100` and the `-50` are reflected. The schedule is safe because the lock forces T2 to start from T1's *committed* result instead of a value T1 was about to overwrite.

**Without any lock (unsafe, shown for contrast):** both T1 and T2 read `1000` at the same time, write `900` and `950` independently, and whichever commits last overwrites the other — final balance `950` or `900`, with one of the two debits silently lost.

## Part 2 — Distributed Database Planning (High-Level)

Three branches: **Tunis**, **Sousse**, **Sfax**.

### Horizontal fragmentation of `Customers`

Split by branch, same schema everywhere, different rows per site:

- Site Tunis → `Customers WHERE branch = 'Tunis'`
- Site Sousse → `Customers WHERE branch = 'Sousse'`
- Site Sfax → `Customers WHERE branch = 'Sfax'`

This satisfies the three fragmentation criteria from the course: **completeness** (every customer belongs to exactly one branch fragment), **disjointness** (no customer row duplicated across fragments), and **reconstructability** (`UNION` of the three fragments recreates the full table). It also puts each branch's customer data physically close to where it's queried most (tellers and local staff looking up local customers).

### Vertical separation

Split off **login/authentication info** (email used for login, password hash, 2FA secret, etc.) into a separate `CustomerAuth` fragment, keeping `customer_id` in both fragments as the shared key so they can be `JOIN`ed back together. Reasons:

- It's accessed far less often than everyday profile data (name, phone, branch) — most queries (teller lookups, statements) don't need it.
- It has stricter security requirements than the rest of the profile, so isolating it makes it easier to apply tighter access control without affecting normal profile queries.

### What should be replicated across all branches?

- **Customer info (name, contact, branch, ID number, ...): full replication.** It's read-heavy, rarely updated, and a customer or a teller at *any* branch may need to look someone up — availability matters more than strict single-copy consistency here, and conflicts on simple profile edits are rare and low-stakes.
- **Account balances: not freely replicated.** A balance is written on every transaction and must stay strongly consistent (this is literally the Part 1 lost-update problem at a larger scale) — replicating it everywhere with eventual consistency risks the same money being spent twice from two branches before the copies sync. Keep one authoritative copy (at the account's home branch), with other branches reading through to it (or, at most, a tightly-synchronized replica) rather than an independently-writable local copy.
- **Transaction history: not replicated everywhere either.** It's write-once/append-only and high-volume, and almost always queried in the context of its own account (home-branch statements) or centrally (fraud/audit reporting) — replicating the full history to every branch would mostly waste storage and bandwidth for data that branch doesn't need locally.

### Static or dynamic allocation for transaction history?

**Static allocation**, partitioned by the account's home branch (or archived centrally for old records). Transaction history is append-only with a stable, predictable access pattern — it's read from the branch where the account lives, or in bulk for reporting — so there's no shifting "hot spot" that would justify dynamic allocation's extra complexity and data-movement overhead. Dynamic allocation earns its cost when access patterns genuinely change over time; a ledger of past transactions doesn't move around, it just grows.
