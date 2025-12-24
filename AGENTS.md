# AGENTS.md - Instructions for AI Assistants

READ THIS FIRST before making any changes to this repository.

Priority: reviews/comments.md is the highest-priority source of truth. If anything conflicts, follow reviews/comments.md and update this file.

---

## Project Overview

AnyFS is an open standard for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality.

**Key design principle:** Complete separation of concerns via composable layers.

---

## Architecture (Tower-style Middleware)

```
┌─────────────────────────────────────────┐
│  FilesContainer<B>                      │  ← Ergonomics only (std::fs API)
├─────────────────────────────────────────┤
│  Middleware (optional, composable):     │
│    Quota<B>                    │  ← Quota enforcement
│    Tracing<B>                    │  ← Instrumentation (tracing ecosystem)
│    FeatureGuard<B>               │  ← Symlink/hardlink whitelist
├─────────────────────────────────────────┤
│  VfsBackend                             │  ← Pure storage + fs semantics
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

Each layer has **exactly one responsibility**:

| Layer | Responsibility |
|-------|----------------|
| `VfsBackend` | Storage + filesystem semantics |
| `Quota<B>` | Quota enforcement |
| `Tracing<B>` | Instrumentation / audit trail (tracing ecosystem) |
| `FeatureGuard<B>` | Feature whitelist (symlinks, hard links, permissions) |
| `FilesContainer<B>` | Ergonomic std::fs-aligned API (thin wrapper) |

---

## Crate Structure

```
anyfs-backend/              # Crate 1: trait + types
  Cargo.toml
  src/
    lib.rs
    backend.rs              # VfsBackend trait
    layer.rs                # Layer trait (Tower-style composition)
    ext.rs                  # VfsBackendExt (extension methods)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # VfsError (with context)

anyfs/                      # Crate 2: backends + middleware
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend [feature: memory, default]
      sqlite.rs             # SqliteBackend [feature: sqlite]
      vrootfs.rs            # VRootFsBackend [feature: vrootfs]
    middleware/
      quota.rs              # Quota<B> + QuotaLayer
      tracing.rs            # Tracing<B> + TracingLayer
      feature_guard.rs      # FeatureGuard<B> + FeatureGuardLayer

anyfs-container/            # Crate 3: ergonomic wrapper
  Cargo.toml
  src/
    lib.rs
    container.rs            # FilesContainer<B> (thin std::fs wrapper)
    stack.rs                # BackendStack (fluent builder)
```

---

## Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware)
     |     ^-- vrootfs feature may use strict-path
     |
     +-- anyfs-container (ergonomic wrapper)
```

---

## VfsBackend Trait (in anyfs-backend)

Pure storage + filesystem semantics. No policy, no limits.

```rust
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub trait VfsBackend: Send {
    // ═══════════════════════════════════════════════════════════════════════
    // READ OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // WRITE OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // LINKS
    // ═══════════════════════════════════════════════════════════════════════
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // STREAMING I/O
    // ═══════════════════════════════════════════════════════════════════════
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // FILE SIZE
    // ═══════════════════════════════════════════════════════════════════════
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // DURABILITY
    // ═══════════════════════════════════════════════════════════════════════
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // FILESYSTEM INFO
    // ═══════════════════════════════════════════════════════════════════════
    fn statfs(&self) -> Result<StatFs, VfsError>;
}
```

---

## Layer Trait (in anyfs-backend)

Standardizes middleware composition (inspired by Tower):

```rust
pub trait Layer<B: VfsBackend> {
    type Backend: VfsBackend;
    fn layer(self, backend: B) -> Self::Backend;
}
```

Each middleware provides a corresponding Layer:
- `QuotaLayer` → `Quota<B>`
- `TracingLayer` → `Tracing<B>`
- `FeatureGuardLayer` → `FeatureGuard<B>`

---

## VfsBackendExt (in anyfs-backend)

Extension methods auto-implemented for all backends:

```rust
pub trait VfsBackendExt: VfsBackend {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError>;
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError>;
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
}

impl<B: VfsBackend> VfsBackendExt for B {}
```

---

## Middleware (in anyfs)

Each middleware implements `VfsBackend` by wrapping another `VfsBackend`.

### Quota<B>

Enforces quota limits. Tracks usage and rejects operations that would exceed limits.

```rust
pub struct Quota<B: VfsBackend> {
    inner: B,
    limits: Limits,
    usage: Usage,
}

impl<B: VfsBackend> Quota<B> {
    pub fn new(inner: B) -> Self;
    pub fn with_max_total_size(self, bytes: u64) -> Self;
    pub fn with_max_file_size(self, bytes: u64) -> Self;
    pub fn with_max_node_count(self, count: u64) -> Self;
    pub fn with_max_dir_entries(self, count: u32) -> Self;
    pub fn with_max_path_depth(self, depth: u32) -> Self;

    pub fn usage(&self) -> &Usage;
    pub fn limits(&self) -> &Limits;
    pub fn remaining(&self) -> Remaining;
}

impl<B: VfsBackend> VfsBackend for Quota<B> {
    // Delegates to inner, checking limits before writes
}
```

### Tracing<B>

Integrates with the [tracing](https://docs.rs/tracing) ecosystem for structured logging.

```rust
pub struct Tracing<B: VfsBackend> {
    inner: B,
    target: &'static str,
    level: tracing::Level,
}

impl<B: VfsBackend> Tracing<B> {
    pub fn new(inner: B) -> Self;
    pub fn with_target(self, target: &'static str) -> Self;
    pub fn with_level(self, level: tracing::Level) -> Self;
}

impl<B: VfsBackend> VfsBackend for Tracing<B> {
    // Delegates to inner, emitting tracing spans/events
}
```

**Why tracing instead of custom logging?**
- Works with existing tracing subscribers
- Structured logging with spans
- Compatible with OpenTelemetry, Jaeger, etc.

### FeatureGuard<B>

Enforces least-privilege by disabling features by default.

```rust
pub struct FeatureGuard<B: VfsBackend> {
    inner: B,
    allow_symlinks: bool,
    allow_hard_links: bool,
    allow_permissions: bool,
    max_symlink_resolution: u32,
}

impl<B: VfsBackend> FeatureGuard<B> {
    pub fn new(inner: B) -> Self;  // All features disabled by default
    pub fn with_symlinks(self) -> Self;
    pub fn with_hard_links(self) -> Self;
    pub fn with_permissions(self) -> Self;
    pub fn with_max_symlink_resolution(self, max: u32) -> Self;
}

impl<B: VfsBackend> VfsBackend for FeatureGuard<B> {
    // Delegates to inner, rejecting disabled operations
}
```

---

## FilesContainer (in anyfs-container)

**Thin ergonomic wrapper only.** No policy, no limits - just std::fs-aligned API.

```rust
use std::path::{Path, PathBuf};
use anyfs_backend::VfsBackend;

pub struct FilesContainer<B: VfsBackend> {
    backend: B,
}

impl<B: VfsBackend> FilesContainer<B> {
    pub fn new(backend: B) -> Self;

    // All methods delegate directly to backend
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    // ... etc (mirrors VfsBackend exactly)
}
```

---

## Usage Examples

### Simple (no middleware)

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut fs = FilesContainer::new(MemoryBackend::new());
fs.write("/hello.txt", b"Hello!")?;
```

### With limits

```rust
use anyfs::{SqliteBackend, Quota};
use anyfs_container::FilesContainer;

let backend = Quota::new(SqliteBackend::open("data.db")?)
    .with_max_total_size(100 * 1024 * 1024)
    .with_max_file_size(10 * 1024 * 1024);

let mut fs = FilesContainer::new(backend);
```

### Full stack (limits + feature gates + tracing)

```rust
use anyfs::{SqliteBackend, Quota, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

let backend = SqliteBackend::open("data.db")?;
let limited = Quota::new(backend)
    .with_max_total_size(100 * 1024 * 1024);
let gated = FeatureGuard::new(limited)
    .with_symlinks();
let traced = Tracing::new(gated);

let mut fs = FilesContainer::new(traced);
```

### Layer-based composition

```rust
use anyfs::{SqliteBackend, QuotaLayer, FeatureGuardLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGuardLayer::new().allow_symlinks())
    .layer(TracingLayer::new());
```

### BackendStack builder (fluent API)

```rust
use anyfs_container::BackendStack;

let fs = BackendStack::new(SqliteBackend::open("data.db")?)
    .limited(|l| l.max_total_size(100 * 1024 * 1024).max_file_size(10 * 1024 * 1024))
    .feature_gated(|g| g.allow_symlinks().allow_hard_links())
    .traced()
    .into_container();
```

---

## Built-in Backends

| Backend | Feature | Description |
|---------|---------|-------------|
| `MemoryBackend` | `memory` (default) | In-memory storage |
| `SqliteBackend` | `sqlite` | Single-file portable database |
| `VRootFsBackend` | `vrootfs` | Host filesystem (contained via strict-path) |

---

## Key Dependencies

| Crate | Used by | Purpose |
|-------|---------|---------|
| `thiserror` | `anyfs-backend` | Error types |
| `tracing` | `anyfs` | Instrumentation (Tracing) |
| `strict-path` | `anyfs` [vrootfs] | VirtualRoot containment |
| `rusqlite` | `anyfs` [sqlite] | SQLite database access |
| `bytes` | `anyfs` [bytes] | Zero-copy byte handling (optional) |
| `serde` | `anyfs-backend` | JSON support in VfsBackendExt |

---

## Optional Features

| Feature | Crate | Description |
|---------|-------|-------------|
| `memory` | `anyfs` | MemoryBackend (default) |
| `sqlite` | `anyfs` | SqliteBackend |
| `vrootfs` | `anyfs` | VRootFsBackend |
| `bytes` | `anyfs` | Use `Bytes` instead of `Vec<u8>` for zero-copy |

---

## Common Mistakes to Avoid

- Do NOT put quota/limit logic in FilesContainer - use Quota
- Do NOT put feature gates in FilesContainer - use FeatureGuard
- Do NOT implement custom logging - use Tracing with tracing ecosystem
- FilesContainer is a thin wrapper only - no policy logic
- Middleware order matters: innermost applies first
- Use `VfsBackendExt` for convenience methods, don't add them to VfsBackend

---

## Async Support

**Decision:** Sync-first, async-ready (ADR-010).

- `VfsBackend` is **synchronous** for v1
- All built-in backends are naturally sync (rusqlite, std::fs, memory)
- API is designed to allow `AsyncVfsBackend` trait later without breaking changes

**Async-ready principles:**
- Trait requires `Send` - works with async executors
- Return types are `Result<T, VfsError>` - compatible with async
- No hidden blocking state
- Users can wrap in `spawn_blocking` if needed

**Future:** When async is needed, add parallel `AsyncVfsBackend` trait with blanket impl for sync backends.

---

## When in Doubt

1. **Where do limits go?** `Quota<B>` middleware
2. **Where do feature gates go?** `FeatureGuard<B>` middleware
3. **Where does logging go?** `Tracing<B>` middleware (uses tracing crate)
4. **What does FilesContainer do?** Thin std::fs-aligned wrapper only
5. **Path type everywhere?** `impl AsRef<Path>` (std::fs aligned)
6. **Is strict-path used?** Only internally by VRootFsBackend
7. **Sync or async?** Sync for v1, async-ready for future
8. **How to compose middleware?** Use `Layer` trait or `BackendStack` builder
9. **Where do convenience methods go?** `VfsBackendExt` (extension trait)
10. **Zero-copy bytes?** Enable `bytes` feature for `Bytes` instead of `Vec<u8>`
