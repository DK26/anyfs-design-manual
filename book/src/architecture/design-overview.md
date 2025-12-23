# AnyFS - Design Overview

**Status:** Current  
**Last updated:** 2025-12-23

---

## What This Project Is

AnyFS is a small Rust ecosystem for **virtual filesystem backends** with an optional **policy/container layer**.

It is designed for:
- portable application storage (SQLite backend = single `.db` file)
- tenant isolation (one container per tenant)
- quotas/limits (enforced in the container layer)
- containment and safe path handling (`strict-path`)

It is not a POSIX emulator and does not try to expose OS-specific behavior.

---

## Crates

| Crate | Purpose | Who uses it |
|------|---------|-------------|
| `anyfs-traits` | Minimal contract: `VfsBackend` + core types + `VirtualPath` re-export | Backend implementers |
| `anyfs` | Re-exports `anyfs-traits` + built-in backends (feature-gated) | Most users |
| `anyfs-container` | `FilesContainer<B: VfsBackend>` + limits + feature whitelist (least privilege) | Application code |

### Dependency Graph

```
strict-path
   -> anyfs-traits
        -> anyfs (built-in backends)
        -> anyfs-container (policy/quotas)
```

---

## Two-Layer Path Handling

AnyFS uses two path types on purpose:

1. **User-facing API (`FilesContainer`)** accepts `impl AsRef<Path>` for ergonomics.
2. **Backend API (`VfsBackend`)** uses `&VirtualPath` (from `strict-path`) for type safety.

The container validates and normalizes the user path once, then passes a `VirtualPath` to the backend.

---

## Core Trait: `VfsBackend` (in `anyfs-traits`)

`VfsBackend` is a path-based trait aligned with `std::fs` naming.

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

**Semantics (high-level):**
- `read`/`write`/`metadata`/`exists`/`copy` follow symlinks.
- `symlink_metadata` and `read_link` do not follow.
- `remove_file` removes the symlink itself, not the target.

---

## Built-in Backends (in `anyfs`)

These are provided by the `anyfs` crate and selected via Cargo features:

- `memory` (default): `MemoryBackend` (test/reference backend)
- `vrootfs`: `VRootFsBackend` (contained real filesystem via `strict_path::VirtualRoot`)
- `sqlite`: `SqliteBackend` (single-file portable database backend)

---

## `FilesContainer` (in `anyfs-container`)

`FilesContainer<B>` wraps any backend and provides:
- ergonomic paths (`impl AsRef<Path>`)
- quota enforcement (limits)
- a **feature whitelist** for advanced behavior (least privilege)

### Feature Whitelist (Least Privilege)

Advanced features are **disabled by default**. Enable only what you need:

- `symlinks` gates symlink creation and symlink-following behavior
- `hard_links` gates hard-link creation
- `permissions` gates permission mutation via `set_permissions`

```rust
use anyfs_container::ContainerBuilder;

let mut container = ContainerBuilder::new(backend)
    .symlinks()
    .max_symlink_resolution(40)
    .hard_links()
    .permissions()
    .max_total_size(100 * 1024 * 1024)
    .build()?;
```

When a feature is disabled, operations that require it return `ContainerError::FeatureNotEnabled("...")`.

### Limits

Limits are enforced by the container layer (not by backends):
- `max_total_size`
- `max_file_size`
- `max_node_count`
- `max_dir_entries`
- `max_path_depth`

---

## Security Model (Summary)

- **Containment:** `VirtualPath` ensures validated paths cannot escape the virtual root.
- **Least privilege:** the container disables advanced features by default and requires explicit opt-in.
- **No host paths:** application code interacts only with virtual paths; backends decide storage.

For more detail, see `book/src/comparisons/security.md`.