# VFS Container — API Quick Reference

**Condensed reference for developers**

---

## Installation

```toml
[dependencies]
anyfs-container = "0.1"
anyfs = "0.1"
```

With specific backends:

```toml
[dependencies]
anyfs-container = "0.1"
anyfs = { version = "0.1", features = ["sqlite", "memory"] }
```

---

## Creating a Container

```rust
use anyfs_container::FilesContainer;
use anyfs::{SqliteBackend, MemoryBackend, VirtualPath};

// In-memory (testing)
let container = FilesContainer::new(MemoryBackend::new());

// SQLite (production)
let container = FilesContainer::new(
    SqliteBackend::open_or_create("data.db")?
);

// Full configuration
let container = FilesContainer::builder()
    .backend(SqliteBackend::open_or_create("data.db")?)
    .symlinks(true)
    .hard_links(false)
    .max_total_size(100 * 1024 * 1024)  // 100 MB
    .max_file_size(10 * 1024 * 1024)    // 10 MB
    .max_node_count(10_000)
    .max_dir_entries(1_000)
    .max_path_depth(64)
    .max_name_length(255)
    .build()?;
```

---

## Path Handling

```rust
use anyfs::VirtualPath;

// Create paths
let path = VirtualPath::new("/documents/report.pdf")?;
let root = VirtualPath::root();  // "/"

// Path operations
path.parent()           // Some("/documents")
path.file_name()        // Some("report.pdf")
path.depth()            // 2
path.components()       // ["documents", "report.pdf"]
path.join("v2")?        // "/documents/report.pdf/v2"
path.as_str()           // "/documents/report.pdf"

// Normalization (automatic)
VirtualPath::new("//a/../b/./c")  // → "/b/c"
VirtualPath::new("/../../../a")   // → "/a" (can't escape root)
```

---

## File Operations

### Reading

```rust
// Check existence
container.exists(&path)                    // → bool

// Get metadata
let meta = container.metadata(&path)?;
meta.size                                  // file size
meta.kind                                  // File | Directory | Symlink
meta.created_at                            // Option<Timestamp>
meta.modified_at
meta.permissions                           // Option<Permissions>

// Read file
let bytes = container.read(&path)?;        // → Vec<u8>
let chunk = container.read_range(&path, 0, 1024)?;  // partial read

// List directory
let entries = container.list(&path)?;      // → Vec<DirEntry>
for entry in entries {
    println!("{}: {:?}", entry.name, entry.kind);
}

// Read symlink target (without following)
let target = container.read_link(&path)?;  // → VirtualPath
```

### Writing

```rust
// Create directory
container.mkdir(&path)?;                   // single level
container.mkdir_all(&path)?;               // recursive

// Write file (create or overwrite)
container.write(&path, b"content")?;

// Append to file
container.append(&path, b"more")?;

// Create symlink (requires .symlinks(true))
container.symlink(&link_path, &target_path)?;

// Create hard link (requires .hard_links(true))
container.hard_link(&link_path, &target_path)?;

// Delete
container.remove(&path)?;                  // file or empty dir
container.remove_all(&path)?;              // recursive

// Rename/move
container.rename(&from, &to)?;

// Copy
container.copy(&from, &to)?;               // single file
container.copy_all(&from, &to)?;           // recursive
```

### Metadata Operations

```rust
// Permissions (requires .permissions(true))
container.set_permissions(&path, Permissions { mode: 0o644 })?;

// Timestamps
container.set_times(&path, Times {
    accessed: Some(Timestamp::now()),
    modified: Some(Timestamp::now()),
})?;

// Extended attributes (requires .extended_attrs(true))
container.set_xattr(&path, "user.tag", b"important")?;
let val = container.get_xattr(&path, "user.tag")?;  // Option<Vec<u8>>
container.remove_xattr(&path, "user.tag")?;
let names = container.list_xattrs(&path)?;          // Vec<String>
```

---

## Import / Export

```rust
use std::path::Path;

// Import from host filesystem
let stats = container.import_from_host(
    Path::new("/real/path/to/folder"),
    &VirtualPath::new("/imported")?,
)?;
println!("Imported {} files, {} bytes", stats.files_imported, stats.bytes_imported);

// Export to host filesystem
let stats = container.export_to_host(
    &VirtualPath::new("/documents")?,
    Path::new("/real/export/path"),
)?;
```

---

## Capacity Management

```rust
// Check usage
let usage = container.usage()?;
usage.total_size       // bytes used
usage.node_count       // total nodes
usage.file_count
usage.directory_count
usage.symlink_count

// Check limits
let limits = container.limits();
limits.max_total_size  // Option<u64>
limits.max_file_size

// Check remaining
let remaining = container.remaining()?;
remaining.bytes        // Option<u64>
remaining.nodes
remaining.can_write    // quick check

// Clear all content
container.clear()?;

// Destroy backing storage (if supported)
container.destroy()?;  // consumes container
```

---

## Error Handling

```rust
use anyfs_container::{ContainerError, CapacityError};
use anyfs::{VfsError, PathError};

match container.write(&path, &data) {
    Ok(()) => println!("Written"),
    
    Err(VfsError::NotFound(p)) => println!("Path not found: {}", p),
    Err(VfsError::AlreadyExists(p)) => println!("Already exists: {}", p),
    Err(VfsError::NotADirectory(p)) => println!("Not a directory: {}", p),
    Err(VfsError::NotAFile(p)) => println!("Not a file: {}", p),
    Err(VfsError::DirectoryNotEmpty(p)) => println!("Not empty: {}", p),
    Err(VfsError::SymlinkLoop(p)) => println!("Symlink loop at: {}", p),
    
    Err(VfsError::Capacity(CapacityError::TotalSizeExceeded { used, limit })) => {
        println!("Storage full: {} / {} bytes", used, limit);
    }
    Err(VfsError::Capacity(CapacityError::FileSizeExceeded { size, limit })) => {
        println!("File too large: {} > {} bytes", size, limit);
    }
    
    Err(VfsError::FeatureNotEnabled(feature)) => {
        println!("Feature not enabled: {}", feature);
    }
    
    Err(e) => println!("Other error: {}", e),
}
```

---

## Backend Lifecycle

```rust
use anyfs::{SqliteBackend, BackendLifecycle};

// Create new (fails if exists)
let backend = SqliteBackend::create("new.db")?;

// Open existing (fails if doesn't exist)
let backend = SqliteBackend::open("existing.db")?;

// Open or create
let backend = SqliteBackend::open_or_create("data.db")?;

// Destroy
backend.destroy()?;  // deletes the file
```

---

## Quick Patterns

### Ensure parent directories exist

```rust
use anyfs::VfsBackend;

fn write_with_parents(
    container: &mut FilesContainer<impl VfsBackend>,
    path: &VirtualPath,
    data: &[u8],
) -> Result<(), ContainerError> {
    if let Some(parent) = path.parent() {
        container.mkdir_all(&parent)?;
    }
    container.write(path, data)
}
```

### Walk directory tree

```rust
fn walk(
    container: &FilesContainer<impl VfsBackend>,
    path: &VirtualPath,
    f: &mut impl FnMut(&VirtualPath, &DirEntry),
) -> Result<(), ContainerError> {
    for entry in container.list(path)? {
        let child_path = path.join(entry.name.as_str())?;
        f(&child_path, &entry);
        if entry.kind == NodeKind::Directory {
            walk(container, &child_path, f)?;
        }
    }
    Ok(())
}
```

### Calculate directory size

```rust
fn dir_size(
    container: &FilesContainer<impl VfsBackend>,
    path: &VirtualPath,
) -> Result<u64, ContainerError> {
    let mut total = 0u64;
    for entry in container.list(path)? {
        let child = path.join(entry.name.as_str())?;
        match entry.kind {
            NodeKind::File { size, .. } => total += size,
            NodeKind::Directory => total += dir_size(container, &child)?,
            _ => {}
        }
    }
    Ok(total)
}
```

---

## Type Reference

| Type | Description |
|------|-------------|
| `FilesContainer<B>` | Main API, generic over backend |
| `VirtualPath` | Validated, normalized path |
| `NodeId` | Unique node identifier |
| `ContentId` | Unique content identifier |
| `Name` | Validated filename component |
| `NodeKind` | `File`, `Directory`, `Symlink` |
| `Metadata` | File/directory metadata |
| `DirEntry` | Directory listing entry |
| `Permissions` | Unix-style mode bits |
| `Timestamp` | Milliseconds since epoch |
| `CapacityLimits` | Quota configuration |
| `CapacityUsage` | Current usage stats |

| Trait | Crate | Description |
|-------|-------|-------------|
| `VfsBackend` | `anyfs` | Core storage operations |
| `BackendLifecycle` | `anyfs` | Create/open/destroy |

| Error | Crate | Description |
|-------|-------|-------------|
| `VfsError` | `anyfs` | Backend-level errors |
| `PathError` | `anyfs` | Path parsing errors |
| `ContainerError` | `anyfs-container` | Container-level errors |
| `CapacityError` | `anyfs-container` | Quota exceeded errors |

---

*For full details, see the [Design Document](./anyfs-container-design.md).*
