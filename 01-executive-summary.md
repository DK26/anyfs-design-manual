# VFS Container — Executive Summary

**One-page overview for stakeholders and decision-makers**

---

## What Is It?

VFS Container is a **virtual filesystem library** for Rust that stores files in a portable database (SQLite) instead of the operating system's filesystem.

```
Traditional Filesystem          VFS Container
─────────────────────          ─────────────────
/home/user/docs/               ┌─────────────────┐
├── report.pdf        →        │  tenant_1.db    │  ← Single portable file
├── notes.txt                  │  (SQLite)       │
└── images/                    └─────────────────┘
    └── photo.jpg
```

---

## Why Does It Matter?

| Problem | How VFS Container Solves It |
|---------|----------------------------|
| **Multi-tenant isolation** | Each tenant gets their own database file. Complete namespace isolation. No chance of data leakage. |
| **Portability** | A SQLite file works on Windows, Mac, Linux. Copy it, email it, version it. |
| **Security** | No connection to host filesystem. Path traversal attacks are structurally impossible. |
| **Resource control** | Built-in quotas: max storage, max file size, max files. Prevents abuse. |
| **Testing** | In-memory backend for fast, deterministic tests. No temp file cleanup. |

---

## Who Is It For?

- **SaaS platforms** needing per-tenant file storage
- **AI/ML systems** requiring sandboxed file operations
- **Desktop applications** wanting portable user data
- **Testing frameworks** needing reproducible filesystem state
- **Embedded systems** needing portable storage without OS dependencies

---

## Key Properties

| Property | Guarantee |
|----------|-----------|
| **Isolated** | Virtual paths never touch the host filesystem |
| **Portable** | Single-file database, cross-platform |
| **Transactional** | All operations are atomic (via SQLite) |
| **Bounded** | Configurable limits on size, file count, depth |
| **Extensible** | Pluggable backends (SQLite, memory, custom) |

---

## What It's NOT

- ❌ Not a replacement for OS filesystems
- ❌ Not a container runtime (Docker, etc.)
- ❌ Not a distributed filesystem
- ❌ Not optimized for maximum throughput
- ❌ Cannot execute code from stored files

---

## Technical Approach

The library separates **filesystem logic** from **storage**:

```
┌─────────────────────────────────────┐
│  FilesContainer (API)               │  ← Your code talks to this
│  - read, write, copy, delete        │
│  - path resolution, symlinks        │
│  - quota enforcement                │
├─────────────────────────────────────┤
│  StorageBackend (trait)             │  ← Pluggable storage
│  - SQLite (default)                 │
│  - In-memory (testing)              │
│  - Custom (your implementation)     │
└─────────────────────────────────────┘
```

This means:
- Swapping storage backends doesn't change application code
- Custom backends can be implemented in hours, not weeks
- All filesystem complexity is handled once, in the core

---

## Project Status

| Phase | Status |
|-------|--------|
| Design | ✅ Complete (this document) |
| Core types | 🔲 Not started |
| Memory backend | 🔲 Not started |
| SQLite backend | 🔲 Not started |
| Full API | 🔲 Not started |
| Documentation | 🔲 Not started |
| Release | 🔲 Not started |

**Estimated timeline:** ~11 weeks to v1.0

---

## Quick Example

```rust
use vfs::{FilesContainer, SqliteBackend, VirtualPath};

// Create a container backed by SQLite
let container = FilesContainer::builder()
    .backend(SqliteBackend::open_or_create("tenant_123.db")?)
    .max_total_size(100 * 1024 * 1024)  // 100 MB quota
    .build()?;

// Use familiar filesystem operations
container.mkdir(&VirtualPath::new("/documents")?)?;
container.write(&VirtualPath::new("/documents/hello.txt")?, b"Hello!")?;

let content = container.read(&VirtualPath::new("/documents/hello.txt")?)?;
// content == b"Hello!"
```

---

## Decision Needed

We are seeking review and approval of the design before implementation begins.

**Review deadline:** ____-__-__

**Reviewers:**
- [ ] _______________
- [ ] _______________
- [ ] _______________

---

*For technical details, see the full [Design Document](./anyfs-container-design.md).*
