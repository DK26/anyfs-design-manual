# Which Crate Should I Use?

---

## Decision Guide

| You want to... | Use |
|----------------|-----|
| Build an application | `anyfs` + `anyfs-container` |
| Use built-in backends (Memory, SQLite, VRootFs) | `anyfs` |
| Use built-in middleware (Quota, PathFilter, etc.) | `anyfs` |
| Implement a custom backend | `anyfs-backend` only |
| Implement custom middleware | `anyfs-backend` only |

---

## Quick Examples

### Simple usage

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut fs = FilesContainer::new(MemoryBackend::new());
fs.create_dir_all("/data")?;
fs.write("/data/file.txt", b"hello")?;
```

### With middleware (quotas, sandboxing, tracing)

```rust
use anyfs::{SqliteBackend, Quota, FeatureGuard, PathFilter, Tracing};
use anyfs_container::FilesContainer;

let backend = SqliteBackend::open("tenant.db")?;

let stack = Tracing::new(
    PathFilter::new(
        FeatureGuard::new(
            Quota::new(backend)
                .with_max_total_size(100 * 1024 * 1024)
        )
    )
    .allow("/workspace/**")
    .deny("**/.env")
);

let mut fs = FilesContainer::new(stack);
```

### Using Layer trait (alternative syntax)

```rust
use anyfs::{SqliteBackend, QuotaLayer, FeatureGuardLayer, TracingLayer};
use anyfs_container::FilesContainer;

let backend = SqliteBackend::open("tenant.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGuardLayer::new())
    .layer(TracingLayer::new());

let mut fs = FilesContainer::new(backend);
```

### Custom backend implementation

```rust
use anyfs_backend::{VfsBackend, VfsError, Metadata, DirEntry};
use std::path::Path;

pub struct MyBackend;

impl VfsBackend for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        todo!()
    }

    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        todo!()
    }

    // ... implement all 25 methods
}
```

### Custom middleware implementation

```rust
use anyfs_backend::{VfsBackend, Layer, VfsError};
use std::path::Path;

pub struct MyMiddleware<B: VfsBackend> {
    inner: B,
}

impl<B: VfsBackend> VfsBackend for MyMiddleware<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        // Intercept, transform, or delegate
        self.inner.read(path)
    }
    // ... implement all methods
}

pub struct MyMiddlewareLayer;

impl<B: VfsBackend> Layer<B> for MyMiddlewareLayer {
    type Backend = MyMiddleware<B>;
    fn layer(self, backend: B) -> Self::Backend {
        MyMiddleware { inner: backend }
    }
}
```

---

## Common Mistakes

- **Don't depend on `anyfs`** if you're only implementing a backend or middleware. Use `anyfs-backend`.
- **Don't put policy in backends.** Use middleware (Quota, PathFilter, etc.).
- **Don't put policy in FilesContainer.** It's just an ergonomic wrapper.
