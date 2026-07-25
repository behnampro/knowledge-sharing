# Leases and fencing tokens

A lease is a time-bounded distributed lock. Expiry lets another worker recover when the owner crashes, but a delayed old owner may wake up later. Clock skew, garbage-collection pauses, and network partitions make this normal rather than exceptional.

## Fencing-token solution

Each successful acquisition gets a strictly increasing number.

```text
Worker A acquires lease -> fencing token 101
Worker B acquires after A expires -> fencing token 102
Resource accepts operations only with a token >= the highest token it saw
```

```mermaid
sequenceDiagram
  participant A as A (token 101)
  participant B as B (token 102)
  participant R as Protected storage
  B->>R: write, token 102
  R-->>B: accepted; highest=102
  A->>R: delayed write, token 101
  R-->>A: rejected as stale
```

The protected resource—not merely the lock service—must enforce the token. For example, a database update can include `WHERE last_fence < :token`, then persist the new fence atomically with the change.

```sql
UPDATE export_target
SET payload = :payload, last_fence = :token
WHERE id = :id AND last_fence < :token;
```

## Renewal

Renew before expiry only if the owner token still matches. Treat a failed or uncertain renewal as loss of ownership: stop work, do not “try one last write,” and recover using idempotent state.
