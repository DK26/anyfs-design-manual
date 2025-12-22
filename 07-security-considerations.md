# VFS Container — Security Considerations

**Security model, threat analysis, and containment guarantees**

---

## Overview

VFS Container is designed with security as a primary concern. This document outlines the security model, potential threats, and the guarantees provided by the architecture.

---

## Threat Model

### In Scope

| Threat | Description | Mitigation |
|--------|-------------|------------|
| **Path traversal** | Attacker attempts to access files outside the container via `../` | Lexical path resolution in `VirtualPath` |
| **Symlink attacks** | Attacker creates symlinks pointing outside container | Symlinks are contained; targets validated |
| **Resource exhaustion** | Attacker fills storage or creates excessive files | Capacity limits enforced by `FilesContainer` |
| **Tenant data leakage** | One tenant accesses another's data | Complete isolation via separate containers |

### Out of Scope

| Threat | Reason |
|--------|--------|
| **Side-channel attacks** | Timing attacks, cache analysis — requires OS-level mitigations |
| **Physical access** | Disk encryption is application's responsibility |
| **SQLite vulnerabilities** | Upstream dependency; update regularly |
| **Denial of service** | Rate limiting is application's responsibility |

---

## Security Architecture

### 1. Path Containment (via strict-path)

All paths are validated through `VirtualPath` from the `strict-path` crate:

```rust
// User input is validated immediately
let path = VirtualPath::new(user_input)?;

// These attacks are structurally prevented:
VirtualPath::new("/../../../etc/passwd")  // → "/etc/passwd" (clamped to root)
VirtualPath::new("/data/../../../../tmp") // → "/tmp" (clamped to root)
```

**Guarantee**: No path can escape the virtual root, regardless of `..` sequences.

### 2. Lexical Path Resolution

Unlike POSIX filesystems, paths are resolved **lexically** without consulting the filesystem:

```rust
// POSIX behavior (dangerous):
// /foo/bar/.. where bar → /etc would resolve to /etc/../ → /

// VFS Container behavior (safe):
// /foo/bar/.. always resolves to /foo (pure string manipulation)
```

**Guarantee**: Path resolution is deterministic and cannot be influenced by symlinks or filesystem state.

### 3. Symlink Safety

When symlinks are enabled (`.symlinks(true)`):

- Symlink targets are validated as `VirtualPath`
- Resolution has a configurable depth limit (default: 20)
- Symlinks cannot point outside the container

```rust
// Symlink creation validates the target
container.symlink(
    &VirtualPath::new("/shortcut")?,
    &VirtualPath::new("/deeply/nested/path")?  // Must be valid VirtualPath
)?;

// Following symlinks respects depth limit
container.read(&VirtualPath::new("/shortcut")?)?;  // Max 20 hops
```

**Guarantee**: Symlink loops are detected; symlinks cannot escape containment.

### 4. Capacity Limits

`FilesContainer` enforces configurable limits:

```rust
let container = FilesContainer::builder()
    .backend(backend)
    .max_total_size(100 * 1024 * 1024)  // 100 MB total
    .max_file_size(10 * 1024 * 1024)    // 10 MB per file
    .max_node_count(10_000)              // 10K files/directories
    .max_dir_entries(1_000)              // 1K entries per directory
    .max_path_depth(64)                  // Max directory depth
    .build()?;
```

**Guarantee**: Resource exhaustion attacks are mitigated.

### 5. Backend Isolation

Each backend provides isolation differently:

| Backend | Isolation Mechanism |
|---------|---------------------|
| `VRootFsBackend` | `strict-path::VirtualRoot` clamps all paths to root directory |
| `SqliteBackend` | Each container is a separate `.db` file |
| `MemoryBackend` | Each container is a separate process memory region |

**Guarantee**: Operations on one container cannot affect another.

---

## Secure Usage Patterns

### Multi-Tenant Isolation

```rust
// Each tenant gets a completely separate container
fn create_tenant_storage(tenant_id: &str) -> FilesContainer<SqliteBackend> {
    let db_path = format!("tenants/{}.db", tenant_id);
    let backend = SqliteBackend::create(&db_path).unwrap();

    FilesContainer::builder()
        .backend(backend)
        .max_total_size(tenant_quota(tenant_id))
        .build()
        .unwrap()
}

// No path can cross tenant boundaries — they're different files entirely
```

### Untrusted Input Handling

```rust
fn handle_user_upload(
    container: &mut FilesContainer<impl VfsBackend>,
    user_filename: &str,  // Untrusted input
    data: &[u8],
) -> Result<(), ContainerError> {
    // VirtualPath::new validates the input
    // - Rejects invalid UTF-8
    // - Normalizes path (removes .., .)
    // - Ensures absolute path
    let path = VirtualPath::new(&format!("/uploads/{}", user_filename))?;

    // Safe to use — already validated
    container.write(&path, data)?;
    Ok(())
}
```

### Sandboxed Execution

```rust
// Create an isolated sandbox for untrusted code
let sandbox = FilesContainer::builder()
    .backend(MemoryBackend::new())
    .max_total_size(10 * 1024 * 1024)  // 10 MB limit
    .max_file_size(1024 * 1024)         // 1 MB per file
    .max_node_count(100)                // Max 100 files
    .symlinks(false)                    // Disable symlinks
    .hard_links(false)                  // Disable hard links
    .build()?;

// Untrusted code can only access this sandbox
// Cannot escape, cannot exhaust resources
```

---

## Security Checklist

### For Application Developers

- [ ] Set appropriate capacity limits for your use case
- [ ] Use separate containers for separate tenants/users
- [ ] Validate user input before creating `VirtualPath`
- [ ] Disable symlinks/hard links if not needed
- [ ] Keep `strict-path` and `rusqlite` dependencies updated

### For Backend Implementers

- [ ] Never bypass `VirtualPath` validation
- [ ] Ensure all paths are contained within the backend's scope
- [ ] Handle concurrent access safely (if supporting `Sync`)
- [ ] Report errors without leaking internal paths

---

## Known Limitations

1. **No encryption at rest**: Use OS-level encryption (LUKS, FileVault) or encrypted SQLite extensions
2. **No access control lists**: Permissions are opt-in and simple (Unix mode bits only)
3. **No audit logging**: Application must implement its own logging
4. **Time-of-check-time-of-use (TOCTOU)**: Like all filesystems, check-then-act patterns may race

---

## Reporting Security Issues

If you discover a security vulnerability, please report it responsibly:

1. Do not open a public issue
2. Email security concerns to the maintainers
3. Allow time for a fix before public disclosure

---

*For implementation details, see [Architecture Decision Records](./04-architecture-decision-records.md).*
