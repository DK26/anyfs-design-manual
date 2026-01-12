# PathResolver: The Simple Explanation

## What Problem Does It Solve?

Imagine you're giving someone directions to a room in a building:

> "Go to the **office**, then into the **storage closet**, then **back out**, then into the **conference room**."

That's a lot of steps! A smart person would simplify it:

> "Just go to the **conference room**."

**PathResolver does exactly this for file paths.**

---

## The Problem: Messy Paths

When programs work with files, they often create messy paths like:

```
/home/user/../user/./documents/../documents/report.txt
```

This path says:
- Go to `/home/user`
- Go back up (`..`)
- Go to `user` again
- Stay here (`.`)
- Go to `documents`
- Go back up (`..`)
- Go to `documents` again
- Finally, `report.txt`

**That's exhausting!** The simple answer is just:

```
/home/user/documents/report.txt
```

**PathResolver's job is to figure out the simple answer.**

---

## Why Can't the Backend Just Do This?

Good question! Here's why we separated it:

### 1. Different Backends, Same Logic

Think of backends like different types of filing cabinets:
- **MemoryBackend** = Files in your brain (RAM)
- **SqliteBackend** = Files in a database
- **VRootFsBackend** = Files on your hard drive

The path simplification logic is **the same** for all of them:
- `..` means "go up one level"
- `.` means "stay here"
- Symlinks mean "actually go over there instead"

Why write this logic three times? **Write it once, use it everywhere.**

### 2. We Can Test It Alone

If path resolution is buried inside each backend, testing is hard:

```
❌ To test path resolution, you need:
   - A real backend
   - Real files
   - Complex setup
```

With PathResolver separated:

```
✅ To test path resolution, you need:
   - Just the resolver
   - Simple inputs and outputs
   - No files required!
```

### 3. We Can Benchmark It

"Is our path resolution fast enough?"

If it's mixed with everything else, you can't measure it. Separated, you can:

```rust
// Easy to benchmark!
let resolver = IterativeResolver::new();
benchmark(|| resolver.canonicalize("/a/b/../c", &mock_fs));
```

### 4. We Can Swap It

Different situations need different approaches:

| Resolver            | Best For                                |
| ------------------- | --------------------------------------- |
| `IterativeResolver` | General use (walks path step by step)   |
| `CachingResolver`   | Repeated paths (remembers answers)      |
| `NoOpResolver`      | Real filesystem (OS already handles it) |

With separation, switching is one line:

```rust
// Default
let fs = FileStorage::new(backend);

// With caching (for performance)
let fs = FileStorage::with_resolver(backend, CachingResolver::new());
```

---

## The Analogy: GPS Navigation

Think of PathResolver like a GPS system separate from your car:

| Component      | In AnyFS                               | In a Car             |
| -------------- | -------------------------------------- | -------------------- |
| **Storage**    | Backend (MemoryBackend, SqliteBackend) | The roads themselves |
| **Navigation** | PathResolver                           | GPS device           |
| **Interface**  | FileStorage                            | Dashboard            |

Why is GPS a separate device?
- ✅ You can upgrade the GPS without changing the car
- ✅ You can test the GPS in a simulator
- ✅ Different GPS apps can work in the same car
- ✅ The car maker doesn't need to be a GPS expert

**Same reasons we separated PathResolver!**

---

## What PathResolver Actually Does

```
Input:  /home/user/../admin/./config.txt
                  ↓
         [PathResolver]
                  ↓
Output: /home/admin/config.txt
```

Step by step:
1. `/home/user` → go to user's home
2. `..` → go back up to `/home`
3. `admin` → go into admin folder
4. `.` → stay here (ignore)
5. `config.txt` → the file!

Result: `/home/admin/config.txt`

### Symlinks Make It Interesting

Symlinks are like shortcuts. If `/home/admin` is actually a symlink pointing to `/users/administrator`, the resolver follows it:

```
Input:  /home/admin/config.txt
        (but /home/admin → /users/administrator)
                  ↓
         [PathResolver]
                  ↓
Output: /users/administrator/config.txt
```

---

## The Two Main Methods

### `canonicalize()` - Strict Mode

"Give me the real, final path. Everything must exist."

```rust
resolver.canonicalize("/a/b/../c/file.txt", &fs)
// Returns: /a/c/file.txt (if it exists)
// Error: if any part doesn't exist
```

### `soft_canonicalize()` - Relaxed Mode

"Resolve what you can, but the last part doesn't need to exist yet."

```rust
resolver.soft_canonicalize("/a/b/../c/new_file.txt", &fs)
// Returns: /a/c/new_file.txt (even if new_file.txt doesn't exist)
// Error: only if /a/c doesn't exist
```

This is useful for creating new files—you need to know WHERE to create them, but they don't exist yet!

---

## Summary: Why Separate?

| Benefit            | Explanation                                  |
| ------------------ | -------------------------------------------- |
| **Testable**       | Test path logic without touching real files  |
| **Benchmarkable**  | Measure performance in isolation             |
| **Swappable**      | Different resolvers for different needs      |
| **Maintainable**   | One place to fix bugs, benefits all backends |
| **Understandable** | Each piece has one job                       |

---

## In Code

```rust
// The trait (the "job description")
pub trait PathResolver: Send + Sync {
    fn canonicalize(&self, path: &Path, fs: &dyn Fs) -> Result<PathBuf, FsError>;
    fn soft_canonicalize(&self, path: &Path, fs: &dyn Fs) -> Result<PathBuf, FsError>;
}

// FileStorage uses it
pub struct FileStorage<B, R = IterativeResolver, M = ()> {
    backend: B,      // Where files live
    resolver: R,     // How to simplify paths
    _marker: PhantomData<M>,
}
```

That's it! **PathResolver answers one question: "What's the real, simple path?"**

Everything else—reading files, writing files, listing directories—is the backend's job.
