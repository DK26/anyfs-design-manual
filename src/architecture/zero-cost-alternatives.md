# Zero-Cost Alternatives for I/O Operations

This document analyzes alternatives to dynamic dispatch (`Box<dyn Trait>`) for streaming I/O and directory iteration.

> **Decision:** See [ADR-025: Strategic Boxing](./adrs.md#adr-025-strategic-boxing-tower-style) for the formal decision.
>
> **TL;DR:** We follow Tower/Axum's approach - zero-cost on hot path (`read()`, `write()`), box at cold path boundaries (`open_read()`, `read_dir()`). The boxing cost (<1% of I/O time) is negligible; the ergonomic benefits are substantial.

---

## Current Design (Dynamic Dispatch)

```rust
pub trait FsRead: Send + Sync {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}

pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError>;
}

// Where ReadDirIter is:
pub struct ReadDirIter(Box<dyn Iterator<Item = Result<DirEntry, FsError>> + Send>);
```

**Cost:** One heap allocation per `open_read()`, `open_write()`, or `read_dir()` call.

---

## Option 1: Associated Types (Classic Approach)

```rust
pub trait FsRead: Send + Sync {
    type Reader: Read + Send;

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Self::Reader, FsError>;
}

pub trait FsDir: Send + Sync {
    type DirIter: Iterator<Item = Result<DirEntry, FsError>> + Send;

    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Self::DirIter, FsError>;
}
```

### Implementation

```rust
impl FsRead for MemoryBackend {
    type Reader = std::io::Cursor<Vec<u8>>;

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Self::Reader, FsError> {
        let data = self.read(path)?;
        Ok(std::io::Cursor::new(data))
    }
}

impl FsDir for MemoryBackend {
    type DirIter = std::vec::IntoIter<Result<DirEntry, FsError>>;

    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Self::DirIter, FsError> {
        let entries = self.collect_entries(path)?;
        Ok(entries.into_iter())
    }
}
```

### Middleware Propagation Problem

```rust
impl<B: FsRead> FsRead for Quota<B> {
    // Must define our own Reader type that wraps B::Reader
    type Reader = QuotaReader<B::Reader>;

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Self::Reader, FsError> {
        let inner = self.inner.open_read(path)?;
        Ok(QuotaReader::new(inner, self.usage.clone()))
    }
}

// Every middleware needs a custom wrapper type
struct QuotaReader<R> {
    inner: R,
    usage: Arc<RwLock<QuotaUsage>>,
}

impl<R: Read> Read for QuotaReader<R> {
    fn read(&mut self, buf: &mut [u8]) -> std::io::Result<usize> {
        // Track bytes read if needed
        self.inner.read(buf)
    }
}
```

### The Type Explosion

With a middleware stack like `Quota<PathFilter<Tracing<MemoryBackend>>>`:

```rust
type FinalReader = QuotaReader<PathFilterReader<TracingReader<Cursor<Vec<u8>>>>>;
type FinalDirIter = QuotaIter<PathFilterIter<TracingIter<IntoIter<Result<DirEntry, FsError>>>>>;
```

### Verdict

| Aspect | Assessment |
|--------|------------|
| Heap allocations | ✅ None |
| Type complexity | ❌ Exponential growth |
| Middleware authoring | ❌ Every middleware needs wrapper types |
| User ergonomics | ⚠️ Type annotations become unwieldy |
| Compile times | ❌ Longer due to monomorphization |

**Not recommended** as the primary API due to complexity explosion.

---

## Option 2: RPITIT (Rust 1.75+)

Return Position Impl Trait in Traits allows:

```rust
pub trait FsRead: Send + Sync {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<impl Read + Send, FsError>;
}

pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: impl AsRef<Path>)
        -> Result<impl Iterator<Item = Result<DirEntry, FsError>> + Send, FsError>;
}
```

### How It Works

The compiler infers a unique anonymous type for each implementor:

```rust
impl FsRead for MemoryBackend {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<impl Read + Send, FsError> {
        let data = self.read(path)?;
        Ok(std::io::Cursor::new(data))  // Returns Cursor<Vec<u8>>, but caller sees impl Read
    }
}

impl FsRead for SqliteBackend {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<impl Read + Send, FsError> {
        Ok(SqliteReader::new(self.conn.clone(), path))  // Different type, same interface
    }
}
```

### Middleware Still Works

```rust
impl<B: FsRead> FsRead for Tracing<B> {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<impl Read + Send, FsError> {
        let span = tracing::span!(Level::DEBUG, "open_read");
        let _guard = span.enter();
        self.inner.open_read(path)  // Just forward - return type is inferred
    }
}
```

### The Catch: Object Safety

**RPITIT makes traits non-object-safe.** You cannot do:

```rust
// This won't compile with RPITIT
let backends: Vec<Box<dyn FsRead>> = vec![...];
```

### Verdict

| Aspect | Assessment |
|--------|------------|
| Heap allocations | ✅ None |
| Type complexity | ✅ Hidden behind `impl Trait` |
| Middleware authoring | ✅ Simple forwarding |
| User ergonomics | ✅ Clean API |
| Object safety | ❌ Lost - can't use `dyn FsRead` |
| Rust version | ⚠️ Requires 1.75+ |

**Good for performance-critical paths** but sacrifices `dyn` usage.

---

## Option 3: Generic Associated Types (GATs)

For readers that borrow from the backend:

```rust
pub trait FsRead: Send + Sync {
    type Reader<'a>: Read + Send where Self: 'a;

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Self::Reader<'_>, FsError>;
}
```

### Use Case: Zero-Copy Reads

```rust
impl FsRead for MemoryBackend {
    type Reader<'a> = &'a [u8];  // Borrow directly from internal storage!

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Self::Reader<'_>, FsError> {
        let data = self.storage.read().unwrap();
        let bytes = data.get(path.as_ref())
            .ok_or(FsError::NotFound { path: path.as_ref().to_path_buf() })?;
        Ok(bytes.as_slice())
    }
}
```

### Complexity

GATs are powerful but add significant complexity:
- Lifetime parameters propagate through middleware
- Not all backends can provide borrowed data (SQLite must copy)
- Makes trait definitions harder to understand

### Verdict

| Aspect | Assessment |
|--------|------------|
| Heap allocations | ✅ Can be zero-copy |
| Type complexity | ❌ High (lifetimes everywhere) |
| Middleware authoring | ❌ Complex lifetime handling |
| Use case fit | ⚠️ Only benefits backends with owned data |

**Overkill for most use cases.** Consider only for specialized zero-copy scenarios.

---

## Option 4: Hybrid Approach (Recommended)

Provide **both** dynamic and static APIs:

```rust
pub trait FsRead: Send + Sync {
    /// Dynamic dispatch version (simple, flexible)
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}

/// Extension trait for zero-cost static dispatch
pub trait FsReadTyped: FsRead {
    type Reader: Read + Send;

    /// Static dispatch version (zero-cost, less flexible)
    fn open_read_typed(&self, path: impl AsRef<Path>) -> Result<Self::Reader, FsError>;
}

// Blanket impl for convenience when types align
impl<T: FsReadTyped> FsRead for T {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError> {
        Ok(Box::new(self.open_read_typed(path)?))
    }
}
```

### Usage

```rust
// Default: dynamic dispatch (works everywhere)
let reader = fs.open_read("/file.txt")?;

// Performance-critical: static dispatch
let reader: MemoryReader = fs.open_read_typed("/file.txt")?;
```

### Verdict

| Aspect | Assessment |
|--------|------------|
| Heap allocations | ✅ Optional (use `_typed` to avoid) |
| Type complexity | ✅ Hidden unless you opt-in |
| Middleware authoring | ✅ Only implement base trait |
| User ergonomics | ✅ Simple default, power when needed |
| Object safety | ✅ Base trait remains object-safe |

**Best of both worlds** - simple default, zero-cost opt-in.

---

## Option 5: Callback-Based Iteration

Avoid returning iterators entirely:

```rust
pub trait FsDir: Send + Sync {
    fn for_each_entry<F>(&self, path: impl AsRef<Path>, f: F) -> Result<(), FsError>
    where
        F: FnMut(DirEntry) -> ControlFlow<(), ()>;
}
```

### Usage

```rust
fs.for_each_entry("/dir", |entry| {
    println!("{}", entry.name);
    ControlFlow::Continue(())
})?;
```

### Verdict

| Aspect | Assessment |
|--------|------------|
| Heap allocations | ✅ None |
| Ergonomics | ❌ Callbacks are awkward |
| Early exit | ✅ Via `ControlFlow::Break` |
| Composability | ❌ Can't chain iterator methods |

**Not recommended** as primary API. Could be added as optimization option.

---

## Option 6: Stack-Allocated Small Buffer

For directory iteration, most directories are small:

```rust
use smallvec::SmallVec;

pub struct ReadDirIter {
    // Stack-allocate up to 32 entries, heap only if larger
    entries: SmallVec<[Result<DirEntry, FsError>; 32]>,
    index: usize,
}
```

### Verdict

| Aspect | Assessment |
|--------|------------|
| Heap allocations | ⚠️ Avoided for small directories |
| Memory overhead | ⚠️ Larger stack frames |
| Dependencies | ⚠️ Adds `smallvec` crate |

**Reasonable optimization** for directory iteration specifically.

---

## Recommendation

### Primary API: Keep Dynamic Dispatch

```rust
pub trait FsRead: Send + Sync {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}

pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError>;
}
```

**Why:**
1. **Simplicity** - One type to learn, one API
2. **Object safety** - Can use `Box<dyn Fs>` for runtime polymorphism
3. **Middleware simplicity** - No wrapper types needed
4. **Actual cost is low** - One allocation per stream open, not per read

### Optional: Static Dispatch Extension

For performance-critical code, offer typed variants:

```rust
pub trait FsReadTyped: FsRead {
    type Reader: Read + Send;
    fn open_read_typed(&self, path: impl AsRef<Path>) -> Result<Self::Reader, FsError>;
}
```

### Future: RPITIT When Object Safety Not Needed

If a user doesn't need `dyn Fs`, they can define their own trait:

```rust
pub trait FsReadStatic: Send + Sync {
    fn open_read(&self, path: impl AsRef<Path>) -> Result<impl Read + Send, FsError>;
}
```

---

## Cost Analysis: Is It Actually a Problem?

### Heap Allocation Cost

| Operation | Allocations | Typical Size | Cost |
|-----------|-------------|--------------|------|
| `open_read()` | 1 | ~24-48 bytes (vtable + pointer) | ~20-50ns |
| `read()` (data) | 0-1 | File size | Dominates |
| `read_dir()` | 1 | ~24-48 bytes | ~20-50ns |
| Iteration | 0 | - | - |

**The allocation is dwarfed by actual I/O time.** For a 4KB file read from SQLite or disk, the Box allocation is <0.1% of total time.

### When It Matters

| Scenario | Matters? |
|----------|----------|
| Reading large files | No - I/O dominates |
| Reading many small files | Maybe - consider batching |
| Hot loop micro-benchmarks | Yes |
| Real-world applications | Rarely |

### Conclusion

**Dynamic dispatch is the right default.** The cost is negligible for real workloads, and the ergonomic benefits are substantial. Offer static dispatch as an opt-in escape hatch for the rare cases where it matters.

---

## Summary Decision Matrix

| Approach | Alloc-Free | Simple | Object-Safe | Recommended |
|----------|------------|--------|-------------|-------------|
| Current (`Box<dyn>`) | ❌ | ✅ | ✅ | ✅ Default |
| Associated Types | ✅ | ❌ | ✅ | ❌ Too complex |
| RPITIT | ✅ | ✅ | ❌ | ⚠️ When no dyn needed |
| GATs | ✅ | ❌ | ❌ | ❌ Overkill |
| Hybrid | ✅ opt-in | ✅ | ✅ | ✅ Best of both |
| Callbacks | ✅ | ❌ | ✅ | ❌ Awkward API |
| SmallVec | ⚠️ | ✅ | ✅ | ⚠️ For ReadDirIter |
