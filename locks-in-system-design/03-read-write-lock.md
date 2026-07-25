# Read-write locks

An RW lock lets multiple readers proceed together, but a writer is exclusive. It helps only when the data is read frequently, writes are relatively rare, and reads must see a stable in-memory structure.

```go
type ConfigStore struct {
  mu sync.RWMutex
  rules map[string]Rule
}

func (s *ConfigStore) Get(name string) (Rule, bool) {
  s.mu.RLock(); defer s.mu.RUnlock()
  r, ok := s.rules[name]
  return r, ok
}

func (s *ConfigStore) Replace(next map[string]Rule) {
  s.mu.Lock(); defer s.mu.Unlock()
  s.rules = next
}
```

## Scheduling policies

Implementations may prefer readers, prefer writers, or aim for fairness. Reader preference can starve a writer during constant traffic. Writer preference can delay readers. Check the language/runtime guarantees rather than assuming a policy.

## Avoid read-to-write upgrades

This pattern is risky:

```text
acquire read lock -> notice missing value -> acquire write lock
```

Two readers can both decide to upgrade, creating a deadlock or a race. Release the read lock, acquire the write lock, then re-check the condition; or use an API with an explicitly safe upgrade operation.

## When not to use one

For tiny maps or frequent writes, a simple mutex may perform better. For configuration, an immutable snapshot atomically swapped into place can let readers avoid locking altogether. See [alternatives](13-alternatives.md).
