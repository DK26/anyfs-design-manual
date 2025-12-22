# VFS Container — Build vs. Reuse Analysis

**Can your goals be achieved with existing crates?**

---

## Your Core Requirements

1. **Multi-tenant isolation** — Each tenant gets their own namespace
2. **Portable single-file storage** — Move, copy, backup as one file
3. **SQLite backing** — The "single file" is a SQLite database
4. **Standard filesystem operations** — read, write, mkdir, etc.

---

## What Exists in the Rust Ecosystem

### 1. `vfs` Crate (VFS Abstraction)

**What it does:** Provides a `FileSystem` trait with multiple backends (Physical, Memory, Overlay, Embedded).

**Does it have SQLite backend?** ❌ **NO**

**Could you add one?** Yes — implement `FileSystem` trait with SQLite storage.

```rust
// vfs crate's trait (simplified)
pub trait FileSystem: Send + Sync {
    fn read_dir(&self, path: &str) -> VfsResult<Box<dyn Iterator<Item = String>>>;
    fn create_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndWrite>>;
    fn open_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndRead>>;
    fn metadata(&self, path: &str) -> VfsResult<VfsMetadata>;
    // ... ~15 methods total
}
```

### 2. `rusqlite` (SQLite Bindings)

**What it does:** Ergonomic SQLite bindings for Rust.

**Relevant features:**
- Transactions ✅
- Blob streaming ✅ (`feature = "blob"`)
- In-memory databases ✅

**Does it provide filesystem abstraction?** ❌ **NO** — just raw SQLite access.

### 3. `sqlite-vfs` / `sqlite-plugin` (SQLite VFS)

**What it does:** Lets you implement SQLite's internal VFS — i.e., where SQLite stores its `.db` file.

**Is this what you want?** ❌ **NO** — This is the opposite direction. You want to store filesystem data *inside* SQLite, not control where SQLite stores itself.

### 4. `strict-path` (Safe Path Handling)

**What it does:** Type-safe path boundaries with `VirtualRoot`/`VirtualPath`.

**Relevant features:**
- Path traversal protection ✅
- Built-in I/O operations ✅
- Sandboxing ✅

**Does it provide SQLite storage?** ❌ **NO** — backs onto real filesystem.

### 5. SQLAR (SQLite Archive Format)

**What it is:** SQLite's official archive format — simple table with `(name, mode, mtime, sz, data)`.

**Rust crate?** ❌ **NO production-ready crate** — would need to build yourself.

**Limitations:** Flat namespace (paths as keys), no directory semantics, no transactions across operations.

---

## Gap Analysis

| Requirement | vfs | rusqlite | strict-path | SQLAR |
|-------------|-----|----------|-------------|-------|
| Filesystem API | ✅ | ❌ | ✅ | ❌ |
| SQLite storage | ❌ | ✅ (raw) | ❌ | ✅ (schema) |
| Single-file portable | ❌ | ✅ | ❌ | ✅ |
| Multi-tenant | ❌ | Manual | ❌ | ❌ |
| Path validation | ❌ | N/A | ✅ | ❌ |
| Capacity limits | ❌ | ❌ | ❌ | ❌ |

**The gap:** There is NO existing crate that provides:
> "A filesystem API backed by SQLite with tenant isolation"

---

## Your Options

### Option A: Implement `vfs::FileSystem` for SQLite

**Approach:** Create a `SqliteFS` that implements the `vfs` crate's `FileSystem` trait.

```rust
use vfs::{FileSystem, VfsResult, VfsMetadata};

pub struct SqliteFS {
    conn: rusqlite::Connection,
}

impl FileSystem for SqliteFS {
    fn create_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndWrite + Send>> {
        // Validate path (we must do this ourselves)
        // Insert into nodes/edges tables
        // Return a wrapper that writes to SQLite blob
    }
    // ... implement ~15 methods
}
```

**Pros:**
- Ecosystem compatibility — works with existing `vfs` code
- Less API design work
- Users familiar with `vfs` can adopt easily

**Cons:**
- Paths are `&str` — we must validate in every method
- No built-in capacity limits — we implement ourselves
- Streaming handles (`Box<dyn Read>`) add complexity for SQLite blobs
- No transaction support across operations in `vfs` trait

**Effort:** ~2-3 weeks

### Option B: Build Our Own Thin Wrapper

**Approach:** Create a minimal abstraction specifically for SQLite-backed storage.

```rust
pub struct FilesContainer {
    conn: rusqlite::Connection,
}

impl FilesContainer {
    pub fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    pub fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    pub fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    // ... ~14 methods
}
```

**Pros:**
- Full control over API
- Validated paths (`VirtualPath`) by design
- Built-in capacity limits
- Transactions naturally fit
- Simpler (no streaming handles to wrap)

**Cons:**
- No `vfs` ecosystem compatibility
- Users learn new API
- More design work

**Effort:** ~2-3 weeks (similar to Option A)

### Option C: Minimal rusqlite Wrapper

**Approach:** Just wrap rusqlite with basic filesystem helpers, no abstraction layer.

```rust
pub fn write_file(conn: &Connection, path: &str, data: &[u8]) -> Result<()> {
    conn.execute("INSERT OR REPLACE INTO files (path, data) VALUES (?, ?)", params![path, data])
}

pub fn read_file(conn: &Connection, path: &str) -> Result<Vec<u8>> {
    conn.query_row("SELECT data FROM files WHERE path = ?", params![path], |row| row.get(0))
}
```

**Pros:**
- Fastest to implement
- Direct SQLite access when needed
- No abstraction overhead

**Cons:**
- No type safety
- No path validation
- No capacity limits
- Ad-hoc API, hard to maintain
- Every user re-invents the wheel

**Effort:** ~1 week

### Option D: Hybrid — Our Abstraction + vfs Compatibility Layer

**Approach:** Build our own `FilesContainer`, then optionally wrap it to implement `vfs::FileSystem`.

```rust
// Our core
pub struct FilesContainer { ... }

// vfs compatibility (optional feature)
impl vfs::FileSystem for FilesContainerVfsAdapter { ... }
```

**Pros:**
- Best of both worlds
- Users can choose API style
- Migration path for `vfs` users

**Cons:**
- More code to maintain
- Two APIs to document

**Effort:** ~3-4 weeks

---

## My Recommendation

### For Your Specific Goals:

| Goal | Best Option |
|------|-------------|
| Multi-tenant isolation | **Option B** (own abstraction) |
| Move SQLite file = move tenant | **Any** (this is just SQLite) |
| Backup/replicate easily | **Any** (this is just SQLite) |
| Capacity limits per tenant | **Option B** (built-in) |
| Type-safe paths | **Option B** (VirtualPath) |
| Ecosystem compatibility | **Option A or D** |

### Concrete Recommendation:

**Go with Option B (Build Our Own) because:**

1. **Your use case is specific** — tenant isolation + quotas + portability. The `vfs` crate isn't designed for this.

2. **SQLite backend doesn't exist anyway** — You're building something new regardless of abstraction choice.

3. **Validated paths matter** — `VirtualPath` catches bugs at compile time, not runtime.

4. **Capacity limits are core to multi-tenant** — Building them in from the start is cleaner than bolting on.

5. **`vfs` compatibility can come later** — If there's demand, add adapter as Option D.

### What We DO Reuse:

| Crate | What We Use It For |
|-------|-------------------|
| `rusqlite` | All SQLite operations |
| `strict-path` | Filesystem backend (MVP validation) |
| `thiserror` | Error types |

We're not building everything from scratch — we're building a **thin layer** on top of battle-tested crates.

---

## Implementation Path

```
Week 1: Core types (VirtualPath, errors) + FS backend via strict-path
Week 2: FilesContainer API + tests
Week 3: SQLite backend (schema + implementation)
Week 4: Capacity limits + polish
```

### Minimal Viable Product:

```rust
use vfs::{FilesContainer, SqliteBackend, VirtualPath};

// Create tenant storage
let tenant_db = format!("tenants/{}.db", tenant_id);
let container = FilesContainer::builder()
    .backend(SqliteBackend::open_or_create(&tenant_db)?)
    .max_total_size(100 * 1024 * 1024)  // 100 MB quota
    .build()?;

// Use standard filesystem operations
container.mkdir(&VirtualPath::new("/documents")?)?;
container.write(&VirtualPath::new("/documents/report.pdf")?, &pdf_bytes)?;

// Move tenant = move SQLite file
std::fs::rename(&tenant_db, &new_location)?;

// Backup tenant = copy SQLite file  
std::fs::copy(&tenant_db, &backup_path)?;
```

---

## Summary

| Question | Answer |
|----------|--------|
| Can existing crates achieve your goals? | **Partially** — rusqlite + strict-path help, but no complete solution |
| Does SQLite filesystem backend exist? | **No** — this is the gap |
| Should you use `vfs` crate? | **Optional** — you can, but it doesn't add much for your use case |
| Should you build from scratch? | **Thin layer only** — reuse rusqlite, strict-path |
| What's the effort? | **~3-4 weeks** to working MVP |

**Bottom line:** You need to build something, but it's a thin layer on top of existing crates, not a ground-up implementation. The core insight is that your specific combination (SQLite + filesystem API + tenant isolation + quotas) doesn't exist, and composing existing crates doesn't get you there cleanly.
