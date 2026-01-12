# FileStorage<B> (anyfs)

**Ergonomic wrapper for std::fs-aligned API**

---

## Overview

`FileStorage<B>` is a **thin wrapper** that provides a familiar std::fs-aligned API with:
- **`B`** - Backend type (the only generic)
- Built-in path resolution via boxed `PathResolver` (swappable at runtime)

It is the intended application-facing API: std::fs-style paths with object-safe core traits under the hood.

**It does TWO things:**
1. Ergonomics (std::fs-aligned API with `impl AsRef<Path>` convenience)
2. Path resolution for virtual backends (via boxed `PathResolver` - cold path, boxing is acceptable)

All policy (limits, feature gates, logging) is handled by middleware, not FileStorage.

---

## Why Only One Generic?

Previous designs used `FileStorage<B, R, M>` with three type parameters. We simplified to `FileStorage<B>`:

| Old Param | Purpose | Why Removed |
|-----------|---------|-------------|
| `R` (Resolver) | Swappable path resolution | Boxed internally—resolution is a cold path (ADR-025) |
| `M` (Marker) | Compile-time safety | Users can create wrapper newtypes if needed |

**Result:** Simpler API for 90% of users. Those who need type-safe markers wrap `FileStorage` themselves.

---

## Creating a Container

```rust
use anyfs::{MemoryBackend, FileStorage};

// Simple: ergonomics + default path resolution
let fs = FileStorage::new(MemoryBackend::new());
```

With middleware (layer-based):

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, FileStorage};

let fs = FileStorage::new(
    SqliteBackend::open("data.db")?
        .layer(QuotaLayer::builder()
            .max_total_size(100 * 1024 * 1024)
            .build())
        .layer(RestrictionsLayer::builder()
            .deny_permissions()
            .build())
);
```

With custom resolver:

```rust
use anyfs::{MemoryBackend, FileStorage};
use anyfs::resolvers::CachingResolver;

// Custom resolver for read-heavy workloads
let fs = FileStorage::with_resolver(
    MemoryBackend::new(),
    CachingResolver::new()
);
```

---

## Type-Safe Markers (User-Defined Wrappers)

If you need compile-time safety to prevent mixing filesystems, create wrapper newtypes:

```rust
use anyfs::{MemoryBackend, SqliteBackend, FileStorage};

// Define your own wrapper types
struct SandboxFs(FileStorage<MemoryBackend>);
struct UserDataFs(FileStorage<SqliteBackend>);

impl SandboxFs {
    fn new() -> Self {
        SandboxFs(FileStorage::new(MemoryBackend::new()))
    }
}

// Type-safe function signatures prevent mixing
fn process_sandbox(fs: &SandboxFs) {
    // Can only accept SandboxFs
}

fn save_user_file(fs: &UserDataFs, name: &str, data: &[u8]) {
    // Can only accept UserDataFs
}

// Compile-time safety:
let sandbox = SandboxFs::new();
process_sandbox(&sandbox);     // OK
// process_sandbox(&userdata); // Compile error! Wrong type
```

### When to Use Wrapper Types

| Scenario                       | Use Wrapper? | Why                               |
| ------------------------------ | ------------ | --------------------------------- |
| Single container               | No           | `FileStorage<B>` is sufficient    |
| Multiple containers, same type | **Yes**      | Prevent accidental mixing         |
| Multi-tenant systems           | **Yes**      | Compile-time tenant isolation     |
| Sandbox + user data            | **Yes**      | Never write user data to sandbox  |

---

## std::fs-aligned Methods

FileStorage mirrors std::fs naming:

| FileStorage          | std::fs                     |
| -------------------- | --------------------------- |
| `read()`             | `std::fs::read`             |
| `read_to_string()`   | `std::fs::read_to_string`   |
| `write()`            | `std::fs::write`            |
| `read_dir()`         | `std::fs::read_dir`         |
| `create_dir()`       | `std::fs::create_dir`       |
| `create_dir_all()`   | `std::fs::create_dir_all`   |
| `remove_file()`      | `std::fs::remove_file`      |
| `remove_dir()`       | `std::fs::remove_dir`       |
| `remove_dir_all()`   | `std::fs::remove_dir_all`   |
| `rename()`           | `std::fs::rename`           |
| `copy()`             | `std::fs::copy`             |
| `metadata()`         | `std::fs::metadata`         |
| `symlink_metadata()` | `std::fs::symlink_metadata` |
| `read_link()`        | `std::fs::read_link`        |
| `set_permissions()`  | `std::fs::set_permissions`  |

When the backend implements extended traits (e.g., `FsLink`, `FsInode`, `FsHandles`), FileStorage forwards those methods too and keeps the same `impl AsRef<Path>` ergonomics for path parameters.

---

## What FileStorage Does NOT Do

| Concern           | Use Instead                                         |
| ----------------- | --------------------------------------------------- |
| Quota enforcement | `Quota<B>`                                          |
| Feature gating    | `Restrictions<B>`                                   |
| Audit logging     | `Tracing<B>`                                        |
| Path containment  | PathFilter middleware or VRootFsBackend containment |

FileStorage is **not a policy layer**. If you need policy, compose middleware.

---

## FileStorage Implementation

```rust
use anyfs_backend::{Fs, PathResolver};
use anyfs::resolvers::IterativeResolver;

/// Ergonomic wrapper with single generic parameter.
pub struct FileStorage<B> {
    backend: B,
    resolver: Box<dyn PathResolver>,  // Boxed: resolution is cold path
}

impl<B: Fs> FileStorage<B> {
    /// Create with default resolver (IterativeResolver).
    pub fn new(backend: B) -> Self {
        FileStorage {
            backend,
            resolver: Box::new(IterativeResolver::new()),
        }
    }

    /// Create with custom resolver.
    pub fn with_resolver(backend: B, resolver: impl PathResolver + 'static) -> Self {
        FileStorage {
            backend,
            resolver: Box::new(resolver),
        }
    }

    /// Type-erase the backend for simpler types (opt-in boxing).
    pub fn boxed(self) -> FileStorage<Box<dyn Fs>> {
        FileStorage {
            backend: Box::new(self.backend),
            resolver: self.resolver,
        }
    }
}
```

### Path Resolution

FileStorage handles path resolution for virtual backends via the boxed `PathResolver`. The default `IterativeResolver` provides symlink-aware canonicalization.

Backends implementing `SelfResolving` (like `VRootFsBackend`) skip resolution since the OS handles it.

---

## Type Erasure (Opt-in)

When you need simpler types (e.g., storing in collections), use `.boxed()`:

```rust
use anyfs::{MemoryBackend, SqliteBackend, FileStorage, Fs};

// Type-erased for uniform storage
let filesystems: Vec<FileStorage<Box<dyn Fs>>> = vec![
    FileStorage::new(MemoryBackend::new()).boxed(),
    FileStorage::new(SqliteBackend::open("a.db")?).boxed(),
    FileStorage::new(SqliteBackend::open("b.db")?).boxed(),
];
```

**When to use `.boxed()`:**

| Situation                        | Use Generic     | Use `.boxed()` |
| -------------------------------- | --------------- | -------------- |
| Local variables                  | Yes             | No             |
| Function params                  | Yes (`impl Fs`) | No             |
| Return types                     | Yes (`impl Fs`) | No             |
| Collections of mixed backends    | No              | **Yes**        |
| Struct fields (want simple type) | Maybe           | **Yes**        |

---

## Direct Backend Access

If you don't need the wrapper, use backends directly:

```rust
use anyfs::{MemoryBackend, QuotaLayer, FileStorage};

let backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build());

// Use FileStorage for std::fs-style paths
let fs = FileStorage::new(backend);
fs.write("/file.txt", b"data")?;
```

`FileStorage<B>` is part of the `anyfs` crate, not a separate crate.
