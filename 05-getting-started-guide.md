# VFS Container — Getting Started Guide

**A practical introduction with examples**

---

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
anyfs-container = "0.1"
anyfs = "0.1"
```

By default, `anyfs` includes the SQLite and in-memory backends. For specific backends only:

```toml
[dependencies]
anyfs-container = "0.1"
anyfs = { version = "0.1", default-features = false, features = ["sqlite"] }
```

Available features for `anyfs`:
- `sqlite` — SQLite-backed persistent storage
- `memory` — In-memory storage (for testing)
- `vrootfs` — Host filesystem backend (via strict-path)

---

## Quick Start

### Hello World

```rust
use anyfs_container::FilesContainer;
use anyfs::{MemoryBackend, VirtualPath};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create an in-memory container
    let mut container = FilesContainer::new(MemoryBackend::new());
    
    // Write a file
    let path = VirtualPath::new("/hello.txt")?;
    container.write(&path, b"Hello, VFS!")?;
    
    // Read it back
    let content = container.read(&path)?;
    println!("{}", String::from_utf8_lossy(&content));
    // Output: Hello, VFS!
    
    Ok(())
}
```

### Persistent Storage with SQLite

```rust
use anyfs_container::FilesContainer;
use anyfs::{SqliteBackend, VirtualPath};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create or open a SQLite-backed container
    let mut container = FilesContainer::new(
        SqliteBackend::open_or_create("my_data.db")?
    );
    
    // Create a directory structure
    container.mkdir_all(&VirtualPath::new("/documents/work")?)?;
    
    // Write some files
    container.write(
        &VirtualPath::new("/documents/work/notes.txt")?,
        b"Meeting notes for Monday"
    )?;
    
    // Data persists across program runs
    Ok(())
}
```

---

## Common Tasks

### Creating Directories

```rust
// Single directory (parent must exist)
container.mkdir(&VirtualPath::new("/documents")?)?;

// Recursive (creates parents as needed)
container.mkdir_all(&VirtualPath::new("/documents/projects/2024/q1")?)?;
```

### Reading and Writing Files

```rust
let path = VirtualPath::new("/data.txt")?;

// Write (creates or overwrites)
container.write(&path, b"line 1\n")?;

// Append
container.append(&path, b"line 2\n")?;

// Read entire file
let content = container.read(&path)?;

// Read partial (offset 0, length 6)
let partial = container.read_range(&path, 0, 6)?;
```

### Listing Directories

```rust
let path = VirtualPath::new("/documents")?;

for entry in container.list(&path)? {
    match entry.kind {
        NodeKind::File { size, .. } => {
            println!("📄 {} ({} bytes)", entry.name, size);
        }
        NodeKind::Directory => {
            println!("📁 {}/", entry.name);
        }
        NodeKind::Symlink { .. } => {
            println!("🔗 {}", entry.name);
        }
    }
}
```

### Checking Existence and Metadata

```rust
let path = VirtualPath::new("/maybe-exists.txt")?;

if container.exists(&path) {
    let meta = container.metadata(&path)?;
    println!("Size: {} bytes", meta.size);
    println!("Created: {:?}", meta.created_at);
    println!("Modified: {:?}", meta.modified_at);
}
```

### Copying and Moving

```rust
let src = VirtualPath::new("/original.txt")?;
let dst = VirtualPath::new("/copy.txt")?;

// Copy a file
container.copy(&src, &dst)?;

// Copy a directory recursively
container.copy_all(
    &VirtualPath::new("/documents")?,
    &VirtualPath::new("/backup/documents")?
)?;

// Move (rename)
container.rename(&src, &VirtualPath::new("/renamed.txt")?)?;
```

### Deleting

```rust
// Delete a file or empty directory
container.remove(&VirtualPath::new("/old-file.txt")?)?;

// Delete directory and all contents
container.remove_all(&VirtualPath::new("/old-folder")?)?;

// Clear everything (reset to empty)
container.clear()?;
```

---

## Configuration

### Builder Pattern

```rust
let container = FilesContainer::builder()
    .backend(SqliteBackend::open_or_create("data.db")?)
    
    // Enable optional features
    .symlinks(true)
    .hard_links(true)
    
    // Set capacity limits
    .max_total_size(500 * 1024 * 1024)  // 500 MB total
    .max_file_size(50 * 1024 * 1024)    // 50 MB per file
    .max_node_count(100_000)             // 100K files/dirs
    .max_dir_entries(5_000)              // 5K entries per directory
    
    // Set depth limits
    .max_path_depth(32)
    .max_symlink_resolution(20)
    
    .build()?;
```

### Checking Capacity

```rust
// Current usage
let usage = container.usage()?;
println!("Using {} of {} bytes", usage.total_size, container.limits().max_total_size.unwrap_or(u64::MAX));
println!("Files: {}", usage.file_count);
println!("Directories: {}", usage.directory_count);

// Remaining capacity
let remaining = container.remaining()?;
if !remaining.can_write {
    println!("Storage is full!");
}
```

---

## Optional Features

### Symbolic Links

Enable with `.symlinks(true)`:

```rust
let container = FilesContainer::builder()
    .backend(MemoryBackend::new())
    .symlinks(true)
    .build()?;

// Create a symlink
container.symlink(
    &VirtualPath::new("/shortcut")?,      // link location
    &VirtualPath::new("/deep/nested/dir")? // target
)?;

// Read the target (without following)
let target = container.read_link(&VirtualPath::new("/shortcut")?)?;

// Regular operations follow symlinks automatically
container.list(&VirtualPath::new("/shortcut")?)?;  // lists /deep/nested/dir
```

### Hard Links

Enable with `.hard_links(true)`:

```rust
let container = FilesContainer::builder()
    .backend(MemoryBackend::new())
    .hard_links(true)
    .build()?;

// Create a file
container.write(&VirtualPath::new("/original.txt")?, b"content")?;

// Create a hard link (same file, different path)
container.hard_link(
    &VirtualPath::new("/link.txt")?,
    &VirtualPath::new("/original.txt")?
)?;

// Both paths refer to the same content
// Deleting one doesn't affect the other
container.remove(&VirtualPath::new("/original.txt")?)?;
let content = container.read(&VirtualPath::new("/link.txt")?)?;  // Still works
```

---

## Import and Export

### Import from Host Filesystem

```rust
use std::path::Path;

// Import a single file
container.import_from_host(
    Path::new("/home/user/document.pdf"),
    &VirtualPath::new("/imported/document.pdf")?
)?;

// Import an entire directory
let stats = container.import_from_host(
    Path::new("/home/user/photos"),
    &VirtualPath::new("/photos")?
)?;

println!("Imported {} files ({} bytes)", 
    stats.files_imported, 
    stats.bytes_imported
);
```

### Export to Host Filesystem

```rust
// Export a file
container.export_to_host(
    &VirtualPath::new("/reports/annual.pdf")?,
    Path::new("/tmp/annual.pdf")
)?;

// Export a directory
let stats = container.export_to_host(
    &VirtualPath::new("/backup")?,
    Path::new("/mnt/external/backup")
)?;

println!("Exported {} files", stats.files_exported);
```

---

## Error Handling

### Pattern Matching on Errors

```rust
use anyfs_container::{ContainerError, CapacityError};

match container.write(&path, &large_data) {
    Ok(()) => println!("Written successfully"),

    Err(ContainerError::NotFound(p)) => {
        println!("Parent directory doesn't exist: {}", p);
    }

    Err(ContainerError::Capacity(CapacityError::FileSizeExceeded { size, limit })) => {
        println!("File too large: {} bytes (limit: {})", size, limit);
    }

    Err(ContainerError::Capacity(CapacityError::TotalSizeExceeded { used, limit })) => {
        println!("Storage full: {} / {} bytes", used, limit);
    }

    Err(e) => println!("Other error: {}", e),
}
```

### Common Error Scenarios

| Error | Cause | Solution |
|-------|-------|----------|
| `NotFound` | Path doesn't exist | Check with `exists()` first, or use `mkdir_all` |
| `AlreadyExists` | Creating over existing | Remove first, or check existence |
| `NotADirectory` | Operating on file as dir | Check `metadata().kind` |
| `DirectoryNotEmpty` | Removing non-empty dir | Use `remove_all()` instead |
| `FeatureNotEnabled` | Using disabled feature | Enable in builder (e.g., `.symlinks(true)`) |
| `FileSizeExceeded` | File too large | Increase limit or split file |
| `TotalSizeExceeded` | Storage full | Delete files or increase quota |

---

## Testing

### Use In-Memory Backend for Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use anyfs_container::FilesContainer;
    use anyfs::{MemoryBackend, VirtualPath};

    fn test_container() -> FilesContainer<MemoryBackend> {
        FilesContainer::new(MemoryBackend::new())
    }
    
    #[test]
    fn test_write_and_read() {
        let mut container = test_container();
        let path = VirtualPath::new("/test.txt").unwrap();
        
        container.write(&path, b"test data").unwrap();
        let content = container.read(&path).unwrap();
        
        assert_eq!(content, b"test data");
    }
    
    #[test]
    fn test_directory_operations() {
        let mut container = test_container();
        
        container.mkdir_all(&VirtualPath::new("/a/b/c").unwrap()).unwrap();
        
        assert!(container.exists(&VirtualPath::new("/a").unwrap()));
        assert!(container.exists(&VirtualPath::new("/a/b").unwrap()));
        assert!(container.exists(&VirtualPath::new("/a/b/c").unwrap()));
    }
}
```

### Test with Capacity Limits

```rust
#[test]
fn test_storage_limit() {
    let mut container = FilesContainer::builder()
        .backend(MemoryBackend::new())
        .max_total_size(1024)  // 1 KB limit
        .build()
        .unwrap();
    
    let path = VirtualPath::new("/big.bin").unwrap();
    let big_data = vec![0u8; 2048];  // 2 KB
    
    let result = container.write(&path, &big_data);
    
    assert!(matches!(
        result,
        Err(ContainerError::Capacity(CapacityError::TotalSizeExceeded { .. }))
    ));
}
```

---

## Best Practices

### 1. Use Path Constants

```rust
mod paths {
    use anyfs::VirtualPath;
    use once_cell::sync::Lazy;
    
    pub static CONFIG: Lazy<VirtualPath> = Lazy::new(|| {
        VirtualPath::new("/config").unwrap()
    });
    
    pub static DATA: Lazy<VirtualPath> = Lazy::new(|| {
        VirtualPath::new("/data").unwrap()
    });
}

// Usage
container.mkdir(&*paths::CONFIG)?;
```

### 2. Handle Errors Gracefully

```rust
use anyfs::VfsBackend;

fn ensure_parent_exists(
    container: &mut FilesContainer<impl VfsBackend>,
    path: &VirtualPath,
) -> Result<(), ContainerError> {
    if let Some(parent) = path.parent() {
        if !container.exists(&parent) {
            container.mkdir_all(&parent)?;
        }
    }
    Ok(())
}
```

### 3. Use Appropriate Backend for Use Case

| Use Case | Recommended Backend |
|----------|---------------------|
| Unit tests | `MemoryBackend` |
| Integration tests | `MemoryBackend` or temp `SqliteBackend` |
| Production | `SqliteBackend` |
| High-security | `SqliteBackend` with encryption |

### 4. Set Reasonable Limits

```rust
// Production defaults suggestion
FilesContainer::builder()
    .backend(backend)
    .max_total_size(1024 * 1024 * 1024)  // 1 GB
    .max_file_size(100 * 1024 * 1024)    // 100 MB
    .max_node_count(1_000_000)            // 1M nodes
    .max_dir_entries(10_000)              // 10K per dir
    .max_path_depth(64)
    .build()
```

---

## Next Steps

- [API Quick Reference](./02-api-quick-reference.md) — Condensed API overview
- [Backend Implementer's Guide](./03-backend-implementers-guide.md) — Create custom backends
- [Architecture Decision Records](./04-architecture-decision-records.md) — Understand design choices
- [Design Document](./anyfs-container-design.md) — Full technical specification

---

*Happy coding! 🗂️*
