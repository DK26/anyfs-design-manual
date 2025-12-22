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

## How to Use This Manual

This manual is organized into logical sections for different audiences:

| Section | Audience | Purpose |
|---------|----------|---------|
| [Overview](./overview/executive-summary.md) | Stakeholders, Decision-makers | High-level understanding |
| [Getting Started](./getting-started/guide.md) | Developers | Practical introduction |
| [Design & Architecture](./architecture/design-overview.md) | Architects, Contributors | Technical deep-dive |
| [Comparisons](./comparisons/positioning.md) | Evaluators | How we compare to alternatives |
| [Implementation](./implementation/backend-guide.md) | Backend Implementers | Custom backend creation |
| [Review & Decisions](./review/decisions.md) | Contributors | Design decisions and rationale |

## Status

| Component | Status |
|-----------|--------|
| Design | Completed |
| Implementation | Not started |

## Authoritative Documents

The following documents are authoritative sources of truth:

1. **[VFS Container Design (RFC)](./architecture/vfs-container-design.md)** - Full technical specification
2. **[Architecture Decision Records](./architecture/adrs.md)** - Key design decisions
3. **[Review Response & Decisions](./review/decisions.md)** - Final decisions after review

---

*For questions about specific features, start with the [Getting Started Guide](./getting-started/guide.md).*
