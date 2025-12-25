# FileStorage<M> (anyfs-container)

**Type-safe ergonomic wrapper for std::fs-aligned API**

---

## Overview

`FileStorage<M>` is a **thin wrapper** that provides a familiar std::fs-aligned API with an optional **marker type** for compile-time safety.

The backend is type-erased (`Box<dyn Vfs>`) for a clean API.

**It does TWO things:**
1. Ergonomics (std::fs-aligned API)
2. Optional type-safety via marker types

All policy (limits, feature gates, logging) is handled by middleware, not FileStorage.

---

## Creating a Container

```rust
use anyfs::MemoryBackend;
use anyfs_container::FileStorage;

// Simple: just ergonomics
let mut fs = FileStorage::new(MemoryBackend::new());
```

With middleware (layer-based):

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer};
use anyfs_container::FileStorage;

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new()
        .max_total_size(100 * 1024 * 1024))
    .layer(RestrictionsLayer::new()
        .deny_hard_links()
        .deny_permissions());

let mut fs = FileStorage::new(backend);
```

---

## Marker Types (Compile-Time Safety)

The type parameter `M` (default: `()`) enables compile-time container differentiation:

```rust
use anyfs::{MemoryBackend, SqliteBackend};
use anyfs_container::FileStorage;

// Define marker types for your domains
struct Sandbox;
struct UserData;
struct TempFiles;

// Create typed containers - backend type is erased!
let sandbox: FileStorage<Sandbox> = FileStorage::with_marker(MemoryBackend::new());
let userdata: FileStorage<UserData> = FileStorage::with_marker(SqliteBackend::open("data.db")?);

// Type-safe function signatures prevent mixing containers
fn process_sandbox(fs: &FileStorage<Sandbox>) {
    // Can only accept Sandbox-marked containers
}

fn save_user_file(fs: &mut FileStorage<UserData>, name: &str, data: &[u8]) {
    // Can only accept UserData-marked containers
}

// Compile-time safety:
process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // Compile error! Type mismatch
```

### When to Use Markers

| Scenario | Use Markers? | Why |
|----------|--------------|-----|
| Single container | No | `FileStorage` is sufficient |
| Multiple containers, same type | **Yes** | Prevent accidental mixing |
| Multi-tenant systems | **Yes** | Compile-time tenant isolation |
| Sandbox + user data | **Yes** | Never write user data to sandbox |
| Testing | Maybe | Tag test vs production containers |

---

## std::fs-aligned Methods

FileStorage mirrors std::fs naming:

| FileStorage | std::fs |
|-------------|---------|
| `read()` | `std::fs::read` |
| `read_to_string()` | `std::fs::read_to_string` |
| `write()` | `std::fs::write` |
| `read_dir()` | `std::fs::read_dir` |
| `create_dir()` | `std::fs::create_dir` |
| `create_dir_all()` | `std::fs::create_dir_all` |
| `remove_file()` | `std::fs::remove_file` |
| `remove_dir()` | `std::fs::remove_dir` |
| `remove_dir_all()` | `std::fs::remove_dir_all` |
| `rename()` | `std::fs::rename` |
| `copy()` | `std::fs::copy` |
| `metadata()` | `std::fs::metadata` |
| `symlink_metadata()` | `std::fs::symlink_metadata` |
| `read_link()` | `std::fs::read_link` |
| `set_permissions()` | `std::fs::set_permissions` |

---

## What FileStorage Does NOT Do

| Concern | Use Instead |
|---------|-------------|
| Quota enforcement | `Quota<B>` |
| Feature gating | `Restrictions<B>` |
| Audit logging | `Tracing<B>` |
| Path containment | Backend-specific (VRootFsBackend) |

FileStorage is **purely ergonomic**. If you need policy, compose middleware.

---

## FileStorage Implementation

```rust
use std::marker::PhantomData;
use anyfs_backend::Vfs;

/// Ergonomic filesystem wrapper with optional type marker.
/// Backend is type-erased to `Box<dyn Vfs>` for a clean API.
pub struct FileStorage<M = ()> {
    backend: Box<dyn Vfs>,
    _marker: PhantomData<M>,
}

impl<M> FileStorage<M> {
    /// Create a new FileStorage (no marker).
    pub fn new(backend: impl Vfs + 'static) -> FileStorage {
        FileStorage { backend: Box::new(backend), _marker: PhantomData }
    }

    /// Create a new FileStorage with a specific marker type.
    pub fn with_marker<N>(backend: impl Vfs + 'static) -> FileStorage<N> {
        FileStorage { backend: Box::new(backend), _marker: PhantomData }
    }

    /// Change the marker type (zero-cost, compile-time only).
    pub fn retype<N>(self) -> FileStorage<N> {
        FileStorage { backend: self.backend, _marker: PhantomData }
    }
}
```

**Note:** `FileStorage` uses `Box<dyn Vfs>` which provides Layer 1 operations (read, write, dirs). If you need `VfsFull`, `VfsFuse`, or `VfsPosix` operations, use the backend directly without type erasure.

---

## Direct Backend Access

If you don't need ergonomics, want zero-cost abstractions, or need higher-level traits (`VfsFull`, `VfsFuse`), use backends directly:

```rust
use anyfs::{MemoryBackend, Quota, Vfs};

let mut backend = Quota::new(MemoryBackend::new())
    .with_max_total_size(100 * 1024 * 1024);

// Use Vfs trait methods directly
backend.write("/file.txt", b"data")?;
```

For FUSE mounting (requires `VfsFuse`):

```rust
use anyfs::{SqliteBackend, VfsFuse};
use anyfs_fuse::FuseMount;

let backend = SqliteBackend::open("data.db")?;
// backend implements VfsFuse, so we can mount it
FuseMount::mount(backend, "/mnt/myfs")?;
```
