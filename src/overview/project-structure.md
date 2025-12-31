# AnyFS - Project Structure

**Status:** Current
**Last updated:** 2025-12-24

---

## Repository Layout

```
anyfs-backend/              # Crate 1: trait + types
  Cargo.toml
  src/
    lib.rs
    backend.rs              # Fs traits (FsRead, FsWrite, FsDir, etc.)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # FsError

anyfs/                      # Crate 2: backends + middleware + ergonomics
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend [feature: memory, default]
      sqlite.rs             # SqliteBackend [feature: sqlite]
      sqlite_cipher.rs      # SqliteCipherBackend [feature: sqlite-cipher]
      stdfs.rs              # StdFsBackend [feature: stdfs]
      vrootfs.rs            # VRootFsBackend [feature: vrootfs]
    middleware/
      quota.rs              # Quota<B>
      restrictions.rs       # Restrictions<B>
      path_filter.rs        # PathFilter<B>
      read_only.rs          # ReadOnly<B>
      rate_limit.rs         # RateLimit<B>
      tracing.rs            # Tracing<B>
      dry_run.rs            # DryRun<B>
      cache.rs              # Cache<B>
      overlay.rs            # Overlay<B1, B2>
    container.rs            # FileStorage<B, M>
    stack.rs                # BackendStack builder
```

---

## Dependency Model

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware + ergonomics)
           ^-- vrootfs feature uses strict-path
```

**Key points:**
- Custom backends depend only on `anyfs-backend`
- `anyfs` provides built-in backends, middleware, and the ergonomic `FileStorage<B, M>` wrapper

---

## Middleware Pattern

```
FileStorage<B, M>
    wraps -> Tracing<B>
        wraps -> Restrictions<B>
            wraps -> Quota<B>
                wraps -> SqliteBackend (or any Fs)
```

Each layer implements `Fs`, enabling composition.

---

## Cargo Features

### Backends
- `memory` — In-memory storage (default)
- `sqlite` — SQLite-backed persistent storage
- `sqlite-cipher` — Encrypted SQLite via SQLCipher (mutually exclusive with `sqlite`)
- `stdfs` — Direct `std::fs` delegation (no containment)
- `vrootfs` — Host filesystem backend with path containment (uses `strict-path`)

### Middleware
Following the **Tower/Axum** pattern, we use feature flags to keep the core lightweight:

- `quota` — Storage limits (default)
- `path-filter` — Glob-based access control (default)
- `restrictions` — Permission/Link control (default)
- `read-only` — Write blocking (default)
- `rate-limit` — Token bucket throttling (default)
- `metrics` — Prometheus integration (requires `prometheus` crate)
- `tracing` — Detailed audit logging (requires `tracing` crate)

Use `default-features = false` to cherry-pick exactly what you need.

---

## Where To Start

- Application usage: [Getting Started Guide](../getting-started/guide.md)
- Trait details: [Layered Traits](../traits/layered-traits.md)
- Middleware: [Design Overview](../architecture/design-overview.md)
- Decisions: [Architecture Decision Records](../architecture/adrs.md)
