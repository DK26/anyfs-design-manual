# Layered Design: Backends + Middleware + Ergonomics

AnyFS uses a layered architecture that separates concerns:

1. **Backends**: Pure storage + filesystem semantics
2. **Middleware**: Composable policy layers
3. **FilesContainer**: Ergonomic wrapper

---

## Architecture

```
┌─────────────────────────────────────────┐
│  FilesContainer                         │  ← Ergonomics only
├─────────────────────────────────────────┤
│  Middleware Stack (composable):         │  ← Policy enforcement
│    Tracing → PathFilter → FeatureGuard  │
│    → Quota → Backend                    │
├─────────────────────────────────────────┤
│  VfsBackend                             │  ← Pure storage
│  (Memory, SQLite, VRootFs, custom)      │
└─────────────────────────────────────────┘
```

---

## Layer Responsibilities

| Layer | Responsibility | Path Handling |
|-------|----------------|---------------|
| `FilesContainer` | Ergonomic API | Accepts `impl AsRef<Path>` |
| Middleware | Policy enforcement | Accepts `impl AsRef<Path>` |
| Backend | Storage + FS semantics | Accepts `impl AsRef<Path>` |

All layers use `impl AsRef<Path>` for consistency with `std::fs`.

---

## Policy via Middleware

**Old design (rejected):** FilesContainer contained quota/feature logic.

**Current design:** Policy is handled by composable middleware:

```rust
// Middleware enforces policy
let backend = PathFilter::new(
    FeatureGuard::new(
        Quota::new(MemoryBackend::new())
    )
)
.allow("/workspace/**");

// FilesContainer is just ergonomics
let fs = FilesContainer::new(backend);
```

---

## Path Containment

For `VRootFsBackend` (real filesystem), path containment uses `strict-path::VirtualRoot` internally:

```rust
impl VfsBackend for VRootFsBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        // VirtualRoot ensures paths can't escape
        let safe_path = self.root.join(path)?;
        std::fs::read(safe_path).map_err(Into::into)
    }
}
```

For virtual backends (Memory, SQLite), paths are just keys - no OS path traversal possible.

For sandboxing across all backends, use `PathFilter` middleware:

```rust
PathFilter::new(backend)
    .allow("/workspace/**")
    .deny("**/.env")
```

---

## Why This Matters

- **Separation of concerns**: Backends focus on storage, middleware handles policy
- **Composability**: Add/remove policies without touching storage code
- **Flexibility**: Same middleware works with any backend
- **Simplicity**: Each layer has one job
