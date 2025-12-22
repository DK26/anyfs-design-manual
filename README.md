# VFS Ecosystem

**Switchable virtual filesystem backends for Rust**

---

## Overview

Two crates:

| Crate | Purpose |
|-------|---------|
| `anyfs` | VFS trait with swappable backends (Fs, Memory, SQLite) |
| `anyfs-container` | Wraps anyfs, adds capacity limits and isolation |

```
┌─────────────────────────────────────────┐
│  Your Application                       │
├─────────────────────────────────────────┤
│  anyfs-container (quotas, isolation)      │
├─────────────────────────────────────────┤
│  anyfs (VfsBackend trait)      │
├──────────┬──────────┬───────────────────┤
│ VRootFs│  Memory  │  SQLite           │
└──────────┴──────────┴───────────────────┘
```

## Quick Example

```rust
use anyfs::{VfsBackend, MemoryBackend};

fn save(vfs: &mut impl VfsBackend) -> Result<(), VfsError> {
    vfs.create_dir_all("/data")?;
    vfs.write("/data/file.txt", b"hello")?;
    Ok(())
}

let mut mem = MemoryBackend::new();
save(&mut mem)?;
```

## Documentation

**Authoritative design document:** [`book/src/architecture/design-overview.md`](./book/src/architecture/design-overview.md)

Browse the full documentation with `mdbook serve book/`.

## Status

| Component | Status |
|-----------|--------|
| Design | ✅ Complete |
| Implementation | 🔲 Not started |

## Key Design Decisions

See the [Architecture Decision Records](./book/src/architecture/adrs.md) for details:

1. ✅ **Path type**: VfsBackend uses `&VirtualPath`, FilesContainer uses `impl AsRef<Path>`
2. ✅ **Error paths**: Use `VirtualPath`
3. ✅ **Symlink/hardlink support**: Built-in to all backends (simulated for Memory/SQLite, real for VRootFs)
