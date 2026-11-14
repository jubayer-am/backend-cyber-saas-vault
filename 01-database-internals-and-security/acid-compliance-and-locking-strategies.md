# ACID & Locking Strategies

Reference notes on transaction guarantees, schema invariants, and row-level locking in PostgreSQL—focused on concurrency bugs and security exploits.

Related: [db-isolation-levels-and-race-conditions.md](./db-isolation-levels-and-race-conditions.md)

---

## The Core Problem

A balance transfer looks like one action to a user, but executes as separate mutations:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
UPDATE accounts SET balance = balance + 10 WHERE id = 2;
COMMIT;
```

Without proper transaction boundaries and locking, three failure modes show up in production:
1. **Partial writes:** The process crashes after query 1; `$10` is deducted from account 1 and never credited anywhere.
2. **Race conditions (TOCTOU):** Two parallel requests read `balance = 10` at the same microsecond and both withdraw `$10`.
3. **Invariant bypass:** Malformed or negative inputs corrupt ledger state.

---

## ACID Guarantees

### Atomicity
A transaction is indivisible. Either every statement inside `BEGIN ... COMMIT` persists, or the entire transaction aborts (`ROLLBACK`) and reverts via undo/MVCC state.

* By default, most drivers and SQL consoles run in `auto-commit` mode—each query commits independently. If a crash happens mid-flow, earlier writes stay on disk.
* **Security impact:** Attackers intentionally trigger timeouts, dropped connections, or serialization errors between steps. If an endpoint grants credits/items first and deducts payment second without an explicit transaction block, killing the request mid-flight yields free credits.

### Consistency
Consistency guarantees that a transaction transitions the database from one valid state to another. Before `COMMIT` succeeds, all schema-level rules must pass or the transaction aborts.

Enforced at the storage layer via:
* `CHECK (balance >= 0)` — prevents negative balances regardless of application-layer bugs.
* `UNIQUE` / `PRIMARY KEY` — blocks duplicate identities or replayed idempotency keys.
* `FOREIGN KEY` — prevents orphaned rows pointing to non-existent parent records.
* `NOT NULL` and strict column types.

**Security impact:** Application-layer checks (`if balance >= amount:`) are not enough. If an API accepts a negative transfer (`amount = -500`), naive backend math computes `balance - (-500)` and mints funds. Database constraints are the hard boundary that blocks rule-breaking writes even when app validation fails.

> **Atomicity + Consistency coupling:** Consistency only validates within a single transaction boundary. Without Atomicity grouping multi-step writes, single-row `CHECK` constraints can pass on query 1 while global invariants (total money across accounts) silently break when query 2 never runs.

### Isolation
Isolation controls how concurrent transactions see each other's uncommitted changes. Under heavy parallel load, weak isolation lets transactions read stale or half-written state, causing double-spends and lost updates.

* Implemented via **row/table locks** or **MVCC** (snapshot visibility).
* See [db-isolation-levels-and-race-conditions.md](./db-isolation-levels-and-race-conditions.md) for isolation tiers (`Read Committed` vs `Serializable`), write skew, and parallel HTTP race exploits.

### Durability
Once `COMMIT` returns success to the client, the data survives process crashes and power loss.

* **PostgreSQL mechanism (WAL):** Modifying main heap files directly on every commit is too slow, and RAM is volatile. Instead, Postgres appends transaction changes sequentially to a **Write-Ahead Log (WAL)** on disk and calls `fsync()` before acknowledging `COMMIT`. On crash recovery, Postgres replays committed WAL records from the last checkpoint.
* **Security impact:** Prevents repudiation (crashing a service immediately after an unauthorized action to wipe audit records) and guarantees that password resets, key revocations, and ledger commits do not silently revert after an unclean reboot.

---

## Locking Strategies

When concurrent requests target the same row, there are two standard ways to serialize access:

### Pessimistic Locking (`SELECT ... FOR UPDATE`)
Assumes contention is likely. Acquires an exclusive row-level lock on read; concurrent transactions attempting to lock or modify that row block until the lock holder runs `COMMIT` or `ROLLBACK`.

```sql
BEGIN;

-- Blocks concurrent transactions from locking/updating row 1 until COMMIT
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- Application validates balance >= 100, then mutates safely
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

COMMIT;
```

* **When to use:** High-contention targets where collisions are frequent or expensive (wallets, inventory decrements, single-use coupons).
* **Trade-offs:** Reduces throughput under heavy contention and introduces deadlock risk if rows are locked inconsistently.
* **Queue variant:** `FOR UPDATE SKIP LOCKED` lets worker processes claim unlocked rows without waiting—standard pattern for Postgres-backed job queues.

### Optimistic Locking (Compare-and-Swap / `version`)
Assumes collisions are rare. Holds no locks during reads. Each row stores a `version` integer (or `updated_at` timestamp), and the write succeeds only if the version matches what was originally read.

```sql
-- 1. Read state (e.g., balance = 100, version = 4)
SELECT balance, version FROM accounts WHERE id = 1;

-- 2. Atomic conditional update
UPDATE accounts
SET balance = balance - 100,
    version = version + 1
WHERE id = 1 AND version = 4;
```

* **When to use:** Read-heavy workloads with low write contention (profile updates, configuration records, documents).
* **Trade-offs:** If `rowcount == 0`, a concurrent writer incremented `version` first. The application must catch this conflict and retry from step 1.

| | Pessimistic (`FOR UPDATE`) | Optimistic (`version`) |
| :--- | :--- | :--- |
| **Contention fit** | High contention (ledgers, stock, tokens) | Low contention (settings, profiles) |
| **Conflict behavior** | Concurrent requests wait in queue | Losing writer gets `0 rows` and retries |
| **Failure mode** | Lock waits and deadlocks | Retry storms under parallel request spam |

---

## Deadlock Prevention

A deadlock occurs when Transaction A locks `id = 1` and waits for `id = 2`, while Transaction B locks `id = 2` and waits for `id = 1`. Postgres detects the cycle after `deadlock_timeout` (default `1s`) and kills one transaction (`SQLSTATE 40P01`).

**Fix:** Always acquire locks in a deterministic order (e.g., ascending primary key) across all code paths:

```sql
SELECT id, balance
FROM accounts
WHERE id IN (101, 205)
ORDER BY id ASC
FOR UPDATE;
```

---

## Common Prod Footguns

* **Multi-step mutations in auto-commit mode:** Leaving `UPDATE` / `INSERT` sequences outside `BEGIN ... COMMIT` creates partial-state corruption on timeouts or crashes.
* **Read-then-write under `Read Committed` without locks:** Running `SELECT balance` followed by `UPDATE balance` in Postgres's default isolation level is vulnerable to parallel race conditions unless you use `FOR UPDATE` or a single atomic `UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100`.
* **App-only validation:** Relying on Pydantic/Zod/backend `if` checks without SQL `CHECK`, `UNIQUE`, and `FOREIGN KEY` constraints leaves the database open to logic flaws and negative-value payloads.
* **Disabling `synchronous_commit` or `fsync` on critical tables:** Trades durability for benchmark speed; an OS crash will drop recently confirmed commits.
