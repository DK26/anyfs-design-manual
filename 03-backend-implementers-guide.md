# VFS Container — Backend Implementer's Guide

**Step-by-step guide for implementing a custom storage backend**

---

## Overview

This guide walks you through implementing a custom `StorageBackend`. By the end, you'll have a working backend that passes the conformance test suite.

**Time estimate:** 2-4 hours for a basic implementation

---

## What You're Building

A backend is a **typed graph store** with transactions. It stores:

- **Nodes**: Metadata about files, directories, and symlinks
- **Edges**: Named parent→child relationships
- **Chunks**: Binary content in fixed-size pieces

The backend does NOT handle:
- Path parsing or resolution
- Symlink following
- Permission checking
- Capacity enforcement

All of that lives in `FilesContainer`. Your backend just stores what it's told.

---

## Prerequisites

```toml
[dependencies]
anyfs = "0.1"
thiserror = "1"
```

---

## Step 1: Define Your Backend Struct

```rust
use anyfs::{
    StorageBackend, BackendLifecycle, Snapshot, Transaction,
    NodeId, ContentId, ChunkId, Name, Edge,
    NodeRecord, BackendError,
};

pub struct MyBackend {
    // Your storage state here
    // For example, for a Redis backend:
    // client: redis::Client,
    // connection: redis::Connection,
}

pub struct MyBackendConfig {
    // Configuration options
    // e.g., connection string, file path, etc.
}
```

---

## Step 2: Implement BackendLifecycle

This trait handles creation, opening, and destruction:

```rust
impl BackendLifecycle for MyBackend {
    type Config = MyBackendConfig;
    
    fn create(config: Self::Config) -> Result<Self, BackendError> {
        // Create new storage
        // Fail if storage already exists
        
        // Initialize with root node (ID = 1, Directory)
        let mut backend = Self::initialize(config)?;
        backend.create_root_node()?;
        Ok(backend)
    }
    
    fn open(config: Self::Config) -> Result<Self, BackendError> {
        // Open existing storage
        // Fail if storage doesn't exist
        
        Self::connect(config)
    }
    
    fn open_or_create(config: Self::Config) -> Result<Self, BackendError> {
        match Self::open(config.clone()) {
            Ok(backend) => Ok(backend),
            Err(BackendError::NotFound(_)) => Self::create(config),
            Err(e) => Err(e),
        }
    }
    
    fn destroy(self) -> Result<(), BackendError> {
        // Delete all data
        // For file-based: delete the file
        // For database: drop tables or delete database
        
        self.delete_all_data()
    }
}
```

---

## Step 3: Implement Snapshot (Read Operations)

`Snapshot` provides read-only access:

```rust
pub struct MySnapshot<'a> {
    backend: &'a MyBackend,
}

impl<'a> Snapshot for MySnapshot<'a> {
    fn get_node(&self, id: NodeId) -> Result<Option<NodeRecord>, BackendError> {
        // Retrieve node by ID
        // Return None if not found (don't error)
        
        self.backend.fetch_node(id)
    }
    
    fn get_edge(&self, parent: NodeId, name: &Name) -> Result<Option<NodeId>, BackendError> {
        // Look up child ID by parent + name
        // Return None if edge doesn't exist
        
        self.backend.fetch_edge(parent, name.as_str())
    }
    
    fn list_edges(&self, parent: NodeId) -> Result<Vec<Edge>, BackendError> {
        // Return all edges where edge.parent == parent
        // Empty vec if directory is empty (not an error)
        
        self.backend.fetch_children(parent)
    }
    
    fn read_chunk(&self, id: ChunkId) -> Result<Option<Vec<u8>>, BackendError> {
        // Retrieve chunk data
        // Return None if chunk doesn't exist
        
        self.backend.fetch_chunk(id)
    }
    
    // Optional: batch read (default implementation loops)
    fn get_nodes(&self, ids: &[NodeId]) -> Result<Vec<Option<NodeRecord>>, BackendError> {
        // Override for backends that can batch efficiently
        // Default implementation:
        ids.iter().map(|id| self.get_node(*id)).collect()
    }
}
```

---

## Step 4: Implement Transaction (Write Operations)

`Transaction` extends `Snapshot` with write operations:

```rust
pub struct MyTransaction<'a> {
    backend: &'a mut MyBackend,
    // Track changes for rollback if needed
    // changes: Vec<Change>,
}

impl<'a> Snapshot for MyTransaction<'a> {
    // Delegate read operations to snapshot implementation
    fn get_node(&self, id: NodeId) -> Result<Option<NodeRecord>, BackendError> {
        self.backend.fetch_node(id)
    }
    
    fn get_edge(&self, parent: NodeId, name: &Name) -> Result<Option<NodeId>, BackendError> {
        self.backend.fetch_edge(parent, name.as_str())
    }
    
    fn list_edges(&self, parent: NodeId) -> Result<Vec<Edge>, BackendError> {
        self.backend.fetch_children(parent)
    }
    
    fn read_chunk(&self, id: ChunkId) -> Result<Option<Vec<u8>>, BackendError> {
        self.backend.fetch_chunk(id)
    }
}

impl<'a> Transaction for MyTransaction<'a> {
    fn insert_node(&mut self, node: &NodeRecord) -> Result<(), BackendError> {
        // Insert new node
        // Error if node with this ID already exists
        
        if self.backend.node_exists(node.id)? {
            return Err(BackendError::NodeAlreadyExists(node.id));
        }
        self.backend.store_node(node)
    }
    
    fn update_node(&mut self, id: NodeId, node: &NodeRecord) -> Result<(), BackendError> {
        // Update existing node
        // Error if node doesn't exist
        
        if !self.backend.node_exists(id)? {
            return Err(BackendError::NodeNotFound(id));
        }
        self.backend.store_node(node)
    }
    
    fn delete_node(&mut self, id: NodeId) -> Result<(), BackendError> {
        // Delete node
        // Error if node doesn't exist
        
        if !self.backend.node_exists(id)? {
            return Err(BackendError::NodeNotFound(id));
        }
        self.backend.remove_node(id)
    }
    
    fn insert_edge(&mut self, edge: &Edge) -> Result<(), BackendError> {
        // Insert new edge
        // Error if edge already exists (same parent + name)
        
        if self.backend.edge_exists(edge.parent, &edge.name)? {
            return Err(BackendError::EdgeAlreadyExists {
                parent: edge.parent,
                name: edge.name.as_str().to_string(),
            });
        }
        self.backend.store_edge(edge)
    }
    
    fn delete_edge(&mut self, parent: NodeId, name: &Name) -> Result<(), BackendError> {
        // Delete edge
        // Error if edge doesn't exist
        
        if !self.backend.edge_exists(parent, name)? {
            return Err(BackendError::EdgeNotFound {
                parent,
                name: name.as_str().to_string(),
            });
        }
        self.backend.remove_edge(parent, name)
    }
    
    fn write_chunk(&mut self, id: ChunkId, data: &[u8]) -> Result<(), BackendError> {
        // Write chunk data (insert or replace)
        
        self.backend.store_chunk(id, data)
    }
    
    fn delete_content(&mut self, id: ContentId) -> Result<(), BackendError> {
        // Delete all chunks for this content ID
        
        self.backend.remove_all_chunks(id)
    }
    
    fn next_node_id(&mut self) -> Result<NodeId, BackendError> {
        // Generate a new unique node ID
        // Must never return the same ID twice
        
        self.backend.allocate_node_id()
    }
    
    fn next_content_id(&mut self) -> Result<ContentId, BackendError> {
        // Generate a new unique content ID
        
        self.backend.allocate_content_id()
    }
    
    // Optional: batch operations (default implementations loop)
    fn insert_nodes(&mut self, nodes: &[NodeRecord]) -> Result<(), BackendError> {
        for node in nodes {
            self.insert_node(node)?;
        }
        Ok(())
    }
    
    fn insert_edges(&mut self, edges: &[Edge]) -> Result<(), BackendError> {
        for edge in edges {
            self.insert_edge(edge)?;
        }
        Ok(())
    }
}
```

---

## Step 5: Implement StorageBackend

The main trait ties it all together:

```rust
impl StorageBackend for MyBackend {
    fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
    where
        F: FnOnce(&mut dyn Transaction) -> Result<T, BackendError>,
    {
        // Begin transaction
        self.begin_transaction()?;
        
        let mut txn = MyTransaction { backend: self };
        
        match f(&mut txn) {
            Ok(result) => {
                // Commit on success
                self.commit_transaction()?;
                Ok(result)
            }
            Err(e) => {
                // Rollback on error
                self.rollback_transaction()?;
                Err(e)
            }
        }
    }
    
    fn snapshot(&self) -> Box<dyn Snapshot + '_> {
        Box::new(MySnapshot { backend: self })
    }
}
```

---

## Step 6: Initialize Root Node

Every container needs a root directory (NodeId = 1):

```rust
impl MyBackend {
    fn create_root_node(&mut self) -> Result<(), BackendError> {
        use anyfs::{NodeKind, NodeMetadata, Timestamp};
        
        let root = NodeRecord {
            id: NodeId(1),
            kind: NodeKind::Directory,
            metadata: NodeMetadata {
                created_at: Some(Timestamp::now()),
                modified_at: Some(Timestamp::now()),
                accessed_at: None,
                permissions: None,
                link_count: 1,
                extended_attrs: Default::default(),
            },
        };
        
        self.store_node(&root)?;
        
        // Set next_node_id to 2
        self.set_next_node_id(2)?;
        self.set_next_content_id(1)?;
        
        Ok(())
    }
}
```

---

## Step 7: Run Conformance Tests

Use the provided test macro to verify your implementation:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    fn create_test_backend() -> MyBackend {
        MyBackend::create(MyBackendConfig::default()).unwrap()
    }
    
    // This generates ~50 tests
    anyfs::backend_conformance_tests!(create_test_backend);
    
    // Add your own backend-specific tests
    #[test]
    fn test_my_backend_specific_feature() {
        let backend = create_test_backend();
        // ...
    }
}
```

---

## Checklist

Before considering your backend complete:

- [ ] `BackendLifecycle::create` initializes with root node
- [ ] `BackendLifecycle::open` fails gracefully if storage doesn't exist
- [ ] `BackendLifecycle::destroy` cleans up all resources
- [ ] `Snapshot::get_node` returns `None` for missing nodes (not error)
- [ ] `Snapshot::get_edge` returns `None` for missing edges (not error)
- [ ] `Snapshot::list_edges` returns empty vec for empty directories
- [ ] `Snapshot::read_chunk` returns `None` for missing chunks
- [ ] `Transaction::insert_node` errors on duplicate ID
- [ ] `Transaction::update_node` errors on missing node
- [ ] `Transaction::delete_node` errors on missing node
- [ ] `Transaction::insert_edge` errors on duplicate parent+name
- [ ] `Transaction::delete_edge` errors on missing edge
- [ ] `Transaction::next_node_id` never returns duplicates
- [ ] `Transaction::next_content_id` never returns duplicates
- [ ] `StorageBackend::transact` rolls back on error
- [ ] All conformance tests pass

---

## Common Pitfalls

### 1. Forgetting the root node

The container expects NodeId(1) to exist and be a directory. Always create it in `BackendLifecycle::create`.

### 2. Returning errors instead of None

`get_node`, `get_edge`, and `read_chunk` should return `Ok(None)` for missing items, not `Err(NotFound)`.

### 3. Not handling transactions properly

If your storage doesn't support transactions natively, you need to implement them. Options:
- Copy-on-write (clone state, apply changes, swap on commit)
- Write-ahead log
- Mutex + rollback journal

### 4. ID collisions

`next_node_id` and `next_content_id` must be monotonically increasing or otherwise guaranteed unique. Store the counter persistently.

### 5. Chunk size assumptions

The core uses a fixed chunk size (`anyfs::CHUNK_SIZE = 64KB`). Your backend doesn't need to enforce this, but it should handle any chunk size ≤ CHUNK_SIZE.

---

## Example: Minimal In-Memory Backend

Here's a complete minimal implementation for reference:

```rust
use std::collections::HashMap;
use anyfs::*;

pub struct MemoryBackend {
    nodes: HashMap<NodeId, NodeRecord>,
    edges: HashMap<(NodeId, String), NodeId>,
    chunks: HashMap<(ContentId, u32), Vec<u8>>,
    next_node_id: u64,
    next_content_id: u64,
}

impl MemoryBackend {
    pub fn new() -> Self {
        let mut backend = Self {
            nodes: HashMap::new(),
            edges: HashMap::new(),
            chunks: HashMap::new(),
            next_node_id: 2,
            next_content_id: 1,
        };
        
        // Create root node
        backend.nodes.insert(
            NodeId(1),
            NodeRecord {
                id: NodeId(1),
                kind: NodeKind::Directory,
                metadata: NodeMetadata::default(),
            },
        );
        
        backend
    }
}

impl StorageBackend for MemoryBackend {
    fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
    where
        F: FnOnce(&mut dyn Transaction) -> Result<T, BackendError>,
    {
        // Simple approach: clone state, rollback on error
        let snapshot = (
            self.nodes.clone(),
            self.edges.clone(),
            self.chunks.clone(),
            self.next_node_id,
            self.next_content_id,
        );
        
        let mut txn = MemoryTransaction { backend: self };
        
        match f(&mut txn) {
            Ok(result) => Ok(result),
            Err(e) => {
                // Rollback
                self.nodes = snapshot.0;
                self.edges = snapshot.1;
                self.chunks = snapshot.2;
                self.next_node_id = snapshot.3;
                self.next_content_id = snapshot.4;
                Err(e)
            }
        }
    }
    
    fn snapshot(&self) -> Box<dyn Snapshot + '_> {
        Box::new(MemorySnapshot { backend: self })
    }
}

struct MemorySnapshot<'a> {
    backend: &'a MemoryBackend,
}

impl Snapshot for MemorySnapshot<'_> {
    fn get_node(&self, id: NodeId) -> Result<Option<NodeRecord>, BackendError> {
        Ok(self.backend.nodes.get(&id).cloned())
    }
    
    fn get_edge(&self, parent: NodeId, name: &Name) -> Result<Option<NodeId>, BackendError> {
        Ok(self.backend.edges.get(&(parent, name.as_str().to_string())).copied())
    }
    
    fn list_edges(&self, parent: NodeId) -> Result<Vec<Edge>, BackendError> {
        Ok(self.backend.edges
            .iter()
            .filter(|((p, _), _)| *p == parent)
            .map(|((p, n), c)| Edge {
                parent: *p,
                name: Name::new(n).unwrap(),
                child: *c,
            })
            .collect())
    }
    
    fn read_chunk(&self, id: ChunkId) -> Result<Option<Vec<u8>>, BackendError> {
        Ok(self.backend.chunks.get(&(id.content, id.index)).cloned())
    }
}

struct MemoryTransaction<'a> {
    backend: &'a mut MemoryBackend,
}

impl Snapshot for MemoryTransaction<'_> {
    // ... delegate to backend (same as MemorySnapshot)
}

impl Transaction for MemoryTransaction<'_> {
    fn insert_node(&mut self, node: &NodeRecord) -> Result<(), BackendError> {
        if self.backend.nodes.contains_key(&node.id) {
            return Err(BackendError::NodeAlreadyExists(node.id));
        }
        self.backend.nodes.insert(node.id, node.clone());
        Ok(())
    }
    
    fn update_node(&mut self, id: NodeId, node: &NodeRecord) -> Result<(), BackendError> {
        if !self.backend.nodes.contains_key(&id) {
            return Err(BackendError::NodeNotFound(id));
        }
        self.backend.nodes.insert(id, node.clone());
        Ok(())
    }
    
    fn delete_node(&mut self, id: NodeId) -> Result<(), BackendError> {
        self.backend.nodes.remove(&id)
            .ok_or(BackendError::NodeNotFound(id))?;
        Ok(())
    }
    
    fn insert_edge(&mut self, edge: &Edge) -> Result<(), BackendError> {
        let key = (edge.parent, edge.name.as_str().to_string());
        if self.backend.edges.contains_key(&key) {
            return Err(BackendError::EdgeAlreadyExists {
                parent: edge.parent,
                name: edge.name.as_str().to_string(),
            });
        }
        self.backend.edges.insert(key, edge.child);
        Ok(())
    }
    
    fn delete_edge(&mut self, parent: NodeId, name: &Name) -> Result<(), BackendError> {
        let key = (parent, name.as_str().to_string());
        self.backend.edges.remove(&key)
            .ok_or(BackendError::EdgeNotFound {
                parent,
                name: name.as_str().to_string(),
            })?;
        Ok(())
    }
    
    fn write_chunk(&mut self, id: ChunkId, data: &[u8]) -> Result<(), BackendError> {
        self.backend.chunks.insert((id.content, id.index), data.to_vec());
        Ok(())
    }
    
    fn delete_content(&mut self, id: ContentId) -> Result<(), BackendError> {
        self.backend.chunks.retain(|(c, _), _| *c != id);
        Ok(())
    }
    
    fn next_node_id(&mut self) -> Result<NodeId, BackendError> {
        let id = NodeId(self.backend.next_node_id);
        self.backend.next_node_id += 1;
        Ok(id)
    }
    
    fn next_content_id(&mut self) -> Result<ContentId, BackendError> {
        let id = ContentId(self.backend.next_content_id);
        self.backend.next_content_id += 1;
        Ok(id)
    }
}
```

---

## Need Help?

If you run into issues implementing a backend:

1. Check the conformance test output for specific failures
2. Compare against the `MemoryBackend` reference implementation
3. Review the full design document for edge cases

---

*For architecture details, see the [Design Document](./anyfs-container-design.md).*
