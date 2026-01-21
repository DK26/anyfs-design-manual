# Backends Guide

This guide explains each backend available for AnyFS—both built-in (in `anyfs`) and ecosystem crates (`anyfs-sqlite`, `anyfs-indexed`)—how they work internally, when to use them, and the trade-offs involved.

---

## Quick Reference: Which Backend Should You Use?

> **TL;DR** — Pick the first match from top to bottom:

| Your Situation                                 | Best Choice                       | Why                                                |
| ---------------------------------------------- | --------------------------------- | -------------------------------------------------- |
| Writing tests                                  | **MemoryBackend**                 | Fast, isolated, no cleanup                         |
| Running in WASM/browser                        | **MemoryBackend**                 | Simplest option                                    |
| Need encrypted single-file storage             | **anyfs-sqlite: SqliteBackend**   | AES-256 via `encryption` feature (ecosystem crate) |
| Need portable single-file database             | **anyfs-sqlite: SqliteBackend**   | Cross-platform, ACID (ecosystem crate)             |
| Large files (>100MB) with path isolation       | **anyfs-indexed: IndexedBackend** | Virtual paths + native disk I/O (ecosystem crate)  |
| Containing untrusted code to a directory       | **VRootFsBackend**                | Prevents path traversal attacks                    |
| Working with real files in trusted environment | **StdFsBackend**                  | Direct OS operations                               |
| Need layered filesystem (container-like)       | **Overlay** (middleware)          | Base + writable upper layer                        |

⚠️ **Security Warning:** `StdFsBackend` provides **NO isolation**. Never use with untrusted input.

> **Ecosystem Crates:** Complex backends like `SqliteBackend` and `IndexedBackend` live in separate crates (`anyfs-sqlite`, `anyfs-indexed`) because they require internal runtime complexity (connection pooling, sharding, chunking).

---

## Backend Categories

AnyFS backends fall into two fundamental categories based on **who resolves paths**:

| Category                       | Path Resolution          | Symlink Handling   | Isolation    |
| ------------------------------ | ------------------------ | ------------------ | ------------ |
| **Type 1: Virtual Filesystem** | PathResolver (pluggable) | Simulated by AnyFS | Complete     |
| **Type 2: Real Filesystem**    | Operating System         | Delegated to OS    | Partial/None |

### Type 1: Virtual Filesystem Backends

These backends store filesystem data in an abstract format (memory, database, etc.). **AnyFS handles path resolution via pluggable `PathResolver`** (see ADR-033), including:

- Path traversal (`..`, `.`)
- Symlink following (simulated)
- Hard link tracking (simulated)
- Path normalization

**Key benefit:** Complete isolation from the host OS. Identical behavior across all platforms.

### Type 2: Real Filesystem Backends

These backends delegate operations to the actual operating system. **The OS handles path resolution**, which means:

- Native symlink behavior
- Native permission enforcement
- Platform-specific edge cases
- Potential security considerations (path escapes)

**Key benefit:** Native performance and compatibility with existing files.

---

## Type 1: Virtual Filesystem Backends

### MemoryBackend

An in-memory filesystem. All data lives in RAM and is lost when the process exits.

```rust
use anyfs::{MemoryBackend, FileStorage};

let fs = FileStorage::new(MemoryBackend::new());
fs.write("/data.txt", b"Hello, World!")?;
```

#### How It Works

- Files and directories stored in a tree structure (`HashMap` or similar)
- Symlinks stored as data pointing to target paths
- Hard links share the same underlying data node
- All operations are memory-only (no disk I/O)
- Supports snapshots via `Clone` and persistence via `save_to()`/`load_from()`

#### Performance

| Operation       | Speed              | Notes                          |
| --------------- | ------------------ | ------------------------------ |
| Read/Write      | ⚡ **Very Fast**    | No I/O, pure memory operations |
| Path Resolution | ⚡ **Very Fast**    | In-memory tree traversal       |
| Large Files     | ⚠️ **Memory-bound** | Limited by available RAM       |

#### Advantages

- **Fastest backend** - no disk I/O overhead
- **Deterministic** - perfect for testing
- **Portable** - works on all platforms including WASM
- **Snapshots** - `Clone` creates instant backups
- **No cleanup** - no temp files to delete

#### Disadvantages

- **Volatile** - data lost on process exit (unless serialized)
- **Memory-limited** - large filesystems consume RAM
- **No persistence** - must explicitly save/load state

#### When to Use

| Use Case             | Recommendation                                |
| -------------------- | --------------------------------------------- |
| Unit tests           | ✅ **Ideal** - fast, isolated, deterministic   |
| Integration tests    | ✅ **Ideal** - no filesystem pollution         |
| Temporary workspaces | ✅ **Good** - fast scratch space               |
| Build caches         | ✅ **Good** - if fits in memory                |
| WASM/Browser         | ✅ **Ideal** - simplest option                 |
| Large file storage   | ❌ **Avoid** - use anyfs-sqlite or disk        |
| Persistent data      | ❌ **Avoid** - unless you handle serialization |

**✅ USE MemoryBackend when:**
- Writing unit tests (fast, isolated, deterministic)
- Writing integration tests (no filesystem pollution)
- Building temporary workspaces or scratch space
- Caching data that fits in memory
- Running in WASM/browser environments (simplest option)
- Need instant snapshots via `Clone`

**❌ DON'T USE MemoryBackend when:**
- Storing files larger than available RAM
- Data must survive process restart (use anyfs-sqlite)
- Working with existing files on disk (use VRootFsBackend)

---

### SqliteBackend (Ecosystem Crate)

> **Crate:** `anyfs-sqlite`
>
> Complex backends live in separate crates. See AGENTS.md "Crate Ecosystem" section.

Stores the entire filesystem in a single SQLite database file.

```rust
use anyfs_sqlite::SqliteBackend;
use anyfs::FileStorage;

let fs = FileStorage::new(SqliteBackend::open("myfs.db")?);
fs.write("/documents/report.txt", b"Annual Report")?;
```

#### How It Works

- Single `.db` file contains all files, directories, and metadata
- Schema: `nodes` table (path, type, content, permissions, timestamps)
- Symlinks stored as rows with target path in content
- Hard links share the same `inode` (row ID)
- Uses WAL mode for concurrent read access
- Connection pooling: multiple readers, single writer with batching
- Write batching: groups operations into transactions for efficiency
- Transactions ensure atomic operations

**Key insight:** "Writes are expensive." SqliteBackend batches writes internally because one transaction per batch is far more efficient than one transaction per operation.

#### Performance

| Operation       | Speed        | Notes                          |
| --------------- | ------------ | ------------------------------ |
| Read/Write      | 🐢 **Slower** | SQLite query overhead          |
| Path Resolution | 🐢 **Slower** | Database lookups per component |
| Transactions    | ✅ **Atomic** | ACID guarantees                |
| Large Files     | 🟡 **Varies** | See note below                 |

**Large file behavior:** SQLite streams BLOB content incrementally via `sqlite3_blob_read`/`write`, so files don't need to fit entirely in RAM. However, very large BLOBs (>100MB) can cause higher memory pressure during I/O operations due to SQLite's internal buffering and page management. For frequent large file operations, consider `IndexedBackend` which uses native file I/O.

**Performance note:** SQLite performance varies significantly based on hardware, configuration, and workload. With proper tuning (WAL mode, connection pooling, write batching), a single SQLite database on modern hardware can achieve high throughput. See [sqlite-operations.md](../implementation/sqlite-operations.md) for tuning guidance.

#### Advantages

- **Single-file portability** - entire filesystem in one `.db` file
- **ACID transactions** - atomic operations, crash recovery
- **Cross-platform** - works on all platforms including WASM
- **Complete isolation** - no interaction with host filesystem
- **Queryable** - can inspect with SQLite tools
- **Optional encryption** - AES-256 via SQLCipher with `encryption` feature

#### Disadvantages

- **Slower than memory** - database overhead on every operation
- **Single-writer** - SQLite's write lock limits concurrency
- **Large file overhead** - very large BLOBs (>100MB) have higher memory pressure due to SQLite buffering

#### When to Use

| Use Case               | Recommendation                                      |
| ---------------------- | --------------------------------------------------- |
| Portable storage       | ✅ **Ideal** - single file, works everywhere         |
| Embedded databases     | ✅ **Ideal** - self-contained                        |
| Sandboxed environments | ✅ **Good** - complete isolation                     |
| Encrypted storage      | ✅ **Good** - use `open_encrypted()` with feature    |
| Archive/backup         | ✅ **Good** - atomic, portable                       |
| Large media files      | ✅ **Works** - higher memory pressure during I/O     |
| High-throughput I/O    | ⚠️ **Tradeoff** - database overhead vs MemoryBackend |
| External tool access   | ❌ **Avoid** - files not on real filesystem          |

**✅ USE SqliteBackend when:**
- Need portable, single-file storage (easy to copy, backup, share)
- Building embedded/self-contained applications
- Complete isolation from host filesystem is required
- Want encryption (use `open_encrypted()` with `encryption` feature)
- Need ACID transactions and crash recovery
- Cross-platform consistency is critical

**❌ DON'T USE SqliteBackend when:**
- Files must be accessible to external tools (use VRootFsBackend)
- Minimizing memory pressure for very large files is critical (use anyfs-indexed: IndexedBackend)

> 💡 **SqliteBackend vs IndexedBackend:** Both provide complete path isolation. Choose SqliteBackend for single-file portability and portable storage. Choose IndexedBackend (anyfs-indexed) for very large files (>100MB) that need native disk streaming performance.

---

### IndexedBackend (Ecosystem Crate)

> **Crate:** `anyfs-indexed`
>
> Complex backends live in separate crates. See AGENTS.md "Crate Ecosystem" section.

A hybrid backend: **virtual paths** with **disk-based content storage**. Paths, directories, symlinks, and metadata are stored in an index database. File content is stored on the real filesystem as opaque blobs.

**Key insight:** Same isolation model as SqliteBackend, but file content stored externally for native I/O performance with large files.

```rust
use anyfs_indexed::IndexedBackend;
use anyfs::FileStorage;

// Files stored in ./storage/, index in ./storage/index.db
let fs = FileStorage::new(IndexedBackend::open("./storage")?);
fs.write("/documents/report.pdf", &pdf_bytes)?;
// Actually stored as: ./storage/a1b2c3d4-5678-...-1704067200.bin
```

#### How It Works

```
Virtual Path                    Real Storage
─────────────────────────────────────────────────────
/documents/report.pdf    →    ./storage/blobs/a1b2c3d4-...-1704067200.bin
/images/photo.jpg        →    ./storage/blobs/b2c3d4e5-...-1704067201.bin
/config.json             →    ./storage/blobs/c3d4e5f6-...-1704067202.bin

index.db contains:
┌─────────────────────────┬──────────────────────────────┬──────────┐
│ virtual_path            │ blob_name                    │ metadata │
├─────────────────────────┼──────────────────────────────┼──────────┤
│ /documents/report.pdf   │ a1b2c3d4-...-1704067200.bin  │ {...}    │
│ /images/photo.jpg       │ b2c3d4e5-...-1704067201.bin  │ {...}    │
└─────────────────────────┴──────────────────────────────┴──────────┘
```

- **Virtual filesystem, real content:** Directory structure, paths, symlinks, and metadata are virtual (stored in `index.db`). Only raw file content lives on disk as opaque blobs.
- Files stored with UUID + timestamp names (flat, meaningless filenames)
- `index.db` SQLite database maps virtual paths to blob names
- Symlinks and hard links are simulated in the index (not OS symlinks)
- Path resolution handled by AnyFS framework (Type 1 backend)
- File content streamed directly from disk (native I/O performance)

#### Performance

| Operation       | Speed           | Notes                       |
| --------------- | --------------- | --------------------------- |
| Read/Write      | 🟢 **Fast**      | Native disk I/O for content |
| Path Resolution | 🟡 **Moderate**  | Index lookup + disk access  |
| Large Files     | ✅ **Excellent** | Streamed directly from disk |
| Metadata Ops    | 🟢 **Fast**      | Index-only, no disk I/O     |

##### Index Optimization

The SQLite index benefits from the same performance tuning as `SqliteBackend`:

| Setting        | Default     | Purpose                                      |
| -------------- | ----------- | -------------------------------------------- |
| `journal_mode` | WAL         | Concurrent reads during metadata updates     |
| `synchronous`  | FULL        | Index integrity on power loss (safe default) |
| `cache_size`   | 16MB        | Smaller cache for metadata-only index        |
| `busy_timeout` | 5000ms      | Gracefully handle lock contention            |
| `auto_vacuum`  | INCREMENTAL | Reclaim space from deleted entries           |

**Connection pooling:** 4-8 reader connections for concurrent index queries; single writer for metadata updates. Blob I/O bypasses SQLite entirely, so the bottleneck is typically blob disk throughput, not index performance.

See [anyfs-indexed#9](https://github.com/DK26/anyfs-indexed/issues/9) for detailed performance guidance.

#### Advantages

- **Native file I/O** - content stored as raw files, fast streaming
- **Large file support** - uses OS file I/O, avoids SQLite BLOB buffering overhead
- **Complete path isolation** - virtual paths, same as SqliteBackend
- **Inspectable** - can see blob files on disk (though with opaque names)
- **Cross-platform** - works identically on all platforms

#### Disadvantages

- **Index dependency** - losing `index.db` = losing virtual structure (blobs become orphaned)
- **Two-component backup** - must copy directory + index.db together
- **Content exposure** - blob files are readable on disk (paths are hidden, content is not)
- **Not single-file portable** - unlike SqliteBackend

#### When to Use

| Use Case                | Recommendation                                        |
| ----------------------- | ----------------------------------------------------- |
| Large file storage      | ✅ **Ideal** - native I/O performance                  |
| Media libraries         | ✅ **Ideal** - stream large videos/images              |
| Document management     | ✅ **Good** - virtual paths + fast I/O                 |
| Sandboxed + large files | ✅ **Ideal** - virtual paths, real I/O                 |
| Single-file portability | ❌ **Avoid** - use anyfs-sqlite: SqliteBackend         |
| Content confidentiality | ⚠️ **Wrap** - use Encryption middleware for protection |
| WASM/Browser            | ❌ **Avoid** - requires real filesystem                |

**✅ USE IndexedBackend when:**
- Storing large files (videos, images, documents >100MB)
- Need native I/O performance for streaming content
- Building media libraries or document management systems
- Want virtual path isolation but with real disk performance
- Files are large but path structure should be sandboxed

**❌ DON'T USE IndexedBackend when:**
- Need single-file portability (use anyfs-sqlite: SqliteBackend)
- Content must be hidden from host filesystem (use anyfs-sqlite: SqliteBackend with encryption)
- Need WASM/browser support (use MemoryBackend)

> 🔒 **Encryption Tip:** If you need large file performance but content confidentiality matters, you can implement an `Encryption<B>` middleware wrapper to encrypt blob contents at rest. This is a user-defined middleware pattern (not built-in) - see the [middleware implementation guide](../implementation/middleware-implementation.md) for how to create custom middleware. Alternatively, use `SqliteBackend` with SQLCipher encryption for simpler encrypted storage.

---

## Type 2: Real Filesystem Backends

### StdFsBackend

Direct delegation to `std::fs`. Every call maps 1:1 to the standard library.

```rust
use anyfs::{StdFsBackend, FileStorage, NoOpResolver};

// SelfResolving backends require explicit NoOpResolver
let fs = FileStorage::with_resolver(StdFsBackend::new(), NoOpResolver);
fs.write("/tmp/data.txt", b"Hello")?; // Actually writes to /tmp/data.txt
```

#### How It Works

- Every method directly calls the equivalent `std::fs` function
- Paths passed through unchanged
- OS handles all resolution, symlinks, permissions
- Implements `SelfResolving` marker (use `NoOpResolver` to skip virtual resolution)

#### Performance

| Operation       | Speed        | Notes                |
| --------------- | ------------ | -------------------- |
| Read/Write      | 🟢 **Normal** | Native OS speed      |
| Path Resolution | ⚡ **Fast**   | OS kernel handles it |
| Symlinks        | ✅ **Native** | OS behavior          |

#### Advantages

- **Zero overhead** - direct OS calls
- **Full compatibility** - works with all existing files
- **Native features** - OS permissions, ACLs, xattrs
- **Middleware-ready** - add Quota, Tracing, etc. to real filesystem

#### Disadvantages

- **No isolation** - full filesystem access
- **No containment** - paths can escape anywhere
- **Platform differences** - Windows vs Unix behavior
- **Security risk** - must trust path inputs

#### When to Use

| Use Case                     | Recommendation                            |
| ---------------------------- | ----------------------------------------- |
| Adding middleware to real FS | ✅ **Ideal** - wrap with Quota, Tracing    |
| Trusted environments         | ✅ **Good** - when isolation not needed    |
| Migration path               | ✅ **Good** - gradually add AnyFS features |
| Full host FS features        | ✅ **Good** - ACLs, xattrs, etc.           |
| Untrusted input              | ❌ **Never** - use VRootFsBackend          |
| Sandboxing                   | ❌ **Never** - no containment whatsoever   |
| Multi-tenant systems         | ❌ **Avoid** - use virtual backends        |

**✅ USE StdFsBackend when:**
- Adding middleware (Quota, Tracing, etc.) to real filesystem operations
- Operating in a fully trusted environment with controlled inputs
- Migrating existing code to AnyFS incrementally
- Need full access to host filesystem features (ACLs, xattrs)
- Building tools that work with user's actual files

**❌ DON'T USE StdFsBackend when:**
- Handling untrusted path inputs (use VRootFsBackend)
- Any form of sandboxing is required (no containment!)
- Building multi-tenant systems (use virtual backends)
- Security isolation matters at all

> ⚠️ **Security Warning:** StdFsBackend provides **ZERO isolation**. Paths like `../../etc/passwd` will work. Only use with fully trusted, controlled inputs.

---

### VRootFsBackend

Sets a directory as a virtual root. All operations are contained within it.

> **Feature:** `vrootfs`

```rust
use anyfs::{VRootFsBackend, FileStorage, NoOpResolver};

// /home/user/sandbox becomes the virtual "/"
// SelfResolving backends require explicit NoOpResolver
let fs = FileStorage::with_resolver(
    VRootFsBackend::new("/home/user/sandbox")?,
    NoOpResolver
);

fs.write("/data.txt", b"Hello")?; 
// Actually writes to: /home/user/sandbox/data.txt

fs.read("/../../../etc/passwd")?;
// Resolves to: /home/user/sandbox/etc/passwd (clamped!)
```

#### How It Works

- Configured with a real directory as the "virtual root"
- All paths are validated and clamped to stay within root
- Uses `strict-path` crate for escape prevention
- Symlinks are followed but targets validated
- Implements `SelfResolving` marker (OS handles resolution after validation)

```
Virtual Path          Validation              Real Path
───────────────────────────────────────────────────────────────
/data.txt        →   validate & join    →   /home/user/sandbox/data.txt
/../../../etc    →   clamp to root      →   /home/user/sandbox/etc
/link → /tmp     →   validate target    →   ERROR or clamped
```

#### Performance

| Operation         | Speed          | Notes                        |
| ----------------- | -------------- | ---------------------------- |
| Read/Write        | 🟡 **Moderate** | Validation overhead          |
| Path Resolution   | 🐢 **Slower**   | Extra I/O for symlink checks |
| Symlink Following | 🐢 **Slower**   | Must validate each hop       |

#### Advantages

- **Path containment** - cannot escape virtual root
- **Real file access** - native OS performance for content
- **Symlink safety** - targets validated against root
- **Drop-in sandboxing** - wrap existing directories

#### Disadvantages

- **Performance overhead** - validation on every operation
- **Extra I/O** - symlink following requires `lstat` calls
- **Platform quirks** - symlink behavior varies (especially Windows)
- **Theoretical edge cases** - TOCTOU races exist but are difficult to exploit

#### When to Use

| Use Case                   | Recommendation                            |
| -------------------------- | ----------------------------------------- |
| User uploads directory     | ✅ **Ideal** - contain user content        |
| Plugin sandboxing          | ✅ **Good** - limit plugin file access     |
| Chroot-like isolation      | ✅ **Good** - without actual chroot        |
| AI agent workspaces        | ✅ **Good** - bound agent to directory     |
| Real FS + path containment | ✅ **Ideal** - native I/O with boundaries  |
| Maximum security           | ⚠️ **Careful** - theoretical TOCTOU exists |
| Cross-platform symlinks    | ⚠️ **Careful** - Windows behavior differs  |
| Complete host isolation    | ❌ **Avoid** - use SqliteBackend instead   |

**✅ USE VRootFsBackend when:**
- Containing user-uploaded content to a specific directory
- Sandboxing plugins, extensions, or untrusted code
- Need chroot-like isolation without actual chroot privileges
- Building AI agent workspaces with filesystem boundaries
- Want real filesystem performance with path containment

**❌ DON'T USE VRootFsBackend when:**
- Maximum security required (theoretical TOCTOU edge cases exist - use MemoryBackend)
- Need highest I/O performance (validation adds overhead)
- Cross-platform symlink consistency is critical (Windows differs)
- Want complete isolation from host (use SqliteBackend)

> 🔒 **Encryption Tip:** For sensitive data in sandboxed directories (user uploads, plugin workspaces, AI agent data), consider implementing an `Encryption<B>` middleware wrapper. This is a user-defined middleware pattern - you would create a custom Layer that encrypts data before delegating to the inner backend. See the [middleware implementation guide](../implementation/middleware-implementation.md) for the pattern.

---

## Composition Middleware

### Overlay<Base, Upper>

Union filesystem middleware combining a read-only base with a writable upper layer.

> **Note:** Overlay is middleware (in `anyfs/middleware/overlay.rs`), not a standalone backend. It composes two backends into a layered view.

```rust
use anyfs::{VRootFsBackend, MemoryBackend, Overlay, FileStorage};

// Base: read-only template
let base = VRootFsBackend::new("/var/templates")?;

// Upper: writable scratch layer  
let upper = MemoryBackend::new();

let fs = FileStorage::new(Overlay::new(base, upper));

// Read: checks upper first, falls back to base
let data = fs.read("/config.txt")?;

// Write: always goes to upper
fs.write("/config.txt", b"modified")?;

// Delete: creates "whiteout" in upper, shadows base
fs.remove_file("/unwanted.txt")?;
```

#### How It Works

```
┌─────────────────────────────────────────────────┐
│                  Overlay<B, U>                  │
├─────────────────────────────────────────────────┤
│  Read:   upper.exists(path)?                    │
│            → upper.read(path)                   │
│            : base.read(path)                    │
│                                                 │
│  Write:  upper.write(path, data)                │
│          (base unchanged)                       │
│                                                 │
│  Delete: upper.mark_whiteout(path)              │
│          (shadows base, doesn't delete it)      │
│                                                 │
│  List:   merge(base.read_dir(), upper.read_dir())│
│          - exclude whiteouts                    │
└─────────────────────────────────────────────────┘

         ┌──────────────┐
         │    Upper     │  ← Writes go here
         │ (MemoryFs)   │  ← Modifications stored here
         │              │  ← Whiteouts (deletions) here
         └──────┬───────┘
                │ if not found
                ▼
         ┌──────────────┐
         │     Base     │  ← Read-only layer
         │ (SqliteFs)   │  ← Original/template data
         │              │  ← Never modified
         └──────────────┘
```

- **Reads:** Check upper layer first, fall back to base
- **Writes:** Always go to upper layer (base is read-only)
- **Deletes:** Create "whiteout" marker in upper (shadows base file)
- **Directory listing:** Merge both layers, exclude whiteouts

#### Performance

| Operation            | Speed            | Notes                  |
| -------------------- | ---------------- | ---------------------- |
| Read (upper hit)     | ⚡ **Fast**       | Single layer lookup    |
| Read (base fallback) | 🟡 **Moderate**   | Two-layer lookup       |
| Write                | Depends on upper | Upper layer speed      |
| Directory listing    | 🐢 **Slower**     | Must merge both layers |

#### Advantages

- **Copy-on-write semantics** - modifications don't affect base
- **Instant rollback** - discard upper layer to reset
- **Space efficient** - only changes stored in upper
- **Template pattern** - share base across multiple instances
- **Testing isolation** - test against real data without modifying it

#### Disadvantages

- **Complexity** - whiteout handling, merge logic
- **Directory listing overhead** - must combine and filter
- **Two backends to manage** - lifecycle of both layers
- **Not true CoW** - doesn't deduplicate at block level

#### When to Use

| Use Case               | Recommendation                            |
| ---------------------- | ----------------------------------------- |
| Container images       | ✅ **Ideal** - base image + writable layer |
| Template filesystems   | ✅ **Ideal** - shared base, per-user upper |
| Testing with real data | ✅ **Ideal** - modify without consequences |
| Rollback capability    | ✅ **Good** - discard upper to reset       |
| Git-like branching     | ✅ **Good** - branch = new upper layer     |
| Simple use cases       | ❌ **Overkill** - use single backend       |
| Block-level CoW        | ❌ **Avoid** - Overlay is file-level       |
| Dir listing perf       | ❌ **Avoid** - merge overhead on listings  |

**✅ USE Overlay when:**
- Building container-like systems (base image + writable layer)
- Sharing a template filesystem across multiple instances
- Testing against production data without modifying it
- Need instant rollback capability (discard upper layer)
- Implementing git-like branching at filesystem level

**❌ DON'T USE Overlay when:**
- Simple, single-purpose filesystem (unnecessary complexity)
- Need block-level copy-on-write (Overlay is file-level)
- Directory listing performance is critical (merge overhead)
- Don't need layered semantics (use single backend)

---

## Backend Selection Guide

### Quick Decision Tree

```
Do you need persistence?
├─ No → MemoryBackend
└─ Yes
   ├─ Single portable file? → SqliteBackend
   ├─ Large files + path isolation? → IndexedBackend
   └─ Access existing files on disk?
      ├─ Need containment? → VRootFsBackend  
      └─ Trusted environment? → StdFsBackend
```

### Comparison Matrix

| Backend        | Speed       | Isolation  | Persistence   | Large Files   | WASM   |
| -------------- | ----------- | ---------- | ------------- | ------------- | ------ |
| MemoryBackend  | ⚡ Very Fast | ✅ Complete | ❌ None        | ⚠️ RAM-limited | ✅      |
| SqliteBackend  | 🐢 Slower    | ✅ Complete | ✅ Single file | ✅ Supported   | ✅      |
| IndexedBackend | 🟢 Fast      | ✅ Complete | ✅ Directory   | ✅ Native I/O  | ❌      |
| StdFsBackend   | 🟢 Normal    | ❌ None     | ✅ Native      | ✅ Native      | ❌      |
| VRootFsBackend | 🟡 Moderate  | ✅ Strong   | ✅ Native      | ✅ Native      | ❌      |
| Overlay        | Varies      | Varies     | Varies        | Varies        | Varies |

### By Use Case

| Use Case                     | Recommended                               |
| ---------------------------- | ----------------------------------------- |
| Unit testing                 | MemoryBackend                             |
| Integration testing          | MemoryBackend or SqliteBackend            |
| Portable application data    | SqliteBackend                             |
| Encrypted storage            | SqliteBackend (with `encryption` feature) |
| Large file + isolation       | IndexedBackend                            |
| Media libraries              | IndexedBackend                            |
| Plugin/agent sandboxing      | VRootFsBackend                            |
| Adding middleware to real FS | StdFsBackend                              |
| Container-like isolation     | Overlay<SqliteBackend, MemoryBackend>     |
| Template with modifications  | Overlay<Base, Upper>                      |
| WASM/Browser                 | MemoryBackend or SqliteBackend            |

---

## Platform Compatibility

| Backend        | Windows | Linux | macOS |  WASM  |
| -------------- | :-----: | :---: | :---: | :----: |
| MemoryBackend  |    ✅    |   ✅   |   ✅   |   ✅    |
| SqliteBackend  |    ✅    |   ✅   |   ✅   |   ✅*   |
| IndexedBackend |    ✅    |   ✅   |   ✅   |   ❌    |
| StdFsBackend   |    ✅    |   ✅   |   ✅   |   ❌    |
| VRootFsBackend |   ✅**   |   ✅   |   ✅   |   ❌    |
| Overlay        |    ✅    |   ✅   |   ✅   | Varies |

\* Requires wasm32-compatible SQLite build  
\** Windows symlinks require elevated privileges or Developer Mode

---

## Common Mistakes to Avoid

| ❌ Mistake                                                                   | ✅ Instead                                                                     |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Using `StdFsBackend` with user-provided paths                               | Use `VRootFsBackend` - it prevents `../../etc/passwd` attacks                 |
| Using `MemoryBackend` for data that must survive restart                    | Use `SqliteBackend` for persistence, or call `save_to()` to serialize         |
| Expecting identical symlink behavior across platforms with `VRootFsBackend` | Use `MemoryBackend` or `SqliteBackend` for consistent cross-platform symlinks |
| Using `Overlay` when a simple backend would suffice                         | Keep it simple - use `Overlay` only when you need true layered semantics      |
