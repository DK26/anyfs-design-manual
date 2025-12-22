# VFS Container — Comparison & Positioning

**How VFS Container compares to existing solutions**

---

## Executive Comparison

| Solution | Type | Isolation | Portable | Multi-tenant | Backend Agnostic |
|----------|------|-----------|----------|--------------|------------------|
| **VFS Container** | Library | ✅ Complete | ✅ Single file | ✅ Built-in | ✅ Trait-based |
| SQLAR | Archive format | ❌ None | ✅ Single file | ❌ No | ❌ SQLite only |
| libsqlfs | FUSE filesystem | ⚠️ Via mount | ✅ Single file | ❌ No | ❌ SQLite only |
| AgentFS | Agent sandbox | ⚠️ Namespace-based | ✅ Single file | ⚠️ Partial | ❌ SQLite only |
| memfs | In-memory FS | ✅ Complete | ❌ No persistence | ❌ No | ❌ Memory only |
| vfs crate | Abstraction | ⚠️ Depends on backend | ⚠️ Backend-dependent | ❌ No | ✅ Trait-based |
| Docker/Containers | Runtime | ✅ Complete | ❌ Complex | ✅ Yes | N/A |

---

## Detailed Comparisons

### vs. SQLAR (SQLite Archive)

**What it is:** SQLite's official archive format. Stores files in a simple table with optional compression.

**Schema:**
```sql
CREATE TABLE sqlar(
  name TEXT PRIMARY KEY,
  mode INT,
  mtime INT,
  sz INT,
  data BLOB
);
```

| Aspect | SQLAR | VFS Container |
|--------|-------|---------------|
| **Abstraction** | None (raw SQL) | High-level API |
| **Transactions** | Manual | Automatic |
| **Backend flexibility** | SQLite only | Pluggable |
| **Features** | Basic files | Symlinks, hard links, xattrs |
| **Capacity limits** | None | Built-in |
| **Path handling** | String keys | Validated VirtualPath |

**When to use SQLAR:** Simple archive/restore of files. No application logic needed.

**When to use VFS Container:** Application needs filesystem-like operations, isolation, or multi-tenancy.

---

### vs. libsqlfs (Guardian Project)

**What it is:** FUSE filesystem backed by SQLite, developed for secure mobile storage.

| Aspect | libsqlfs | VFS Container |
|--------|----------|---------------|
| **Language** | C | Rust |
| **Interface** | FUSE mount | Library API |
| **Host integration** | Mounts to host FS | No host access |
| **POSIX compliance** | Full | Intentionally not |
| **Encryption** | Via SQLCipher | Backend concern |
| **Use case** | Encrypted filesystem | Application data container |

**When to use libsqlfs:** Need a real mounted filesystem with encryption (e.g., mobile app secure storage).

**When to use VFS Container:** Application controls all file access, no mount needed.

---

### vs. AgentFS (Turso)

**What it is:** SQLite-backed virtual filesystem for AI agent sandboxing.

| Aspect | AgentFS | VFS Container |
|--------|---------|---------------|
| **Focus** | AI agent sandboxing | General-purpose |
| **Features** | POSIX-like + audit log + K/V store | Core filesystem + optional features |
| **Isolation** | Linux namespaces | Structural (no host paths) |
| **FUSE support** | Yes | No |
| **Backend** | SQLite only | Pluggable |
| **Maturity** | Alpha | Design phase |

**When to use AgentFS:** AI/ML agents needing auditable, sandboxed operations with FUSE mount capability.

**When to use VFS Container:** Need backend flexibility, simpler model, or don't need FUSE.

---

### vs. vfs Crate (Rust)

**What it is:** Rust crate providing virtual filesystem abstraction with multiple backends.

| Aspect | vfs crate | VFS Container |
|--------|-----------|---------------|
| **Trait design** | Path-based operations | Node/edge operations |
| **Backend complexity** | Must handle paths, resolution | Just stores nodes/edges |
| **Transactions** | Not enforced | Mandatory |
| **Feature flags** | No | Yes (symlinks, etc.) |
| **Capacity limits** | No | Yes |
| **SQLite backend** | Not included | Reference implementation |

**When to use vfs crate:** Need drop-in replacement for `std::fs` with same API.

**When to use VFS Container:** Need simpler backend implementation, transactions, limits.

---

### vs. Docker/Containers

**What it is:** OS-level virtualization with isolated filesystem namespaces.

| Aspect | Containers | VFS Container |
|--------|------------|---------------|
| **Isolation level** | OS process | Library |
| **Overhead** | Process + namespace | None |
| **Portability** | Requires runtime | Single file |
| **Use case** | Run applications | Store data |
| **Complexity** | High | Low |

**When to use containers:** Running untrusted code, full environment isolation.

**When to use VFS Container:** Just need isolated data storage, no code execution.

---

### vs. In-Memory Filesystems (memfs, pyfakefs)

**What they are:** Complete in-memory filesystem implementations for testing.

| Aspect | memfs/pyfakefs | VFS Container |
|--------|----------------|---------------|
| **Persistence** | None | Yes (SQLite) |
| **Primary use** | Testing | Production + testing |
| **API** | Patches OS APIs | Own API |
| **Backend** | Memory only | Pluggable |

**When to use memfs:** Testing code that uses `fs` module directly.

**When to use VFS Container:** Need persistence, or code designed for VFS Container API.

---

## Positioning Matrix

```
                        Backend Agnostic
                              │
                              │
         vfs crate ───────────┼─────────── VFS Container ◄── YOU ARE HERE
                              │                 │
                              │                 │
    Filesystem Integration    │                 │    Application Integration
              ▲               │                 │              ▲
              │               │                 │              │
              │               │                 │              │
         libsqlfs ────────────┼─────────── AgentFS
              │               │                 │
              │               │                 │
              └───────────────┼─────────────────┘
                              │
                        SQLite Only
```

---

## Feature Matrix

| Feature | VFS Container | SQLAR | libsqlfs | AgentFS | vfs crate |
|---------|--------------|-------|----------|---------|-----------|
| Directories | ✅ | ✅ | ✅ | ✅ | ✅ |
| Files | ✅ | ✅ | ✅ | ✅ | ✅ |
| Symlinks | ✅ Optional | ❌ | ✅ | ✅ | ⚠️ Backend |
| Hard links | ✅ Optional | ❌ | ✅ | ❌ | ⚠️ Backend |
| Permissions | ✅ Optional | ⚠️ mode only | ✅ | ✅ | ⚠️ Backend |
| Extended attrs | ✅ Optional | ❌ | ✅ | ❌ | ❌ |
| Transactions | ✅ Required | Manual | ✅ | ✅ | ❌ |
| Capacity limits | ✅ Built-in | ❌ | ❌ | ❌ | ❌ |
| Streaming I/O | 🔮 Future | ❌ | ✅ | ✅ | ⚠️ Backend |
| Async API | 🔮 Future | N/A | ❌ | ⚠️ Partial | ⚠️ Backend |
| FUSE mount | ❌ Intentionally | ❌ | ✅ | ✅ | ⚠️ Backend |
| Compression | Backend | ✅ deflate | ❌ | ❌ | ⚠️ Backend |
| Encryption | Backend | ❌ | ✅ SQLCipher | ❌ | ⚠️ Backend |

Legend: ✅ Yes | ❌ No | ⚠️ Partial/depends | 🔮 Planned

---

## When to Use VFS Container

### ✅ Good Fit

- **Multi-tenant SaaS** — Each tenant gets isolated storage with quotas
- **Desktop applications** — Portable user data in a single file
- **Testing** — Deterministic, isolated filesystem for tests
- **Embedded systems** — No OS filesystem dependency
- **AI/ML pipelines** — Sandboxed data storage without execution risk
- **Plugin systems** — Isolated storage per plugin
- **Game save data** — Self-contained, portable saves

### ❌ Not a Good Fit

- **Need to mount to host OS** — Use libsqlfs or FUSE-based solution
- **Need to execute code from storage** — Use containers
- **Need maximum throughput** — Use native filesystem
- **Need distributed storage** — Use distributed database
- **Need POSIX compatibility** — Use libsqlfs

---

## Migration Paths

### From raw SQLite/SQLAR

```rust
// Old: raw SQL
conn.execute("INSERT INTO sqlar(name, data) VALUES (?, ?)", [path, data])?;

// New: VFS Container
container.write(&VirtualPath::new(path)?, data)?;
```

### From std::fs

```rust
// Old: host filesystem
std::fs::write("/data/file.txt", content)?;

// New: VFS Container
container.write(&VirtualPath::new("/data/file.txt")?, content)?;
```

### From vfs crate

```rust
// Old: vfs crate
let fs: VfsPath = MemoryFS::new().into();
fs.join("file.txt")?.create_file()?.write_all(b"content")?;

// New: VFS Container  
let container = FilesContainer::new(MemoryBackend::new());
container.write(&VirtualPath::new("/file.txt")?, b"content")?;
```

---

## Summary

**VFS Container occupies a unique niche:**

1. **More structured than SQLAR** — Proper API, transactions, limits
2. **More portable than libsqlfs** — No FUSE, no mount, pure library
3. **More flexible than AgentFS** — Pluggable backends, simpler model
4. **More opinionated than vfs crate** — Transactions required, features opt-in
5. **Lighter than containers** — Just data storage, no runtime

If you need a **safe, portable, application-level virtual filesystem** with **pluggable storage** and **built-in resource limits**, VFS Container is designed for you.

---

*For technical details, see the [Design Document](./vfs-container-design.md).*
