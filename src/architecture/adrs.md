# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Path-based `VfsBackend` trait | Accepted |
| ADR-002 | Three-crate structure | Accepted |
| ADR-003 | `impl AsRef<Path>` for all path parameters | Accepted |
| ADR-004 | Tower-style middleware pattern | Accepted |
| ADR-005 | `std::fs`-aligned method names | Accepted |
| ADR-006 | LimitedBackend for quota enforcement | Accepted |
| ADR-007 | FeatureGatedBackend for least-privilege | Accepted |
| ADR-008 | FilesContainer as thin ergonomic wrapper | Accepted |
| ADR-009 | Built-in backends are feature-gated | Accepted |
| ADR-010 | Sync-first, async-ready design | Accepted |
| ADR-011 | Layer trait for standardized composition | Accepted |
| ADR-012 | TracingBackend for instrumentation | Accepted |
| ADR-013 | VfsBackendExt for extension methods | Accepted |
| ADR-014 | Optional Bytes support | Accepted |
| ADR-015 | Contextual VfsError | Accepted |

---

## ADR-001: Path-based `VfsBackend` trait

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Three-crate structure

**Decision:**

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait, `Layer` trait, `VfsBackendExt`, types |
| `anyfs` | Backends + middleware (LimitedBackend, FeatureGatedBackend, TracingBackend) |
| `anyfs-container` | Ergonomic wrapper: `FilesContainer<B>`, `BackendStack` builder |

**Why:**
- Backend authors only need `anyfs-backend` (no heavy dependencies).
- Middleware is composable and lives with backends in `anyfs`.
- `FilesContainer` is purely ergonomic - no policy logic.

---

## ADR-003: `impl AsRef<Path>` for all path parameters

**Decision:** Both `VfsBackend` and `FilesContainer` accept `impl AsRef<Path>` for all path parameters.

**Why:**
- Aligned with `std::fs` API conventions.
- Works across all platforms (not limited to UTF-8).
- Ergonomic: accepts `&str`, `String`, `&Path`, `PathBuf`.

---

## ADR-004: Tower-style middleware pattern

**Decision:** Use composable middleware (decorator pattern) for cross-cutting concerns like limits, logging, and feature gates. Each middleware implements `VfsBackend` by wrapping another `VfsBackend`.

**Why:**
- Complete separation of concerns - each layer has one job.
- Composable - use only what you need.
- Familiar pattern (Axum/Tower use the same approach).
- No code duplication - middleware written once, works with any backend.
- Testable - each layer can be tested in isolation.

**Example:**
```rust
let backend = TracingBackend::new(
    FeatureGatedBackend::new(
        LimitedBackend::new(SqliteBackend::open("data.db")?)
    )
);
```

---

## ADR-005: `std::fs`-aligned method names

**Decision:** Prefer `read_dir`, `create_dir_all`, `remove_file`, etc.

**Why:** Familiarity and reduced cognitive overhead.

---

## ADR-006: LimitedBackend for quota enforcement

**Decision:** Quota/limit enforcement is handled by `LimitedBackend<B>` middleware, not by backends or FilesContainer.

**Configuration:**
- `with_max_total_size(bytes)` - total storage limit
- `with_max_file_size(bytes)` - per-file limit
- `with_max_node_count(count)` - max files/directories
- `with_max_dir_entries(count)` - max entries per directory
- `with_max_path_depth(depth)` - max directory nesting

**Why:**
- Limits are policy, not storage semantics.
- Written once, works with any backend.
- Optional - users who don't need limits skip this middleware.

---

## ADR-007: FeatureGatedBackend for least-privilege

**Decision:** Dangerous features (symlinks, hard links, permission mutation) are disabled by default via `FeatureGatedBackend<B>` middleware.

**Configuration:**
- `.with_symlinks()` - enable symlink creation/following
- `.with_hard_links()` - enable hard link creation
- `.with_permissions()` - enable `set_permissions`
- `.with_max_symlink_resolution(n)` - limit symlink hops (default: 40)

When disabled, operations return `VfsError::FeatureNotEnabled`.

**Why:**
- Reduces attack surface by default.
- Explicit opt-in for dangerous features.
- Separate from backend - works with any backend.

---

## ADR-008: FilesContainer as thin ergonomic wrapper

**Decision:** `FilesContainer<B>` is a thin wrapper that provides std::fs-aligned ergonomics only. It contains NO policy logic.

**What it does:**
- Provides familiar method names
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to the wrapped backend

**What it does NOT do:**
- Quota enforcement (use LimitedBackend)
- Feature gating (use FeatureGatedBackend)
- Instrumentation (use TracingBackend)
- Any other policy

**Why:**
- Single responsibility - ergonomics only.
- Users who don't need ergonomics can use backends directly.
- Policy is composable via middleware, not hardcoded.

---

## ADR-009: Built-in backends are feature-gated

**Decision:** `anyfs` uses Cargo features so users only pull the dependencies they need.

- `memory` (default)
- `sqlite` (optional)
- `vrootfs` (optional)

**Why:** Minimizes binary size and compile time for users who don't need all backends.

---

## ADR-010: Sync-first, async-ready design

**Decision:** VfsBackend is synchronous for v1. The API is designed to allow adding `AsyncVfsBackend` later without breaking changes.

**Rationale:**
- All built-in backends are naturally synchronous:
  - `MemoryBackend` - in-memory, instant
  - `SqliteBackend` - rusqlite is sync
  - `VRootFsBackend` - std::fs is sync
- Sync is simpler - no runtime dependency (tokio/async-std)
- Users can wrap sync backends in `spawn_blocking` if needed

**Async-ready design principles:**
- Trait requires `Send` - compatible with async executors
- Return types are `Result<T, VfsError>` - works with async
- No internal blocking assumptions
- Methods are stateless per-call - no hidden blocking state

**Future async path (Option 2):**
When async is needed (e.g., network-backed storage), add a parallel trait:

```rust
// In anyfs-backend
pub trait AsyncVfsBackend: Send + Sync {
    async fn read(&self, path: impl AsRef<Path> + Send) -> Result<Vec<u8>, VfsError>;
    async fn write(&mut self, path: impl AsRef<Path> + Send, data: &[u8]) -> Result<(), VfsError>;
    // ... mirrors VfsBackend with async

    // Streaming uses AsyncRead/AsyncWrite
    async fn open_read(&self, path: impl AsRef<Path> + Send)
        -> Result<Box<dyn AsyncRead + Send + Unpin>, VfsError>;
}
```

**Migration notes:**
- `AsyncVfsBackend` would be a separate trait, not replacing `VfsBackend`
- Blanket impl possible: `impl<T: VfsBackend> AsyncVfsBackend for T` using `spawn_blocking`
- Middleware would need async variants: `AsyncLimitedBackend<B>`, etc.
- No breaking changes to existing sync API

**Why not async now:**
- Complexity without benefit - all current backends are sync
- Rust 1.75 makes async traits easy, so adding later is low-cost
- Better to wait for real async backend requirements

---

## ADR-011: Layer trait for standardized composition

**Decision:** Provide a `Layer` trait (inspired by Tower) that standardizes middleware composition.

```rust
pub trait Layer<B: VfsBackend> {
    type Backend: VfsBackend;
    fn layer(self, backend: B) -> Self::Backend;
}
```

**Why:**
- Standardized composition pattern familiar to Tower/Axum users.
- IDE autocomplete for available layers.
- Enables `BackendStack` fluent builder in anyfs-container.
- Each middleware provides a corresponding `*Layer` type.

**Example:**
```rust
let backend = SqliteBackend::open("data.db")?
    .layer(LimitedLayer::new().max_total_size(100_000))
    .layer(TracingLayer::new());
```

---

## ADR-012: TracingBackend for instrumentation

**Decision:** Use `TracingBackend<B>` integrated with the `tracing` ecosystem instead of a custom logging solution.

**Why:**
- Works with existing tracing infrastructure (tracing-subscriber, OpenTelemetry, Jaeger).
- Structured logging with spans for each operation.
- Users choose their subscriber - no logging framework lock-in.
- Consistent with modern Rust ecosystem practices.

**Configuration:**
```rust
TracingBackend::new(backend)
    .with_target("anyfs")
    .with_level(tracing::Level::DEBUG)
```

---

## ADR-013: VfsBackendExt for extension methods

**Decision:** Provide `VfsBackendExt` trait with convenience methods, auto-implemented for all backends.

```rust
pub trait VfsBackendExt: VfsBackend {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError>;
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError>;
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
}

impl<B: VfsBackend> VfsBackendExt for B {}
```

**Why:**
- Adds convenience without bloating `VfsBackend` trait.
- Blanket impl means all backends get these methods for free.
- Users can define their own extension traits for domain-specific operations.
- Follows Rust convention (e.g., `IteratorExt`, `StreamExt`).

---

## ADR-014: Optional Bytes support

**Decision:** Support the `bytes` crate via an optional feature for zero-copy efficiency.

```toml
anyfs = { version = "0.1", features = ["bytes"] }
```

**Why:**
- `Bytes` provides O(1) slicing via reference counting.
- Beneficial for large file handling, network backends, streaming.
- Optional - users who don't need it avoid the dependency.
- Default remains `Vec<u8>` for simplicity.

**Trade-off:** With `bytes` feature, `read` returns `Bytes` instead of `Vec<u8>`. This is a compile-time choice.

---

## ADR-015: Contextual VfsError

**Decision:** `VfsError` variants include context for better debugging.

```rust
VfsError::NotFound {
    path: PathBuf,
    operation: &'static str,  // "read", "metadata", etc.
}

VfsError::QuotaExceeded {
    limit: u64,
    requested: u64,
    usage: u64,
}
```

**Why:**
- Error messages include enough context to understand what failed.
- No need for separate error context crate (like anyhow) for basic usage.
- Operation field helps distinguish "file not found during read" vs "during metadata".
- Quota errors include all relevant numbers for debugging.
