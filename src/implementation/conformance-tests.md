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
3. **Thread safety verification** for concurrent access
4. **No-panic guarantees** for invalid inputs
5. **Extended tests** for `FsFull` and `FsFuse` traits
6. **Middleware testing patterns**

Copy the relevant test modules, implement `create_backend()`, and run the tests. If they all pass, your backend/middleware is AnyFS-compatible.
