# AnyFS Implementation Patterns

**Purpose:** Quick reference for implementing backends, middleware, and adapters.

---

## Trait Hierarchy (Pick Your Level)

```
VfsPosix  ← Full POSIX (handles, locks, xattr)
    ↑
VfsFuse   ← FUSE-mountable (+ inodes)
    ↑
VfsFull   ← std::fs features (+ links, permissions, sync, stats)
    ↑
  Vfs     ← Basic filesystem (90% of use cases)
    ↑
VfsRead + VfsWrite + VfsDir  ← Core traits
```

**Rule:** Implement the lowest level you need. Higher levels include all below.

---

## Pattern 1: Implement a Backend

### Minimum: Implement `Vfs` (3 traits)

```rust
use anyfs_backend::{VfsRead, VfsWrite, VfsDir, VfsError, Metadata, DirEntry, FileType, Permissions};
use std::path::Path;

pub struct MyBackend { /* your storage */ }

impl VfsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        todo!("return file contents")
    }
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError> {
        String::from_utf8(self.read(path)?).map_err(|e| VfsError::Backend(e.to_string()))
    }
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError> {
        todo!("return slice of file")
    }
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        todo!("check if path exists")
    }
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError> {
        todo!("return Metadata { inode, nlink, file_type, size, permissions, created, modified, accessed }")
    }
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Read + Send>, VfsError> {
        Ok(Box::new(std::io::Cursor::new(self.read(path)?)))
    }
}

impl VfsWrite for MyBackend {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        todo!("create or overwrite file")
    }
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        todo!("append to file")
    }
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("delete file")
    }
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("move/rename")
    }
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("copy file")
    }
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError> {
        todo!("resize file")
    }
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Write + Send>, VfsError> {
        todo!("return writer")
    }
}

impl VfsDir for MyBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError> {
        todo!("return Vec<DirEntry { name, inode, file_type }>")
    }
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("create single directory")
    }
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("create directory and parents")
    }
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("remove empty directory")
    }
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("remove directory recursively")
    }
}
// MyBackend now implements Vfs automatically!
```

### Add Links/Permissions: Implement `VfsFull` (add 4 traits)

```rust
impl VfsLink for MyBackend {
    fn symlink(&mut self, target: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("create symlink pointing to target")
    }
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("create hard link (same inode)")
    }
    fn read_link(&self, path: impl AsRef<Path>) -> Result<std::path::PathBuf, VfsError> {
        todo!("return symlink target")
    }
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError> {
        todo!("metadata without following symlink")
    }
}

impl VfsPermissions for MyBackend {
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError> {
        todo!("set file permissions")
    }
}

impl VfsSync for MyBackend {
    fn sync(&mut self) -> Result<(), VfsError> {
        todo!("flush all writes to storage")
    }
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!("flush writes for one file")
    }
}

impl VfsStats for MyBackend {
    fn statfs(&self) -> Result<anyfs_backend::StatFs, VfsError> {
        todo!("return StatFs { total_bytes, available_bytes, ... }")
    }
}
// MyBackend now implements VfsFull!
```

### Add FUSE Support: Implement `VfsFuse` (add 1 trait)

```rust
impl VfsInode for MyBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        todo!("return unique inode for path")
    }
    fn inode_to_path(&self, inode: u64) -> Result<std::path::PathBuf, VfsError> {
        todo!("return path for inode")
    }
    fn lookup(&self, parent_inode: u64, name: &std::ffi::OsStr) -> Result<u64, VfsError> {
        todo!("find child inode by name")
    }
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError> {
        todo!("get metadata by inode (fast path)")
    }
}
// MyBackend now implements VfsFuse!
```

---

## Pattern 2: Implement Middleware

**Rule:** Middleware implements the same traits as its inner backend, intercepting some methods.

### Template

```rust
use anyfs_backend::{VfsRead, VfsWrite, VfsDir, VfsError, Layer};

pub struct MyMiddleware<B> {
    inner: B,
    // your state
}

impl<B> MyMiddleware<B> {
    pub fn new(inner: B) -> Self {
        Self { inner }
    }
}

// Implement each trait, delegating or intercepting as needed
impl<B: VfsRead> VfsRead for MyMiddleware<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        // INTERCEPT: add your logic
        let data = self.inner.read(path)?;  // DELEGATE
        // INTERCEPT: modify result
        Ok(data)
    }

    // For passthrough methods, just delegate:
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.inner.exists(path)
    }
    // ... implement all VfsRead methods
}

impl<B: VfsWrite> VfsWrite for MyMiddleware<B> {
    // ... implement all VfsWrite methods
}

impl<B: VfsDir> VfsDir for MyMiddleware<B> {
    // ... implement all VfsDir methods
}

// Optional: Layer for .layer() syntax
pub struct MyMiddlewareLayer { /* config */ }

impl<B: Vfs> Layer<B> for MyMiddlewareLayer {
    type Backend = MyMiddleware<B>;
    fn layer(self, backend: B) -> Self::Backend {
        MyMiddleware::new(backend)
    }
}
```

### Common Middleware Patterns

| Pattern | Intercept | Delegate | Example |
|---------|-----------|----------|---------|
| **Logging** | All (before/after) | All | `Tracing` |
| **Block writes** | Write methods → return error | Read methods | `ReadOnly` |
| **Transform data** | `read`/`write` | Everything else | `Encryption` |
| **Check permissions** | All (before) | All | `PathFilter` |
| **Enforce limits** | Write methods (check size) | Read methods | `Quota` |
| **Count operations** | All (increment counter) | All | `Counter` |

### Example: ReadOnly Middleware

```rust
impl<B: VfsWrite> VfsWrite for ReadOnly<B> {
    fn write(&mut self, _: impl AsRef<Path>, _: &[u8]) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "write" })
    }
    fn remove_file(&mut self, _: impl AsRef<Path>) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "remove_file" })
    }
    // ... all write methods return ReadOnly error
}
```

### Example: Encryption Middleware

```rust
impl<B: VfsRead> VfsRead for Encrypted<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let encrypted = self.inner.read(path)?;
        Ok(self.decrypt(&encrypted))
    }
}

impl<B: VfsWrite> VfsWrite for Encrypted<B> {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        let encrypted = self.encrypt(data);
        self.inner.write(path, &encrypted)
    }
}
```

---

## Pattern 3: Implement an Adapter

### Adapter FROM another crate's trait

```rust
// Wrap external crate's filesystem to use as AnyFS backend
pub struct ExternalCompat<F>(F);

impl<F: external::FileSystem> VfsRead for ExternalCompat<F> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        self.0.read_file(path).map_err(|e| VfsError::Backend(e.to_string()))
    }
    // Map each method, converting errors
}

impl<F: external::FileSystem> VfsWrite for ExternalCompat<F> { /* ... */ }
impl<F: external::FileSystem> VfsDir for ExternalCompat<F> { /* ... */ }
// Only implement traits the external crate supports
```

### Adapter TO another crate's trait

```rust
// Wrap AnyFS backend to use with external crate
pub struct AnyFsCompat<B: Vfs>(B);

impl<B: Vfs> external::FileSystem for AnyFsCompat<B> {
    fn read_file(&self, path: &Path) -> external::Result<Vec<u8>> {
        self.0.read(path).map_err(|e| external::Error::from(e.to_string()))
    }
    // Only expose methods the external trait requires
}
```

---

## Error Handling

**Rule:** Always return `VfsError`, never panic.

```rust
// Common error returns
VfsError::NotFound { path, operation }      // Path doesn't exist
VfsError::AlreadyExists { path, operation } // Path already exists
VfsError::NotAFile { path }                 // Expected file, got directory
VfsError::NotADirectory { path }            // Expected directory, got file
VfsError::DirectoryNotEmpty { path }        // Can't remove non-empty dir
VfsError::ReadOnly { operation }            // Write blocked by ReadOnly middleware
VfsError::AccessDenied { path, reason }     // Blocked by PathFilter
VfsError::QuotaExceeded { limit, requested, usage }
VfsError::NotSupported { operation }        // Backend doesn't support this
VfsError::Backend(String)                   // Backend-specific error
```

---

## Quick Reference: What to Implement

| You want to... | Implement |
|----------------|-----------|
| Basic file operations | `VfsRead` + `VfsWrite` + `VfsDir` (= `Vfs`) |
| + Symlinks/hardlinks | + `VfsLink` |
| + Permissions | + `VfsPermissions` |
| + Durability (sync) | + `VfsSync` |
| + Filesystem stats | + `VfsStats` |
| All of above | `Vfs` + `VfsLink` + `VfsPermissions` + `VfsSync` + `VfsStats` (= `VfsFull`) |
| + FUSE mounting | + `VfsInode` (= `VfsFuse`) |
| + File handles/locks | + `VfsHandles` + `VfsLock` + `VfsXattr` (= `VfsPosix`) |

---

## Usage Examples

### Create a backend stack

```rust
use anyfs::{MemoryBackend, Quota, PathFilter, Tracing};

let backend = MemoryBackend::new();
let limited = Quota::new(backend).with_max_total_size(100 * 1024 * 1024);
let filtered = PathFilter::new(limited).allow("/workspace/**").deny("**/.env");
let traced = Tracing::new(filtered);
```

### Using Layer syntax

```rust
let backend = MemoryBackend::new()
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(PathFilterLayer::new().allow("/workspace/**"))
    .layer(TracingLayer::new());
```

### With FileStorage wrapper

```rust
use anyfs_container::FileStorage;

let mut fs = FileStorage::new(backend);
fs.write("/file.txt", b"hello")?;
let data = fs.read("/file.txt")?;
```

### Type-safe containers

```rust
struct Sandbox;
struct UserData;

let sandbox: FileStorage<Sandbox> = FileStorage::with_marker(MemoryBackend::new());
let userdata: FileStorage<UserData> = FileStorage::with_marker(SqliteBackend::open("data.db")?);

fn process(fs: &FileStorage<Sandbox>) { /* only accepts Sandbox */ }
```
