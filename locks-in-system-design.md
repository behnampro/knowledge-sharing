# Locks in System Design

This guide has moved into a topic-by-topic reference: [Locks in System Design guide](locks-in-system-design/README.md).

It includes individual pages for every lock type, code examples, decision guidance, and diagrams.

---

## Legacy overview

Locks coordinate concurrent work over a shared resource. They help preserve **correctness**—for example, preventing two buyers from both receiving the final item in inventory—but can reduce availability and throughput when used carelessly.

## Core concepts

Before choosing a lock, define the resource being protected (a memory object, database row, file, or distributed business entity) and the invariant that must remain true.

- **Critical section:** the code or transaction that must not run concurrently with conflicting work.
- **Contention:** multiple workers attempting to access the same resource at once.
- **Granularity:** how much data one lock covers. Fine-grained locks improve concurrency but are harder to manage; coarse-grained locks are simpler but create more contention.
- **Deadlock:** two or more workers permanently wait for locks held by each other.
- **Starvation:** a worker continually loses access to a lock.
- **Timeout / lease:** a bound on how long lock ownership is valid.

## 1. Mutex (mutual-exclusion lock)

A mutex permits exactly one owner at a time. Other threads wait until the owner releases it.

```text
Thread A: acquire(cartLock) -> update cart -> release(cartLock)
Thread B: acquire(cartLock) -> waits until A releases it
```

### Best for

- Protecting in-memory state inside one process.
- Short, synchronous critical sections such as updating a shared cache map or counter.

### Benefits

- Simple and familiar.
- Guarantees exclusive access when correctly used.

### Risks

- Holding it during slow I/O, network calls, or database work blocks other workers.
- Forgetting to release it can stall the process.
- A mutex does **not** coordinate separate application instances.

### Practical rule

Acquire it as late as possible, release it as early as possible, and always release it with a language construct such as `defer`, `finally`, or RAII.

## 2. Read-write lock (RW lock)

A read-write lock distinguishes reads from writes. Multiple readers may hold the lock together, but a writer needs exclusive access.

```text
Readers R1, R2, R3: allowed concurrently
Writer W: waits for readers; blocks new conflicting access while writing
```

### Best for

- Read-heavy, in-process data: configuration snapshots, routing tables, or reference data.

### Benefits

- Better concurrency than a mutex when reads greatly outnumber writes.

### Risks

- More overhead than a mutex; it can be slower when writes are frequent or critical sections are tiny.
- Some policies can starve writers (constant readers) or readers (frequent writers).
- Upgrading a held read lock to a write lock can deadlock unless the implementation explicitly supports it.

## 3. Spinlock

A spinlock makes the waiting worker repeatedly check the lock instead of sleeping.

```text
while !tryAcquire(lock):
    keep checking
```

### Best for

- Very short waits in low-level code, often kernels or runtime internals.
- Situations where sleeping and waking would cost more than waiting briefly.

### Benefits

- Extremely low handoff latency for tiny critical sections.

### Risks

- Wastes CPU while waiting.
- Dangerous around network calls, disk I/O, or unpredictable work.
- Usually inappropriate for application-level system design.

## 4. Semaphore

A semaphore maintains a number of permits. A worker must acquire one permit before entering; it releases the permit when finished.

- **Binary semaphore:** one permit; behaves similarly to a mutex, though ownership semantics may differ.
- **Counting semaphore:** `N` permits; permits up to `N` concurrent users.

```text
Image processor semaphore: 10 permits
First 10 jobs run; job 11 waits for a permit.
```

### Best for

- Limiting concurrency to a scarce resource: database connections, third-party API calls, CPU-heavy tasks, or uploads.

### Benefits

- Provides backpressure and protects dependencies from overload.
- Models a pool naturally.

### Risks

- A leaked permit gradually reduces capacity, potentially to zero.
- It limits concurrency but does not necessarily protect correctness of shared mutable data.

## 5. Reentrant (recursive) lock

A reentrant lock lets its current owner acquire the same lock again. Internally it tracks an ownership count; releases must match acquisitions.

```text
processOrder() acquires orderLock
  -> validateOrder() also acquires orderLock
  -> both calls return and release once
```

### Best for

- Layered code where public methods can call other lock-taking methods on the same object.

### Benefits

- Avoids self-deadlock for the owning thread.

### Risks

- Can conceal unclear lock boundaries and excessive nesting.
- It does not solve deadlocks between different locks or threads.

## 6. Optimistic locking

Optimistic locking assumes conflicts are uncommon. A worker reads a record with a version, and writes only if that version has not changed.

```sql
UPDATE inventory
SET available = available - 1, version = version + 1
WHERE product_id = :id
  AND version = :previous_version
  AND available > 0;
```

If zero rows are updated, another writer changed the record (or stock ran out). The caller reloads, retries, or returns a conflict.

### Best for

- APIs and databases with relatively rare write conflicts.
- Editing user profiles, documents, or records where a conflict can be surfaced or retried.

### Benefits

- No lock is held while a user thinks or an application performs slow work.
- Scales well under low contention.

### Risks

- High contention creates retries and can cause a retry storm.
- Every write path must correctly handle conflict outcomes.
- A version check must cover the actual invariant; checking the wrong row can still permit anomalies.

## 7. Pessimistic locking

Pessimistic locking prevents conflicts by locking data before an update, usually within a database transaction.

```sql
BEGIN;
SELECT * FROM inventory WHERE product_id = :id FOR UPDATE;
-- validate stock and decrement it
COMMIT;
```

### Best for

- High-conflict resources where retries are expensive or unacceptable.
- Short transactional operations such as allocating a limited seat or balance transfer.

### Benefits

- Clear serialization of conflicting updates.
- Avoids repeated failed work under contention.

### Risks

- Transactions must be short; waiting locks consume database capacity.
- Multiple locked rows can deadlock.
- Behavior and lock scope vary by database and isolation level.

### Practical rule

Lock rows in a consistent order, keep transactions small, set sensible lock timeouts, and retry only known transient database errors.

## 8. Database row, page, table, and advisory locks

Databases offer several lock granularities.

| Lock | What it protects | Typical use | Trade-off |
| --- | --- | --- | --- |
| Row lock | One logical record | Update one order or inventory item | Usually high concurrency |
| Page lock | Physical group of rows | Database-managed optimization | Can block nearby records |
| Table lock | Entire table | DDL, bulk operations, simple engines | Low concurrency |
| Advisory lock | Application-defined key | One worker performs a named job | All participants must honor it |

### Advisory locks

An advisory lock is often obtained with a numeric or string-derived key such as `daily-settlement:tenant-42`. The database enforces exclusivity, but it does not automatically attach the lock to a particular row or protect arbitrary queries. Your application must consistently acquire it before the protected operation.

They are useful for singleton jobs, migrations, or coordinating work among services that already share a database.

## 9. File locks

File locks coordinate access to a file, usually between processes on the same machine.

### Types

- **Shared lock:** many readers may access the file.
- **Exclusive lock:** one writer has exclusive access.
- **Advisory file lock:** cooperating processes voluntarily honor it.
- **Mandatory file lock:** the operating system blocks non-cooperating access; uncommon and platform-dependent.

### Best for

- Local logs, PID files, cache files, or single-host batch processes.

### Risks

- Semantics vary by operating system and network filesystem.
- Not a dependable coordination mechanism across containers or multiple hosts.

## 10. Distributed lock

A distributed lock coordinates work across processes or machines. Typical backends include Redis, ZooKeeper, etcd, Consul, or a database.

```text
Worker A acquires lock:invoice:123
Worker B cannot acquire it, so it waits, retries, or handles the job later.
```

### Best for

- Preventing duplicate execution of a cross-instance operation.
- Electing one active scheduler or leader.
- Coordinating a resource that cannot be protected with a database transaction alone.

### Why it is difficult

Network partitions, delayed messages, process pauses, and backend failover can make an old owner believe it still owns a lock after another worker acquires it. A simple “set a key if absent” is not enough for every correctness-critical workflow.

### Requirements for a safe design

- Use an owner token: only the owner that acquired the lock may release it.
- Use expiration: an abandoned lock must eventually clear.
- Avoid assuming expiration proves the old owner stopped.
- Make the protected operation idempotent.
- Use a strongly coordinated service when correctness demands it.
- Prefer **fencing tokens** for external resources.

## 11. Lease locks

A lease lock is a lock with an expiry time. The owner must finish or renew its lease before expiration.

```text
00:00 Worker A receives lease until 00:30
00:35 Worker A is paused by a long GC pause
00:36 Worker B obtains the expired lease
00:40 Worker A resumes and must not continue as if it still owns the lock
```

Leases improve availability after crashes, but expiration alone cannot prevent a delayed former owner from acting later.

### Fencing tokens

The lock service issues monotonically increasing tokens:

```text
Worker A: token 41
Worker B: token 42
Storage service rejects operations with token 41 after seeing token 42
```

The downstream resource must validate the token. This prevents stale owners from overwriting newer work and is the standard protection for lease-based distributed coordination.

## 12. Lock-free and alternative patterns

Locks are not always the best first choice.

| Pattern | Use it when | Example |
| --- | --- | --- |
| Atomic compare-and-swap | Small independent state transition | Increment a versioned counter |
| Database constraint | Invariant maps to data model | Unique `(user_id, event_id)` registration |
| Conditional update | One-row invariant | Decrement inventory only when positive |
| Idempotency key | Request may be retried | Charge a payment once per request key |
| Queue / single consumer | Work can be serialized per key | Process one account’s events in order |
| Partitioning / sharding | Natural ownership exists | Route all customer updates to one partition |
| Immutable snapshots | Readers dominate | Atomically replace configuration object |

These approaches often reduce lock duration, eliminate distributed coordination, and make failure recovery easier.

## Choosing the right approach

1. Start with the invariant, not the lock: for example, “available inventory never becomes negative.”
2. Keep coordination as close to the data as possible. A database constraint or conditional update is usually preferable to a separate distributed lock for database data.
3. Choose optimistic concurrency for infrequent conflicts; use pessimistic transactions for short, high-conflict operations.
4. Use semaphores to protect capacity, not as a substitute for correctness.
5. Use a distributed lock only when the operation truly spans instances and cannot be made safe with transactions, idempotency, or partitioned ownership.
6. For distributed locks, use timeouts, owner tokens, idempotent handlers, observability, and fencing where stale owners could cause damage.

## Common failure modes and mitigations

| Failure mode | Mitigation |
| --- | --- |
| Deadlock | Consistent lock order, short critical sections, timeouts, retry transient failures |
| Lock never released | `finally`/RAII, leases, monitoring, process cleanup |
| Hot key / excessive contention | Partition the resource, queue by key, reduce lock scope, cache reads |
| Stale distributed-lock owner | Fencing tokens and idempotent writes |
| Lost update | Version checks, conditional updates, suitable transaction isolation |
| Duplicate request | Idempotency keys and durable request-result records |
| Thundering herd after release | Backoff with jitter, queues, bounded concurrency |

## Example: reserving the last item

For inventory stored in one relational database, a conditional update is normally simpler and safer than a distributed lock:

```sql
UPDATE inventory
SET available = available - 1
WHERE sku = :sku AND available > 0;
```

If one row changes, reserve the item in the same transaction. If zero rows change, return “out of stock.” This lets the database serialize the contested state transition without holding a separate application-level lock.

Use a distributed lease only if the workflow must coordinate external systems that cannot participate in that database transaction. Even then, design each external step to be idempotent and protect it against stale owners.

## Summary

- Use **mutexes** and **RW locks** for shared memory within one process.
- Use **semaphores** to cap concurrent use of a resource.
- Use **optimistic locking** for low-conflict writes; **pessimistic locking** for brief, high-conflict transactions.
- Use **database constraints and conditional writes** whenever the invariant lives in the database.
- Treat **distributed locks and leases** as advanced coordination tools: add owner tokens, expiry, idempotency, and fencing when correctness is critical.
