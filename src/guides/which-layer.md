# Which Crate Should I Use?

---

## Decision Guide

- Building an application? Use `anyfs-container` (`FilesContainer`).
- Need a built-in backend (memory/sqlite/vrootfs)? Use `anyfs`.
- Implementing your own backend? Depend only on `anyfs-traits`.

---

## Quick Examples

### Application code (recommended)

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut container = FilesContainer::new(MemoryBackend::new());
container.create_dir_all("/data")?;
container.write("/data/file.txt", b"hello")?;
```

### Application code with policy (quotas + feature whitelist)

```rust
use anyfs::SqliteBackend;
use anyfs_container::ContainerBuilder;

let mut container = ContainerBuilder::new(SqliteBackend::open_or_create("tenant.db")?)
    .max_total_size(100 * 1024 * 1024)
    .symlinks()
    .max_symlink_resolution(40)
    .build()?;
```

### Custom backend implementation

```rust
use anyfs_traits::{VfsBackend, VirtualPath};

pub struct MyBackend;

impl VfsBackend for MyBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        todo!()
    }

    // ... implement the remaining methods
}
```

---

## Common Mistake

If you are implementing a backend, avoid depending on `anyfs` unless you specifically need built-in backends.
Use `anyfs-traits` as your dependency.
