# AnyFS Ecosystem

Pluggable virtual filesystem backends for Rust.

---

## Overview

AnyFS is a three-crate ecosystem:

| Crate | Purpose |
|-------|---------|
| `anyfs-traits` | Minimal contract: `VfsBackend` + core types; re-exports `VirtualPath` |
| `anyfs` | Re-exports `anyfs-traits` + built-in backends (feature-gated) |
| `anyfs-container` | `FilesContainer<B: VfsBackend>` policy layer (limits + least-privilege feature whitelist) |

High-level data flow:

```text
Your application
  -> anyfs-container (FilesContainer: ergonomic paths + policy)
      -> anyfs (built-in backends)
          -> anyfs-traits (VfsBackend + types)
              -> strict-path (VirtualPath, VirtualRoot)
```

---

## Quick example

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut container = FilesContainer::new(MemoryBackend::new());

    container.create_dir_all("/data")?;
    container.write("/data/file.txt", b"hello")?;

    Ok(())
}
```

---

## How to use this manual

| Section | Audience | Purpose |
|---------|----------|---------|
| Overview | Stakeholders | One-page understanding |
| Getting Started | Developers | Practical examples |
| Design & Architecture | Contributors | Detailed design |
| Traits & APIs | Backend authors | Contract and types |
| Implementation | Implementers | Plan + backend guide |
| Review | Contributors | Historical review record |

---

## Status

| Component | Status |
|-----------|--------|
| Design | Complete |
| Implementation | Not started |

---

## Authoritative documents

1. `book/src/architecture/design-overview.md`
2. `book/src/architecture/adrs.md`

If something conflicts with `AGENTS.md`, treat `AGENTS.md` as authoritative.
