# VFS Container — Architecture Decision Records

**Key design decisions with context and rationale**

---

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](#adr-001-storage-agnostic-backend-trait) | Storage-Agnostic Backend Trait | Accepted |
| [ADR-002](#adr-002-node-edge-data-model) | Node/Edge Data Model | Accepted |
| [ADR-003](#adr-003-mandatory-transactions) | Mandatory Transactions | Accepted |
| [ADR-004](#adr-004-lexical-path-resolution) | Lexical Path Resolution | Accepted |
| [ADR-005](#adr-005-opt-in-filesystem-features) | Opt-in Filesystem Features | Accepted |
| [ADR-006](#adr-006-fixed-chunk-size) | Fixed Chunk Size | Accepted |
| [ADR-007](#adr-007-capacity-enforcement-in-core) | Capacity Enforcement in Core | Accepted |
| [ADR-008](#adr-008-synchronous-first-api) | Synchronous-First API | Accepted |
| [ADR-009](#adr-009-newtype-ids) | Newtype IDs | Accepted |
| [ADR-010](#adr-010-no-posix-compliance) | No POSIX Compliance | Accepted |

---

## ADR-001: Storage-Agnostic Backend Trait

### Status
Accepted

### Context
We need to support multiple storage backends (SQLite, memory, filesystem, potentially cloud storage). Existing VFS abstractions like `vfs` crate and FUSE require backends to understand filesystem semantics (paths, symlinks, permissions).

### Decision
Create a minimal `StorageBackend` trait that operates on **nodes and edges**, not filesystem concepts. The backend is essentially a typed graph store.

```rust
pub trait StorageBackend {
    fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
    where F: FnOnce(&mut dyn Transaction) -> Result<T, BackendError>;
    
    fn snapshot(&self) -> Box<dyn Snapshot + '_>;
}
```

The backend knows nothing about:
- Paths (just node IDs)
- Symlinks (just nodes with a target string)
- Permissions (just metadata bytes)
- Resolution logic (handled by FilesContainer)

### Consequences
**Positive:**
- Backend implementation is simple (~13 methods)
- Any key-value or graph store can be a backend
- All filesystem complexity is centralized in FilesContainer
- Testing is easier (backends are just data stores)

**Negative:**
- Some operations that could be optimized at the storage level (e.g., recursive delete) require multiple round trips
- Backends can't enforce filesystem invariants themselves

### Alternatives Considered
1. **Path-based trait** (like `vfs` crate): Rejected because it pushes too much complexity to backends
2. **Full POSIX trait** (like FUSE): Rejected because backends shouldn't need to understand POSIX semantics

---

## ADR-002: Node/Edge Data Model

### Status
Accepted

### Context
Filesystems have a hierarchical structure. We need to model files, directories, symlinks, and their relationships.

### Decision
Model the filesystem as a **directed graph**:

- **Nodes**: Entities with metadata (files, directories, symlinks)
- **Edges**: Named parent→child relationships
- **Content**: Binary data stored separately (for files)

```rust
pub struct NodeRecord {
    pub id: NodeId,
    pub kind: NodeKind,      // File | Directory | Symlink
    pub metadata: NodeMetadata,
}

pub struct Edge {
    pub parent: NodeId,
    pub name: Name,
    pub child: NodeId,
}
```

### Consequences
**Positive:**
- Clean separation between structure and content
- Hard links are natural (multiple edges to same node)
- Efficient directory listings (just query edges by parent)
- Content deduplication is possible (multiple nodes share ContentId)

**Negative:**
- Path resolution requires traversing edges (not a single lookup)
- Moving directories requires updating edges, not just renaming

### Alternatives Considered
1. **Path-as-key model**: Store each file under its full path. Rejected because rename/move is O(n) for directories.
2. **Nested document model**: Embed children in parent. Rejected because updates are complex and transactions are harder.

---

## ADR-003: Mandatory Transactions

### Status
Accepted

### Context
Filesystem operations often involve multiple steps (e.g., rename = create new edge + delete old edge). Without atomicity, crashes can corrupt the filesystem.

### Decision
All write operations must happen inside a transaction:

```rust
fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
where F: FnOnce(&mut dyn Transaction) -> Result<T, BackendError>;
```

There is no way to call `insert_node` or `delete_edge` outside of a transaction. The type system enforces this.

### Consequences
**Positive:**
- Crash-safe by default
- Backends can't be misused
- Complex operations (copy directory tree) are atomic

**Negative:**
- Slight API complexity (must wrap operations in closure)
- Backends that don't support transactions must implement rollback

### Alternatives Considered
1. **Optional transactions**: Let callers choose. Rejected because it's easy to forget and corrupt data.
2. **Auto-commit per operation**: Each call is its own transaction. Rejected because multi-step operations become non-atomic.

---

## ADR-004: Lexical Path Resolution

### Status
Accepted

### Context
Path handling is a major source of security vulnerabilities (path traversal, symlink attacks). We need a safe approach.

### Decision
Paths are **lexical only** — they are normalized without consulting the filesystem:

```rust
// Normalization is pure string manipulation
VirtualPath::new("/../../../etc/passwd") → "/etc/passwd"
```

Key properties:
- No host filesystem interaction
- No `realpath` or `std::fs::canonicalize`
- `..` is resolved lexically, cannot escape root
- Symlinks are followed after normalization (with depth limit)

### Consequences
**Positive:**
- Path traversal attacks are structurally impossible
- Deterministic behavior regardless of filesystem state
- No race conditions in path resolution

**Negative:**
- Some POSIX behaviors aren't replicated (e.g., `/foo/bar/..` where `bar` is a symlink)
- Users expecting POSIX semantics may be surprised

### Alternatives Considered
1. **POSIX-style resolution**: Follow symlinks during `..` resolution. Rejected because it's complex and has security implications.
2. **No `..` support**: Reject paths with `..`. Rejected because it's too restrictive for usability.

---

## ADR-005: Opt-in Filesystem Features

### Status
Accepted

### Context
Full filesystem semantics (symlinks, hard links, permissions, xattrs) add complexity. Not all use cases need them.

### Decision
Advanced features are disabled by default and enabled via builder:

```rust
FilesContainer::builder()
    .backend(backend)
    .symlinks(true)        // default: false
    .hard_links(true)      // default: false
    .permissions(true)     // default: false
    .extended_attrs(true)  // default: false
    .build()
```

Operations on disabled features return `VfsError::FeatureNotEnabled`.

### Consequences
**Positive:**
- Simple use cases stay simple
- Reduced attack surface when features are off
- Clear documentation of what's supported
- Potential for smaller code size (dead code elimination)

**Negative:**
- Users must know to enable features
- Testing matrix is larger (features × backends)

### Alternatives Considered
1. **Everything on by default**: Rejected because it adds complexity for simple use cases.
2. **Compile-time features**: Rejected because runtime configuration is more flexible.

---

## ADR-006: Fixed Chunk Size

### Status
Accepted

### Context
Large files need to be stored efficiently. Options include storing whole files, variable-size chunks, or fixed-size chunks.

### Decision
Content is stored in **fixed 64KB chunks**:

```rust
pub const CHUNK_SIZE: usize = 64 * 1024;

pub struct ChunkId {
    pub content: ContentId,
    pub index: u32,
}
```

### Consequences
**Positive:**
- Simple implementation (no chunk boundary decisions)
- Efficient partial reads (seek to chunk, read subset)
- Predictable storage overhead
- Easy to implement across all backend types

**Negative:**
- Small files waste space (1 byte file = 1 chunk + overhead)
- 64KB might not be optimal for all workloads
- Cannot change chunk size without migration

### Alternatives Considered
1. **Whole file storage**: Store files as single blobs. Rejected because large files become unwieldy.
2. **Variable chunks (content-defined)**: Better dedup but complex. Deferred to future version.
3. **Configurable chunk size**: Adds complexity, rejected for v1.

---

## ADR-007: Capacity Enforcement in Core

### Status
Accepted

### Context
Multi-tenant systems need resource limits. Where should limits be enforced?

### Decision
Capacity limits are enforced in `FilesContainer`, not in backends:

```rust
pub struct CapacityLimits {
    pub max_total_size: Option<u64>,
    pub max_file_size: Option<u64>,
    pub max_node_count: Option<u64>,
    pub max_dir_entries: Option<u32>,
    pub max_path_depth: Option<u16>,
    pub max_name_length: Option<u16>,
}
```

The core tracks usage via metadata stored in the backend.

### Consequences
**Positive:**
- Consistent enforcement across all backends
- Backends stay simple
- Limits are configurable per-container

**Negative:**
- Usage tracking adds overhead (update counters on every write)
- Backend-specific limits (e.g., disk full) need separate handling

### Alternatives Considered
1. **Backend-enforced limits**: Each backend implements limits. Rejected because it's duplicated work and inconsistent behavior.
2. **External enforcement**: Wrapper or middleware. Rejected because it's easy to bypass.

---

## ADR-008: Synchronous-First API

### Status
Accepted

### Context
Rust's async ecosystem is mature but adds complexity. Should the API be sync, async, or both?

### Decision
Start with a **synchronous API**. Async support can be added later as a separate trait.

```rust
// Current: sync
pub trait StorageBackend {
    fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>;
}

// Future: async (separate trait)
pub trait AsyncStorageBackend {
    async fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>;
}
```

### Consequences
**Positive:**
- Simpler implementation and API
- SQLite (primary backend) is sync anyway
- No async runtime dependency
- Easier to reason about transactions

**Negative:**
- Network-backed backends will block threads
- May need significant refactoring to add async later

### Alternatives Considered
1. **Async-first**: Rejected because it adds complexity and most backends are sync.
2. **Both from start**: Rejected because it doubles the API surface.

---

## ADR-009: Newtype IDs

### Status
Accepted

### Context
The system uses multiple ID types (NodeId, ContentId, ChunkId). Raw integers are error-prone.

### Decision
All IDs are **newtypes** with restricted construction:

```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct NodeId(pub(crate) u64);

#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct ContentId(pub(crate) u64);

#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct ChunkId {
    pub content: ContentId,
    pub index: u32,
}
```

The inner value is `pub(crate)` — only the crate can construct them.

### Consequences
**Positive:**
- Compiler catches ID type confusion
- Self-documenting code
- Can add validation in constructor
- Can change internal representation without API change

**Negative:**
- Slight verbosity (`NodeId(1)` vs `1`)
- Backend implementers need to convert to/from their storage format

### Alternatives Considered
1. **Type aliases**: `type NodeId = u64`. Rejected because compiler doesn't distinguish them.
2. **Generic ID type**: `Id<Node>`, `Id<Content>`. Rejected because it's more complex.

---

## ADR-010: No POSIX Compliance

### Status
Accepted

### Context
Most filesystems aim for POSIX compliance. Should we?

### Decision
Explicitly **do not target POSIX compliance**. We implement a simpler, safer subset.

Differences from POSIX:
- No device files, sockets, FIFOs
- No hard links to directories
- Lexical (not physical) path resolution
- No user/group ownership (just optional mode bits)
- No file locking primitives
- Symlinks are optional

### Consequences
**Positive:**
- Simpler implementation
- Smaller attack surface
- Predictable behavior across platforms
- No need to handle obscure POSIX edge cases

**Negative:**
- Cannot be used as a drop-in replacement for OS filesystem
- Some tools/libraries expecting POSIX may not work
- Users familiar with POSIX may have incorrect assumptions

### Alternatives Considered
1. **Full POSIX**: Rejected because it's complex and we don't need it.
2. **POSIX subset with compliance flag**: Rejected because partial compliance is confusing.

---

## Template for Future ADRs

```markdown
## ADR-XXX: [Title]

### Status
[Proposed | Accepted | Deprecated | Superseded by ADR-YYY]

### Context
[What is the issue? Why do we need to make a decision?]

### Decision
[What is the change being proposed? Be specific.]

### Consequences
**Positive:**
- [Good outcome 1]
- [Good outcome 2]

**Negative:**
- [Tradeoff 1]
- [Tradeoff 2]

### Alternatives Considered
1. **[Alternative 1]**: [Why rejected]
2. **[Alternative 2]**: [Why rejected]
```

---

*For full technical details, see the [Design Document](./anyfs-container-design.md).*
