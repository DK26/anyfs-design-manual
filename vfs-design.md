# VFS Ecosystem — Design Document

**Version:** 0.1.0
**Status:** Draft  
**Last Updated:** 2025-12-22

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Project 1: anyfs](#3-project-1-anyfs)
4. [Project 2: anyfs-container](#4-project-2-anyfs-container)
5. [Backend Implementations](#5-backend-implementations)
6. [Open Design Questions](#6-open-design-questions)
7. [Implementation Plan](#7-implementation-plan)
8. [Appendix: Comparison with Alternatives](#appendix-comparison-with-alternatives)

---

## 1. Overview

### 1.1 What Is This?

A layered virtual filesystem ecosystem for Rust:

- **Project 1 (`anyfs`)**: Generic VFS abstraction with swappable backends
- **Project 2 (`anyfs-container`)**: Tenant isolation layer with capacity limits, built on Project 1

### 1.2 Goals

| Goal | Description |
|------|-------------|
| **Switchable I/O** | Write code once, swap storage backends without changes |
| **Tenant Containment** | Isolate tenants via filesystem boundaries and quotas |
| **Backend Extensibility** | Users can implement custom backends |
| **Multiple Storage Options** | Filesystem, SQLite, Memory out of the box |

### 1.3 Non-Goals

- POSIX compliance
- Async (initially)
- Streaming I/O (initially)
- Distributed storage

---

## 2. Architecture

### 2.1 Layer Diagram

```
┌─────────────────────────────────────────┐
│  Your Application                       │  ← Uses clean API
├─────────────────────────────────────────┤
│  anyfs-container                          │  ← Quotas, isolation
│  FilesContainer<B: VfsBackend>          │
├─────────────────────────────────────────┤
│  anyfs                         │  ← Core abstraction
│  VfsBackend trait                       │
├──────────┬──────────┬───────────────────┤
│ Fs  │  Memory  │  SQLite           │  ← Backends
│ Backend  │  Backend │  Backend          │
└──────────┴──────────┴───────────────────┘
     │
     ▼
┌──────────────────┐
│  strict-path     │  ← VirtualRoot for containment
│  VirtualRoot     │
└──────────────────┘
```

### 2.2 Separation of Concerns

| Layer | Responsibility |
|-------|---------------|
| `anyfs` | Defines `VfsBackend` trait, provides backend implementations |
| `anyfs-container` | Wraps any `VfsBackend`, adds capacity limits and usage tracking |
| Application | Uses `FilesContainer`, doesn't know about backend details |

### 2.3 Containment Strategies

All three backends achieve isolation differently:

| Backend | Containment Mechanism |
|---------|----------------------|
| `VRootFsBackend` | `strict-path::VirtualRoot` — real directory acts as root, paths clamped |
| `SqliteBackend` | Each container is a `.db` file — complete isolation by file |
| `MemoryBackend` | Each container is a separate instance — isolation by process memory |

---

## 3. Project 1: anyfs

### 3.1 Purpose

Provide a trait for virtual filesystem operations that any storage backend can implement.

### 3.2 The VfsBackend Trait

```rust
/// A virtual filesystem backend.
///
/// Implementations provide storage; callers get uniform I/O.
pub trait VfsBackend: Send {
    // ─────────────────────────────────────────────────────────
    // Read operations
    // ─────────────────────────────────────────────────────────

    /// Read entire file contents
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;

    /// Read a range of bytes from a file
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;

    /// Get metadata for a path
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;

    /// Check if path exists
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;

    /// List directory contents
    fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;

    // ─────────────────────────────────────────────────────────
    // Write operations
    // ─────────────────────────────────────────────────────────

    /// Write entire file contents (create or overwrite)
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;

    /// Append to file
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;

    /// Create directory
    fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Create directory and all parents
    fn mkdir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove file or empty directory
    fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove directory and all contents
    fn remove_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Rename/move
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    /// Copy file
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
}
```

**Path type rationale:** `&VirtualPath` provides stronger type safety than `impl AsRef<Path>`:
- **Containment guarantee**: `VirtualPath` is pre-validated and normalized — it cannot escape root
- **Lexical resolution**: `..` is resolved lexically, preventing path traversal attacks
- **Backend simplicity**: Backends receive safe paths; no need to re-validate
- **Consistency**: All backends use the same path semantics

Each backend converts `VirtualPath` to its internal representation:
- `VRootFsBackend`: Converts to `strict-path::VirtualPath` for filesystem access
- `MemoryBackend`: Uses `VirtualPath` as HashMap key directly
- `SqliteBackend`: Stores as string in database

### 3.3 VirtualPath (re-exported from strict-path)

`anyfs` re-exports `VirtualPath` from the `strict-path` crate:

```rust
// In anyfs/src/lib.rs
pub use strict_path::VirtualPath;
```

This type provides validated, normalized paths that cannot escape root. See `strict-path` documentation for full API.

### 3.4 Types

```rust
/// File metadata
#[derive(Clone, Debug)]
pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
}

/// Type of filesystem entry
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum FileType {
    File,
    Directory,
    Symlink,
}

/// Directory entry
#[derive(Clone, Debug)]
pub struct DirEntry {
    pub name: String,
    pub file_type: FileType,
}
```

### 3.5 Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("not found: {0}")]
    NotFound(VirtualPath),

    #[error("already exists: {0}")]
    AlreadyExists(VirtualPath),

    #[error("not a file: {0}")]
    NotAFile(VirtualPath),
    
    #[error("not a directory: {0}")]
    NotADirectory(VirtualPath),

    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(VirtualPath),

    #[error("invalid path: {0}")]
    InvalidPath(#[from] strict_path::PathError),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("backend error: {0}")]
    Backend(String),
}
```

### 3.6 Crate Structure

```
anyfs/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Re-exports
│   ├── backend.rs      # VfsBackend trait
│   ├── path.rs         # VirtualPath (validated paths)
│   ├── types.rs        # Metadata, DirEntry, FileType
│   ├── error.rs        # VfsError, PathError
│   │
│   ├── vrootfs/        # Filesystem backend via strict-path
│   │   └── mod.rs      # VRootFsBackend
│   │
│   ├── memory/         # In-memory backend
│   │   └── mod.rs      # MemoryBackend
│   │
│   └── sqlite/         # SQLite backend
│       ├── mod.rs      # SqliteBackend
│       └── schema.rs   # Internal schema
│
└── tests/
    └── conformance.rs  # Tests all backends identically
```

### 3.7 Dependencies

```toml
[dependencies]
thiserror = "1"
strict-path = "..."  # Required: provides VirtualPath

[dependencies.rusqlite]
version = "0.31"
features = ["bundled"]
optional = true

[features]
default = ["vrootfs", "memory"]
vrootfs = []   # VRootFsBackend (uses strict-path for filesystem containment)
memory = []    # MemoryBackend
sqlite = ["rusqlite"]
full = ["vrootfs", "memory", "sqlite"]
```

---

## 4. Project 2: anyfs-container

### 4.1 Purpose

Wrap any `VfsBackend` with:
- Capacity limits (total size, file size, node count, etc.)
- Usage tracking
- (Future: permissions, quotas per directory, etc.)

### 4.2 FilesContainer

```rust
use std::path::Path;
use anyfs::{VfsBackend, VirtualPath};

/// A contained filesystem with capacity limits.
pub struct FilesContainer<B: VfsBackend> {
    backend: B,
    limits: CapacityLimits,
    usage: CapacityUsage,
}

impl<B: VfsBackend> FilesContainer<B> {
    pub fn new(backend: B) -> Self;
    pub fn with_limits(backend: B, limits: CapacityLimits) -> Self;

    // User-facing operations (accept flexible paths, convert internally)
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;  // Validate & convert
        self.check_limits(&vpath)?;
        Ok(self.backend.read(&vpath)?)  // Backend receives &VirtualPath
    }

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;
        self.check_file_size(data.len())?;
        self.check_total_size(data.len())?;
        self.backend.write(&vpath, data)?;
        self.usage.add_bytes(data.len() as u64);
        Ok(())
    }
    // ... other methods follow same pattern

    // Capacity management
    pub fn usage(&self) -> &CapacityUsage;
    pub fn limits(&self) -> &CapacityLimits;
    pub fn remaining(&self) -> CapacityRemaining;
}
```

**Key insight**: User-facing API accepts `impl AsRef<Path>` for ergonomics. Internally, paths are validated and converted to `VirtualPath` before reaching the backend.

### 4.3 Capacity Types

```rust
#[derive(Clone, Debug, Default)]
pub struct CapacityLimits {
    /// Maximum total bytes (None = unlimited)
    pub max_total_size: Option<u64>,
    
    /// Maximum single file size (None = unlimited)
    pub max_file_size: Option<u64>,
    
    /// Maximum number of files + directories (None = unlimited)
    pub max_node_count: Option<u64>,
    
    /// Maximum entries per directory (None = unlimited)
    pub max_dir_entries: Option<u32>,
    
    /// Maximum path depth (None = unlimited)
    pub max_path_depth: Option<u16>,
}

#[derive(Clone, Debug, Default)]
pub struct CapacityUsage {
    pub total_size: u64,
    pub node_count: u64,
    pub file_count: u64,
    pub directory_count: u64,
}

#[derive(Clone, Debug)]
pub struct CapacityRemaining {
    pub bytes: Option<u64>,
    pub nodes: Option<u64>,
    pub can_write: bool,
}
```

### 4.4 Builder

```rust
pub struct ContainerBuilder<B> {
    backend: B,
    limits: CapacityLimits,
}

impl<B: VfsBackend> ContainerBuilder<B> {
    pub fn new(backend: B) -> Self;
    
    pub fn max_total_size(self, bytes: u64) -> Self;
    pub fn max_file_size(self, bytes: u64) -> Self;
    pub fn max_node_count(self, count: u64) -> Self;
    pub fn max_dir_entries(self, count: u32) -> Self;
    pub fn max_path_depth(self, depth: u16) -> Self;
    pub fn unlimited(self) -> Self;
    
    pub fn build(self) -> FilesContainer<B>;
}
```

### 4.5 Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum ContainerError {
    #[error(transparent)]
    Vfs(#[from] VfsError),
    
    #[error("total size limit exceeded: {used} / {limit} bytes")]
    TotalSizeExceeded { used: u64, limit: u64 },
    
    #[error("file size {size} exceeds limit of {limit} bytes")]
    FileSizeExceeded { size: u64, limit: u64 },
    
    #[error("node count limit exceeded: {count} / {limit}")]
    NodeCountExceeded { count: u64, limit: u64 },
    
    #[error("directory entry limit exceeded: {count} / {limit}")]
    DirEntriesExceeded { count: u32, limit: u32 },
    
    #[error("path depth {depth} exceeds limit of {limit}")]
    PathDepthExceeded { depth: u16, limit: u16 },
}
```

### 4.6 Crate Structure

```
anyfs-container/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── container.rs    # FilesContainer
│   ├── builder.rs      # ContainerBuilder
│   ├── limits.rs       # CapacityLimits
│   ├── usage.rs        # CapacityUsage, CapacityRemaining
│   └── error.rs        # ContainerError
│
└── tests/
    └── limits.rs       # Capacity enforcement tests
```

### 4.7 Dependencies

```toml
[dependencies]
anyfs = { version = "0.1", path = "../anyfs" }
thiserror = "1"
```

---

## 5. Backend Implementations

### 5.1 VRootFsBackend

Uses `strict-path::VirtualRoot` for containment.

```rust
use std::path::Path;
use strict_path::{VirtualRoot, VirtualPath};

pub struct VRootFsBackend {
    vroot: VirtualRoot,
}

impl VRootFsBackend {
    /// Create backend with given directory as virtual root.
    /// The directory is created if it doesn't exist.
    pub fn new(root_dir: &Path) -> Result<Self, VfsError> {
        let vroot = VirtualRoot::try_new_create(root_dir)
            .map_err(|e| VfsError::Backend(e.to_string()))?;
        Ok(Self { vroot })
    }

    /// Open existing directory as virtual root.
    pub fn open(root_dir: &Path) -> Result<Self, VfsError> {
        let vroot = VirtualRoot::try_new(root_dir)
            .map_err(|e| VfsError::Backend(e.to_string()))?;
        Ok(Self { vroot })
    }
}

impl VfsBackend for VRootFsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        // VirtualPath maps directly to strict-path operations
        self.vroot.read(path).map_err(VfsError::Io)
    }

    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        self.vroot.write(path, data).map_err(VfsError::Io)
    }

    // ... delegate other methods via VirtualRoot
}
```

**Key behavior:**
- `VirtualPath` is already validated — backend just delegates to `VirtualRoot`
- Symlink escapes are prevented by `strict-path`
- Uses real filesystem for storage

### 5.2 MemoryBackend

In-memory storage using HashMap.

```rust
use std::collections::HashMap;
use strict_path::VirtualPath;

pub struct MemoryBackend {
    entries: HashMap<VirtualPath, Entry>,
}

enum Entry {
    File { data: Vec<u8>, modified: SystemTime },
    Directory { modified: SystemTime },
}

impl MemoryBackend {
    pub fn new() -> Self {
        let mut entries = HashMap::new();
        // Root directory always exists
        entries.insert(VirtualPath::root(), Entry::Directory {
            modified: SystemTime::now(),
        });
        Self { entries }
    }
}

impl VfsBackend for MemoryBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        match self.entries.get(path) {
            Some(Entry::File { data, .. }) => Ok(data.clone()),
            Some(Entry::Directory { .. }) => Err(VfsError::NotAFile(path.clone())),
            None => Err(VfsError::NotFound(path.clone())),
        }
    }
    
    // ... implement other methods
}
```

**Key behavior:**
- `VirtualPath` is already normalized — use directly as HashMap key
- No persistence — data lost when dropped
- Useful for testing

### 5.3 SqliteBackend

Single-file storage using SQLite.

```rust
use rusqlite::Connection;

pub struct SqliteBackend {
    conn: Connection,
}

impl SqliteBackend {
    pub fn create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
}
```

**Schema:**

```sql
CREATE TABLE entries (
    id INTEGER PRIMARY KEY,
    path TEXT NOT NULL UNIQUE,
    is_dir INTEGER NOT NULL,
    size INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER,
    modified_at INTEGER
);

CREATE TABLE content (
    entry_id INTEGER PRIMARY KEY,
    data BLOB NOT NULL,
    FOREIGN KEY (entry_id) REFERENCES entries(id) ON DELETE CASCADE
);

-- For large files, could use chunking:
-- CREATE TABLE chunks (
--     entry_id INTEGER NOT NULL,
--     chunk_index INTEGER NOT NULL,
--     data BLOB NOT NULL,
--     PRIMARY KEY (entry_id, chunk_index)
-- );

CREATE INDEX idx_entries_path ON entries(path);
```

**Key behavior:**
- Entire container is a single `.db` file
- Portable — copy file to move container
- Supports large files via chunking (optional)

---

## 6. Open Design Questions

### 6.1 Path Type in Trait — ✅ RESOLVED

**Decision:** Two-layer approach:
- **VfsBackend trait**: Uses `&VirtualPath` (from `strict-path`)
- **FilesContainer API**: Uses `impl AsRef<Path>` for ergonomics

**Rationale:**
- User-facing API is idiomatic Rust — accepts `&str`, `String`, `&Path`, `PathBuf`, etc.
- Backend trait receives pre-validated `VirtualPath` — simpler, safer backends
- Validation happens once in `FilesContainer`, not in every backend
- `VirtualPath` guarantees containment (can't escape root)

**How it works:**
```rust
// User calls with any path-like type
container.read("/data/file.txt")?;

// FilesContainer validates and converts
let vpath = VirtualPath::new(path.as_ref())?;

// Backend receives validated VirtualPath
self.backend.read(&vpath)
```

### 6.2 Error Path Representation — ✅ RESOLVED

**Decision:** `VirtualPath`

**Rationale:**
- Errors originate from operations on validated paths
- `VirtualPath` is already owned and cloneable
- Consistent with the internal path representation

### 6.3 Symlink Support

**Question:** Should the trait support symlinks?

- `VRootFsBackend`: The underlying filesystem has symlinks; `strict-path` handles them
- `MemoryBackend`: Could support symlinks as a stored target path
- `SqliteBackend`: Could store symlink targets

**Current leaning:** Defer symlinks to v2. Keep v1 simple.

### 6.4 Backend Discovery of Usage

**Question:** How does `FilesContainer` know current usage when wrapping an existing backend?

Options:
1. Scan on construction (slow for large containers)
2. Backend provides `usage()` method
3. Store usage metadata separately

**Current leaning:** Backend provides optional `fn usage(&self) -> Option<CapacityUsage>`. If `None`, container scans.

---

## 7. Implementation Plan

### Phase 1: Core Types (Week 1)

- [x] ~~Decide path type~~ → `impl AsRef<Path>`
- [ ] `anyfs`: `VfsBackend` trait
- [ ] `anyfs`: `Metadata`, `DirEntry`, `FileType`
- [ ] `anyfs`: `VfsError`
- [ ] `anyfs`: `MemoryBackend` (simplest, for testing)

### Phase 2: VRootFsBackend (Week 2)

- [ ] Integrate `strict-path`
- [ ] Implement `VRootFsBackend`
- [ ] Conformance tests (same tests as MemoryBackend)

### Phase 3: SqliteBackend (Week 3)

- [ ] Schema design
- [ ] Implement `SqliteBackend`
- [ ] Conformance tests

### Phase 4: anyfs-container (Week 4)

- [ ] `FilesContainer`
- [ ] `CapacityLimits`, `CapacityUsage`
- [ ] `ContainerBuilder`
- [ ] Limit enforcement
- [ ] Tests

### Phase 5: Polish (Week 5)

- [ ] Documentation
- [ ] Examples
- [ ] Publish

---

## Appendix: Comparison with Alternatives

### vs. `vfs` crate

| Feature | vfs | anyfs |
|---------|-----|----------------|
| Path type | `&str` | TBD |
| Backends | Physical, Memory, Overlay, Embedded | Fs, Memory, SQLite |
| Capacity limits | ❌ | ✅ via anyfs-container |
| Containment | ❌ | ✅ via strict-path |
| Overlay/layering | ✅ | ❌ |

### vs. `OpenDAL`

| Feature | OpenDAL | anyfs |
|---------|---------|----------------|
| Focus | Cloud/object storage | Local/embedded storage |
| API style | Object storage (get/put) | Filesystem (read/write/mkdir) |
| Backends | 50+ (S3, GCS, Azure, etc.) | 3 (Fs, Memory, SQLite) |
| Async | ✅ | ❌ (sync only) |

### When to Use What

- **anyfs**: Local storage, tenant isolation, portable containers
- **vfs crate**: Overlay filesystems, embedded resources
- **OpenDAL**: Cloud storage, async workloads

---

## Appendix: Usage Examples

### Basic Usage (anyfs only)

```rust
use anyfs::{VfsBackend, VRootFsBackend, MemoryBackend};

fn save_data(vfs: &mut impl VfsBackend, name: &str, data: &[u8]) -> Result<(), VfsError> {
    vfs.create_dir_all("/data")?;
    vfs.write(&format!("/data/{}", name), data)?;
    Ok(())
}

// With filesystem
let mut fs = VRootFsBackend::new("/var/myapp")?;
save_data(&mut fs, "report.txt", b"...")?;

// With memory (for tests)
let mut mem = MemoryBackend::new();
save_data(&mut mem, "report.txt", b"...")?;
```

### With Container (anyfs-container)

```rust
use anyfs_container::{FilesContainer, ContainerBuilder};
use anyfs::SqliteBackend;

// Create tenant with 100 MB quota
let backend = SqliteBackend::create("tenant_123.db")?;
let mut container = ContainerBuilder::new(backend)
    .max_total_size(100 * 1024 * 1024)
    .build();

// Use it
container.write("/uploads/doc.pdf", &pdf_data)?;

// Check usage
println!("Used: {} bytes", container.usage().total_size);

// Move tenant = move file
std::fs::rename("tenant_123.db", "archive/tenant_123.db")?;
```

### Multi-Tenant Setup

```rust
use anyfs_container::{FilesContainer, ContainerBuilder};
use anyfs::VRootFsBackend;
use std::path::Path;

fn create_tenant_container(tenant_id: &str) -> Result<FilesContainer<VRootFsBackend>, Error> {
    let root = format!("/data/tenants/{}", tenant_id);
    let backend = VRootFsBackend::new(Path::new(&root))?;

    Ok(ContainerBuilder::new(backend)
        .max_total_size(100 * 1024 * 1024)
        .build())
}
```

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2025-12-22 | Initial design (graph-store trait) |
| 0.2.0 | 2025-12-22 | Restructured: two projects, path-based trait, three backends |
