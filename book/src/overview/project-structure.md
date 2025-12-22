# VFS Ecosystem — Restructured Design

**Date:** 2025-12-22  
**Status:** Revised based on corrected understanding

---

## Project Structure

```
┌─────────────────────────────────────────┐
│  Your App                               │  ← Project 3: Consumer
│  (just calls FilesContainer API)        │
├─────────────────────────────────────────┤
│  anyfs-container                          │  ← Project 2: Isolation layer
│  (quotas, tenant isolation, safety)     │
├─────────────────────────────────────────┤
│  anyfs                         │  ← Project 1: Foundation
│  (generic VFS with swappable backends)  │
├──────────┬──────────┬───────────────────┤
│ FsBackend│MemBackend│  SqliteBackend    │
└──────────┴──────────┴───────────────────┘
```

---

# Project 1: anyfs

**Purpose:** Generic virtual filesystem abstraction with swappable backends.

**Value proposition:** Call `read()`, `write()`, `mkdir()` — backend is interchangeable.

## Crate Structure

```
anyfs/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── backend.rs      # VfsBackend trait
    ├── path.rs         # VirtualPath (validated paths)
    ├── types.rs        # Metadata, DirEntry, FileType
    ├── error.rs        # VfsError
    │
    ├── fs/             # Filesystem backend
    │   └── mod.rs      # FsBackend (via strict-path)
    │
    ├── memory/         # In-memory backend
    │   └── mod.rs      # MemoryBackend
    │
    └── sqlite/         # SQLite backend
        ├── mod.rs      # SqliteBackend
        └── schema.rs   # Internal schema
```

## Core Trait

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

## VirtualPath

```rust
/// A validated virtual path that cannot escape its root.
/// 
/// - Always absolute (starts with `/`)
/// - No `..` that escapes root
/// - No empty components
/// - Valid UTF-8
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct VirtualPath(String);

impl VirtualPath {
    pub fn new(path: &str) -> Result<Self, PathError>;
    pub fn root() -> Self;
    pub fn join(&self, component: &str) -> Result<Self, PathError>;
    pub fn parent(&self) -> Option<Self>;
    pub fn file_name(&self) -> Option<&str>;
    pub fn as_str(&self) -> &str;
}
```

## Types

```rust
#[derive(Clone, Debug)]
pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum FileType {
    File,
    Directory,
    Symlink,
}

#[derive(Clone, Debug)]
pub struct DirEntry {
    pub name: String,
    pub file_type: FileType,
}
```

## Errors

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
    InvalidPath(#[from] PathError),
    
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("backend error: {0}")]
    Backend(String),
}
```

## Backend Implementations

### FsBackend (via strict-path)

```rust
use strict_path::{VirtualRoot, VirtualPath as StrictPath};

pub struct FsBackend {
    root: VirtualRoot,
}

impl FsBackend {
    pub fn new(root_dir: &Path) -> Result<Self, VfsError> {
        Ok(Self {
            root: VirtualRoot::new(root_dir)?,
        })
    }
}

impl VfsBackend for FsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        let strict_path = to_strict_path(path)?;
        Ok(self.root.read(&strict_path)?)
    }
    
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        let strict_path = to_strict_path(path)?;
        Ok(self.root.write(&strict_path, data)?)
    }
    
    // ... straightforward delegation
}
```

### MemoryBackend

```rust
pub struct MemoryBackend {
    entries: HashMap<VirtualPath, Entry>,
}

enum Entry {
    File(Vec<u8>),
    Directory,
}

impl MemoryBackend {
    pub fn new() -> Self {
        let mut entries = HashMap::new();
        entries.insert(VirtualPath::root(), Entry::Directory);
        Self { entries }
    }
}

impl VfsBackend for MemoryBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        match self.entries.get(path) {
            Some(Entry::File(data)) => Ok(data.clone()),
            Some(Entry::Directory) => Err(VfsError::NotAFile(path.clone())),
            None => Err(VfsError::NotFound(path.clone())),
        }
    }
    
    // ... HashMap operations
}
```

### SqliteBackend

```rust
pub struct SqliteBackend {
    conn: rusqlite::Connection,
}

impl SqliteBackend {
    pub fn open(path: &Path) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
    pub fn create(path: &Path) -> Result<Self, VfsError>;
}

impl VfsBackend for SqliteBackend {
    // Internal implementation can use node/edge model
    // but that's hidden from consumers
}
```

## Usage Example

```rust
use anyfs::{VfsBackend, FsBackend, MemoryBackend, SqliteBackend, VirtualPath};

// Pick any backend — same API
fn process_files(vfs: &mut impl VfsBackend) -> Result<(), VfsError> {
    let path = VirtualPath::new("/data/file.txt")?;
    
    vfs.mkdir_all(&VirtualPath::new("/data")?)?;
    vfs.write(&path, b"hello world")?;
    
    let content = vfs.read(&path)?;
    println!("Read {} bytes", content.len());
    
    Ok(())
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Filesystem backend
    let mut fs = FsBackend::new(Path::new("/tmp/myapp"))?;
    process_files(&mut fs)?;
    
    // Memory backend (for tests)
    let mut mem = MemoryBackend::new();
    process_files(&mut mem)?;
    
    // SQLite backend (portable)
    let mut db = SqliteBackend::create(Path::new("data.db"))?;
    process_files(&mut db)?;
    
    Ok(())
}
```

---

# Project 2: anyfs-container

**Purpose:** Tenant isolation and containment layer built on anyfs.

**Depends on:** `anyfs`

## Crate Structure

```
anyfs-container/
├── Cargo.toml          # depends on anyfs
└── src/
    ├── lib.rs
    ├── container.rs    # FilesContainer<B>
    ├── builder.rs      # ContainerBuilder
    ├── limits.rs       # CapacityLimits
    ├── usage.rs        # CapacityUsage tracking
    └── error.rs        # ContainerError
```

## FilesContainer

```rust
use anyfs::{VfsBackend, VirtualPath, VfsError};

/// A contained filesystem with quotas and isolation.
pub struct FilesContainer<B: VfsBackend> {
    backend: B,
    limits: CapacityLimits,
    usage: CapacityUsage,
}

impl<B: VfsBackend> FilesContainer<B> {
    pub fn new(backend: B) -> Self {
        Self::with_limits(backend, CapacityLimits::default())
    }
    
    pub fn with_limits(backend: B, limits: CapacityLimits) -> Self {
        Self {
            backend,
            limits,
            usage: CapacityUsage::default(),
        }
    }
    
    // ─────────────────────────────────────────────────────────
    // Delegated operations (with limit enforcement)
    // ─────────────────────────────────────────────────────────
    
    pub fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, ContainerError> {
        Ok(self.backend.read(path)?)
    }
    
    pub fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), ContainerError> {
        // Check limits BEFORE writing
        self.check_file_size(data.len())?;
        self.check_total_size(data.len())?;
        
        self.backend.write(path, data)?;
        self.usage.add_bytes(data.len() as u64);
        Ok(())
    }
    
    pub fn mkdir(&mut self, path: &VirtualPath) -> Result<(), ContainerError> {
        self.check_node_count()?;
        self.check_path_depth(path)?;
        
        self.backend.mkdir(path)?;
        self.usage.add_node();
        Ok(())
    }
    
    // ... other operations with limit checks
    
    // ─────────────────────────────────────────────────────────
    // Capacity management
    // ─────────────────────────────────────────────────────────
    
    pub fn usage(&self) -> &CapacityUsage {
        &self.usage
    }
    
    pub fn limits(&self) -> &CapacityLimits {
        &self.limits
    }
    
    pub fn remaining(&self) -> CapacityRemaining {
        self.limits.remaining(&self.usage)
    }
}
```

## Capacity Limits

```rust
#[derive(Clone, Debug)]
pub struct CapacityLimits {
    /// Maximum total bytes (None = unlimited)
    pub max_total_size: Option<u64>,
    
    /// Maximum single file size (None = unlimited)
    pub max_file_size: Option<u64>,
    
    /// Maximum number of nodes (None = unlimited)
    pub max_node_count: Option<u64>,
    
    /// Maximum entries per directory (None = unlimited)
    pub max_dir_entries: Option<u32>,
    
    /// Maximum path depth (None = unlimited)
    pub max_path_depth: Option<u16>,
}

impl Default for CapacityLimits {
    fn default() -> Self {
        Self {
            max_total_size: None,
            max_file_size: Some(1 << 30),    // 1 GB
            max_node_count: None,
            max_dir_entries: Some(10_000),
            max_path_depth: Some(64),
        }
    }
}

impl CapacityLimits {
    pub fn unlimited() -> Self {
        Self {
            max_total_size: None,
            max_file_size: None,
            max_node_count: None,
            max_dir_entries: None,
            max_path_depth: None,
        }
    }
    
    pub fn tenant(max_bytes: u64) -> Self {
        Self {
            max_total_size: Some(max_bytes),
            ..Default::default()
        }
    }
}
```

## Builder

```rust
pub struct ContainerBuilder<B> {
    backend: B,
    limits: CapacityLimits,
}

impl<B: VfsBackend> ContainerBuilder<B> {
    pub fn new(backend: B) -> Self {
        Self {
            backend,
            limits: CapacityLimits::default(),
        }
    }
    
    pub fn max_total_size(mut self, bytes: u64) -> Self {
        self.limits.max_total_size = Some(bytes);
        self
    }
    
    pub fn max_file_size(mut self, bytes: u64) -> Self {
        self.limits.max_file_size = Some(bytes);
        self
    }
    
    pub fn unlimited(mut self) -> Self {
        self.limits = CapacityLimits::unlimited();
        self
    }
    
    pub fn build(self) -> FilesContainer<B> {
        FilesContainer::with_limits(self.backend, self.limits)
    }
}
```

## Usage Example

```rust
use anyfs::{FsBackend, SqliteBackend, VirtualPath};
use anyfs_container::{FilesContainer, ContainerBuilder, CapacityLimits};

// Create a tenant container with 100MB quota
fn create_tenant(tenant_id: &str) -> Result<FilesContainer<SqliteBackend>, Error> {
    let db_path = format!("tenants/{}.db", tenant_id);
    let backend = SqliteBackend::create(&db_path)?;
    
    let container = ContainerBuilder::new(backend)
        .max_total_size(100 * 1024 * 1024)  // 100 MB
        .max_file_size(10 * 1024 * 1024)    // 10 MB per file
        .build();
    
    Ok(container)
}

// Use the container
fn tenant_operation(container: &mut FilesContainer<impl VfsBackend>) -> Result<(), Error> {
    let path = VirtualPath::new("/uploads/document.pdf")?;
    
    // Check remaining space
    let remaining = container.remaining();
    println!("Space remaining: {:?} bytes", remaining.bytes);
    
    // Write (will fail if over quota)
    container.mkdir_all(&VirtualPath::new("/uploads")?)?;
    container.write(&path, &pdf_bytes)?;
    
    Ok(())
}
```

---

# Project 3: Your App

**Purpose:** Your application that uses FilesContainer.

**Depends on:** `anyfs-container` (which depends on `anyfs`)

```rust
use anyfs_container::{FilesContainer, ContainerBuilder};
use anyfs::SqliteBackend;

struct TenantManager {
    // Your app just works with FilesContainer
    // No knowledge of backends needed
}

impl TenantManager {
    fn get_tenant_storage(&self, tenant_id: &str) -> FilesContainer<SqliteBackend> {
        // Configuration happens here, once
        let backend = SqliteBackend::open(&format!("data/{}.db", tenant_id)).unwrap();
        ContainerBuilder::new(backend)
            .max_total_size(self.quota_for_tenant(tenant_id))
            .build()
    }
}

// Your app code — clean and simple
fn handle_upload(container: &mut FilesContainer<impl VfsBackend>, file: UploadedFile) {
    let path = VirtualPath::new(&format!("/uploads/{}", file.name)).unwrap();
    container.write(&path, &file.data).unwrap(); // Quota enforced automatically
}
```

---

# Summary

| Project | Crate | Purpose | Depends On |
|---------|-------|---------|------------|
| 1 | `anyfs` | Generic VFS with swappable backends | — |
| 2 | `anyfs-container` | Tenant isolation + quotas | `anyfs` |
| 3 | Your App | Business logic | `anyfs-container` |

**Project 1** is the foundation — pure I/O abstraction.
**Project 2** adds containment — quotas, limits, safety.
**Project 3** is your code — just uses the clean API.

---

# Implementation Plan

## Phase 1: anyfs (Week 1-2)

1. Core types: `VirtualPath`, `Metadata`, `DirEntry`, `VfsError`
2. `VfsBackend` trait
3. `FsBackend` via `strict-path`
4. `MemoryBackend` 
5. Tests

## Phase 2: anyfs + SQLite (Week 3)

1. `SqliteBackend` implementation
2. Schema design (internal)
3. All backends pass same test suite

## Phase 3: anyfs-container (Week 4)

1. `FilesContainer<B>`
2. `CapacityLimits` + enforcement
3. `ContainerBuilder`
4. Tests

## Phase 4: Polish (Week 5)

1. Documentation
2. Examples
3. Publish to crates.io
