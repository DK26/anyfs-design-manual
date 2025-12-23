# Implementation Plan

This plan describes a phased rollout of the AnyFS ecosystem:

- `anyfs-traits`: minimal contract (`VfsBackend` + core types, re-export `VirtualPath`)
- `anyfs`: built-in backends (feature-gated) + re-exports
- `anyfs-container`: `FilesContainer<B: VfsBackend>` policy layer (limits + feature whitelist)

---

## Phase 1: `anyfs-traits` (core contract)

**Goal:** Define the stable backend interface and shared types.

- Re-export `strict_path::VirtualPath`
- Define core types (`Metadata`, `Permissions`, `FileType`, `DirEntry`)
- Define `VfsError` (errors carry `VirtualPath` where relevant)
- Define `VfsBackend` trait (20 `std::fs`-aligned, path-based methods)

**Exit criteria:** `anyfs-traits` stands alone with minimal dependencies (`strict-path`, `thiserror`).

---

## Phase 2: `anyfs` (built-in backends)

**Goal:** Provide a small set of reference/production backends behind Cargo features.

- `memory` (default): `MemoryBackend`
- `vrootfs` (optional): `VRootFsBackend` using `strict_path::VirtualRoot` for containment
- `sqlite` (optional): `SqliteBackend` storing an entire filesystem in a single `.db`

**Exit criteria:** Each backend implements `VfsBackend` and passes the shared conformance suite.

---

## Phase 3: `anyfs-container` (policy + ergonomics)

**Goal:** Provide the user-facing `std::fs`-aligned API with consistent policy enforcement.

- `FilesContainer<B>` accepts `impl AsRef<Path>` for ergonomics
- Centralizes path validation: convert once to `VirtualPath`, then call backend
- Enforces limits (quota and structural constraints):
  - `max_total_size`
  - `max_file_size`
  - `max_node_count`
  - `max_dir_entries`
  - `max_path_depth`
- Implements a least-privilege feature whitelist (default deny):
  - `symlinks()` (+ `max_symlink_resolution`, default 40)
  - `hard_links()`
  - `permissions()`

**Exit criteria:** An application can use `FilesContainer` as a drop-in for common `std::fs` patterns while gaining quotas + default-deny policy.

---

## Phase 4: Conformance test suite

**Goal:** Prevent backend divergence and make semantics explicit.

Recommended structure:

- Backend conformance tests (run against every `VfsBackend` implementation)
  - `read`/`write`/`append` semantics
  - directory operations (`create_dir*`, `read_dir`, `remove_dir*`)
  - `rename` and `copy` semantics
  - link behavior (`symlink`, `read_link`, `hard_link`, `nlink`)
  - permissions (`set_permissions`) where meaningful
- Container policy tests
  - limit enforcement and usage accounting
  - whitelist behavior (`FeatureNotEnabled`)
  - symlink resolution depth limit (when enabled)

**Exit criteria:** All built-in backends pass the same suite; container policy tests are backend-agnostic.

---

## Phase 5: Documentation + examples

- Keep `book/src/architecture/design-overview.md` + `book/src/architecture/adrs.md` authoritative
- Provide at least one complete example per backend
- Provide a backend implementer guide for `anyfs-traits`

---

## Future work (out of scope for MVP)

- Streaming I/O (file handles)
- Async API
- Import/export helpers (host path <-> container path)
- Extended attributes
- Encryption-at-rest helpers (backend-specific)
- Capability negotiation (optional): detect when a backend cannot support certain operations
