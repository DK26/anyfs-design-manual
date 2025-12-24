# Implementation Plan

This plan describes a phased rollout of the AnyFS ecosystem:

- `anyfs-backend`: Core trait (`VfsBackend`, `Layer`) + types
- `anyfs`: Built-in backends + middleware (feature-gated)
- `anyfs-container`: `FilesContainer<B>` ergonomic wrapper

---

## Phase 1: `anyfs-backend` (core contract)

**Goal:** Define the stable backend interface and composition traits.

- Define `VfsBackend` trait (25 `std::fs`-aligned methods)
- Define `Layer` trait (Tower-style middleware composition)
- Define `VfsBackendExt` trait (extension methods)
- Define core types (`Metadata`, `Permissions`, `FileType`, `DirEntry`, `StatFs`)
- Define `VfsError` with contextual variants

**Exit criteria:** `anyfs-backend` stands alone with minimal dependencies (`thiserror`).

---

## Phase 2: `anyfs` (backends + middleware)

**Goal:** Provide reference backends and core middleware.

### Backends (feature-gated)

- `memory` (default): `MemoryBackend`
- `sqlite` (optional): `SqliteBackend`
- `vrootfs` (optional): `VRootFsBackend` using `strict-path` for containment

### Middleware

- `Quota<B>` + `QuotaLayer` - Resource limits
- `FeatureGuard<B>` + `FeatureGuardLayer` - Feature whitelist
- `PathFilter<B>` + `PathFilterLayer` - Path-based access control
- `ReadOnly<B>` + `ReadOnlyLayer` - Block writes
- `RateLimit<B>` + `RateLimitLayer` - Operation throttling
- `Tracing<B>` + `TracingLayer` - Instrumentation
- `DryRun<B>` + `DryRunLayer` - Log without executing
- `Cache<B>` + `CacheLayer` - LRU read cache
- `Overlay<B1,B2>` + `OverlayLayer` - Union filesystem

**Exit criteria:** Each backend implements `VfsBackend` and passes conformance suite. Each middleware wraps any `VfsBackend`.

---

## Phase 3: `anyfs-container` (ergonomics)

**Goal:** Provide user-facing ergonomic wrapper.

- `FilesContainer<B>` - Thin wrapper with `std::fs`-aligned API
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to wrapped backend

**Note:** `FilesContainer` contains NO policy logic. Policy is handled by middleware.

**Exit criteria:** Applications can use `FilesContainer` as drop-in for `std::fs` patterns.

---

## Phase 4: Conformance test suite

**Goal:** Prevent backend divergence and validate middleware behavior.

### Backend conformance tests

- `read`/`write`/`append` semantics
- Directory operations (`create_dir*`, `read_dir`, `remove_dir*`)
- `rename` and `copy` semantics
- Link behavior (`symlink`, `read_link`, `hard_link`)
- Streaming I/O (`open_read`, `open_write`)
- `truncate`, `sync`, `fsync`, `statfs`

### Middleware tests

- `Quota`: Limit enforcement, usage tracking
- `FeatureGuard`: Feature blocking
- `PathFilter`: Glob pattern matching
- `RateLimit`: Throttling behavior
- Middleware composition order

**Exit criteria:** All backends pass same suite; middleware tests are backend-agnostic.

---

## Phase 5: Documentation + examples

- Keep `AGENTS.md` and `src/architecture/design-overview.md` authoritative
- Provide example per backend
- Provide backend implementer guide
- Provide middleware implementer guide

---

## Future work (post-MVP)

- Async API (`AsyncVfsBackend` trait)
- Import/export helpers (host path <-> container)
- Extended attributes
- Encryption middleware
- Compression middleware
- `vfs` crate compatibility adapter
