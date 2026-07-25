# Reentrant locks

A reentrant lock lets the thread that already owns it acquire it again. The runtime maintains an acquisition count; every acquisition must have a matching release.

```java
final ReentrantLock lock = new ReentrantLock();

void publish() {
  lock.lock();
  try { validateAndPublish(); } finally { lock.unlock(); }
}
void validateAndPublish() {
  lock.lock();
  try { /* inspect and update shared state */ } finally { lock.unlock(); }
}
```

Without reentrancy, `publish` would deadlock when its helper attempted to acquire the same lock. This is useful in object-oriented code where public methods call other lock-taking methods.

## Limits

It only prevents self-deadlock. It cannot resolve the classic two-lock deadlock:

```text
Worker 1 holds account-A and waits for account-B
Worker 2 holds account-B and waits for account-A
```

## Design advice

Use it deliberately, not as an excuse for unclear lock ownership. Keep private helpers explicit about whether the caller must already hold the lock, and avoid invoking arbitrary callbacks while holding it: callback code can reenter unexpectedly and extend the critical section.
