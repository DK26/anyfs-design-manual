# VfsBackend Trait (anyfs-traits)

**The core backend contract for AnyFS**

---

## Overview

`VfsBackend` is the minimal interface a storage backend implements.

- It is **path-based** and aligned with `std::fs` naming.
- It uses `&VirtualPath` (from `strict-path`) so all paths are validated.
- It does not include quotas or application policy; that lives in `anyfs-container`.

If you are implementing a custom backend, depend only on `anyfs-traits`.

---

## Trait Surface

```rust
use strict_path::VirtualPath;

pub trait VfsBackend: Send {
    // Read
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;

    // Write
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    // Links
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    // Permissions
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
}
```

---

## Notes on Semantics

- `read`/`write`/`metadata`/`exists`/`copy` follow symlinks.
- `symlink_metadata` and `read_link` do not follow.
- `remove_file` removes the symlink itself, not the target.

The container layer may still deny certain operations via feature whitelisting.

---

## Implementing a Backend

- Depend on `anyfs-traits` only.
- Use `VirtualPath` as your canonical path type.
- Implement the semantics consistently across backends (a shared conformance suite is recommended).

See `book/src/implementation/backend-guide.md` for a step-by-step guide.