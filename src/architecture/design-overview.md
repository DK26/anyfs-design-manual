# AnyFS - Design Overview

**Status:** Current
**Last updated:** 2025-12-24

---

## What This Project Is

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality with complete separation of concerns.

Anyone can:
- Implement a custom backend for their storage needs
- Compose middleware to add limits, logging, feature gates
- Use the ergonomic `FilesContainer` wrapper

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

**Each layer has exactly one responsibility:**

| Layer | Responsibility |
|-------|----------------|
| `VfsBackend` | Storage + filesystem semantics |
| `Quota<B>` | Quota enforcement |
| `Tracing<B>` | Instrumentation / audit trail |
| `FeatureGuard<B>` | Feature whitelist |
| `FilesContainer<B>` | Ergonomic std::fs-aligned API |

---

## Crates

| Crate | Purpose | Contains |
|-------|---------|----------|
| `anyfs-backend` | Minimal contract | `VfsBackend` trait, `Layer` trait, types, `VfsBackendExt` |
| `anyfs` | Backends + middleware | Built-in backends, `Quota`, `Tracing`, `FeatureGuard` |
| `anyfs-container` | Ergonomic wrapper | `FilesContainer<B>`, `BackendStack` builder |

### Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware)
     |     ^-- vrootfs feature may use strict-path
     |
     +-- anyfs-container (ergonomic wrapper)
```

---

## Core Trait: `VfsBackend` (in `anyfs-backend`)

`VfsBackend` is a path-based trait aligned with `std::fs`. It handles **storage + filesystem semantics only**. No policy, no limits.

```rust
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub trait VfsBackend: Send {
    // Read
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // Write
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // Links
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // Permissions
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // Streaming I/O
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;

    // File size
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;

    // Durability
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;

    // Filesystem info
    fn statfs(&self) -> Result<StatFs, VfsError>;
}
```

---

## Middleware (in `anyfs`)

Each middleware is itself a `VfsBackend` that wraps another backend. This enables composition.

### Quota<B>

Enforces quota limits. Tracks usage and rejects operations that would exceed limits.

```rust
use anyfs::{SqliteBackend, Quota};

let backend = Quota::new(SqliteBackend::open("data.db")?)
    .with_max_total_size(100 * 1024 * 1024)   // 100 MB
    .with_max_file_size(10 * 1024 * 1024)     // 10 MB per file
    .with_max_node_count(10_000)               // 10K files/dirs
    .with_max_dir_entries(1_000)               // 1K entries per dir
    .with_max_path_depth(64);

// Check usage
let usage = backend.usage();
let remaining = backend.remaining();
```

### FeatureGuard<B>

Enforces least-privilege by disabling features by default.

```rust
use anyfs::{MemoryBackend, FeatureGuard};

let backend = FeatureGuard::new(MemoryBackend::new())
    .with_symlinks()                          // Enable symlink operations
    .with_max_symlink_resolution(40)          // Max symlink hops
    .with_hard_links()                        // Enable hard links
    .with_permissions();                      // Enable set_permissions
```

When a feature is disabled, operations return `VfsError::FeatureNotEnabled`.

### Tracing<B>

Integrates with the [tracing](https://docs.rs/tracing) ecosystem for structured logging and instrumentation.

```rust
use anyfs::{SqliteBackend, Tracing};

let backend = Tracing::new(SqliteBackend::open("data.db")?)
    .with_target("anyfs")           // tracing target
    .with_level(tracing::Level::DEBUG);

// Users configure tracing subscribers as they prefer
tracing_subscriber::fmt::init();
```

**Why tracing instead of custom logging?**
- Works with existing tracing infrastructure
- Structured logging with spans
- Compatible with OpenTelemetry, Jaeger, etc.
- Users choose their subscriber (console, file, distributed tracing)

---

## FilesContainer (in `anyfs-container`)

`FilesContainer<B>` is a **thin ergonomic wrapper**. It provides a familiar std::fs-aligned API but does nothing else. All policy (limits, feature gates) is handled by middleware.

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut fs = FilesContainer::new(MemoryBackend::new());

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

---

## Layer Trait (in `anyfs-backend`)

The `Layer` trait (inspired by Tower) standardizes middleware composition:

```rust
/// A layer that wraps a backend to add functionality.
pub trait Layer<B: VfsBackend> {
    type Backend: VfsBackend;
    fn layer(self, backend: B) -> Self::Backend;
}
```

Each middleware provides a corresponding `Layer` implementation:

```rust
// QuotaLayer, TracingLayer, FeatureGuardLayer, etc.
pub struct QuotaLayer { /* config */ }

impl<B: VfsBackend> Layer<B> for QuotaLayer {
    type Backend = Quota<B>;
    fn layer(self, backend: B) -> Self::Backend {
        Quota::new(backend).with_limits(self.limits)
    }
}
```

---

## Composing Middleware

Middleware composes by wrapping. Order matters - innermost applies first.

### Manual Composition

```rust
use anyfs::{SqliteBackend, Quota, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

// Build from inside out:
let backend = SqliteBackend::open("data.db")?;

let limited = Quota::new(backend)
    .with_max_total_size(100 * 1024 * 1024);

let gated = FeatureGuard::new(limited)
    .with_symlinks();

let traced = Tracing::new(gated);

let mut fs = FilesContainer::new(traced);
```

### Layer-based Composition

Use the `Layer` trait for Axum-style composition:

```rust
use anyfs::{SqliteBackend, QuotaLayer, FeatureGuardLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGuardLayer::new().allow_symlinks())
    .layer(TracingLayer::new());
```

### BackendStack Builder

For complex stacks, use `BackendStack` for a fluent API:

```rust
use anyfs_container::BackendStack;

let fs = BackendStack::new(SqliteBackend::open("data.db")?)
    .limited(|l| l
        .max_total_size(100 * 1024 * 1024)
        .max_file_size(10 * 1024 * 1024))
    .feature_gated(|g| g
        .allow_symlinks()
        .allow_hard_links())
    .traced()
    .into_container();
```

---

## Built-in Backends

| Backend | Feature | Description |
|---------|---------|-------------|
| `MemoryBackend` | `memory` (default) | In-memory storage |
| `SqliteBackend` | `sqlite` | Single-file portable database |
| `VRootFsBackend` | `vrootfs` | Host filesystem via strict-path |

---

## Path Handling

All layers use `impl AsRef<Path>`, aligned with `std::fs`:

```rust
// All of these work
fs.write("/file.txt", data)?;
fs.write(String::from("/file.txt"), data)?;
fs.write(Path::new("/file.txt"), data)?;
fs.write(PathBuf::from("/file.txt"), data)?;
```

---

## Security Model

Security is achieved through composition:

| Concern | Solution |
|---------|----------|
| Path containment | Backend-specific (VRootFsBackend uses strict-path) |
| Resource exhaustion | `Quota` enforces quotas |
| Feature restriction | `FeatureGuard` disables dangerous features |
| Audit trail | `Tracing` instruments operations |
| Tenant isolation | Separate backend instances |

**Defense in depth:** Compose multiple middleware layers for comprehensive security.

---

## Extension Traits (in `anyfs-backend`)

The `VfsBackendExt` trait provides convenience methods without modifying `VfsBackend`:

```rust
use serde::{Serialize, de::DeserializeOwned};

/// Extension methods for VfsBackend (auto-implemented for all backends).
pub trait VfsBackendExt: VfsBackend {
    /// Read and deserialize JSON.
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError> {
        let bytes = self.read(path)?;
        serde_json::from_slice(&bytes).map_err(|e| VfsError::Deserialization(e.to_string()))
    }

    /// Serialize and write JSON.
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError> {
        let bytes = serde_json::to_vec(value).map_err(|e| VfsError::Serialization(e.to_string()))?;
        self.write(path, &bytes)
    }

    /// Check if path is a file.
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.metadata(path).map(|m| m.file_type == FileType::File)
    }

    /// Check if path is a directory.
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.metadata(path).map(|m| m.file_type == FileType::Directory)
    }
}

// Blanket implementation for all backends
impl<B: VfsBackend> VfsBackendExt for B {}
```

Users can define their own extension traits for domain-specific operations.

---

## Optional Features

### Bytes Support (feature: `bytes`)

For zero-copy efficiency, enable the `bytes` feature to use `Bytes` instead of `Vec<u8>`:

```toml
anyfs = { version = "0.1", features = ["bytes"] }
```

```rust
use bytes::Bytes;

// With bytes feature, read returns Bytes (O(1) slicing)
let data: Bytes = backend.read("/large-file.bin")?;
let slice = data.slice(1000..2000);  // Zero-copy!
```

**When to use:**
- Large file handling with frequent slicing
- Network-backed storage
- Streaming scenarios

**Default:** `Vec<u8>` (no extra dependency)

---

## Error Types

`VfsError` includes context for better debugging:

```rust
pub enum VfsError {
    /// Path not found.
    NotFound {
        path: PathBuf,
        operation: &'static str,  // "read", "metadata", etc.
    },

    /// Path already exists.
    AlreadyExists {
        path: PathBuf,
        operation: &'static str,
    },

    /// Expected a file, found directory.
    NotAFile { path: PathBuf },

    /// Expected a directory, found file.
    NotADirectory { path: PathBuf },

    /// Directory not empty (for remove_dir).
    DirectoryNotEmpty { path: PathBuf },

    /// Quota exceeded.
    QuotaExceeded {
        limit: u64,
        requested: u64,
        usage: u64,
    },

    /// File size limit exceeded.
    FileSizeExceeded {
        path: PathBuf,
        size: u64,
        limit: u64,
    },

    /// Feature not enabled (from FeatureGuard).
    FeatureNotEnabled {
        feature: &'static str,  // "symlinks", "hard_links", "permissions"
        operation: &'static str,
    },

    /// Serialization error (from VfsBackendExt).
    Serialization(String),

    /// Deserialization error (from VfsBackendExt).
    Deserialization(String),

    /// Backend-specific error.
    Backend(String),

    /// I/O error.
    Io(std::io::Error),
}
```

All error variants include enough context for meaningful error messages.
