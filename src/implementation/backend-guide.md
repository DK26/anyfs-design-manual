# Backend Implementer's Guide

This guide walks you through implementing a custom AnyFS backend.

If you are writing a backend for other people to consume, depend only on `anyfs-traits`.

---

## What you are implementing

A backend implements `VfsBackend`: a path-based trait aligned with `std::fs` naming.

Key properties:
- Backends receive `&VirtualPath` (from `strict-path`), not raw strings.
- The policy layer (`anyfs-container`) handles quotas/limits and feature whitelisting.

```rust
use anyfs_traits::{DirEntry, Metadata, Permissions, VirtualPath, VfsBackend, VfsError};

pub struct MyBackend {
    // storage fields...
}

impl VfsBackend for MyBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        todo!()
    }

    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        todo!()
    }

    // ... implement all 20 methods
}
```

---

## Step 1: Pick a data model

A practical internal model is a small "virtual inode" representation:

- **Directory**: owns a mapping `name -> child` (or can be derived from a global path index).
- **File**: points to bytes (inline, blob table, object store key, etc.).
- **Symlink**: stores a *target* `VirtualPath`.
- **Hard links**: multiple paths refer to the same underlying file content.

You do not need to expose any of this publicly; it is an implementation detail.

### Recommended minimum metadata

Your backend needs enough metadata to populate `Metadata` and return correct errors:
- file type (file/dir/symlink)
- size
- link count (`nlink`) for hard links
- timestamps (optional)
- permissions (even if simplified)

---

## Step 2: Implement the "easy" surface first

Start with these operations:

- `exists`
- `create_dir` / `create_dir_all`
- `read_dir`
- `write` / `read` / `append` / `read_range`
- `remove_file` / `remove_dir` / `remove_dir_all`
- `rename` / `copy`

Guidelines:

- Use `VirtualPath` as your canonical key. It is already normalized.
- Match `std::fs`-like error expectations:
  - creating an existing directory should return `AlreadyExists`
  - `remove_dir` should fail on non-empty directories
  - `remove_file` should fail on directories

---

## Step 3: Links (symlinks + hard links)

AnyFS includes link methods in the core trait.

### Symlinks

- `symlink(original, link)` creates a symlink at `link` whose target is `original`.
- `read_link(path)` returns the stored target.
- `symlink_metadata(path)` returns metadata about the symlink itself (not the target).

Note: `anyfs-container` may deny symlink operations unless explicitly enabled (least privilege).

### Hard links

- `hard_link(original, link)` creates a second directory entry pointing at the same underlying file.
- Update `nlink` and any reference-counting you maintain.
- `remove_file` decrements link count and only deletes the underlying content when it reaches zero.

---

## Step 4: Permissions

AnyFS models permissions as a minimal type (`Permissions`) to keep the cross-platform story simple.

- `set_permissions` should mutate the stored permissions.
- Backends may implement stricter semantics internally, but the API stays small.

As with links, `anyfs-container` may deny permission mutation unless enabled.

---

## Step 5: Error mapping

Backends should return `VfsError` variants that are meaningful to callers.

Recommended principles:
- Prefer structured variants like `NotFound(VirtualPath)` over stringly-typed errors.
- Keep backend-specific errors available (e.g., `VfsError::Backend(String)`) but do not leak host paths when possible.

---

## Step 6: Validate with a conformance suite

Backends drift without tests.

A useful conformance suite includes:
- directory creation/removal edge cases
- `rename` semantics for files vs directories
- symlink metadata vs metadata-following behavior
- hard link link-count accounting
- copy semantics (copy follows symlinks)

See `book/src/implementation/plan.md` for the recommended test breakdown.

---

## Checklist

- [ ] Depends only on `anyfs-traits`
- [ ] Implements all 20 `VfsBackend` methods
- [ ] Returns correct `std::fs`-like errors
- [ ] Symlinks store `VirtualPath` targets
- [ ] Hard links update `nlink` and share content
- [ ] Conformance suite passes

---

For the trait definition, see `book/src/traits/vfs-trait.md`.
