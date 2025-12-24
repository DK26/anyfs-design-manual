# AnyFS — API Quick Reference

**Condensed reference for developers**

---

## Installation

```toml
[dependencies]
anyfs = "0.1"
anyfs-container = "0.1"  # Optional
```

With backends and optional features:

```toml
anyfs = { version = "0.1", features = ["sqlite", "vrootfs", "bytes"] }
```

---

## Creating a Backend Stack

```rust
use anyfs::{MemoryBackend, SqliteBackend, Quota, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

// Simple
let fs = FilesContainer::new(MemoryBackend::new());

// With limits
let fs = FilesContainer::new(
    Quota::new(SqliteBackend::open("data.db")?)
        .with_max_total_size(100 * 1024 * 1024)
        .with_max_file_size(10 * 1024 * 1024)
);

// Full stack (manual composition)
let fs = FilesContainer::new(
    Tracing::new(
        FeatureGuard::new(
            Quota::new(SqliteBackend::open("data.db")?)
                .with_max_total_size(100 * 1024 * 1024)
        )
        .with_symlinks()
        .with_hard_links()
    )
);

// Layer-based composition
use anyfs::{QuotaLayer, FeatureGuardLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGuardLayer::new().allow_symlinks())
    .layer(TracingLayer::new());

// BackendStack builder (fluent API)
use anyfs_container::BackendStack;

let fs = BackendStack::new(SqliteBackend::open("data.db")?)
    .limited(|l| l.max_total_size(100 * 1024 * 1024))
    .feature_gated(|g| g.allow_symlinks())
    .traced()
    .into_container();
```

---

## Quota Methods

```rust
Quota::new(backend)
    .with_max_total_size(bytes)      // Total storage limit
    .with_max_file_size(bytes)       // Per-file limit
    .with_max_node_count(count)      // Max files/dirs
    .with_max_dir_entries(count)     // Max entries per dir
    .with_max_path_depth(depth)      // Max nesting

// Query
backend.usage()        // -> Usage { total_size, file_count, ... }
backend.limits()       // -> Limits { max_total_size, ... }
backend.remaining()    // -> Remaining { bytes, can_write, ... }
```

---

## FeatureGuard Methods

```rust
FeatureGuard::new(backend)    // All disabled by default
    .with_symlinks()                  // Enable symlink ops
    .with_max_symlink_resolution(40)  // Max hops
    .with_hard_links()                // Enable hard links
    .with_permissions()               // Enable set_permissions
```

---

## Tracing Methods

```rust
Tracing::new(backend)
    .with_target("anyfs")             // tracing target
    .with_level(tracing::Level::DEBUG)
```

---

## VfsBackendExt Methods

Extension methods available on all backends:

```rust
use anyfs_backend::VfsBackendExt;

// JSON support
let config: Config = fs.read_json("/config.json")?;
fs.write_json("/config.json", &config)?;

// Type checks
if fs.is_file("/path")? { ... }
if fs.is_dir("/path")? { ... }
```

---

## File Operations

```rust
// Check existence
fs.exists("/path")?                     // -> bool

// Metadata
let meta = fs.metadata("/path")?;
meta.size                                // file size
meta.file_type                           // File | Directory | Symlink
meta.permissions
meta.created                             // Option<SystemTime>
meta.modified

// Read
let bytes = fs.read("/path")?;           // -> Vec<u8>
let text = fs.read_to_string("/path")?;  // -> String
let chunk = fs.read_range("/path", 0, 1024)?;

// List directory
let entries = fs.read_dir("/path")?;     // -> Vec<DirEntry>

// Write
fs.write("/path", b"content")?;          // Create or overwrite
fs.append("/path", b"more")?;            // Append

// Directories
fs.create_dir("/path")?;
fs.create_dir_all("/path")?;

// Delete
fs.remove_file("/path")?;
fs.remove_dir("/path")?;                 // Empty only
fs.remove_dir_all("/path")?;             // Recursive

// Move/Copy
fs.rename("/from", "/to")?;
fs.copy("/from", "/to")?;

// Links (requires FeatureGuard)
fs.symlink("/target", "/link")?;
fs.hard_link("/original", "/link")?;
fs.read_link("/link")?;                  // -> PathBuf

// Permissions (requires FeatureGuard)
fs.set_permissions("/path", perms)?;

// File size
fs.truncate("/path", 1024)?;             // Resize to 1024 bytes

// Durability
fs.sync()?;                              // Flush all writes
fs.fsync("/path")?;                      // Flush writes for one file
```

---

## Error Handling

```rust
use anyfs_backend::VfsError;

match result {
    Err(VfsError::NotFound { path, operation }) => {
        // e.g., path="/file.txt", operation="read"
    }
    Err(VfsError::AlreadyExists { path, operation }) => ...
    Err(VfsError::NotADirectory { path }) => ...
    Err(VfsError::NotAFile { path }) => ...
    Err(VfsError::DirectoryNotEmpty { path }) => ...
    Err(VfsError::QuotaExceeded { limit, requested, usage }) => ...
    Err(VfsError::FileSizeExceeded { path, size, limit }) => ...
    Err(VfsError::FeatureNotEnabled { feature, operation }) => ...
    Err(VfsError::Serialization(msg)) => ...   // from VfsBackendExt
    Err(VfsError::Deserialization(msg)) => ... // from VfsBackendExt
    Err(e) => ...
}
```

---

## Built-in Backends

| Type | Feature | Description |
|------|---------|-------------|
| `MemoryBackend` | `memory` (default) | In-memory |
| `SqliteBackend` | `sqlite` | Persistent |
| `VRootFsBackend` | `vrootfs` | Host filesystem |

---

## Middleware

| Type | Purpose |
|------|---------|
| `Quota<B>` | Quota enforcement |
| `FeatureGuard<B>` | Least privilege |
| `Tracing<B>` | Instrumentation (tracing ecosystem) |

---

## Layers

| Layer | Creates |
|-------|---------|
| `QuotaLayer` | `Quota<B>` |
| `FeatureGuardLayer` | `FeatureGuard<B>` |
| `TracingLayer` | `Tracing<B>` |

---

## Type Reference

### From `anyfs-backend`

| Type | Description |
|------|-------------|
| `VfsBackend` | Core trait |
| `Layer` | Middleware composition trait |
| `VfsBackendExt` | Extension methods trait |
| `VfsError` | Error type (with context) |
| `FileType` | `File`, `Directory`, `Symlink` |
| `Metadata` | File/dir metadata |
| `DirEntry` | Directory entry |
| `Permissions` | File permissions |
| `StatFs` | Filesystem stats |

### From `anyfs`

| Type | Description |
|------|-------------|
| `MemoryBackend` | In-memory backend |
| `SqliteBackend` | SQLite backend |
| `VRootFsBackend` | Host FS backend |
| `Quota<B>` | Quota middleware |
| `FeatureGuard<B>` | Feature gate middleware |
| `Tracing<B>` | Tracing middleware |
| `QuotaLayer` | Layer for Quota |
| `FeatureGuardLayer` | Layer for FeatureGuard |
| `TracingLayer` | Layer for Tracing |

### From `anyfs-container`

| Type | Description |
|------|-------------|
| `FilesContainer<B>` | Ergonomic wrapper |
| `BackendStack` | Fluent builder for middleware stacks |
