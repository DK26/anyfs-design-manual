# AnyFS Container - Technical Comparison with Alternatives

This document compares AnyFS (especially `anyfs-container`) with existing Rust filesystem abstractions.

---

## Executive summary

AnyFS is opinionated around three goals:

- **Safety by construction**: container APIs accept `impl AsRef<Path>`, but backends only see validated `&VirtualPath` (from `strict-path`).
- **Least privilege by default**: advanced behaviors (symlinks, hard links, permission mutation) are denied unless explicitly enabled per container.
- **Portable storage**: the SQLite backend turns a tenant filesystem into a single portable `.db` file.

---

## Compared solutions

| Solution | What it is | Where it fits |
|----------|------------|---------------|
| `vfs` | General-purpose VFS trait + backends | Simple path-based abstraction, streaming handles |
| `virtual-filesystem` | std::fs-like interface with sandbox option | Small abstraction for in-process sandboxing |
| OpenDAL | Object storage access layer | Cloud/object storage (async-first) |
| AgentFS | SQLite-backed agent sandbox | OS-level sandboxing + auditing (product-focused) |
| AnyFS | VFS backends + policy layer | Embedded storage, multi-tenant quotas, portability |

---

## 1. Trait / API design

### `vfs` crate (typical shape)

`vfs` exposes a path-based trait taking raw strings and often returns streaming handles:

```rust
pub trait FileSystem: Send + Sync {
    fn read_dir(&self, path: &str) -> VfsResult<Box<dyn Iterator<Item = String> + Send>>;
    fn create_dir(&self, path: &str) -> VfsResult<()>;
    fn open_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndRead + Send>>;
    fn create_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndWrite + Send>>;
    fn metadata(&self, path: &str) -> VfsResult<VfsMetadata>;
    fn exists(&self, path: &str) -> VfsResult<bool>;
    fn remove_file(&self, path: &str) -> VfsResult<()>;
    fn remove_dir(&self, path: &str) -> VfsResult<()>;
}
```

**Implications:**
- Backends must validate/normalize paths themselves.
- The API surface is typically "filesystem flavored", but not standardized on `std::fs` naming.

### AnyFS: two-layer path handling + policy layer

AnyFS splits responsibilities across three crates:

- `anyfs-traits`: minimal `VfsBackend` + core types (backend implementers depend only on this)
- `anyfs`: re-exports `anyfs-traits` and provides built-in backends (feature-gated)
- `anyfs-container`: `FilesContainer<B: VfsBackend>` plus limits and feature whitelist

**Backend trait (`anyfs-traits`):** validated paths only.

```rust
use strict_path::VirtualPath;

pub trait VfsBackend: Send {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
    // ... (20 std::fs-aligned methods total)
}
```

**User-facing API (`anyfs-container`):** ergonomic paths, plus enforcement.

```rust
use std::path::Path;

impl<B: VfsBackend> FilesContainer<B> {
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    // ... names aligned with std::fs
}
```

**Implications:**
- Path validation happens once, centrally, before any backend sees the path.
- Limits and default-deny features are consistent across all backends.

---

## 2. What makes AnyFS Container better

### 2.1 Centralized path safety

- `VirtualPath` normalizes `.` / `..` lexically and prevents escaping the virtual root.
- Backends do not accept raw strings, reducing the attack surface.

### 2.2 Least privilege (feature whitelist)

Advanced behavior is disabled by default and enabled explicitly per container instance:

```rust
use anyfs_container::ContainerBuilder;

let container = ContainerBuilder::new(backend)
    .symlinks()
    .max_symlink_resolution(40)
    .hard_links()
    .permissions()
    .build()?;
```

This supports a default-deny posture for apps handling untrusted input.

### 2.3 Built-in quotas and limits

`FilesContainer` can enforce limits like:
- max total bytes
- max file size
- max node count
- max directory entries
- max path depth

Backends remain focused on storage, while enforcement stays consistent.

### 2.4 Portable "single file" storage (SQLite backend)

With the `sqlite` backend, an entire tenant filesystem is a single `.db` file that can be copied/moved as a unit.

### 2.5 Dependency control

- `anyfs` uses Cargo features (`memory` default, `sqlite`, `vrootfs`) so users only compile what they use.
- Custom backend authors depend only on `anyfs-traits`.

---

## 3. Tradeoffs / downsides

### 3.1 Fewer backends (initially)

AnyFS intentionally starts small: Memory, SQLite, and a contained host filesystem backend.

### 3.2 Sync-only, whole-file I/O

The core API is sync-first and the common read/write APIs are whole-buffer (`Vec<u8>`).
Streaming handles and async APIs are future work.

### 3.3 Not full POSIX

AnyFS aims for `std::fs` alignment, but does not try to exactly emulate OS edge cases.

### 3.4 Cross-platform link semantics

Symlink and hard-link behavior differs across OSes and filesystems. AnyFS exposes links as a virtual concept, but real host filesystem behavior (especially on Windows) may vary.

---

## 4. When to use what

- Use **AnyFS Container** when you need embedded app storage, multi-tenant isolation, quotas, and portability.
- Use **`vfs`** when you want a general-purpose VFS abstraction with streaming handles and you are comfortable with backend-specific path validation.
- Use **OpenDAL** when your "filesystem" is actually object storage (S3/GCS/Azure) and you want async + middleware.

---

## 5. Feature matrix (high level)

| Feature | AnyFS Container | `vfs` | OpenDAL |
|--------|------------------|-------|---------|
| Path validation centralized | Yes (`VirtualPath`) | No | No |
| Quotas/limits | Yes | No | No |
| Least-privilege defaults | Yes (whitelist) | No | N/A |
| SQLite backend | Yes (feature `sqlite`) | No | No |
| Host filesystem containment | Yes (`VRootFsBackend` + `VirtualRoot`) | Backend-dependent | N/A |
| Streaming I/O | Not yet | Often yes | Yes |
| Async API | Not yet | Backend-dependent | Yes |

---

If you spot an inconsistency between this document and `book/src/architecture/design-overview.md` or `AGENTS.md`, treat those as authoritative.
