# Layered Traits (anyfs-backend)

AnyFS uses a **layered trait architecture** for maximum flexibility with minimal complexity.

---

## Trait Hierarchy

```
                    VfsPosix
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   VfsHandles      VfsLock       VfsXattr
        │              │              │
        └──────────────┼──────────────┘
                       │
                    VfsFuse
                       │
                   VfsInode
                       │
                    VfsFull
                       │
        ┌──────┬───────┼───────┬──────┐
        │      │       │       │      │
   VfsLink  VfsPerm  VfsSync VfsStats │
        │      │       │       │      │
        └──────┴───────┼───────┴──────┘
                       │
                      Vfs  ← Most users only need this
                       │
           ┌───────────┼───────────┐
           │           │           │
        VfsRead    VfsWrite     VfsDir
```

**Simple rule:** Import `Vfs` for basic use. Add traits as needed for advanced features.

---

## Layer 1: Core Traits (Required)

### VfsRead

```rust
pub trait VfsRead: Send {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
}
```

### VfsWrite

```rust
pub trait VfsWrite: Send {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;
}
```

### VfsDir

```rust
pub trait VfsDir: Send {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
}
```

---

## Layer 2: Extended Traits (Optional)

### VfsLink

```rust
pub trait VfsLink: Send {
    fn symlink(&mut self, target: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
}
```

### VfsPermissions

```rust
pub trait VfsPermissions: Send {
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;
}
```

### VfsSync

```rust
pub trait VfsSync: Send {
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
}
```

### VfsStats

```rust
pub trait VfsStats: Send {
    fn statfs(&self) -> Result<StatFs, VfsError>;
}
```

---

## Layer 3: Inode Trait (For FUSE)

### VfsInode

```rust
pub trait VfsInode: Send {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, VfsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, VfsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError>;
}
```

---

## Layer 4: POSIX Traits (Full POSIX)

### VfsHandles

```rust
pub trait VfsHandles: Send {
    fn open(&mut self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, VfsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, VfsError>;
    fn write_at(&mut self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, VfsError>;
    fn close(&mut self, handle: Handle) -> Result<(), VfsError>;
}
```

### VfsLock

```rust
pub trait VfsLock: Send {
    fn lock(&mut self, handle: Handle, lock: LockType) -> Result<(), VfsError>;
    fn try_lock(&mut self, handle: Handle, lock: LockType) -> Result<bool, VfsError>;
    fn unlock(&mut self, handle: Handle) -> Result<(), VfsError>;
}
```

### VfsXattr

```rust
pub trait VfsXattr: Send {
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, VfsError>;
    fn set_xattr(&mut self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), VfsError>;
    fn remove_xattr(&mut self, path: impl AsRef<Path>, name: &str) -> Result<(), VfsError>;
    fn list_xattr(&self, path: impl AsRef<Path>) -> Result<Vec<String>, VfsError>;
}
```

---

## Convenience Supertraits

These are automatically implemented via blanket impls:

```rust
/// Basic filesystem - covers 90% of use cases
pub trait Vfs: VfsRead + VfsWrite + VfsDir {}
impl<T: VfsRead + VfsWrite + VfsDir> Vfs for T {}

/// Full filesystem with all std::fs features
pub trait VfsFull: Vfs + VfsLink + VfsPermissions + VfsSync + VfsStats {}
impl<T: Vfs + VfsLink + VfsPermissions + VfsSync + VfsStats> VfsFull for T {}

/// FUSE-mountable filesystem
pub trait VfsFuse: VfsFull + VfsInode {}
impl<T: VfsFull + VfsInode> VfsFuse for T {}

/// Full POSIX filesystem
pub trait VfsPosix: VfsFuse + VfsHandles + VfsLock + VfsXattr {}
impl<T: VfsFuse + VfsHandles + VfsLock + VfsXattr> VfsPosix for T {}
```

---

## When to Use Each Level

| Level | Trait | Use When |
|-------|-------|----------|
| 1 | `Vfs` | Basic file operations (read, write, dirs) |
| 2 | `VfsFull` | Need links, permissions, sync, or stats |
| 3 | `VfsFuse` | FUSE mounting or hardlink support |
| 4 | `VfsPosix` | Full POSIX (file handles, locks, xattr) |

---

## Implementing Functions

Use trait bounds to specify requirements:

```rust
// Works with any backend
fn process_files(fs: &impl Vfs) -> Result<(), VfsError> {
    let data = fs.read("/input.txt")?;
    fs.write("/output.txt", &data)?;
    Ok(())
}

// Requires link support
fn create_backup(fs: &mut (impl Vfs + VfsLink)) -> Result<(), VfsError> {
    fs.hard_link("/data.txt", "/data.txt.bak")?;
    Ok(())
}

// Requires FUSE-level support
fn mount_filesystem(fs: impl VfsFuse) -> Result<(), VfsError> {
    anyfs_fuse::FuseMount::mount(fs, "/mnt/myfs")?;
    Ok(())
}
```

---

## Extension Trait

`VfsBackendExt` provides convenience methods for any `Vfs` backend:

```rust
pub trait VfsBackendExt: Vfs {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError>;
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError>;
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
}

// Blanket implementation
impl<B: Vfs> VfsBackendExt for B {}
```
