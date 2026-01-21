# AnyFS - Project Structure

**Status:** Target layout (design spec)
**Last updated:** 2025-12-24

---

This manual describes the intended code repository layout; this repository contains documentation only.

## Repository Layout

```
anyfs-backend/              # Crate 1: traits + types (minimal dependencies)
  Cargo.toml
  src/
    lib.rs
    traits/
      fs_read.rs            # FsRead trait
      fs_write.rs           # FsWrite trait
      fs_dir.rs             # FsDir trait
      fs_link.rs            # FsLink trait
      fs_permissions.rs     # FsPermissions trait
      fs_sync.rs            # FsSync trait
      fs_stats.rs           # FsStats trait
      fs_path.rs            # FsPath trait (canonicalization, blanket impl)
      fs_inode.rs           # FsInode trait
      fs_handles.rs         # FsHandles trait
      fs_lock.rs            # FsLock trait
      fs_xattr.rs           # FsXattr trait
    layer.rs                # Layer trait (Tower-style)
    ext.rs                  # FsExt (extension methods)
    markers.rs              # SelfResolving marker trait
    path_resolver.rs        # PathResolver trait (pluggable resolution)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # FsError

anyfs/                      # Crate 2: framework glue (simple backends + middleware + ergonomics)
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend (in-memory, simple)
      stdfs.rs              # StdFsBackend (thin std::fs wrapper)
      vrootfs.rs            # VRootFsBackend (std::fs + path containment)
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
    resolvers/
      iterative.rs          # IterativeResolver (default)
      noop.rs               # NoOpResolver (for SelfResolving backends)
      caching.rs            # CachingResolver (LRU cache wrapper)
    container.rs            # FileStorage<B>
    stack.rs                # BackendStack builder
```

### Ecosystem Crates (Separate Repositories)

Complex backends with internal runtime requirements:

```
anyfs-sqlite/               # SQLite backend (connection pooling, WAL, sharding)
anyfs-indexed/              # Hybrid backend (SQLite index + disk blobs)
anyfs-s3/                   # Third-party: S3 backend
anyfs-redis/                # Third-party: Redis backend
```

### Testing Crate

```
anyfs-test/                 # Conformance test suite for backend implementers
  src/
    lib.rs
    conformance/            # Test generators for each trait level
      fs_tests.rs           # Fs-level tests
      fs_full_tests.rs      # FsFull-level tests
      fs_fuse_tests.rs      # FsFuse-level tests
    prelude.rs              # Re-exports for test files
```

---

## Dependency Model

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware + ergonomics)
     |     ^-- vrootfs feature uses strict-path
```

**Key points:**
- Custom backends depend only on `anyfs-backend`
- `anyfs` provides built-in backends, middleware, mounting (behind feature flags), and the ergonomic `FileStorage<B>` wrapper

---

## Middleware Pattern

```
FileStorage<B>
    wraps -> Tracing<B>
        wraps -> Restrictions<B>
            wraps -> Quota<B>
                wraps -> MemoryBackend (or any Fs)
```

Each layer implements `Fs`, enabling composition.

---

## Cargo Features

### Backends (anyfs crate)
- `memory` — In-memory storage (default)
- `stdfs` — Direct `std::fs` delegation (no containment)
- `vrootfs` — Host filesystem backend with path containment (uses `strict-path`)

### Ecosystem Backends (Separate Crates)
Complex backends live in their own crates:
- `anyfs-sqlite` — SQLite-backed persistent storage (pooling, WAL, sharding, encryption)
- `anyfs-indexed` — SQLite index + disk blobs for large file performance

### Middleware (MVP Scope)
Core middleware is always available (no feature flags needed):
- Quota, PathFilter, Restrictions, ReadOnly, RateLimit, Cache, DryRun, Overlay

Optional middleware with external dependencies:
- `tracing` — Detailed audit logging (requires `tracing` crate)

### Mounting (Platform Features)
- `fuse` — Mount as filesystem on Linux/macOS (requires `fuser` crate)
- `winfsp` — Mount as filesystem on Windows (requires `winfsp` crate)

Use `default-features = false` to cherry-pick exactly what you need.

### Middleware (Future Scope)

- `metrics` — Prometheus integration (requires `prometheus` crate)

---

## Where To Start

- Application usage: [Getting Started Guide](../getting-started/guide.md)
- Trait details: [Layered Traits](../traits/layered-traits.md)
- Middleware: [Design Overview](../architecture/design-overview.md)
- Decisions: [Architecture Decision Records](../architecture/adrs.md)
