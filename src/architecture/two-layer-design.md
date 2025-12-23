# Two-Layer Design: Ergonomic Paths + Validated Paths

AnyFS intentionally uses two layers of path handling:

1. **User-facing**: `FilesContainer` methods take `impl AsRef<Path>`.
2. **Backend-facing**: `VfsBackend` methods take `&VirtualPath` (from `strict-path`).

This keeps application code ergonomic while giving backends a single, validated path type.

---

## Why This Matters

- **Least privilege**: validation happens once at the boundary.
- **Backend simplicity**: backends do not re-parse strings or fight `..` traversal edge cases.
- **Consistency**: all backends receive the same normalized representation.

---

## Flow

```
User code:
  container.write("/data/file.txt", b"...")

FilesContainer:
  VirtualPath::new("/data/file.txt") -> VirtualPath
  backend.write(&vpath, data)

Backend:
  receives &VirtualPath (already validated)
```

---

## Important Interaction With Security

`FilesContainer` is the enforcement point for:
- quota/limit checks
- feature whitelisting (deny-by-default for advanced behavior)

The backend provides storage and filesystem semantics, but the container decides what is allowed for a given application.