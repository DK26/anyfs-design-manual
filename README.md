# VFS Ecosystem

**Switchable virtual filesystem backends for Rust**

---

## Overview

Two crates:

| Crate | Purpose |
|-------|---------|
| `vfs-switchable` | VFS trait with swappable backends (VRootFs, Memory, SQLite) |
| `vfs-container` | Wraps vfs-switchable, adds capacity limits and isolation |

```
┌─────────────────────────────────────────┐
│  Your Application                       │
├─────────────────────────────────────────┤
│  vfs-container (quotas, isolation)      │
├─────────────────────────────────────────┤
│  vfs-switchable (VfsBackend trait)      │
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │
└──────────┴──────────┴───────────────────┘
```

## Quick Example

```rust
use vfs_switchable::{VfsBackend, MemoryBackend};

fn save(vfs: &mut impl VfsBackend) -> Result<(), VfsError> {
    vfs.create_dir_all("/data")?;
    vfs.write("/data/file.txt", b"hello")?;
    Ok(())
}

let mut mem = MemoryBackend::new();
save(&mut mem)?;
```

## Documentation

**Authoritative design document:** [`vfs-design.md`](./vfs-design.md)

## Status

| Component | Status |
|-----------|--------|
| Design | ✅ Path type decided (`impl AsRef<Path>`) |
| Implementation | 🔲 Not started |

## Open Questions

See [Section 6 of the design doc](./vfs-design.md#6-open-design-questions) for remaining decisions:

1. ~~**Path type in trait**~~ — ✅ Resolved: `impl AsRef<Path>`
2. **Symlink support** — Include in v1 or defer?
3. **Usage discovery** — How does container know existing usage?
