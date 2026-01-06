# AnyFS Ecosystem

**An open standard for pluggable virtual filesystem backends in Rust.**

---

## Overview

AnyFS is an **open standard** for virtual filesystem backends using a **Tower-style middleware pattern** for composable functionality.

You get:
- A familiar `std::fs`-aligned API
- Composable middleware (limits, logging, security)
- Choice of storage: memory, SQLite, host filesystem, or custom
- A developer-first goal: make storage composition easy, safe, and enjoyable

---

## Architecture

```
┌─────────────────────────────────────────┐
│  FileStorage<B, R, M>                   │  ← Ergonomics + type-safe marker
├─────────────────────────────────────────┤
│  Middleware (composable):               │
│    Quota<B>                             │  ← Quotas
│    Restrictions<B>                      │  ← Security
│    Tracing<B>                           │  ← Audit
├─────────────────────────────────────────┤
│  Fs                             │  ← Storage
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

**Each layer has one job.** Compose only what you need.

---

## Two-Crate Structure

| Crate           | Purpose                                                             |
| --------------- | ------------------------------------------------------------------- |
| `anyfs-backend` | Minimal contract: `Fs` trait + types                                |
| `anyfs`         | Backends + middleware + mounting + ergonomic `FileStorage<B, R, M>` |

**Note:** Mounting (`FsFuse` + `MountHandle`) is part of the `anyfs` crate behind feature flags (`fuse`, `winfsp`), not a separate crate.

---

## Quick Example

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, FileStorage};

// Layer-based composition
let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(RestrictionsLayer::builder()
        .deny_permissions()
        .build());

let fs = FileStorage::new(backend);

fs.create_dir_all("/data")?;
fs.write("/data/file.txt", b"hello")?;
```

---

## How to Use This Manual

| Section               | Audience        | Purpose                |
| --------------------- | --------------- | ---------------------- |
| Overview              | Stakeholders    | One-page understanding |
| Getting Started       | Developers      | Practical examples     |
| Design & Architecture | Contributors    | Detailed design        |
| Traits & APIs         | Backend authors | Contract and types     |
| Implementation        | Implementers    | Plan + backend guide   |

---

## Future Considerations

These are optional extensions that fit the design but are out of scope for initial release:

- URL-based backend registry and bulk helpers (`FsExt`/utilities)
- Async adapter for remote backends
- Companion shell for interactive exploration
- Copy-on-write overlay and archive backends (zip/tar)

See [Design Overview](./architecture/design-overview.md#future-considerations) for the full list and rationale.

---

## Status

| Component      | Status                                 |
| -------------- | -------------------------------------- |
| Design         | Complete                               |
| Implementation | Not started (mounting roadmap defined) |

---

## Authoritative Documents

1. `AGENTS.md` (for AI assistants)
2. `src/architecture/design-overview.md`
3. `src/architecture/adrs.md`
