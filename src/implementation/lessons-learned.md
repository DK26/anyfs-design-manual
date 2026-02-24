# Lessons from Similar Projects

**Analysis of issues from `vfs` and `agentfs` to inform AnyFS design.**

This chapter documents problems encountered by similar projects and how AnyFS addresses them. These lessons are incorporated into our [Implementation Plan](./plan.md) and [Backend Guide](./backend-guide.md).

---

## Summary

| Priority | Issue                      | AnyFS Response                            |
| -------- | -------------------------- | ----------------------------------------- |
| 1        | Panics instead of errors   | No-panic policy, always return `Result`   |
| 2        | Thread safety problems     | Concurrent stress tests required          |
| 3        | Inconsistent path handling | Normalize in one place, test edge cases   |
| 4        | Poor error ergonomics      | `FsError` with context fields             |
| 5        | Missing documentation      | Performance & thread safety docs required |
| 6        | Platform issues            | Cross-platform CI pipeline                |

---

## 1. Thread Safety Issues

### What Happened

| Project | Issue                                                       | Problem                            |
| ------- | ----------------------------------------------------------- | ---------------------------------- |
| vfs     | [#72](https://github.com/manuel-woelker/rust-vfs/issues/72) | RwLock panic in production         |
| vfs     | [#47](https://github.com/manuel-woelker/rust-vfs/issues/47) | `create_dir_all` races with itself |

**Root cause:** Insufficient synchronization in concurrent access patterns.

### AnyFS Response

- **Test concurrent operations explicitly** - stress test with multiple threads
- **Document thread safety guarantees** per backend
- **`Fs: Send`** bound is intentional
- **`MemoryBackend` uses `Arc<RwLock<...>>`** for interior mutability

**Required tests:**
```rust
#[test]
fn test_concurrent_create_dir_all() {
    let backend = Arc::new(RwLock::new(create_backend()));
    let handles: Vec<_> = (0..10).map(|_| {
        let backend = backend.clone();
        std::thread::spawn(move || {
            let mut backend = backend.write().unwrap();
            let _ = backend.create_dir_all(std::path::Path::new("/a/b/c/d"));
        })
    }).collect();
    for handle in handles {
        handle.join().unwrap();
    }
}
```

---

## 2. Panics Instead of Errors

### What Happened

| Project | Issue                                                       | Problem                                  |
| ------- | ----------------------------------------------------------- | ---------------------------------------- |
| vfs     | [#8](https://github.com/manuel-woelker/rust-vfs/issues/8)   | AltrootFS panics when file doesn't exist |
| vfs     | [#23](https://github.com/manuel-woelker/rust-vfs/issues/23) | Unhandled edge cases cause panics        |
| vfs     | [#68](https://github.com/manuel-woelker/rust-vfs/issues/68) | MemoryFS panics in WebAssembly           |

**Root cause:** Using `.unwrap()` or `.expect()` on fallible operations.

### AnyFS Response

**No-panic policy:** Never use `.unwrap()` or `.expect()` in library code.

```rust
// BAD - will panic
let entry = self.entries.get(&path).unwrap();

// GOOD - returns error
let entry = self.entries.get(&path)
    .ok_or_else(|| FsError::NotFound { path: path.to_path_buf() })?;
```

**Edge cases that must return errors (not panic):**
- File doesn't exist
- Directory doesn't exist
- Path is empty string
- Invalid UTF-8 in path
- Parent directory missing
- Type mismatch (file vs directory)
- Concurrent access conflicts

---

## 3. Path Handling Inconsistencies

### What Happened

| Project | Issue                                                       | Problem                                      |
| ------- | ----------------------------------------------------------- | -------------------------------------------- |
| vfs     | [#24](https://github.com/manuel-woelker/rust-vfs/issues/24) | Inconsistent path definition across backends |
| vfs     | [#42](https://github.com/manuel-woelker/rust-vfs/issues/42) | Path join doesn't behave Unix-like           |
| vfs     | [#22](https://github.com/manuel-woelker/rust-vfs/issues/22) | Non-UTF-8 path support questions             |

**Root cause:** Each backend implemented path handling differently.

### AnyFS Response

- **Normalize paths in ONE place** (FileStorage resolver for virtual backends; `SelfResolving` backends delegate to the OS)
- **Consistent semantics:** always absolute, always `/` separator
- **Use `&Path` in core traits** for object safety; provide `impl AsRef<Path>` at the ergonomic layer (FileStorage/FsExt)

**Required conformance tests:**

| Input             | Expected Output |
| ----------------- | --------------- |
| `/foo/../bar`     | `/bar`          |
| `/foo/./bar`      | `/foo/bar`      |
| `//double//slash` | `/double/slash` |
| `/`               | `/`             |
| `` (empty)        | Error           |
| `/foo/bar/`       | `/foo/bar`      |

---

## 4. Static Lifetime Requirements

### What Happened

| Project | Issue                                                       | Problem                                |
| ------- | ----------------------------------------------------------- | -------------------------------------- |
| vfs     | [#66](https://github.com/manuel-woelker/rust-vfs/issues/66) | Why does filesystem require `'static`? |

**Root cause:** Design decision that confused users and limited flexibility.

### AnyFS Response

- **Avoid `'static` bounds** unless necessary
- **Our design:** `Fs: Send` (not `'static`)
- **Document why bounds exist** when needed

---

## 5. Missing Symlink Support

### What Happened

| Project | Issue                                                       | Problem                          |
| ------- | ----------------------------------------------------------- | -------------------------------- |
| vfs     | [#81](https://github.com/manuel-woelker/rust-vfs/issues/81) | Symlink support missing entirely |

**Root cause:** Symlinks are complex and were deferred indefinitely.

### AnyFS Response

- **Symlinks supported via `FsLink` trait** - backends that implement `FsLink` support symlinks
- **Compile-time capability** - no `FsLink` impl = no symlinks (won't compile)
- **Bound resolution depth** (default: 40 hops)
- **`strict-path` prevents symlink escapes** in `VRootFsBackend`

---

## 6. Error Type Ergonomics

### What Happened

| Project | Issue                                                       | Problem                                   |
| ------- | ----------------------------------------------------------- | ----------------------------------------- |
| vfs     | [#33](https://github.com/manuel-woelker/rust-vfs/issues/33) | Error type hard to match programmatically |

**Root cause:** Error enum wasn't designed for pattern matching.

### AnyFS Response

`FsError` includes context and is easy to match:

```rust
#[derive(Debug, thiserror::Error)]
pub enum FsError {
    #[error("not found: {path}")]
    NotFound { path: PathBuf },

    #[error("{operation}: already exists: {path}")]
    AlreadyExists { path: PathBuf, operation: &'static str },

    #[error("quota exceeded: limit {limit}, requested {requested}, usage {usage}")]
    QuotaExceeded { limit: u64, requested: u64, usage: u64 },

    #[error("feature not enabled: {feature} ({operation})")]
    FeatureNotEnabled { feature: &'static str, operation: &'static str },

    #[error("permission denied: {path} ({operation})")]
    PermissionDenied { path: PathBuf, operation: &'static str },

    // ...
}
```

---

## 7. Seek + Write Operations

### What Happened

| Project | Issue                                                       | Problem                           |
| ------- | ----------------------------------------------------------- | --------------------------------- |
| vfs     | [#35](https://github.com/manuel-woelker/rust-vfs/issues/35) | Missing file positioning features |

**Root cause:** Initial API was too simple.

### AnyFS Response

- **Streaming I/O:** `open_read`/`open_write` return `Box<dyn Read/Write + Send>`
- **Seek support varies by backend** - document which support it
- **Consider future:** `open_read_seek` variant or capability query

---

## 8. Read-Only Filesystem Request

### What Happened

| Project | Issue                                                       | Problem                          |
| ------- | ----------------------------------------------------------- | -------------------------------- |
| vfs     | [#58](https://github.com/manuel-woelker/rust-vfs/issues/58) | Request for immutable filesystem |

**Root cause:** No built-in way to enforce read-only access.

### AnyFS Response

**Already solved:** `ReadOnly<B>` middleware blocks all writes.

```rust
use anyfs::{ReadOnly, FileStorage};
use anyfs_sqlite::SqliteBackend;  // Ecosystem crate

let readonly_fs = FileStorage::new(
    ReadOnly::new(SqliteBackend::open("archive.db")?)
);
// All write operations return FsError::ReadOnly
```

This validates our middleware approach.

---

## 9. Performance Issues

### What Happened

| Project | Issue                                                       | Problem               |
| ------- | ----------------------------------------------------------- | --------------------- |
| agentfs | [#130](https://github.com/tursodatabase/agentfs/issues/130) | File deletion is slow |
| agentfs | [#135](https://github.com/tursodatabase/agentfs/issues/135) | Benchmark hangs       |

**Root cause:** SQLite operations not optimized, FUSE overhead.

### AnyFS Response

- **Batch operations** where possible in `SqliteBackend`
- **Use transactions** for multi-file operations
- **Document performance characteristics** per backend
- **Keep mounting optional** - core AnyFS stays a library; mount concerns are behind feature flags (`fuse`, `winfsp`)

**Documentation requirement:**
```rust
/// # Performance Characteristics
///
/// | Operation | Complexity | Notes |
/// |-----------|------------|-------|
/// | `read` | O(1) | Single DB query |
/// | `write` | O(n) | n = data size |
/// | `remove_dir_all` | O(n) | n = descendants |
pub struct SqliteBackend { ... }
```

---

## 10. Signal Handling / Shutdown

### What Happened

| Project | Issue                                                       | Problem                     |
| ------- | ----------------------------------------------------------- | --------------------------- |
| agentfs | [#129](https://github.com/tursodatabase/agentfs/issues/129) | Doesn't shutdown on SIGTERM |

**Root cause:** FUSE mount cleanup issues.

### AnyFS Response

- **Core stays a library** - daemon/mount shutdown concerns are behind feature flags
- **Ensure `Drop` implementations clean up properly**
- **`SqliteBackend` flushes on drop**

```rust
impl Drop for SqliteBackend {
    fn drop(&mut self) {
        if let Err(e) = self.sync() {
            eprintln!("Warning: failed to sync on drop: {}", e);
        }
    }
}
```

---

## 11. Platform Compatibility

### What Happened

| Project | Issue                                                       | Problem                |
| ------- | ----------------------------------------------------------- | ---------------------- |
| agentfs | [#132](https://github.com/tursodatabase/agentfs/issues/132) | FUSE-T support (macOS) |
| agentfs | [#138](https://github.com/tursodatabase/agentfs/issues/138) | virtio-fs support      |

**Root cause:** Platform-specific FUSE variants.

### AnyFS Response

- **We isolate this** - core traits stay pure; FUSE lives behind feature flags (`fuse`, `winfsp`) in the `anyfs` crate
- **Cross-platform by design** - Memory and SQLite work everywhere
- **`VRootFsBackend` uses `strict-path`** which handles Windows/Unix

**CI requirement:**
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

---

## 12. Multiple Sessions / Concurrent Access

### What Happened

| Project | Issue                                                       | Problem                                         |
| ------- | ----------------------------------------------------------- | ----------------------------------------------- |
| agentfs | [#126](https://github.com/tursodatabase/agentfs/issues/126) | Can't have multiple sessions on same filesystem |

**Root cause:** Locking/concurrency design.

### AnyFS Response

- **`SqliteBackend` uses WAL mode** for concurrent readers
- **Document concurrency model** per backend
- **`MemoryBackend` uses `Arc<RwLock<...>>`** for sharing

---

## 13. Patterns Validated by Linux Filesystems & Git

Based on a [detailed analysis](../comparisons/linux-fs-concepts-analysis.md) of LVM, ZFS, XFS, and Git internals, several AnyFS design decisions align with battle-tested patterns from these systems:

### What AnyFS Already Gets Right

| AnyFS Pattern | Validated By | Proven Concept |
|---|---|---|
| IndexedBackend (SQLite metadata + blob store) | ZFS special vdevs, Git LFS | Fast metadata on one tier, bulk data on another |
| Content-addressed blobs with refcounting | ZFS DDT, Git object store, XFS reflink | Automatic dedup, O(1) copy, safe GC |
| Write batching in SqliteBackend | XFS delayed allocation | Defer commits for contiguous allocation and throughput |
| SQLite WAL mode | XFS journaling, ZFS ZIL | Write-ahead logging for crash consistency |
| Two-phase commit (blob upload -> metadata commit) | ZFS COW, XFS intent items | Crash-safe multi-step operations |
| `Overlay` middleware | LVM snapshots, ZFS clones | COW layering without modifying the base |
| `QuotaLayer` | XFS project quotas, ZFS dataset quotas | Per-scope resource limits |
| `Cache` middleware | ZFS ARC | Read caching as a composable layer |

### Key Lessons Learned

**1. SQLite single-file limits are solvable with LVM-style spanning.**
LVM treats physical disks as "Physical Volumes" pooled into "Volume Groups." The same pattern works for SQLite: treat each `.db` file as a PV, aggregate them into a pool, and carve out logical filesystems. This overcomes practical size limits (tens of GB per file) by distributing extents across multiple smaller SQLite files, each staying in its performant range. Optimal extent size for SQLite: 64 KiB to 1 MiB (smaller than LVM's 4 MiB default due to SQLite BLOB overflow page behavior).

**2. Compression should be a middleware layer with early-abort.**
ZFS applies compression per-block and skips it if the result is larger than the original. AnyFS can do the same as a `CompressionLayer<B>` middleware. Already-compressed files (images, videos) get auto-detected and stored as-is. This is a low-effort, high-impact addition.

**3. Integrity verification (scrubbing) is essential for long-lived data.**
ZFS scrubs every block periodically. For IndexedBackend where `blob_id = sha256(content)`, a scrub re-hashes every blob and compares. This catches silent corruption, validates refcounts, and finds orphaned blobs -- all in one pass. Should be a standard backend operation.

**4. GC needs grace periods, not just refcount == 0.**
Git protects recently-unreferenced objects for 30-90 days before deleting. AnyFS should do the same -- a blob with `refcount = 0` might be referenced by an in-flight write. Adding a `created_at` check (e.g., "at least 1 day old") prevents race conditions.

**5. Merkle tree hashing enables efficient sync.**
Git's tree objects hash directory contents recursively. Two filesystems can be compared by checking root hashes -- if equal, nothing changed. If different, descend only into changed subtrees. This is O(changed paths), not O(total files). Valuable for send/receive replication.

**6. Shared blob stores (Git alternates) enable multi-tenant dedup.**
Multiple IndexedBackend instances can share a single blob store read-only ("base image"), each with their own writable store for tenant-specific files. This is how GitLab deduplicates fork storage. In AnyFS, a `ChainedBlobStore` searches local store first, then shared alternates.

---

## Issues We Already Avoid

Our design decisions already prevent these problems:

| Problem in Others        | AnyFS Solution                    |
| ------------------------ | --------------------------------- |
| No middleware pattern    | Tower-style composable middleware |
| No quota enforcement     | `Quota<B>` middleware             |
| No read-only mode        | `ReadOnly<B>` middleware          |
| Symlink complexity       | `FsLink` trait (compile-time)     |
| Path escape via symlinks | `strict-path` canonicalization    |
| FUSE complexity          | Isolated behind feature flags     |
| SQLite-only              | Multiple backends                 |
| Monolithic features      | Composable middleware             |

---

## References

- [rust-vfs Issues](https://github.com/manuel-woelker/rust-vfs/issues)
- [agentfs Issues](https://github.com/tursodatabase/agentfs/issues)
- [Implementation Plan](./plan.md) - incorporates these lessons
- [Backend Guide](./backend-guide.md) - implementation requirements
