# AnyFS - Executive Summary

One-page overview for stakeholders and decision-makers.

---

## What is it?

AnyFS is a virtual filesystem library for Rust with swappable storage backends.

You get a familiar, `std::fs`-aligned API (read/write/create_dir/read_dir/etc.) while choosing where the data lives:
- in-memory (tests)
- a single SQLite database file (portable storage)
- a contained host filesystem directory (sandboxed by a virtual root)

---

## Why does it matter?

| Problem | How AnyFS helps |
|---------|------------------|
| Multi-tenant isolation | One container per tenant, separate namespaces |
| Portability | SQLite backend: a tenant filesystem is a single `.db` file |
| Security | Paths are validated via `VirtualPath` (`strict-path`) |
| Resource control | Built-in limits: max bytes, max file size, max nodes, etc. |
| Testing | In-memory backend is fast and deterministic |

---

## Key design points

- **Three-crate structure**
  - `anyfs-traits`: minimal backend contract (`VfsBackend`) and types
  - `anyfs`: built-in backends (feature-gated), re-exports traits
  - `anyfs-container`: `FilesContainer<B>` policy layer (limits + least privilege)

- **Two-layer path handling**
  - User APIs accept `impl AsRef<Path>` for ergonomics.
  - Backends receive `&VirtualPath` (validated and normalized).

- **Least privilege by default**
  - Advanced behavior is denied unless explicitly enabled per container:
    - symlinks
    - hard links
    - permission mutation

---

## Quick example

```rust
use anyfs::SqliteBackend;
use anyfs_container::ContainerBuilder;

let mut fs = ContainerBuilder::new(SqliteBackend::open_or_create("tenant_123.db")?)
    .max_total_size(100 * 1024 * 1024)
    .build()?;

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

---

## Status

| Phase | Status |
|-------|--------|
| Design | Complete |
| Implementation | Not started |

---

For details, see `book/src/architecture/design-overview.md`.
