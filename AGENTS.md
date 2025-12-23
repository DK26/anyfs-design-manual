# AGENTS.md — Instructions for AI Assistants

READ THIS FIRST before making any changes to this repository.

---

## Project Overview

This is the VFS Ecosystem — three Rust crates for virtual filesystem abstraction:

| Crate | Purpose |
|-------|---------|
| `anyfs-traits` | Minimal crate — trait definition, types, re-exports `VirtualPath` |
| `anyfs` | Core VFS — re-exports traits, provides built-in backends (optional features) |
| `anyfs-container` | Higher-level wrapper — capacity limits, tenant isolation |

---

## CRITICAL: Old vs New Design

This repository contains documentation from multiple design iterations. IGNORE OLD DESIGN DOCUMENTS.

### CURRENT DESIGN (use this)

- Crate names: `anyfs-traits`, `anyfs`, `anyfs-container`
- Trait crate: `anyfs-traits` — minimal, contains `VfsBackend` trait + types
- Path type in `VfsBackend` trait: `&VirtualPath` (from `strict-path` crate)
- Path type in `FilesContainer` API: `impl AsRef<Path>` (for user ergonomics)
- Trait name: `VfsBackend` (defined in `anyfs-traits`)
- Trait style: Path-based methods aligned with `std::fs` (`read`, `write`, `create_dir`, etc.)
- Backends: `VRootFsBackend`, `MemoryBackend`, `SqliteBackend` (in `anyfs`, feature-gated)
- Three crates: `anyfs-traits` (trait), `anyfs` (backends), `anyfs-container` (wrapper with limits)

### OLD DESIGN (ignore this)

If you see any of these, it is from the old design — do not use:

- `vfs-switchable` crate name — WRONG (renamed to `anyfs`)
- `vfs` as single crate name — WRONG (conflicts with existing crates.io package)
- `impl AsRef<Path>` in `VfsBackend` trait — WRONG (`VfsBackend` uses `&VirtualPath`)
- Custom `VirtualPath` type definition — WRONG (use re-export from `strict-path`)
- `NodeId`, `ContentId`, `ChunkId` — WRONG (old graph-store model)
- `StorageBackend` trait with `insert_node`, `insert_edge` — WRONG (old graph-store model)
- `Transaction`, `Snapshot` traits — WRONG (old transactional model)
- `FsBackend` — WRONG name (it is `VRootFsBackend` to convey virtual root containment)
- `FilesContainer` as the only project — WRONG (there are three crates now)
- Two-crate structure (`anyfs` + `anyfs-container`) — OUTDATED (now three crates)
- Any mention of "graph store" or "node/edge" model — WRONG
- `list()` method — WRONG (renamed to `read_dir()` for std::fs alignment)
- `mkdir()` / `mkdir_all()` — WRONG (renamed to `create_dir()` / `create_dir_all()`)
- Single `remove()` method — WRONG (split into `remove_file()` and `remove_dir()`)
- `remove_all()` — WRONG (renamed to `remove_dir_all()`)
- "13 methods" in trait — OUTDATED (now 20 methods with symlinks, hard links, permissions)

---

## The Correct Architecture

```
┌─────────────────────────────────────────┐
│  User Application                       │
├─────────────────────────────────────────┤
│  anyfs-container                        │  ← Capacity limits, tenant isolation
│  FilesContainer<B: VfsBackend>          │     Uses impl AsRef<Path> (ergonomic)
├─────────────────────────────────────────┤
│  anyfs                                  │  ← Built-in backends (feature-gated)
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │  ← Optional backend implementations
│ Backend  │  Backend │  Backend          │
├──────────┴──────────┴───────────────────┤
│  anyfs-traits                           │  ← Minimal: trait + types
│  VfsBackend trait, VfsError, Metadata   │     Re-exports VirtualPath
├─────────────────────────────────────────┤
│  strict-path (external)                 │  ← VirtualPath, VirtualRoot
└─────────────────────────────────────────┘
```

### Dependency Graph

```
strict-path (external)
     ↑
anyfs-traits (trait + types)
     ↑
     ├── anyfs (re-exports traits, provides backends)
     │
     └── anyfs-container (wraps any VfsBackend)
```

Key insight: Two-layer path handling:
1. User-facing (FilesContainer): `impl AsRef<Path>` — ergonomic, accepts any path-like type
2. Internal (VfsBackend): `&VirtualPath` — type-safe, pre-validated

---

## The Correct Trait (in anyfs-traits)

```rust
// anyfs-traits/src/lib.rs
pub use strict_path::VirtualPath;

/// A virtual filesystem backend.
/// All backends implement full filesystem semantics including symlinks and hard links.
/// Method names align with std::fs where applicable.
pub trait VfsBackend: Send {
    // READ OPERATIONS

    /// Read entire file contents as bytes. Follows symlinks. (like std::fs::read)
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;

    /// Read entire file contents as UTF-8 string. Follows symlinks. (like std::fs::read_to_string)
    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;

    /// Read a byte range from a file. Follows symlinks. (extension)
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;

    /// Check if path exists. Follows symlinks. (like Path::exists)
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;

    /// Get metadata, following symlinks. (like std::fs::metadata)
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;

    /// Get metadata without following symlinks. (like std::fs::symlink_metadata)
    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;

    /// Read directory contents. (like std::fs::read_dir)
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;

    /// Read symbolic link target. (like std::fs::read_link)
    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;

    // WRITE OPERATIONS

    /// Write bytes to file, creating or overwriting. Follows symlinks. (like std::fs::write)
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;

    /// Append bytes to file. Follows symlinks. (like OpenOptions::append)
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;

    /// Create directory. (like std::fs::create_dir)
    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Create directory and all parent directories. (like std::fs::create_dir_all)
    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove file. Removes symlink itself, not target. (like std::fs::remove_file)
    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove empty directory. (like std::fs::remove_dir)
    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove directory and all contents recursively. (like std::fs::remove_dir_all)
    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Rename or move file/directory. (like std::fs::rename)
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    /// Copy file. Follows symlinks. (like std::fs::copy)
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    // LINKS

    /// Create symbolic link. `link` will point to `original`. (like std::os::unix::fs::symlink)
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    /// Create hard link. `link` will share content with `original`. (like std::fs::hard_link)
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    // PERMISSIONS

    /// Set permissions on file or directory. (like std::fs::set_permissions)
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
}
```

This is 20 path-based methods aligned with `std::fs`. Not a graph store. Not transactional.

VirtualPath comes from `strict-path` crate and is re-exported by `anyfs-traits` (and `anyfs`).

---

## FilesContainer (User-Facing API)

```rust
use std::path::Path;
use anyfs::{VfsBackend, VirtualPath};

impl<B: VfsBackend> FilesContainer<B> {
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    pub fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, ContainerError>;
    pub fn exists(&self, path: impl AsRef<Path>) -> Result<bool, ContainerError>;
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<VirtualPath, ContainerError>;

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;
}
```

### Least-privilege feature whitelist (container policy)

AnyFS applies a high-security, default-deny posture at the container layer:

- Advanced behavior is disabled by default.
- The crate user explicitly enables only what they need (whitelist).
- Builder methods should use bare names (no `enable_*` prefix): `.symlinks()` not `.enable_symlinks()`.

Whitelisted features:

- `symlinks()` gates symlink creation and symlink-following behavior (bounded by `max_symlink_resolution`, default 40).
- `hard_links()` gates hard link creation.
- `permissions()` gates permission mutation via `set_permissions()`.

Cargo feature selection is separate: `anyfs` backends are feature-gated (`memory` default, `sqlite`, `vrootfs`).

---

## The three backends

### 1. VRootFsBackend

- Uses `strict-path::VirtualRoot` for containment
- A real directory on disk acts as the virtual root
- Paths are clamped (e.g., `/etc/passwd` -> `root_dir/etc/passwd`)
- Name conveys "Virtual Root Filesystem" containment

### 2. MemoryBackend

- In-memory storage
- Uses `VirtualPath` as keys
- For testing

### 3. SqliteBackend

- Single `.db` file contains entire filesystem
- Portable: copy file to move container
- Internal schema is an implementation detail

---

## Key dependencies

| Crate | Used by | Purpose |
|-------|---------|---------|
| `strict-path` | `anyfs-traits` | `VirtualPath` type (re-exported) |
| `thiserror` | `anyfs-traits` | Error types |
| `strict-path` | `anyfs` [vrootfs] | `VirtualRoot` for containment |
| `rusqlite` | `anyfs` [sqlite] | SQLite database access |

---

## Common mistakes to avoid

- Do not use old crate names like `vfs-switchable`.
- Do not use `impl AsRef<Path>` in `VfsBackend`.
- Do not define your own `VirtualPath`.
- Do not use old method names (`list`, `mkdir`, `remove`).

---

## File structure

```
anyfs-traits/
  Cargo.toml
  src/
    lib.rs
    backend.rs
    types.rs
    error.rs

anyfs/
  Cargo.toml
  src/
    lib.rs
    vrootfs/
    memory/
    sqlite/

anyfs-container/
  Cargo.toml
  src/
    lib.rs
    container.rs
    builder.rs
    limits.rs
    error.rs
```

---

## When in doubt

1. Crate names: `anyfs-traits`, `anyfs`, `anyfs-container`
2. Trait location: `anyfs-traits`
3. `VfsBackend` path type: `&VirtualPath`
4. `FilesContainer` path type: `impl AsRef<Path>`
5. Model: path-based methods aligned with `std::fs`

If documentation conflicts with this file, this file is correct.
