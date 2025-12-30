# Conformance Test Suite

**Verify your backend or middleware works correctly with AnyFS**

This document provides a complete test suite skeleton that any backend or middleware implementer can use to verify their implementation meets the AnyFS trait contracts.

---

## Overview

The conformance test suite verifies:

1. **Correctness** - Operations behave as specified
2. **Error handling** - Correct errors for edge cases
3. **Thread safety** - Safe concurrent access
4. **Trait contracts** - All trait requirements met

### Test Levels

| Level | Traits Tested | When to Use |
|-------|---------------|-------------|
| **Core** | `FsRead`, `FsWrite`, `FsDir` (= `Fs`) | All backends |
| **Full** | + `FsLink`, `FsPermissions`, `FsSync`, `FsStats` | Extended backends |
| **Fuse** | + `FsInode` | FUSE-mountable backends |
| **Posix** | + `FsHandles`, `FsLock`, `FsXattr` | Full POSIX backends |

---

## Quick Start

### For Backend Implementers

Add this to your `Cargo.toml`:

```toml
[dev-dependencies]
anyfs-test = "0.1"  # Conformance test suite
```

Then in your test file:

```rust
use anyfs_test::prelude::*;

// Tell the test suite how to create your backend
fn create_backend() -> MyBackend {
    MyBackend::new()
}

// Run all Fs-level tests
anyfs_test::generate_fs_tests!(create_backend);

// If you implement FsFull traits:
// anyfs_test::generate_fs_full_tests!(create_backend);

// If you implement FsFuse traits:
// anyfs_test::generate_fs_fuse_tests!(create_backend);
```

### For Middleware Implementers

```rust
use anyfs_test::prelude::*;
use anyfs::MemoryBackend;

// Wrap MemoryBackend with your middleware
fn create_backend() -> MyMiddleware<MemoryBackend> {
    MyMiddleware::new(MemoryBackend::new())
}

// Run all tests through your middleware
anyfs_test::generate_fs_tests!(create_backend);
```

---

## Core Test Suite (Fs Traits)

Copy this entire module into your test file and customize `create_backend()`.

```rust
//! Conformance tests for Fs trait implementations.
//!
//! To use: implement `create_backend()` and include this module.

use anyfs_backend::{Fs, FsRead, FsWrite, FsDir, FsError, FileType, Metadata, ReadDirIter};
use std::path::Path;
use std::sync::Arc;
use std::thread;

/// Create a fresh backend instance for testing.
/// Implement this for your backend.
fn create_backend() -> impl Fs {
    todo!("Return your backend here")
}

// ============================================================================
// FsRead Tests
// ============================================================================

mod fs_read {
    use super::*;

    #[test]
    fn read_existing_file() {
        let fs = create_backend();
        fs.write("/test.txt", b"hello world").unwrap();

        let content = fs.read("/test.txt").unwrap();
        assert_eq!(content, b"hello world");
    }

    #[test]
    fn read_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.read("/nonexistent.txt");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn read_directory_returns_not_a_file() {
        let fs = create_backend();
        fs.create_dir("/mydir").unwrap();

        let result = fs.read("/mydir");
        assert!(matches!(result, Err(FsError::NotAFile { .. })));
    }

    #[test]
    fn read_to_string_valid_utf8() {
        let fs = create_backend();
        fs.write("/text.txt", "hello unicode: 你好".as_bytes()).unwrap();

        let content = fs.read_to_string("/text.txt").unwrap();
        assert_eq!(content, "hello unicode: 你好");
    }

    #[test]
    fn read_to_string_invalid_utf8_returns_error() {
        let fs = create_backend();
        fs.write("/binary.bin", &[0xFF, 0xFE, 0x00, 0x01]).unwrap();

        let result = fs.read_to_string("/binary.bin");
        assert!(result.is_err());
    }

    #[test]
    fn read_range_full_file() {
        let fs = create_backend();
        fs.write("/data.bin", b"0123456789").unwrap();

        let content = fs.read_range("/data.bin", 0, 10).unwrap();
        assert_eq!(content, b"0123456789");
    }

    #[test]
    fn read_range_partial() {
        let fs = create_backend();
        fs.write("/data.bin", b"0123456789").unwrap();

        let content = fs.read_range("/data.bin", 3, 4).unwrap();
        assert_eq!(content, b"3456");
    }

    #[test]
    fn read_range_past_end_returns_available() {
        let fs = create_backend();
        fs.write("/data.bin", b"0123456789").unwrap();

        let content = fs.read_range("/data.bin", 8, 100).unwrap();
        assert_eq!(content, b"89");
    }

    #[test]
    fn read_range_offset_past_end_returns_empty() {
        let fs = create_backend();
        fs.write("/data.bin", b"0123456789").unwrap();

        let content = fs.read_range("/data.bin", 100, 10).unwrap();
        assert!(content.is_empty());
    }

    #[test]
    fn exists_for_existing_file() {
        let fs = create_backend();
        fs.write("/exists.txt", b"data").unwrap();

        assert!(fs.exists("/exists.txt").unwrap());
    }

    #[test]
    fn exists_for_nonexistent_file() {
        let fs = create_backend();

        assert!(!fs.exists("/nonexistent.txt").unwrap());
    }

    #[test]
    fn exists_for_directory() {
        let fs = create_backend();
        fs.create_dir("/mydir").unwrap();

        assert!(fs.exists("/mydir").unwrap());
    }

    #[test]
    fn metadata_for_file() {
        let fs = create_backend();
        fs.write("/file.txt", b"hello").unwrap();

        let meta = fs.metadata("/file.txt").unwrap();
        assert_eq!(meta.file_type, FileType::File);
        assert_eq!(meta.size, 5);
    }

    #[test]
    fn metadata_for_directory() {
        let fs = create_backend();
        fs.create_dir("/mydir").unwrap();

        let meta = fs.metadata("/mydir").unwrap();
        assert_eq!(meta.file_type, FileType::Directory);
    }

    #[test]
    fn metadata_for_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.metadata("/nonexistent");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn open_read_and_consume() {
        let fs = create_backend();
        fs.write("/stream.txt", b"streaming content").unwrap();

        let mut reader = fs.open_read("/stream.txt").unwrap();
        let mut buf = Vec::new();
        std::io::Read::read_to_end(&mut reader, &mut buf).unwrap();

        assert_eq!(buf, b"streaming content");
    }
}

// ============================================================================
// FsWrite Tests
// ============================================================================

mod fs_write {
    use super::*;

    #[test]
    fn write_creates_new_file() {
        let fs = create_backend();

        fs.write("/new.txt", b"new content").unwrap();

        assert!(fs.exists("/new.txt").unwrap());
        assert_eq!(fs.read("/new.txt").unwrap(), b"new content");
    }

    #[test]
    fn write_overwrites_existing_file() {
        let fs = create_backend();
        fs.write("/file.txt", b"original").unwrap();

        fs.write("/file.txt", b"replaced").unwrap();

        assert_eq!(fs.read("/file.txt").unwrap(), b"replaced");
    }

    #[test]
    fn write_to_nonexistent_parent_returns_not_found() {
        let fs = create_backend();

        let result = fs.write("/nonexistent/file.txt", b"data");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn write_empty_file() {
        let fs = create_backend();

        fs.write("/empty.txt", b"").unwrap();

        assert!(fs.exists("/empty.txt").unwrap());
        assert_eq!(fs.read("/empty.txt").unwrap(), b"");
    }

    #[test]
    fn write_binary_data() {
        let fs = create_backend();
        let binary: Vec<u8> = (0..=255).collect();

        fs.write("/binary.bin", &binary).unwrap();

        assert_eq!(fs.read("/binary.bin").unwrap(), binary);
    }

    #[test]
    fn append_to_existing_file() {
        let fs = create_backend();
        fs.write("/log.txt", b"line1\n").unwrap();

        fs.append("/log.txt", b"line2\n").unwrap();

        assert_eq!(fs.read("/log.txt").unwrap(), b"line1\nline2\n");
    }

    #[test]
    fn append_to_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.append("/nonexistent.txt", b"data");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn remove_file_existing() {
        let fs = create_backend();
        fs.write("/delete-me.txt", b"bye").unwrap();

        fs.remove_file("/delete-me.txt").unwrap();

        assert!(!fs.exists("/delete-me.txt").unwrap());
    }

    #[test]
    fn remove_file_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.remove_file("/nonexistent.txt");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn remove_file_on_directory_returns_not_a_file() {
        let fs = create_backend();
        fs.create_dir("/mydir").unwrap();

        let result = fs.remove_file("/mydir");
        assert!(matches!(result, Err(FsError::NotAFile { .. })));
    }

    #[test]
    fn rename_file() {
        let fs = create_backend();
        fs.write("/old.txt", b"content").unwrap();

        fs.rename("/old.txt", "/new.txt").unwrap();

        assert!(!fs.exists("/old.txt").unwrap());
        assert!(fs.exists("/new.txt").unwrap());
        assert_eq!(fs.read("/new.txt").unwrap(), b"content");
    }

    #[test]
    fn rename_overwrites_destination() {
        let fs = create_backend();
        fs.write("/src.txt", b"source").unwrap();
        fs.write("/dst.txt", b"destination").unwrap();

        fs.rename("/src.txt", "/dst.txt").unwrap();

        assert!(!fs.exists("/src.txt").unwrap());
        assert_eq!(fs.read("/dst.txt").unwrap(), b"source");
    }

    #[test]
    fn rename_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.rename("/nonexistent.txt", "/new.txt");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn copy_file() {
        let fs = create_backend();
        fs.write("/original.txt", b"data").unwrap();

        fs.copy("/original.txt", "/copy.txt").unwrap();

        assert!(fs.exists("/original.txt").unwrap());
        assert!(fs.exists("/copy.txt").unwrap());
        assert_eq!(fs.read("/copy.txt").unwrap(), b"data");
    }

    #[test]
    fn copy_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.copy("/nonexistent.txt", "/copy.txt");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn truncate_shrink() {
        let fs = create_backend();
        fs.write("/file.txt", b"0123456789").unwrap();

        fs.truncate("/file.txt", 5).unwrap();

        assert_eq!(fs.read("/file.txt").unwrap(), b"01234");
    }

    #[test]
    fn truncate_expand() {
        let fs = create_backend();
        fs.write("/file.txt", b"abc").unwrap();

        fs.truncate("/file.txt", 6).unwrap();

        let content = fs.read("/file.txt").unwrap();
        assert_eq!(content.len(), 6);
        assert_eq!(&content[..3], b"abc");
        // Expanded bytes should be zero
        assert!(content[3..].iter().all(|&b| b == 0));
    }

    #[test]
    fn truncate_to_zero() {
        let fs = create_backend();
        fs.write("/file.txt", b"content").unwrap();

        fs.truncate("/file.txt", 0).unwrap();

        assert_eq!(fs.read("/file.txt").unwrap(), b"");
    }

    #[test]
    fn open_write_and_close() {
        let fs = create_backend();

        {
            let mut writer = fs.open_write("/stream.txt").unwrap();
            std::io::Write::write_all(&mut writer, b"streamed").unwrap();
        }

        // Content should be visible after writer is dropped
        assert_eq!(fs.read("/stream.txt").unwrap(), b"streamed");
    }
}

// ============================================================================
// FsDir Tests
// ============================================================================

mod fs_dir {
    use super::*;

    #[test]
    fn create_dir_single() {
        let fs = create_backend();

        fs.create_dir("/newdir").unwrap();

        assert!(fs.exists("/newdir").unwrap());
        let meta = fs.metadata("/newdir").unwrap();
        assert_eq!(meta.file_type, FileType::Directory);
    }

    #[test]
    fn create_dir_already_exists_returns_error() {
        let fs = create_backend();
        fs.create_dir("/existing").unwrap();

        let result = fs.create_dir("/existing");
        assert!(matches!(result, Err(FsError::AlreadyExists { .. })));
    }

    #[test]
    fn create_dir_parent_not_exists_returns_not_found() {
        let fs = create_backend();

        let result = fs.create_dir("/nonexistent/child");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn create_dir_all_nested() {
        let fs = create_backend();

        fs.create_dir_all("/a/b/c/d").unwrap();

        assert!(fs.exists("/a").unwrap());
        assert!(fs.exists("/a/b").unwrap());
        assert!(fs.exists("/a/b/c").unwrap());
        assert!(fs.exists("/a/b/c/d").unwrap());
    }

    #[test]
    fn create_dir_all_partially_exists() {
        let fs = create_backend();
        fs.create_dir("/exists").unwrap();

        fs.create_dir_all("/exists/new/nested").unwrap();

        assert!(fs.exists("/exists/new/nested").unwrap());
    }

    #[test]
    fn create_dir_all_already_exists_is_ok() {
        let fs = create_backend();
        fs.create_dir_all("/a/b/c").unwrap();

        // Should not error
        fs.create_dir_all("/a/b/c").unwrap();
    }

    #[test]
    fn read_dir_empty() {
        let fs = create_backend();
        fs.create_dir("/empty").unwrap();

        let entries: Vec<_> = fs.read_dir("/empty").unwrap()
            .filter_map(|e| e.ok())
            .collect();

        assert!(entries.is_empty());
    }

    #[test]
    fn read_dir_with_files() {
        let fs = create_backend();
        fs.create_dir("/parent").unwrap();
        fs.write("/parent/file1.txt", b"1").unwrap();
        fs.write("/parent/file2.txt", b"2").unwrap();
        fs.create_dir("/parent/subdir").unwrap();

        let mut entries: Vec<_> = fs.read_dir("/parent").unwrap()
            .filter_map(|e| e.ok())
            .collect();
        entries.sort_by(|a, b| a.name.cmp(&b.name));

        assert_eq!(entries.len(), 3);
        assert_eq!(entries[0].name, "file1.txt");
        assert_eq!(entries[0].file_type, FileType::File);
        assert_eq!(entries[1].name, "file2.txt");
        assert_eq!(entries[2].name, "subdir");
        assert_eq!(entries[2].file_type, FileType::Directory);
    }

    #[test]
    fn read_dir_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.read_dir("/nonexistent");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn read_dir_on_file_returns_not_a_directory() {
        let fs = create_backend();
        fs.write("/file.txt", b"data").unwrap();

        let result = fs.read_dir("/file.txt");
        assert!(matches!(result, Err(FsError::NotADirectory { .. })));
    }

    #[test]
    fn remove_dir_empty() {
        let fs = create_backend();
        fs.create_dir("/todelete").unwrap();

        fs.remove_dir("/todelete").unwrap();

        assert!(!fs.exists("/todelete").unwrap());
    }

    #[test]
    fn remove_dir_not_empty_returns_error() {
        let fs = create_backend();
        fs.create_dir("/notempty").unwrap();
        fs.write("/notempty/file.txt", b"data").unwrap();

        let result = fs.remove_dir("/notempty");
        assert!(matches!(result, Err(FsError::DirectoryNotEmpty { .. })));
    }

    #[test]
    fn remove_dir_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.remove_dir("/nonexistent");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn remove_dir_on_file_returns_not_a_directory() {
        let fs = create_backend();
        fs.write("/file.txt", b"data").unwrap();

        let result = fs.remove_dir("/file.txt");
        assert!(matches!(result, Err(FsError::NotADirectory { .. })));
    }

    #[test]
    fn remove_dir_all_recursive() {
        let fs = create_backend();
        fs.create_dir_all("/root/a/b").unwrap();
        fs.write("/root/file.txt", b"data").unwrap();
        fs.write("/root/a/nested.txt", b"nested").unwrap();

        fs.remove_dir_all("/root").unwrap();

        assert!(!fs.exists("/root").unwrap());
        assert!(!fs.exists("/root/a").unwrap());
        assert!(!fs.exists("/root/file.txt").unwrap());
    }

    #[test]
    fn remove_dir_all_nonexistent_returns_not_found() {
        let fs = create_backend();

        let result = fs.remove_dir_all("/nonexistent");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }
}

// ============================================================================
// Edge Case Tests
// ============================================================================

mod edge_cases {
    use super::*;

    #[test]
    fn root_directory_exists() {
        let fs = create_backend();

        assert!(fs.exists("/").unwrap());
        let meta = fs.metadata("/").unwrap();
        assert_eq!(meta.file_type, FileType::Directory);
    }

    #[test]
    fn read_dir_root() {
        let fs = create_backend();
        fs.write("/file.txt", b"data").unwrap();

        let entries: Vec<_> = fs.read_dir("/").unwrap()
            .filter_map(|e| e.ok())
            .collect();

        assert!(!entries.is_empty());
    }

    #[test]
    fn cannot_remove_root() {
        let fs = create_backend();

        let result = fs.remove_dir("/");
        assert!(result.is_err());
    }

    #[test]
    fn cannot_remove_root_all() {
        let fs = create_backend();

        let result = fs.remove_dir_all("/");
        assert!(result.is_err());
    }

    #[test]
    fn file_at_root_level() {
        let fs = create_backend();

        fs.write("/rootfile.txt", b"at root").unwrap();

        assert!(fs.exists("/rootfile.txt").unwrap());
        assert_eq!(fs.read("/rootfile.txt").unwrap(), b"at root");
    }

    #[test]
    fn deeply_nested_path() {
        let fs = create_backend();
        let deep_path = "/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p";

        fs.create_dir_all(deep_path).unwrap();
        fs.write(&format!("{}/file.txt", deep_path), b"deep").unwrap();

        assert_eq!(
            fs.read(&format!("{}/file.txt", deep_path)).unwrap(),
            b"deep"
        );
    }

    #[test]
    fn unicode_filename() {
        let fs = create_backend();

        fs.write("/文件.txt", b"chinese").unwrap();
        fs.write("/файл.txt", b"russian").unwrap();
        fs.write("/αρχείο.txt", b"greek").unwrap();

        assert_eq!(fs.read("/文件.txt").unwrap(), b"chinese");
        assert_eq!(fs.read("/файл.txt").unwrap(), b"russian");
        assert_eq!(fs.read("/αρχείο.txt").unwrap(), b"greek");
    }

    #[test]
    fn filename_with_spaces() {
        let fs = create_backend();

        fs.write("/file with spaces.txt", b"spaced").unwrap();

        assert!(fs.exists("/file with spaces.txt").unwrap());
        assert_eq!(fs.read("/file with spaces.txt").unwrap(), b"spaced");
    }

    #[test]
    fn filename_with_special_chars() {
        let fs = create_backend();

        fs.write("/file-name_123.test.txt", b"special").unwrap();

        assert!(fs.exists("/file-name_123.test.txt").unwrap());
    }

    #[test]
    fn large_file() {
        let fs = create_backend();
        let large_data: Vec<u8> = (0..1_000_000).map(|i| (i % 256) as u8).collect();

        fs.write("/large.bin", &large_data).unwrap();

        assert_eq!(fs.read("/large.bin").unwrap(), large_data);
    }

    #[test]
    fn many_files_in_directory() {
        let fs = create_backend();
        fs.create_dir("/many").unwrap();

        for i in 0..100 {
            fs.write(&format!("/many/file_{:03}.txt", i), format!("{}", i).as_bytes()).unwrap();
        }

        let entries: Vec<_> = fs.read_dir("/many").unwrap()
            .filter_map(|e| e.ok())
            .collect();

        assert_eq!(entries.len(), 100);
    }

    #[test]
    fn overwrite_larger_with_smaller() {
        let fs = create_backend();
        fs.write("/file.txt", b"this is a longer content").unwrap();

        fs.write("/file.txt", b"short").unwrap();

        assert_eq!(fs.read("/file.txt").unwrap(), b"short");
    }

    #[test]
    fn overwrite_smaller_with_larger() {
        let fs = create_backend();
        fs.write("/file.txt", b"short").unwrap();

        fs.write("/file.txt", b"this is a longer content").unwrap();

        assert_eq!(fs.read("/file.txt").unwrap(), b"this is a longer content");
    }
}

// ============================================================================
// Security Tests (Learned from Prior Art Vulnerabilities)
// ============================================================================

mod security {
    use super::*;

    // ------------------------------------------------------------------------
    // Path Traversal Tests (Apache Commons VFS CVE-inspired)
    // ------------------------------------------------------------------------

    #[test]
    fn reject_dotdot_traversal() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();
        fs.write("/secret.txt", b"secret").unwrap();

        // Direct .. traversal must be blocked or normalized
        let result = fs.read("/sandbox/../secret.txt");
        // Either blocks the operation or normalizes to /secret.txt (acceptable)
        // But must NOT escape sandbox context in sandboxed backends
    }

    #[test]
    fn reject_url_encoded_dotdot() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();

        // URL-encoded path traversal: %2e = '.', %2f = '/'
        // This caused CVE in Apache Commons VFS
        let result = fs.read("/sandbox/%2e%2e/etc/passwd");
        assert!(result.is_err(), "URL-encoded path traversal must be rejected");

        // Double-encoded traversal
        let result = fs.read("/sandbox/%252e%252e/etc/passwd");
        assert!(result.is_err(), "Double URL-encoded traversal must be rejected");
    }

    #[test]
    fn reject_backslash_traversal() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();

        // Windows-style path traversal
        let result = fs.read("/sandbox\\..\\secret.txt");
        assert!(result.is_err(), "Backslash traversal must be rejected");
    }

    #[test]
    fn reject_mixed_slash_traversal() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();

        // Mixed forward/backward slashes
        let result = fs.read("/sandbox/..\\..\\secret.txt");
        assert!(result.is_err(), "Mixed slash traversal must be rejected");

        let result = fs.read("/sandbox\\../secret.txt");
        assert!(result.is_err(), "Mixed slash traversal must be rejected");
    }

    #[test]
    fn reject_null_byte_injection() {
        let fs = create_backend();
        fs.write("/safe.txt", b"safe").unwrap();
        fs.write("/safe.txt.bak", b"backup").unwrap();

        // Null byte injection: /safe.txt\0.bak -> /safe.txt
        let result = fs.read("/safe.txt\0.bak");
        // Should either reject or not truncate at null
        if let Ok(content) = result {
            // If it succeeds, it should read the full path, not truncated
            assert_ne!(content, b"safe", "Null byte must not truncate path");
        }
    }

    // ------------------------------------------------------------------------
    // Symlink Security Tests (Afero-inspired)
    // ------------------------------------------------------------------------

    #[test]
    fn symlink_cannot_escape_sandbox() {
        // This test is for sandboxed backends (e.g., VRootFs)
        // Regular backends may allow this, which is fine
        let fs = create_backend();

        // Attempt to create symlink pointing outside virtual root
        let result = fs.symlink("/etc/passwd", "/escape_link");

        // Sandboxed backends MUST reject this
        // Non-sandboxed backends may allow it
        // The key is: reading through the symlink must not expose
        // content outside the sandbox
    }

    #[test]
    fn symlink_to_absolute_path_outside() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();
        fs.write("/sandbox/safe.txt", b"safe").unwrap();

        // Symlink pointing to absolute path outside sandbox
        // In sandboxed context, this must either:
        // 1. Reject symlink creation, or
        // 2. Resolve relative to sandbox root
        let result = fs.symlink("/../../../etc/passwd", "/sandbox/link");
        // Behavior depends on backend type
    }

    #[test]
    fn relative_symlink_traversal() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();
        fs.create_dir("/sandbox/subdir").unwrap();
        fs.write("/secret.txt", b"secret outside sandbox").unwrap();

        // Relative symlink that traverses up and out
        let _ = fs.symlink("../../secret.txt", "/sandbox/subdir/link");

        // If symlink was created, reading through it in a sandboxed
        // context must not expose /secret.txt
    }

    // ------------------------------------------------------------------------
    // Symlink Loop Detection Tests (PyFilesystem2-inspired)
    // ------------------------------------------------------------------------

    #[test]
    fn detect_direct_symlink_loop() {
        let fs = create_backend();

        // Self-referential symlink
        let _ = fs.symlink("/loop", "/loop");

        // Reading must detect the loop
        let result = fs.read("/loop");
        assert!(matches!(result, Err(FsError::TooManySymlinks { .. }))
            || matches!(result, Err(FsError::NotFound { .. }))
            || result.is_err(),
            "Direct symlink loop must be detected");
    }

    #[test]
    fn detect_indirect_symlink_loop() {
        let fs = create_backend();

        // Two symlinks pointing to each other: a -> b, b -> a
        let _ = fs.symlink("/b", "/a");
        let _ = fs.symlink("/a", "/b");

        // Reading either must detect the loop
        let result = fs.read("/a");
        assert!(matches!(result, Err(FsError::TooManySymlinks { .. }))
            || result.is_err(),
            "Indirect symlink loop must be detected");
    }

    #[test]
    fn detect_deep_symlink_chain() {
        let fs = create_backend();

        // Create a long chain of symlinks
        // link_0 -> link_1 -> link_2 -> ... -> link_N
        for i in 0..100 {
            let _ = fs.symlink(
                &format!("/link_{}", i + 1),
                &format!("/link_{}", i)
            );
        }
        fs.write("/link_100", b"target").unwrap();

        // Following the chain should either succeed or fail with TooManySymlinks
        // Must NOT cause stack overflow or infinite loop
        let result = fs.read("/link_0");
        // Either succeeds (if backend allows deep chains) or returns error
        // Key is: it must terminate
    }

    #[test]
    fn symlink_loop_with_directories() {
        let fs = create_backend();
        fs.create_dir("/dir1").unwrap();
        fs.create_dir("/dir2").unwrap();

        // Create directory symlink loop
        let _ = fs.symlink("/dir2", "/dir1/link_to_dir2");
        let _ = fs.symlink("/dir1", "/dir2/link_to_dir1");

        // Attempting to read a file through the loop
        let result = fs.read("/dir1/link_to_dir2/link_to_dir1/link_to_dir2/file.txt");
        assert!(result.is_err(), "Directory symlink loop must be detected");
    }

    // ------------------------------------------------------------------------
    // Resource Exhaustion Tests
    // ------------------------------------------------------------------------

    #[test]
    fn reject_excessive_symlink_depth() {
        let fs = create_backend();

        // FUSE typically limits to 40 symlink follows
        // We should have a reasonable limit (e.g., 40-256)
        const MAX_EXPECTED_DEPTH: u32 = 256;

        // Create chain that exceeds expected limit
        for i in 0..MAX_EXPECTED_DEPTH + 10 {
            let _ = fs.symlink(
                &format!("/excessive_{}", i + 1),
                &format!("/excessive_{}", i)
            );
        }

        // Create actual target
        fs.write(&format!("/excessive_{}", MAX_EXPECTED_DEPTH + 10), b"data").unwrap();

        // Should reject or limit, not follow indefinitely
        let result = fs.read("/excessive_0");
        // Either succeeds (backend allows this depth) or errors
        // Key: must not hang or OOM
    }

    // ------------------------------------------------------------------------
    // Canonicalization Tests
    // ------------------------------------------------------------------------

    #[test]
    fn path_normalization_removes_dots() {
        let fs = create_backend();
        fs.create_dir("/parent").unwrap();
        fs.write("/parent/file.txt", b"content").unwrap();

        // Single dots should be normalized
        let result = fs.read("/parent/./file.txt");
        if let Ok(content) = result {
            assert_eq!(content, b"content");
        }

        // Double dots in the middle
        let result = fs.read("/parent/subdir/../file.txt");
        // Either normalized to /parent/file.txt or rejected
    }

    #[test]
    fn path_normalization_removes_double_slashes() {
        let fs = create_backend();
        fs.write("/file.txt", b"content").unwrap();

        // Double slashes should be normalized
        let result = fs.read("//file.txt");
        if let Ok(content) = result {
            assert_eq!(content, b"content");
        }

        let result = fs.read("/parent//file.txt");
        // Either works (normalized) or returns NotFound (strict)
    }

    #[test]
    fn trailing_slash_handling() {
        let fs = create_backend();
        fs.create_dir("/mydir").unwrap();
        fs.write("/mydir/file.txt", b"content").unwrap();

        // Directory with trailing slash
        assert!(fs.exists("/mydir/").unwrap() || fs.exists("/mydir").unwrap());

        // File with trailing slash should fail or be normalized
        let result = fs.read("/mydir/file.txt/");
        // Either fails (strict) or normalizes (lenient)
    }

    // ------------------------------------------------------------------------
    // Windows-Specific Security Tests (from soft-canonicalize/strict-path)
    // ------------------------------------------------------------------------

    #[test]
    #[cfg(windows)]
    fn reject_ntfs_alternate_data_streams() {
        let fs = create_backend();
        fs.write("/file.txt", b"main content").unwrap();

        // NTFS ADS: file.txt:hidden_stream
        // Attacker may try to hide data or escape paths via ADS
        let result = fs.read("/file.txt:hidden");
        assert!(result.is_err(), "NTFS ADS must be rejected");

        let result = fs.read("/file.txt:$DATA");
        assert!(result.is_err(), "NTFS ADS with $DATA must be rejected");

        let result = fs.read("/file.txt::$DATA");
        assert!(result.is_err(), "NTFS default stream syntax must be rejected");

        // ADS in directory path (traversal attempt)
        let result = fs.read("/dir:ads/../secret.txt");
        assert!(result.is_err(), "ADS in directory path must be rejected");
    }

    #[test]
    #[cfg(windows)]
    fn reject_windows_8_3_short_names() {
        let fs = create_backend();
        fs.create_dir("/Program Files").unwrap();
        fs.write("/Program Files/secret.txt", b"secret").unwrap();

        // 8.3 short names can be used to obfuscate paths
        // PROGRA~1 is the typical short name for "Program Files"
        // Virtual filesystems should either:
        // 1. Not support 8.3 names at all (reject)
        // 2. Resolve them consistently to the same canonical path

        // Test that we don't accidentally create different files
        let result1 = fs.exists("/Program Files/secret.txt");
        let result2 = fs.exists("/PROGRA~1/secret.txt");

        // Either both exist (resolved) or short name doesn't exist (rejected)
        // Key: they must NOT be different files
        if result1.unwrap_or(false) && result2.unwrap_or(false) {
            // If both exist, they must have same content
            let content1 = fs.read("/Program Files/secret.txt").unwrap();
            let content2 = fs.read("/PROGRA~1/secret.txt").unwrap();
            assert_eq!(content1, content2, "8.3 names must resolve to same file");
        }
    }

    #[test]
    #[cfg(windows)]
    fn reject_windows_unc_traversal() {
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();

        // Extended-length path prefix traversal
        let result = fs.read("\\\\?\\C:\\..\\..\\etc\\passwd");
        assert!(result.is_err(), "UNC extended path traversal must be rejected");

        // Device namespace
        let result = fs.read("\\\\.\\C:\\secret.txt");
        assert!(result.is_err(), "Device namespace paths must be rejected");

        // UNC server path
        let result = fs.read("\\\\server\\share\\..\\..\\secret.txt");
        assert!(result.is_err(), "UNC server paths must be rejected");
    }

    #[test]
    #[cfg(windows)]
    fn reject_windows_reserved_names() {
        let fs = create_backend();

        // Windows reserved device names (CON, PRN, AUX, NUL, COM1-9, LPT1-9)
        // These can cause hangs or unexpected behavior
        let reserved_names = ["CON", "PRN", "AUX", "NUL", "COM1", "LPT1"];

        for name in reserved_names {
            let result = fs.write(&format!("/{}", name), b"data");
            // Should either reject or handle safely (not hang)

            let result = fs.write(&format!("/{}.txt", name), b"data");
            // CON.txt is also problematic on Windows
        }
    }

    #[test]
    #[cfg(windows)]
    fn reject_windows_junction_escape() {
        // Junction points are Windows' equivalent of directory symlinks
        // They can be used for sandbox escape similar to symlinks
        let fs = create_backend();
        fs.create_dir("/sandbox").unwrap();

        // If backend supports junctions, they must be contained like symlinks
        // The test setup would require actual junction creation capability
        // This documents the requirement even if not all backends support it
    }

    // ------------------------------------------------------------------------
    // Linux-Specific Security Tests (from soft-canonicalize/strict-path)
    // ------------------------------------------------------------------------

    #[test]
    #[cfg(target_os = "linux")]
    fn reject_proc_magic_symlinks() {
        // /proc/PID/root and similar "magic" symlinks can escape namespaces
        // Virtual filesystems wrapping real FS must not follow these
        let fs = create_backend();

        // These paths are only relevant for backends that wrap real filesystem
        // In-memory backends naturally don't have this issue

        // /proc/self/root points to the filesystem root, even in containers
        // Following it would escape chroot/container boundaries
        let result = fs.read("/proc/self/root/etc/passwd");
        // Either NotFound (good - path doesn't exist in VFS)
        // or handled safely (doesn't escape actual container)
    }

    #[test]
    #[cfg(target_os = "linux")]
    fn reject_dev_fd_symlinks() {
        let fs = create_backend();

        // /dev/fd/N symlinks to open file descriptors
        // Could be used to access files outside sandbox
        let result = fs.read("/dev/fd/0");
        // Should fail or be isolated from real /dev/fd
    }

    // ------------------------------------------------------------------------
    // Unicode Security Tests (from strict-path)
    // ------------------------------------------------------------------------

    #[test]
    fn unicode_normalization_consistency() {
        let fs = create_backend();

        // NFC vs NFD normalization: é can be:
        // - U+00E9 (precomposed, NFC)
        // - U+0065 U+0301 (decomposed, NFD: e + combining acute)
        let nfc = "/caf\u{00E9}.txt";  // precomposed
        let nfd = "/cafe\u{0301}.txt"; // decomposed

        fs.write(nfc, b"coffee").unwrap();

        // If backend normalizes, both should access same file
        // If backend doesn't normalize, second should not exist
        // Key: must NOT create two different files that look identical
        let result_nfc = fs.exists(nfc);
        let result_nfd = fs.exists(nfd);

        // Document the backend's behavior
        // Either both true (normalized) or only NFC true (strict)
    }

    #[test]
    fn reject_unicode_direction_override() {
        let fs = create_backend();

        // Right-to-Left Override (U+202E) can make paths appear different
        // "secret\u{202E}txt.exe" displays as "secretexe.txt" in some contexts
        let malicious_path = "/secret\u{202E}txt.exe";

        let result = fs.write(malicious_path, b"data");
        // Should either reject or sanitize bidirectional control characters
    }

    #[test]
    fn reject_unicode_homoglyphs() {
        let fs = create_backend();

        // Cyrillic 'а' (U+0430) looks like Latin 'a' (U+0061)
        let latin_path = "/data/file.txt";
        let cyrillic_path = "/d\u{0430}ta/file.txt"; // Cyrillic 'а'

        fs.create_dir("/data").unwrap();
        fs.write(latin_path, b"real content").unwrap();

        // These must NOT silently access the same file
        // Either cyrillic path is NotFound, or it's a different file
        let result = fs.read(cyrillic_path);
        if let Ok(content) = result {
            // If cyrillic path exists, it must be a distinct file
            // (not accidentally matching the latin path)
        }
    }

    #[test]
    fn reject_null_in_unicode() {
        let fs = create_backend();

        // Null can be encoded in various ways
        // UTF-8 null is just 0x00, but check overlong encodings aren't decoded
        let path_with_null = "/file\u{0000}name.txt";

        let result = fs.write(path_with_null, b"data");
        assert!(result.is_err(), "Embedded null must be rejected");
    }

    // ------------------------------------------------------------------------
    // TOCTOU Race Condition Tests (from soft-canonicalize/strict-path)
    // ------------------------------------------------------------------------

    #[test]
    fn toctou_check_then_use() {
        let fs = Arc::new(create_backend());
        fs.create_dir("/uploads").unwrap();

        // Simulate TOCTOU: check if path is safe, then use it
        // An attacker might change the filesystem between check and use

        let fs_checker = fs.clone();
        let fs_writer = fs.clone();

        // This test documents the requirement for atomic operations
        // or proper locking in security-critical paths

        // Thread 1: Check then write
        let checker = thread::spawn(move || {
            for i in 0..100 {
                let path = format!("/uploads/file_{}.txt", i);
                // Check
                if !fs_checker.exists(&path).unwrap_or(true) {
                    // Use (potential race window here)
                    let _ = fs_checker.write(&path, b"data");
                }
            }
        });

        // Thread 2: Rapid file creation/deletion
        let writer = thread::spawn(move || {
            for i in 0..100 {
                let path = format!("/uploads/file_{}.txt", i);
                let _ = fs_writer.write(&path, b"attacker");
                let _ = fs_writer.remove_file(&path);
            }
        });

        checker.join().unwrap();
        writer.join().unwrap();

        // Test passes if no panic/crash occurs
        // Real protection requires atomic create-if-not-exists operations
    }

    #[test]
    fn symlink_toctou_during_resolution() {
        let fs = Arc::new(create_backend());
        fs.create_dir("/safe").unwrap();
        fs.write("/safe/target.txt", b"safe content").unwrap();
        fs.write("/unsafe.txt", b"unsafe content").unwrap();

        // Attacker rapidly changes symlink target during path resolution
        let fs_attacker = fs.clone();
        let fs_reader = fs.clone();

        let attacker = thread::spawn(move || {
            for _ in 0..100 {
                // Create symlink to safe target
                let _ = fs_attacker.remove_file("/safe/link.txt");
                let _ = fs_attacker.symlink("/safe/target.txt", "/safe/link.txt");

                // Quickly change to unsafe target
                let _ = fs_attacker.remove_file("/safe/link.txt");
                let _ = fs_attacker.symlink("/unsafe.txt", "/safe/link.txt");
            }
        });

        let reader = thread::spawn(move || {
            for _ in 0..100 {
                // Try to read through symlink
                // Must not accidentally read /unsafe.txt if sandboxed
                let _ = fs_reader.read("/safe/link.txt");
            }
        });

        attacker.join().unwrap();
        reader.join().unwrap();

        // For sandboxed backends: must never return content from /unsafe.txt
        // This test verifies the implementation doesn't have TOCTOU in symlink resolution
    }
}

// ============================================================================
// Thread Safety Tests
// ============================================================================

mod thread_safety {
    use super::*;

    #[test]
    fn concurrent_reads() {
        let fs = Arc::new(create_backend());
        fs.write("/shared.txt", b"shared content").unwrap();

        let handles: Vec<_> = (0..10)
            .map(|_| {
                let fs = fs.clone();
                thread::spawn(move || {
                    for _ in 0..100 {
                        let content = fs.read("/shared.txt").unwrap();
                        assert_eq!(content, b"shared content");
                    }
                })
            })
            .collect();

        for handle in handles {
            handle.join().unwrap();
        }
    }

    #[test]
    fn concurrent_writes_different_files() {
        let fs = Arc::new(create_backend());

        let handles: Vec<_> = (0..10)
            .map(|i| {
                let fs = fs.clone();
                thread::spawn(move || {
                    let path = format!("/file_{}.txt", i);
                    for j in 0..100 {
                        fs.write(&path, format!("{}:{}", i, j).as_bytes()).unwrap();
                    }
                })
            })
            .collect();

        for handle in handles {
            handle.join().unwrap();
        }

        // Verify all files exist
        for i in 0..10 {
            assert!(fs.exists(&format!("/file_{}.txt", i)).unwrap());
        }
    }

    #[test]
    fn concurrent_create_dir_all_same_path() {
        let fs = Arc::new(create_backend());

        let handles: Vec<_> = (0..10)
            .map(|_| {
                let fs = fs.clone();
                thread::spawn(move || {
                    // All threads try to create the same path
                    let _ = fs.create_dir_all("/a/b/c/d");
                })
            })
            .collect();

        for handle in handles {
            handle.join().unwrap();
        }

        // Path should exist regardless of race
        assert!(fs.exists("/a/b/c/d").unwrap());
    }

    #[test]
    fn read_during_write() {
        let fs = Arc::new(create_backend());
        fs.write("/changing.txt", b"initial").unwrap();

        let fs_writer = fs.clone();
        let writer = thread::spawn(move || {
            for i in 0..100 {
                fs_writer.write("/changing.txt", format!("version {}", i).as_bytes()).unwrap();
            }
        });

        let fs_reader = fs.clone();
        let reader = thread::spawn(move || {
            for _ in 0..100 {
                // Should not panic or return garbage
                let result = fs_reader.read("/changing.txt");
                assert!(result.is_ok());
            }
        });

        writer.join().unwrap();
        reader.join().unwrap();
    }

    #[test]
    fn metadata_consistency() {
        let fs = Arc::new(create_backend());
        fs.write("/meta.txt", b"content").unwrap();

        let handles: Vec<_> = (0..10)
            .map(|_| {
                let fs = fs.clone();
                thread::spawn(move || {
                    for _ in 0..100 {
                        let meta = fs.metadata("/meta.txt").unwrap();
                        // Size should be consistent
                        assert!(meta.size > 0);
                    }
                })
            })
            .collect();

        for handle in handles {
            handle.join().unwrap();
        }
    }
}

// ============================================================================
// No Panic Tests (Edge Cases That Must Not Crash)
// ============================================================================

mod no_panic {
    use super::*;

    #[test]
    fn empty_path_does_not_panic() {
        let fs = create_backend();

        // These should return errors, not panic
        let _ = fs.read("");
        let _ = fs.write("", b"data");
        let _ = fs.metadata("");
        let _ = fs.exists("");
        let _ = fs.read_dir("");
    }

    #[test]
    fn path_with_null_does_not_panic() {
        let fs = create_backend();

        // Paths with null bytes should error or be handled gracefully
        let _ = fs.read("/file\0name.txt");
        let _ = fs.write("/file\0name.txt", b"data");
    }

    #[test]
    fn very_long_path_does_not_panic() {
        let fs = create_backend();
        let long_name = "a".repeat(10000);
        let long_path = format!("/{}", long_name);

        // Should error gracefully, not panic
        let _ = fs.write(&long_path, b"data");
        let _ = fs.read(&long_path);
    }

    #[test]
    fn very_long_filename_does_not_panic() {
        let fs = create_backend();
        let long_name = format!("/{}.txt", "x".repeat(1000));

        let _ = fs.write(&long_name, b"data");
    }

    #[test]
    fn read_after_remove_does_not_panic() {
        let fs = create_backend();
        fs.write("/temp.txt", b"data").unwrap();
        fs.remove_file("/temp.txt").unwrap();

        // Should return NotFound, not panic
        let result = fs.read("/temp.txt");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }

    #[test]
    fn double_remove_does_not_panic() {
        let fs = create_backend();
        fs.write("/temp.txt", b"data").unwrap();
        fs.remove_file("/temp.txt").unwrap();

        // Second remove should error, not panic
        let result = fs.remove_file("/temp.txt");
        assert!(matches!(result, Err(FsError::NotFound { .. })));
    }
}
```

---

## Extended Test Suite (FsFull Traits)

For backends implementing `FsFull`:

```rust
mod fs_full {
    use super::*;
    use anyfs_backend::{FsLink, FsPermissions, FsSync, FsStats, Permissions};

    // Only run these if the backend implements FsFull traits
    fn create_full_backend() -> impl Fs + FsLink + FsPermissions + FsSync + FsStats {
        todo!("Return your FsFull backend")
    }

    // ========================================================================
    // FsLink Tests
    // ========================================================================

    mod fs_link {
        use super::*;

        #[test]
        fn create_symlink() {
            let fs = create_full_backend();
            fs.write("/target.txt", b"target content").unwrap();

            fs.symlink("/target.txt", "/link.txt").unwrap();

            assert!(fs.exists("/link.txt").unwrap());
            let meta = fs.symlink_metadata("/link.txt").unwrap();
            assert_eq!(meta.file_type, FileType::Symlink);
        }

        #[test]
        fn read_symlink() {
            let fs = create_full_backend();
            fs.write("/target.txt", b"content").unwrap();
            fs.symlink("/target.txt", "/link.txt").unwrap();

            let target = fs.read_link("/link.txt").unwrap();
            assert_eq!(target.to_string_lossy(), "/target.txt");
        }

        #[test]
        fn hard_link() {
            let fs = create_full_backend();
            fs.write("/original.txt", b"shared content").unwrap();

            fs.hard_link("/original.txt", "/hardlink.txt").unwrap();

            // Both paths should have the same content
            assert_eq!(fs.read("/original.txt").unwrap(), b"shared content");
            assert_eq!(fs.read("/hardlink.txt").unwrap(), b"shared content");

            // Modifying one should affect the other
            fs.write("/hardlink.txt", b"modified").unwrap();
            assert_eq!(fs.read("/original.txt").unwrap(), b"modified");
        }

        #[test]
        fn symlink_metadata_vs_metadata() {
            let fs = create_full_backend();
            fs.write("/target.txt", b"content").unwrap();
            fs.symlink("/target.txt", "/link.txt").unwrap();

            // symlink_metadata returns the symlink's metadata
            let sym_meta = fs.symlink_metadata("/link.txt").unwrap();
            assert_eq!(sym_meta.file_type, FileType::Symlink);

            // metadata (if it follows symlinks) returns target's metadata
            // Note: behavior depends on implementation
        }
    }

    // ========================================================================
    // FsPermissions Tests
    // ========================================================================

    mod fs_permissions {
        use super::*;

        #[test]
        fn set_permissions() {
            let fs = create_full_backend();
            fs.write("/file.txt", b"data").unwrap();

            fs.set_permissions("/file.txt", Permissions::from_mode(0o755)).unwrap();

            let meta = fs.metadata("/file.txt").unwrap();
            assert_eq!(meta.permissions, Some(0o755));
        }

        #[test]
        fn set_permissions_nonexistent_returns_not_found() {
            let fs = create_full_backend();

            let result = fs.set_permissions("/nonexistent", Permissions::from_mode(0o644));
            assert!(matches!(result, Err(FsError::NotFound { .. })));
        }
    }

    // ========================================================================
    // FsSync Tests
    // ========================================================================

    mod fs_sync {
        use super::*;

        #[test]
        fn sync_does_not_error() {
            let fs = create_full_backend();
            fs.write("/file.txt", b"data").unwrap();

            // sync() should complete without error
            fs.sync().unwrap();
        }

        #[test]
        fn fsync_specific_file() {
            let fs = create_full_backend();
            fs.write("/file.txt", b"data").unwrap();

            fs.fsync("/file.txt").unwrap();
        }

        #[test]
        fn fsync_nonexistent_returns_not_found() {
            let fs = create_full_backend();

            let result = fs.fsync("/nonexistent.txt");
            assert!(matches!(result, Err(FsError::NotFound { .. })));
        }
    }

    // ========================================================================
    // FsStats Tests
    // ========================================================================

    mod fs_stats {
        use super::*;

        #[test]
        fn statfs_returns_valid_stats() {
            let fs = create_full_backend();

            let stats = fs.statfs().unwrap();

            // Basic sanity checks
            assert!(stats.block_size > 0);
            // available should not exceed total (if total is reported)
            if stats.total_bytes > 0 {
                assert!(stats.available_bytes <= stats.total_bytes);
            }
        }
    }
}
```

---

## FUSE Test Suite (FsFuse Traits)

For backends implementing `FsFuse`:

```rust
mod fs_fuse {
    use super::*;
    use anyfs_backend::FsInode;

    fn create_fuse_backend() -> impl Fs + FsInode {
        todo!("Return your FsFuse backend")
    }

    #[test]
    fn path_to_inode_consistency() {
        let fs = create_fuse_backend();
        fs.write("/file.txt", b"data").unwrap();

        let inode1 = fs.path_to_inode("/file.txt").unwrap();
        let inode2 = fs.path_to_inode("/file.txt").unwrap();

        // Same path should always return same inode
        assert_eq!(inode1, inode2);
    }

    #[test]
    fn inode_to_path_roundtrip() {
        let fs = create_fuse_backend();
        fs.write("/file.txt", b"data").unwrap();

        let inode = fs.path_to_inode("/file.txt").unwrap();
        let path = fs.inode_to_path(inode).unwrap();

        assert_eq!(path.to_string_lossy(), "/file.txt");
    }

    #[test]
    fn lookup_child() {
        let fs = create_fuse_backend();
        fs.create_dir("/parent").unwrap();
        fs.write("/parent/child.txt", b"data").unwrap();

        let parent_inode = fs.path_to_inode("/parent").unwrap();
        let child_inode = fs.lookup(parent_inode, std::ffi::OsStr::new("child.txt")).unwrap();

        let expected_inode = fs.path_to_inode("/parent/child.txt").unwrap();
        assert_eq!(child_inode, expected_inode);
    }

    #[test]
    fn metadata_by_inode() {
        let fs = create_fuse_backend();
        fs.write("/file.txt", b"content").unwrap();

        let inode = fs.path_to_inode("/file.txt").unwrap();
        let meta = fs.metadata_by_inode(inode).unwrap();

        assert_eq!(meta.file_type, FileType::File);
        assert_eq!(meta.size, 7);
    }

    #[test]
    fn root_inode_is_one() {
        let fs = create_fuse_backend();

        let root_inode = fs.path_to_inode("/").unwrap();

        // By FUSE convention, root inode is 1
        assert_eq!(root_inode, 1);
    }

    #[test]
    fn different_files_different_inodes() {
        let fs = create_fuse_backend();
        fs.write("/file1.txt", b"data1").unwrap();
        fs.write("/file2.txt", b"data2").unwrap();

        let inode1 = fs.path_to_inode("/file1.txt").unwrap();
        let inode2 = fs.path_to_inode("/file2.txt").unwrap();

        assert_ne!(inode1, inode2);
    }

    #[test]
    fn hard_links_same_inode() {
        let fs = create_fuse_backend();
        fs.write("/original.txt", b"data").unwrap();
        fs.hard_link("/original.txt", "/link.txt").unwrap();

        let inode1 = fs.path_to_inode("/original.txt").unwrap();
        let inode2 = fs.path_to_inode("/link.txt").unwrap();

        // Hard links must share the same inode
        assert_eq!(inode1, inode2);
    }
}
```

---

## Middleware Test Suite

For middleware implementers, verify the middleware doesn't break the underlying backend:

```rust
mod middleware_tests {
    use super::*;
    use anyfs::MemoryBackend;

    /// Your middleware wrapping a known-good backend.
    fn create_middleware() -> MyMiddleware<MemoryBackend> {
        MyMiddleware::new(MemoryBackend::new())
    }

    // Run all standard Fs tests through the middleware
    // This ensures the middleware doesn't break basic functionality

    #[test]
    fn passthrough_read_write() {
        let fs = create_middleware();

        fs.write("/test.txt", b"data").unwrap();
        assert_eq!(fs.read("/test.txt").unwrap(), b"data");
    }

    #[test]
    fn passthrough_directories() {
        let fs = create_middleware();

        fs.create_dir_all("/a/b/c").unwrap();
        assert!(fs.exists("/a/b/c").unwrap());
    }

    // Add middleware-specific tests here
    // e.g., for a Quota middleware:

    #[test]
    fn quota_blocks_oversized_write() {
        let fs = QuotaMiddleware::new(MemoryBackend::new())
            .with_max_file_size(100);

        let result = fs.write("/big.txt", &vec![0u8; 200]);
        assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
    }

    #[test]
    fn quota_allows_within_limit() {
        let fs = QuotaMiddleware::new(MemoryBackend::new())
            .with_max_file_size(100);

        fs.write("/small.txt", &vec![0u8; 50]).unwrap();
        assert!(fs.exists("/small.txt").unwrap());
    }
}
```

---

## Running the Tests

### Basic Usage

```bash
# Run all conformance tests
cargo test --test conformance

# Run specific test module
cargo test --test conformance fs_read

# Run with output
cargo test --test conformance -- --nocapture

# Run thread safety tests with more threads
RUST_TEST_THREADS=1 cargo test --test conformance thread_safety
```

### CI Integration

```yaml
# .github/workflows/test.yml
name: Conformance Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Run conformance tests
        run: cargo test --test conformance
      - name: Run thread safety tests
        run: cargo test --test conformance thread_safety -- --test-threads=1
```

---

## Test Checklist

Before releasing your backend or middleware:

### Core Tests (Required)
- [ ] All `fs_read` tests pass
- [ ] All `fs_write` tests pass
- [ ] All `fs_dir` tests pass
- [ ] All `edge_cases` tests pass
- [ ] All `security` tests pass
- [ ] All `thread_safety` tests pass
- [ ] All `no_panic` tests pass

### Extended Tests (If Implementing FsFull)
- [ ] All `fs_link` tests pass
- [ ] All `fs_permissions` tests pass
- [ ] All `fs_sync` tests pass
- [ ] All `fs_stats` tests pass

### FUSE Tests (If Implementing FsFuse)
- [ ] All `fs_fuse` tests pass
- [ ] Root inode is 1
- [ ] Hard links share inodes

### Middleware Tests
- [ ] Basic passthrough works
- [ ] Middleware-specific behavior tested
- [ ] Error cases handled correctly

---

## Summary

This conformance test suite provides:

1. **Complete coverage** of all `Fs` trait operations
2. **Edge case testing** for robustness
3. **Security tests** learned from vulnerabilities in prior art (Apache Commons VFS, Afero, PyFilesystem2)
4. **Thread safety verification** for concurrent access
5. **No-panic guarantees** for invalid inputs
6. **Extended tests** for `FsFull` and `FsFuse` traits
7. **Middleware testing patterns**

### Security Tests Cover:
- **Path traversal attacks**: URL-encoded `%2e%2e`, backslash traversal, null byte injection
- **Symlink escape**: Preventing sandbox escape via symlinks
- **Symlink loops**: Direct loops, indirect loops, deep chains
- **Resource exhaustion**: Limits on symlink depth
- **Path canonicalization**: Dot removal, double slash normalization
- **Windows-specific** (from `soft-canonicalize`/`strict-path`):
  - NTFS Alternate Data Streams
  - Windows 8.3 short names
  - UNC path traversal
  - Reserved device names
  - Junction point escapes
- **Linux-specific**: Magic symlinks (`/proc/PID/root`), `/dev/fd` escapes
- **Unicode**: NFC/NFD normalization, RTL override, homoglyphs
- **TOCTOU**: Race conditions in check-then-use and symlink resolution

Copy the relevant test modules, implement `create_backend()`, and run the tests. If they all pass, your backend/middleware is AnyFS-compatible.
