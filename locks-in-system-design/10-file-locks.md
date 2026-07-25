# File locks

File locks coordinate access to a file between processes, primarily on one host. They may be shared (multiple readers) or exclusive (one writer), advisory or mandatory.

```python
# Conceptual POSIX-style pattern; exact API differs by platform
with open("/var/run/worker.lock", "w") as f:
    acquire_exclusive_file_lock(f)
    run_singleton_worker()
    release_file_lock(f)
```

## Good uses

- A local singleton process.
- Avoiding simultaneous writes to a cache or generated artifact.
- Coordinating a legacy batch job on one machine.

## Caveats

- Most file locks are advisory: a process that ignores the protocol can still access the file.
- Semantics differ across Windows, Unix, containers, and NFS/network filesystems.
- A lock file's existence is not necessarily proof that its owner is alive. Prefer OS-managed lock ownership or store owner/lease metadata.

Do not rely on file locks as cross-pod distributed coordination. Use a database, queue, or a coordination service instead.
