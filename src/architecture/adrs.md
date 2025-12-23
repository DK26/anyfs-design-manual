# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Path-based `VfsBackend` trait (20 methods) | Accepted |
| ADR-002 | Three-crate structure (`anyfs-traits`, `anyfs`, `anyfs-container`) | Accepted |
| ADR-003 | Two-layer path handling (`AsRef<Path>` -> `VirtualPath`) | Accepted |
| ADR-004 | Use `strict-path` for `VirtualPath` / containment | Accepted |
| ADR-005 | `std::fs`-aligned method names | Accepted |
| ADR-006 | Least-privilege feature whitelist in `FilesContainer` | Accepted |
| ADR-007 | Limits enforced in container (not backend) | Accepted |
| ADR-008 | Built-in backends are feature-gated | Accepted |

---

## ADR-001: Path-based `VfsBackend` trait (20 methods)

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Three-crate structure

**Decision:**
- `anyfs-traits` is minimal and dependency-light
- `anyfs` provides optional built-in backends and re-exports the trait
- `anyfs-container` adds quotas/isolation/policy and should not force backend dependencies on custom backend authors

---

## ADR-003: Two-layer path handling

**Decision:**
- `FilesContainer` accepts `impl AsRef<Path>` for ergonomics
- `VfsBackend` uses `&VirtualPath` so backends receive validated paths

---

## ADR-004: `VirtualPath` from `strict-path`

**Decision:** Re-export `strict_path::VirtualPath` instead of defining a custom path type.

**Why:** Centralizes containment and normalization in a tested dependency.

---

## ADR-005: `std::fs`-aligned method names

**Decision:** Prefer `read_dir`, `create_dir_all`, `remove_file`, etc.

**Why:** Familiarity and reduced cognitive overhead.

---

## ADR-006: Least-privilege feature whitelist in `FilesContainer`

**Decision:** Advanced behavior is disabled by default and explicitly enabled per container instance.

- `symlinks()` enables symlink creation and symlink-following behavior (bounded by `max_symlink_resolution`, default 40)
- `hard_links()` enables hard link creation
- `permissions()` enables permission mutation via `set_permissions`

When disabled, the relevant operations return `ContainerError::FeatureNotEnabled("...")`.

---

## ADR-007: Limits enforced in container (not backend)

**Decision:** Capacity limits are enforced at the `anyfs-container` layer.

**Why:** Limits are a policy decision and should be consistent across all backends.

---

## ADR-008: Built-in backends are feature-gated

**Decision:** `anyfs` uses Cargo features so users only pull the dependencies they need.

- `memory` default
- `sqlite` optional
- `vrootfs` optional

---

## Historical Notes

Older docs referencing a graph-store (`StorageBackend`, `NodeId`, transactions) or inode-based traits are kept only as history and should not be treated as current.