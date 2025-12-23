# AnyFS - Virtual filesystem backends for Rust

AnyFS is a small Rust ecosystem for virtual filesystem backends with an optional policy/container layer.

It is designed for:
- portable application storage (SQLite backend = single `.db` file)
- tenant isolation (one container per tenant)
- quotas/limits (enforced in the container layer)
- containment and safe path handling (`strict-path`)

---

## Crates

| Crate | Purpose |
|------|---------|
| `anyfs-traits` | Minimal contract: `VfsBackend` + core types + `VirtualPath` re-export |
| `anyfs` | Re-exports `anyfs-traits` + built-in backends (feature-gated) |
| `anyfs-container` | `FilesContainer<B: VfsBackend>` policy layer (limits + least-privilege feature whitelist) |

---

## Quick example

```rust
use anyfs::SqliteBackend;
use anyfs_container::ContainerBuilder;

let mut fs = ContainerBuilder::new(SqliteBackend::open_or_create("tenant.db")?)
    .max_total_size(100 * 1024 * 1024)
    // advanced features are opt-in (default deny)
    .symlinks()
    .hard_links()
    .build()?;

fs.create_dir_all("/docs")?;
fs.write("/docs/hello.txt", b"hello")?;
let bytes = fs.read("/docs/hello.txt")?;
```

---

## Feature selection

There are two independent knobs:

- Compile-time (Cargo features on `anyfs`): `memory` (default), `sqlite`, `vrootfs`
- Runtime policy (per-container whitelist): `.symlinks()`, `.hard_links()`, `.permissions()`

---

## Documentation

The manual lives in `book/`:
- `book/src/architecture/design-overview.md`
- `book/src/architecture/adrs.md`

Build locally: `mdbook serve book`

---

## Status

Design complete; implementation not started.
