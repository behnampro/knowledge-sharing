# Database and advisory locks

The database chooses and manages many locks automatically, but lock scope depends on its engine, indexes, isolation level, and query plan.

| Type | Scope | Typical consequence |
| --- | --- | --- |
| Row | One logical row | High concurrency for keyed updates |
| Page / range | Nearby physical or indexed keys | Can block inserts/updates that appear unrelated |
| Table | Whole table | Strong coordination, low concurrency |
| Metadata | Schema object | DDL or long queries can block deployment |
| Advisory | Application-defined key | Only code that agrees to use it is coordinated |

## Isolation is related but different

`READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE` define which concurrent effects a transaction may observe. They may use locks, MVCC, or both. Do not assume `SELECT` locks rows without checking database documentation and the exact query form.

## Advisory-lock example

Use an advisory lock for “only one worker runs tenant 42's daily settlement.” The lock key should be stable and namespaced, such as a hash of `settlement:tenant:42`.

```text
tryAcquire(advisoryKey) -> run job -> release in finally
```

Advisory locks do not automatically protect a row or make a transaction correct. Every writer participating in that protocol must acquire the same key, and job effects should still be idempotent.

## Observe production behavior

Track lock wait time, deadlock count, slow transactions, and the most-contended tables/keys. A missing index can expand lock ranges and look like an application concurrency bug.
