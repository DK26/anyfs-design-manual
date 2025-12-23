# AnyFS - Project Structure

**Status:** Current  
**Last updated:** 2025-12-23

---

## Repository Layout

```
anyfs-traits/              # Crate 1: core trait + types
  Cargo.toml
  src/
    lib.rs
    backend.rs
    types.rs
    error.rs

anyfs/                     # Crate 2: built-in backends (feature-gated)
  Cargo.toml
  src/
    lib.rs                 # re-exports anyfs-traits::*
    memory/                # [feature: memory] (default)
    vrootfs/               # [feature: vrootfs]
    sqlite/                # [feature: sqlite]

anyfs-container/           # Crate 3: policy layer (quotas + isolation)
  Cargo.toml
  src/
    lib.rs
    container.rs           # FilesContainer<B: VfsBackend>
    builder.rs             # ContainerBuilder
    limits.rs              # CapacityLimits
    usage.rs               # CapacityUsage, CapacityRemaining
    error.rs               # ContainerError
```

---

## Dependency Model

```
strict-path (VirtualPath, VirtualRoot)
    -> anyfs-traits
         -> anyfs (optional backends)
         -> anyfs-container (policy/quotas)
```

**Key point:** custom backends depend only on `anyfs-traits`.

---

## Path Flow

```
User code:  container.read("/a/b.txt")
   -> FilesContainer validates into VirtualPath
   -> backend.read(&VirtualPath)
```

This keeps validation centralized and keeps backends simple.

---

## Feature Controls

There are two kinds of feature selection:

1. **Cargo features (compile-time)** select built-in backends in `anyfs` (`memory`, `sqlite`, `vrootfs`).
2. **Container feature whitelist (runtime policy)** enables advanced filesystem behavior per container instance:
   - `symlinks()`
   - `hard_links()`
   - `permissions()`

---

## Where To Start

- Application usage: `book/src/getting-started/guide.md`
- Trait details: `book/src/traits/vfs-trait.md`
- Decisions and rationale: `book/src/architecture/adrs.md`