# Linux Filesystem & Git Concepts: Application to AnyFS

**Status:** Research Complete
**Last Updated:** 2026-02-24

This document analyzes concepts from Linux LVM, ZFS, XFS, and Git internals to identify patterns that could enhance AnyFS -- particularly around overcoming SQLite size limits via multi-file spanning, and advanced storage management features.

---

## Motivation

AnyFS uses SQLite as a primary backend for portable, single-file virtual filesystems. However, SQLite has practical limits:

- **Database size**: While the theoretical max is ~281 TB, practical performance degrades well before that (tens of GB+)
- **Single-writer constraint**: Only one writer at a time, even in WAL mode
- **BLOB overhead**: Large inline BLOBs cause WAL growth and page cache pressure

Linux storage subsystems have solved analogous problems for decades. This analysis maps their proven concepts to AnyFS's architecture.

---

## Table of Contents

- [Part 1: LVM (Logical Volume Manager)](#part-1-lvm-logical-volume-manager)
- [Part 2: ZFS (Zettabyte File System)](#part-2-zfs-zettabyte-file-system)
- [Part 3: XFS](#part-3-xfs)
- [Part 4: Git Internals](#part-4-git-internals)
- [Unified Impact Matrix](#unified-impact-matrix)
- [Recommended Priorities](#recommended-priorities)
- [Sources](#sources)

---

## Part 1: LVM (Logical Volume Manager)

LVM operates as a two-layer system: userspace tools (LVM2, source at [gitlab.com/lvmteam/lvm2](https://gitlab.com/lvmteam/lvm2)) manage metadata and orchestrate operations, while the kernel device-mapper (`drivers/md/` in Linux) provides block-level virtual device mapping via pluggable targets.

### 1.1 Physical Volumes as SQLite Files

**LVM concept**: A Physical Volume (PV) is a block device initialized for LVM use. Each PV has a label, metadata area, and data area divided into fixed-size Physical Extents (PEs). Crucially, each PV carries a *full redundant copy* of the entire Volume Group metadata.

**Source**: `lib/metadata/pv.h`, `lib/format_text/format-text.c`, `lib/label/label.c`

**AnyFS mapping**: Each SQLite database file acts as a Physical Volume. Just as a PV is self-contained with its own metadata header, a SQLite file is self-contained with its own schema.

```rust
/// A Physical Volume is a single SQLite database file
pub struct PhysicalVolume {
    uuid: Uuid,
    path: PathBuf,
    connection: rusqlite::Connection,
    total_bytes: u64,
    pe_size: u64,      // Physical Extent size (e.g., 1 MiB)
    pe_count: u32,
    allocated: BitVec,  // Bitmap of allocated extents
}
```

```sql
-- Schema inside each PV SQLite file
CREATE TABLE pv_header (
    uuid TEXT NOT NULL,
    pe_size INTEGER NOT NULL,
    pe_count INTEGER NOT NULL,
    vg_uuid TEXT,              -- NULL if not assigned to a VG
    created_at TEXT NOT NULL
);
CREATE TABLE extents (
    pe_index INTEGER PRIMARY KEY,
    le_index INTEGER,          -- NULL if free
    lv_uuid TEXT,              -- which LV owns this extent
    data BLOB                  -- the actual chunk data
);
CREATE TABLE vg_metadata (
    version INTEGER NOT NULL,
    metadata_json TEXT NOT NULL -- full VG layout (redundant across all PVs)
);
```

**Key insight**: Each PV SQLite file carries a full copy of the VG metadata, so any single PV can reconstruct the entire volume layout -- mirroring LVM's reliability model.

**Practical limits**: SQLite's `ATTACH` limit is 125 databases per connection (10 by default). This caps simultaneous PV access from a single connection.

### 1.2 Volume Groups: Aggregating SQLite Files

**LVM concept**: A Volume Group (VG) pools one or more PVs into a single storage namespace. It defines a uniform PE size and maintains a complete allocation map.

**Source**: `lib/metadata/vg.h`, `lib/metadata/metadata.c`, `tools/vgcreate.c`

**AnyFS mapping**: A VG coordinator aggregates multiple SQLite PV files into a unified pool.

```rust
/// Aggregates multiple SQLite PV files into a storage pool
pub struct VolumeGroup {
    uuid: Uuid,
    name: String,
    pe_size: u64,                    // Uniform across all PVs
    pvs: Vec<PhysicalVolume>,
    total_pe_count: u32,
    free_pe_count: u32,
    lvs: HashMap<Uuid, LogicalVolume>,
    seqno: u64,                      // Metadata version (incremented on every change)
}

impl VolumeGroup {
    /// Add a new SQLite PV file (analogous to `vgextend`)
    pub fn extend(&mut self, pv_path: &Path) -> Result<()> {
        let mut pv = PhysicalVolume::open(pv_path)?;
        pv.assign_to_vg(self.uuid, self.pe_size)?;
        self.free_pe_count += pv.pe_count;
        self.total_pe_count += pv.pe_count;
        self.pvs.push(pv);
        self.seqno += 1;
        self.write_metadata_to_all_pvs()?; // Redundancy
        Ok(())
    }

    /// Remove an empty PV (analogous to `vgreduce`)
    pub fn reduce(&mut self, pv_uuid: Uuid) -> Result<()> {
        // PV must have no allocated extents; pvmove first if needed
        // ...
    }
}
```

**Use cases**:
- Combine five 1 GB SQLite files into a 5 GB virtual filesystem
- Start with one PV, add more as needed without restructuring
- Different VGs for different environments (dev, staging, prod)

### 1.3 Logical Volumes: Virtual Filesystems from Pooled Storage

**LVM concept**: A Logical Volume (LV) is a virtual block device carved from a VG's extent pool. An LV is composed of segments, each mapping Logical Extents (LEs) to Physical Extents (PEs) on specific PVs.

**Source**: `lib/metadata/lv.h`, `lib/metadata/segtype.h`, `lib/metadata/lv_manip.c`

**AnyFS mapping**: An LV becomes an `Fs` trait implementor backed by extents across multiple SQLite PVs.

```rust
pub struct LogicalVolume {
    uuid: Uuid,
    name: String,
    vg: Arc<RwLock<VolumeGroup>>,
    segments: Vec<LvSegment>,
    size: u64,
}

pub struct LvSegment {
    le_start: u32,
    le_count: u32,
    mapping: SegmentMapping,
}

pub enum SegmentMapping {
    /// Contiguous mapping to a single PV
    Linear { pv_uuid: Uuid, pe_start: u32 },
    /// Striped across multiple PVs (RAID-0)
    Striped { stripe_size: u64, stripes: Vec<(Uuid, u32)> },
    /// Mirrored across multiple PVs (RAID-1)
    Mirror { mirrors: Vec<(Uuid, u32)> },
}
```

**Use cases**:
- Separate LVs for "documents", "media", "temp" -- each with different middleware stacks
- Grow an LV by allocating more extents without moving existing data
- Mix segment types: a "critical" LV uses mirrored segments, "temp" uses linear

### 1.4 Physical/Logical Extents

**LVM concept**: Fixed-size allocation units. PEs on PVs map 1:1 to LEs in LVs. PE size is set at VG creation (default 4 MiB).

**AnyFS mapping**: Extents become fixed-size BLOB rows in SQLite.

```sql
CREATE TABLE extents (
    pe_index    INTEGER PRIMARY KEY,
    lv_uuid     TEXT,            -- NULL = free
    le_index    INTEGER,
    data        BLOB NOT NULL,   -- exactly pe_size bytes
    checksum    INTEGER          -- CRC32 for integrity
);
CREATE INDEX idx_extents_free ON extents(lv_uuid) WHERE lv_uuid IS NULL;
```

**Optimal extent size for SQLite**: Unlike block devices, SQLite has per-row overhead. BLOBs >100 KiB use overflow pages. Practical PE size: **64 KiB to 1 MiB** (vs LVM's 4 MiB default). Use SQLite's incremental BLOB I/O (`sqlite3_blob_open`) for larger extents.

### 1.5 Striping (Parallel SQLite Writers)

**LVM concept**: RAID-0 spreads data across PVs in round-robin fashion for parallel I/O.

**Source**: Kernel `drivers/md/dm-stripe.c`

**AnyFS mapping**: Distribute writes across multiple SQLite connections (each PV has its own single-writer).

```rust
pub struct StripedMapping {
    stripe_size: u64,
    stripes: Vec<Arc<PhysicalVolume>>,
}

impl StripedMapping {
    fn write_striped(&self, offset: u64, data: &[u8]) -> Result<()> {
        // Partition data into per-PV chunks
        let mut pv_writes: HashMap<usize, Vec<(u64, &[u8])>> = HashMap::new();
        // ... distribute based on stripe_size ...

        // Execute in parallel (each PV has its own connection)
        std::thread::scope(|s| {
            for (pv_idx, writes) in &pv_writes {
                s.spawn(|| self.stripes[*pv_idx].batch_write(writes));
            }
        });
        Ok(())
    }
}
```

**Key insight**: SQLite's single-writer bottleneck is *per-file*. With 4 striped PVs, large writes distribute across 4 independent SQLite writers.

### 1.6 Mirroring (Redundant SQLite Files)

**LVM concept**: RAID-1 writes identical data to N PVs. Reads come from any mirror. Dirty region log enables fast crash recovery.

**Source**: Kernel `drivers/md/dm-raid1.c`

**AnyFS mapping**: Write extents to N SQLite PV files. Read from any (round-robin for load distribution).

**Use cases**:
- Critical tenant data mirrored across two disks
- Read scaling: 2-mirror setup doubles read throughput
- Live migration foundation: temporarily mirror, then remove old PV (this is how `pvmove` works)

### 1.7 Thin Provisioning (Overcommitted SQLite Pools)

**LVM concept**: Decouple advertised size from allocated size. Thin volumes collectively claim more space than physically exists. Space allocated on first write (lazy allocation).

**Source**: Kernel `drivers/md/dm-thin.c`, `drivers/md/dm-thin-metadata.c`

**AnyFS mapping**: Highly natural for SQLite -- files grow on demand. An empty database is a few KiB regardless of virtual capacity.

```rust
pub struct ThinProvisionedFs {
    uuid: Uuid,
    virtual_size: u64,              // Advertised: 100 GB
    mapping: BTreeMap<u32, u64>,    // Sparse: only populated extents exist
    pool: Arc<RwLock<ThinPool>>,
}

impl ThinProvisionedFs {
    fn total_space(&self) -> u64 { self.virtual_size }   // 100 GB
    fn used_space(&self) -> u64 { self.mapping.len() as u64 * EXTENT_SIZE }  // 2 GB
}
```

**Integration with existing middleware**:
```rust
let pool = ThinPool::new("pool.db", 50 * GB)?;
let thin_vol = pool.create_volume("tenant-a", 1 * TB)?;

// QuotaLayer prevents any single tenant from exhausting the shared pool
let backend = thin_vol.layer(QuotaLayer::builder()
    .max_total_size(10 * GB)
    .build());
```

**Use case**: 100 tenants, each advertised 10 GB, but only 200 GB actual storage. Most tenants use under 1 GB.

### 1.8 pvmove: Live Data Migration

**LVM concept**: Migrate data between PVs without downtime. Creates a temporary mirror, syncs, breaks mirror keeping only destination.

**Source**: `tools/pvmove.c`

**AnyFS mapping**: Move extents between SQLite PV files while the filesystem remains operational, with crash-safe checkpointing.

```rust
pub struct PvMoveOperation {
    source_pv: Arc<PhysicalVolume>,
    dest_pv: Arc<PhysicalVolume>,
    extent_list: Vec<u32>,
    checkpoint: u32,  // Persisted for crash recovery
}

impl PvMoveOperation {
    pub fn execute(&mut self) -> Result<()> {
        for &pe_idx in &self.extent_list[self.checkpoint as usize..] {
            let data = self.source_pv.read_extent(pe_idx)?;
            let dest_pe = self.dest_pv.allocate_extent()?;
            self.dest_pv.write_extent(dest_pe, &data)?;
            self.update_lv_mapping(pe_idx, &self.dest_pv, dest_pe)?;
            self.source_pv.free_extent(pe_idx)?;
            self.checkpoint += 1;
            self.save_checkpoint()?;
        }
        Ok(())
    }
}
```

**Use cases**:
- Migrate tenant from SSD-backed PV to HDD-backed PV without downtime
- Rebalance extents after adding a new PV
- Replace a degraded PV before it fails

### 1.9 LVM Cache (Tiered Storage)

**LVM concept**: Place a fast device (SSD) in front of a slow device (HDD). dm-cache uses SMQ policy (Stochastic Multi-Queue) for promotion/demotion.

**Source**: Kernel `drivers/md/dm-cache-target.c`, `drivers/md/dm-cache-policy-smq.c`

**AnyFS mapping**: MemoryBackend (or RAM-disk SQLite) caching in front of disk-backed SqliteBackend.

```rust
let slow_backend = SqliteBackend::open("archive.db")?;
let cached = slow_backend
    .layer(CacheLayer::builder()
        .cache_backend(MemoryBackend::new())
        .capacity(256 * MB)
        .policy(CachePolicy::WriteBack)
        .build());
let fs = FileStorage::new(cached);
```

---

## Part 2: ZFS (Zettabyte File System)

ZFS is an integrated volume manager and filesystem. Source at [github.com/openzfs/zfs](https://github.com/openzfs/zfs).

### 2.1 Storage Pools (zpools)

**ZFS concept**: A zpool aggregates physical devices (vdevs) into a single namespace. Self-describing via uberblocks and labels.

**Source**: `module/zfs/spa.c`, `module/zfs/vdev.c`, `module/zfs/vdev_label.c`

**AnyFS mapping**: Similar to LVM Volume Groups but with integrated filesystem semantics. A `BackendPool` aggregates multiple backend instances.

### 2.2 Datasets and zvols

**ZFS concept**: Filesystems within a pool, each with independent properties, quotas, and snapshot history. Hierarchical: `pool/parent/child`.

**Source**: `module/zfs/dsl_dataset.c`, `module/zfs/dsl_dir.c`

**AnyFS mapping**: Each `FileStorage<B>` instance is effectively a dataset. The hierarchy can be modeled in SQLite:

```sql
CREATE TABLE datasets (
    dataset_id    INTEGER PRIMARY KEY,
    name          TEXT NOT NULL UNIQUE,     -- 'pool/users/alice'
    parent_id     INTEGER REFERENCES datasets(dataset_id),
    backend_type  TEXT NOT NULL,
    quota         INTEGER,                  -- inherited from parent if NULL
    compression   TEXT DEFAULT 'none',
    encryption    TEXT DEFAULT 'none',
    created_at    INTEGER NOT NULL,
    referenced    INTEGER NOT NULL DEFAULT 0,
    used_by_self  INTEGER NOT NULL DEFAULT 0,
    used_by_snaps INTEGER NOT NULL DEFAULT 0
);
```

### 2.3 Copy-on-Write (COW)

**ZFS concept**: Never overwrites data in place. Every write allocates new blocks, updates block pointers bottom-up, commits atomically via transaction groups. The on-disk state is always a consistent tree.

**Source**: `module/zfs/dbuf.c`, `module/zfs/dmu.c`, `module/zfs/txg.c`

**AnyFS mapping**: Already present in IndexedBackend. Writing creates a new blob; old blob stays referenced by snapshots. The `blobs` table with refcounting is COW by design. The two-phase commit pattern (blob upload -> SQLite metadata commit) provides crash consistency.

### 2.4 Block-Level Checksumming (Merkle Trees)

**ZFS concept**: Every block has a checksum stored in its parent block pointer, forming a Merkle tree from uberblock to leaf data. Verified on every read. Self-healing from mirrors/parity on mismatch.

**Source**: `module/zfs/zio_checksum.c` (supports Fletcher-2/4, SHA-256, SHA-512, Skein, EDONR, BLAKE3)

**AnyFS mapping**: For IndexedBackend where `blob_id = sha256(content)`, the content hash already serves as a checksum. A verification middleware adds read-time checking:

```rust
pub struct ChecksumLayer<B> {
    inner: B,
    algorithm: ChecksumAlgorithm,
}

impl<B: FsRead> FsRead for ChecksumLayer<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let data = self.inner.read(path)?;
        let expected = self.get_stored_checksum(path)?;
        let actual = self.algorithm.compute(&data);
        if expected != actual {
            return Err(FsError::IntegrityError { path, expected, actual });
        }
        Ok(data)
    }
}
```

**Use case**: Verify data integrity on untrusted storage backends (S3, remote blob stores).

### 2.5 Snapshots and Clones

**ZFS concept**: Snapshot freezes dataset state at a transaction group boundary. Instantaneous because of COW. Clones are writable copies of snapshots.

**Source**: `module/zfs/dsl_dataset.c` (`dsl_dataset_snapshot_sync_impl`, `dsl_dataset_clone_sync`)

**AnyFS mapping**: Already documented in `hybrid-backend-design.md`. Enhancement: make snapshots first-class with clones:

```sql
CREATE TABLE snapshots (
    snap_id     INTEGER PRIMARY KEY,
    dataset_id  INTEGER NOT NULL,
    name        TEXT NOT NULL,
    created_at  INTEGER NOT NULL,
    created_txg INTEGER NOT NULL,
    UNIQUE(dataset_id, name)
);

CREATE TABLE clones (
    clone_id    INTEGER PRIMARY KEY,
    origin_snap INTEGER NOT NULL REFERENCES snapshots(snap_id),
    dataset_id  INTEGER NOT NULL REFERENCES datasets(dataset_id),
    created_at  INTEGER NOT NULL
);
```

### 2.6 Send/Receive (Incremental Replication)

**ZFS concept**: `zfs send` generates a stream of Data Replication Records (DRR). Incremental sends traverse blocks born after the "from" snapshot's transaction group. `zfs receive` applies the stream on the receiving side.

**Source**: `module/zfs/dmu_send.c`, `module/zfs/dmu_recv.c`

**AnyFS mapping**: Define a replication stream format with a change log:

```rust
enum ReplicationRecord {
    Begin { dataset: String, from_txg: Option<u64>, to_txg: u64 },
    CreateNode { inode: u64, parent: u64, name: String, node_type: NodeType },
    WriteBlob { blob_id: String, data: Vec<u8>, checksum: [u8; 32] },
    UpdateNode { inode: u64, changes: NodeDiff },
    RemoveNode { inode: u64 },
    End { checksum: [u8; 32] },
}
```

```sql
-- Change tracking for incremental sends
CREATE TABLE change_log (
    seq         INTEGER PRIMARY KEY AUTOINCREMENT,
    txg         INTEGER NOT NULL,
    operation   TEXT NOT NULL,
    inode       INTEGER,
    path        TEXT,
    blob_id     TEXT,
    timestamp   INTEGER NOT NULL
);
```

Incremental send: `SELECT * FROM change_log WHERE txg > bookmark.txg ORDER BY seq`.

**Use cases**:
- Replicate AnyFS backend to a remote backup server
- Edge-to-cloud sync
- Migrate between backend types (SqliteBackend -> IndexedBackend)

### 2.7 Deduplication

**ZFS concept**: DDT (Dedup Table) uses cryptographic checksums as keys. On write, if checksum exists, increment refcount and skip write. Notoriously memory-intensive.

**Source**: `module/zfs/ddt.c`, `module/zfs/ddt_log.c`, `module/zfs/ddt_zap.c`

**AnyFS mapping**: Already implemented in IndexedBackend (whole-file dedup). Enhancement: block-level dedup:

```sql
CREATE TABLE chunks (
    chunk_hash  TEXT PRIMARY KEY,
    chunk_data  BLOB NOT NULL,
    size        INTEGER NOT NULL,
    refcount    INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE file_chunks (
    file_id     INTEGER NOT NULL,
    chunk_index INTEGER NOT NULL,
    chunk_hash  TEXT NOT NULL REFERENCES chunks(chunk_hash),
    PRIMARY KEY (file_id, chunk_index)
);
```

**Key difference**: Two 1GB files differing by 1 byte share 0% storage with whole-file dedup, but ~99.998% with block-level dedup using 128KB chunks.

### 2.8 Compression

**ZFS concept**: Transparent per-block compression during write pipeline. Early-abort skips compression if result exceeds original size.

**Source**: `module/zfs/zio_compress.c` (LZ4, ZSTD, GZIP, LZJB, ZLE)

**AnyFS mapping**: Tower-style compression middleware:

```rust
pub struct CompressionLayer<B> {
    inner: B,
    algorithm: CompressionAlgorithm, // Lz4, Zstd { level }, Gzip { level }
}

impl<B: FsWrite> FsWrite for CompressionLayer<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let compressed = self.algorithm.compress(data);
        if compressed.len() >= data.len() {
            self.inner.write(path, data)     // Early abort: store uncompressed
        } else {
            self.inner.write(path, &compressed)
        }
    }
}
```

**Caveat**: Compression breaks `read_range()` -- must decompress entire blob. Per-extent compression avoids this.

### 2.9 ARC (Adaptive Replacement Cache)

**ZFS concept**: Seven-state cache based on Megiddo-Modha algorithm. Ghost lists track evicted entries' metadata (no data) to adaptively balance MRU vs MFU. Responds to memory pressure.

**Source**: `module/zfs/arc.c`

**AnyFS mapping**: Replace simple LRU in Cache middleware (ADR-020) with ARC:

```rust
pub struct ArcCache<B> {
    inner: B,
    mru: LruCache<PathBuf, CacheEntry>,       // Recently used
    mfu: LruCache<PathBuf, CacheEntry>,       // Frequently used
    mru_ghost: LruCache<PathBuf, ()>,          // Evicted MRU metadata (no data)
    mfu_ghost: LruCache<PathBuf, ()>,          // Evicted MFU metadata (no data)
    target_mru_size: AtomicUsize,              // Adaptive target
}
```

Ghost MRU hit -> shift toward recency. Ghost MFU hit -> shift toward frequency. No manual tuning needed.

**Caveat**: Double-caching problem if used above SQLite's own page cache.

### 2.10 ZIL (ZFS Intent Log)

**ZFS concept**: Write-ahead log for synchronous operations. Records intents for crash replay. Separate from the main transaction group pipeline.

**Source**: `module/zfs/zil.c`

**AnyFS mapping**: Intent log bridging blob store and metadata DB in IndexedBackend:

```sql
CREATE TABLE intent_log (
    seq         INTEGER PRIMARY KEY AUTOINCREMENT,
    operation   TEXT NOT NULL,
    path        TEXT NOT NULL,
    blob_id     TEXT,
    metadata    TEXT,
    created_at  INTEGER NOT NULL,
    committed   BOOLEAN NOT NULL DEFAULT FALSE
);
```

Record intent before operation, mark committed after. On crash recovery, replay uncommitted intents.

### 2.11 Dataset Properties (Inheritable Configuration)

**ZFS concept**: Per-dataset properties (compression, quota, encryption, recordsize) with inheritance. Values: local, inherited, received, default.

**Source**: `module/zfs/dsl_prop.c`

**AnyFS mapping**: Properties drive runtime middleware composition:

```sql
CREATE TABLE dataset_properties (
    dataset_id  INTEGER NOT NULL,
    property    TEXT NOT NULL,
    value       TEXT NOT NULL,
    source      TEXT NOT NULL DEFAULT 'local',
    PRIMARY KEY (dataset_id, property)
);
```

```rust
fn build_stack(config: &DatasetConfig) -> impl Fs {
    let mut stack = BackendStack::new(SqliteBackend::open(&config.db_path)?);
    if let Some(quota) = config.resolve("quota", parent).as_u64() {
        stack = stack.layer(QuotaLayer::new(quota));
    }
    if config.resolve("compression", parent).as_str() != "none" {
        stack = stack.layer(CompressionLayer::new(config.compression()));
    }
    stack
}
```

### 2.12 Scrubbing (Background Integrity Verification)

**ZFS concept**: Traverse all blocks, verify checksums, self-heal from mirrors. Throttled to avoid starving foreground I/O.

**Source**: `module/zfs/dsl_scan.c`

**AnyFS mapping**: Verify every blob referenced by metadata actually exists and matches its hash:

```rust
pub fn scrub(&self, throttle: ScrubThrottle) -> Result<ScrubResult, FsError> {
    // Phase 1: Verify all nodes reference valid blobs
    for node in all_file_nodes() {
        throttle.wait();
        let data = self.blobs.get(&node.blob_id)?;
        if sha256(&data) != node.blob_id {
            result.checksum_errors.push(/* ... */);
        }
    }
    // Phase 2: Find orphaned blobs
    for blob in self.blobs.list_all()? {
        if !referenced_by_any_node(&blob) {
            result.orphaned_blobs.push(blob);
        }
    }
}
```

### 2.13 Bookmarks (Lightweight Send/Receive Markers)

**ZFS concept**: Record a snapshot's transaction group without holding block references. Enable incremental sends after destroying the source snapshot.

**Source**: `module/zfs/dsl_bookmark.c`

**AnyFS mapping**: Tiny metadata entries that remember "last sent" state:

```sql
CREATE TABLE bookmarks (
    name            TEXT PRIMARY KEY,
    dataset_id      INTEGER NOT NULL,
    creation_txg    INTEGER NOT NULL,
    creation_time   INTEGER NOT NULL,
    last_change_seq INTEGER NOT NULL  -- Points into change_log
);
```

Create snapshot -> send to replica -> bookmark -> destroy snapshot (frees storage) -> next incremental send uses bookmark.

---

## Part 3: XFS

XFS is a high-performance journaling filesystem in the Linux kernel at `fs/xfs/`.

### 3.1 Allocation Groups (Parallel Regions)

**XFS concept**: Divides filesystem into equally-sized Allocation Groups, each with independent inodes, free space B+ trees, and metadata. Enables concurrent I/O without contention.

**Source**: `fs/xfs/libxfs/xfs_ag.h`, `fs/xfs/libxfs/xfs_alloc.c`

**AnyFS mapping**: Database sharding by path prefix -- partition the `nodes` table across multiple SQLite files:

```
shard_0.db: /users/a-m/
shard_1.db: /users/n-z/
shard_2.db: /system/
```

Each shard has its own SQLite writer, enabling true parallel writes across path prefixes.

**Caveat**: Cross-shard operations (rename across shards) lose atomicity.

### 3.2 Extent-Based Allocation (Chunked Blobs)

**XFS concept**: Records contiguous block ranges as `(startblock, startoff, blockcount)` tuples instead of tracking individual blocks.

**Source**: `fs/xfs/libxfs/xfs_bmap.c`, `fs/xfs/libxfs/xfs_bmap.h`

**AnyFS mapping**: Store large files as extent-like chunks:

```sql
CREATE TABLE extents (
    inode       INTEGER NOT NULL,
    offset      INTEGER NOT NULL,
    length      INTEGER NOT NULL,
    blob_id     TEXT NOT NULL,
    PRIMARY KEY (inode, offset)
);
```

Enables `read_range()` to fetch only relevant chunks. Append-only logs create new extents without rewriting existing blobs.

### 3.3 Delayed Allocation (Write Batching)

**XFS concept**: Defer block allocation until writeback. Allows the allocator to see total write size and allocate contiguously.

**Source**: `fs/xfs/xfs_file.c` (`xfs_file_buffered_write`), `fs/xfs/xfs_iomap.c`

**AnyFS mapping**: Already implemented as the write batching pattern in `sqlite-operations.md`. Flush triggers: batch size, timeout, explicit `sync()`, read-after-write consistency.

### 3.4 Online Defragmentation

**XFS concept**: `xfs_fsr` defragments files by allocating a temp file, copying data contiguously, then atomically swapping extent maps.

**Source**: `fs/xfs/xfs_bmap_util.c` (`xfs_swapext`)

**AnyFS mapping**: SQLite `VACUUM` and `incremental_vacuum`. For blob stores: consolidate small blobs, repack chunked files, rebuild indexes.

### 3.5 Project Quotas (Per-Directory-Tree Quotas)

**XFS concept**: Quotas applied to directory trees. A project ID is assigned to a directory hierarchy; all children inherit it.

**Source**: `fs/xfs/xfs_qm.c`

**AnyFS mapping**: Extend `QuotaLayer` beyond global limits to per-path-prefix quotas:

```rust
let backend = backend.layer(ProjectQuotaLayer::builder()
    .project("/users/alice", QuotaPolicy { max_size: 1_GB, max_files: 10_000 })
    .project("/users/bob",   QuotaPolicy { max_size: 500_MB, max_files: 5_000 })
    .build());
```

### 3.6 Reflink/CoW (Lightweight Copies)

**XFS concept**: Share physical blocks between files via reference counting. On write, COW creates new blocks for modified regions only.

**Source**: `fs/xfs/xfs_reflink.c`, `fs/xfs/libxfs/xfs_refcount_btree.c`

**AnyFS mapping**: Already implemented in IndexedBackend's `copy()`:

```rust
fn copy(&self, from: &Path, to: &Path) -> Result<(), FsError> {
    // Just increment refcount -- no blob copy!
}
```

Enhancement: extend to sub-file (extent-level) COW for partial updates.

### 3.7 Reverse Mapping

**XFS concept**: Maps physical blocks back to owning files. Enables online fsck, error reporting (which files affected by bad sector), and reflink validation.

**Source**: `fs/xfs/libxfs/xfs_rmap_btree.c`

**AnyFS mapping**: The `idx_nodes_blob` index already answers "which files reference blob X?". A full reverse mapping table:

```sql
CREATE TABLE blob_owners (
    blob_id TEXT NOT NULL,
    inode   INTEGER NOT NULL,
    PRIMARY KEY (blob_id, inode)
);
```

Enables smart GC (verify refcounts by counting actual references) and impact analysis (which files affected by corrupted blob).

### 3.8 Log-Structured Journaling

**XFS concept**: Write-ahead log for metadata. Deferred operations (EFI) enable atomic multi-step operations.

**Source**: `fs/xfs/xfs_log.c`, kernel.org [delayed logging design](https://www.kernel.org/doc/html/v6.1/filesystems/xfs-delayed-logging-design.html)

**AnyFS mapping**: SQLite WAL mode + the audit table pattern:

```sql
CREATE TABLE audit (
    seq         INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   INTEGER NOT NULL,
    operation   TEXT NOT NULL,
    path        TEXT,
    details     TEXT  -- JSON with before/after state
);
```

Enables crash recovery, undo/redo, and replication (stream audit log to replica).

---

## Part 4: Git Internals

Git source at [github.com/git/git](https://github.com/git/git).

### 4.1 Content-Addressable Object Store

**Git concept**: Every object (blob, tree, commit) stored by SHA hash of content. Identical content = same hash = automatic dedup.

**Source**: `object-file.c`, `hash-object.c`

**AnyFS mapping**: Already implemented in IndexedBackend's `LocalCasBackend` with SHA-256 and the same `xx/hash` directory layout as Git.

### 4.2 Pack Files (Delta Compression)

**Git concept**: Similar objects stored as deltas from a base object. Dramatically reduces storage for repositories with many versions of the same files.

**Source**: `builtin/pack-objects.c`, `packfile.c`

**AnyFS mapping**: Delta-compressed blob storage:

```sql
CREATE TABLE blobs (
    blob_id     TEXT PRIMARY KEY,
    base_id     TEXT,              -- NULL if full blob, else delta base
    delta       BLOB,             -- Delta from base (if base_id set)
    full_size   INTEGER NOT NULL,
    stored_size INTEGER NOT NULL,  -- Actual bytes on disk
    refcount    INTEGER NOT NULL DEFAULT 0
);
```

**Use case**: Document management with many revisions. Background "pack" job identifies similar blobs and computes deltas.

**Tradeoff**: Random access to delta-compressed blobs requires reconstructing from base + delta chain.

### 4.3 Tree Objects (Merkle Trees)

**Git concept**: Directory listing where each entry contains a hash of either a blob or subtree. Changing any file changes hashes up to root.

**Source**: `tree.c`, `tree-walk.c`

**AnyFS mapping**: Add `tree_hash` to the `nodes` table:

```sql
ALTER TABLE nodes ADD COLUMN tree_hash TEXT;
-- Directories: SHA-256 of sorted(child_name + child_hash)
-- Files: same as blob_id
```

**Use cases**:
- Efficient sync: compare root hashes to detect any change; descend only into changed directories
- Snapshot diff in O(changed paths) time
- Integrity verification: recompute tree hashes bottom-up and compare

**Caveat**: Hash invalidation cascades up to root on every file change. Must be lazy/incremental.

### 4.4 Refs and Reflog (Named Filesystem States)

**Git concept**: Named pointers to commits. Reflog records every change to each ref, enabling recovery.

**Source**: `refs.c`, `refs/files-backend.c`, `reflog.c`

**AnyFS mapping**: Named snapshots with change history:

```sql
CREATE TABLE snapshot_log (
    snapshot    TEXT NOT NULL,
    old_hash    TEXT,
    new_hash    TEXT NOT NULL,
    timestamp   INTEGER NOT NULL,
    operation   TEXT NOT NULL,
    message     TEXT
);
```

"Branches" = mutable snapshot refs. "Tags" = immutable snapshot refs.

### 4.5 Garbage Collection with Grace Periods

**Git concept**: Unreachable objects removed, but protected by reflog for configurable period (30-90 days).

**Source**: `builtin/gc.c`, `builtin/prune.c`

**AnyFS mapping**: Already implemented. Enhancement: add grace period:

```sql
SELECT blob_id FROM blobs
WHERE refcount = 0
  AND created_at < strftime('%s', 'now') - 86400;  -- At least 1 day old
```

### 4.6 Worktrees (Shared Blob Stores)

**Git concept**: Multiple working directories linked to the same repository. Each has its own HEAD but shares the object store.

**Source**: `worktree.c`

**AnyFS mapping**: Multiple `FileStorage` instances sharing a single blob store:

```rust
let blobs = Arc::new(LocalCasBackend::new("./shared-blobs"));

// Worktree 1: "main"
let wt1 = IndexedBackend::with_blobs("wt1-index.db", blobs.clone());
let fs1 = FileStorage::new(wt1);

// Worktree 2: "feature" -- shares blobs, independent metadata
let wt2 = IndexedBackend::with_blobs("wt2-index.db", blobs.clone());
let fs2 = FileStorage::new(wt2);
```

**Use case**: Multi-tenant systems where tenants share common files but have independent directory structures.

### 4.7 Alternates (Chained Blob Stores)

**Git concept**: `.git/objects/info/alternates` lists paths to other repos' object dirs. Objects searched locally first, then in alternates.

**Source**: GitLab uses this extensively for [fork deduplication](https://docs.gitlab.com/18.6/development/git_object_deduplication/)

**AnyFS mapping**: Chained blob store pattern:

```rust
pub struct ChainedBlobStore {
    primary: Arc<dyn BlobStore>,          // Local, writable
    alternates: Vec<Arc<dyn BlobStore>>,  // Shared, read-only
}

impl BlobStore for ChainedBlobStore {
    fn get(&self, blob_id: &str) -> Result<Vec<u8>, BlobError> {
        if let Ok(data) = self.primary.get(blob_id) { return Ok(data); }
        for alt in &self.alternates {
            if let Ok(data) = alt.get(blob_id) { return Ok(data); }
        }
        Err(BlobError::NotFound)
    }

    fn put(&self, data: &[u8]) -> Result<String, BlobError> {
        let blob_id = sha256_hex(data);
        // Skip write if already in any alternate
        for alt in &self.alternates {
            if alt.exists(&blob_id)? { return Ok(blob_id); }
        }
        self.primary.put(data)
    }
}
```

**Use case**: "Base image" blob store shared read-only; each tenant has a private writable store.

### 4.8 Bitmap Indexes (Fast Reachability)

**Git concept**: Compressed bitset per commit indicating which objects are reachable. Enables O(bitwise OR) reachability vs O(graph traversal).

**Source**: `pack-bitmap.c`, [bitmap format docs](https://git-scm.com/docs/bitmap-format)

**AnyFS mapping**: Accelerate GC across many snapshots:

```rust
fn reachable_blobs(&self, snapshots: &[&str]) -> RoaringBitmap {
    let mut result = RoaringBitmap::new();
    for snap in snapshots {
        result |= self.load_bitmap(snap);
    }
    result
}

fn gc(&self) {
    let reachable = self.reachable_blobs(&self.active_snapshots());
    let orphans = self.all_blob_bitmap() - reachable;
    // Delete orphans
}
```

Fast snapshot diff: XOR two bitmaps to find changed blobs.

### 4.9 Grafts and Replace Objects

**Git concept**: Transparent object substitution. When reading object X, return replacement Y instead.

**Source**: `refs/replace/`, [git-replace docs](https://git-scm.com/docs/git-replace)

**AnyFS mapping**: Virtual blob replacement table:

```sql
CREATE TABLE replacements (
    original_blob_id    TEXT PRIMARY KEY,
    replacement_blob_id TEXT NOT NULL,
    reason TEXT,
    created_at INTEGER NOT NULL
);
```

**Use cases**: Content migration (re-encode blobs without changing metadata), redaction, A/B testing.

---

## Unified Impact Matrix

### Already Present in AnyFS (Enhance)

| Concept | Source | AnyFS Component | Enhancement Opportunity |
|---|---|---|---|
| Content-addressed storage | Git | IndexedBackend | Block-level dedup (chunks) |
| Copy-on-Write | ZFS, XFS | IndexedBackend (refcount blobs) | Sub-file COW via extents |
| Reflink/CoW copies | XFS | `copy()` via refcount++ | Extent-level partial COW |
| Deduplication | ZFS, Git | `blob_id = sha256(content)` | Block-level dedup tables |
| Garbage collection | Git, ZFS | `refcount = 0` pruning | Grace periods, bitmap indexes |
| Write batching | XFS | Write queue pattern | Configurable flush policies |
| WAL journaling | XFS, ZFS | SQLite WAL mode | Intent log for two-phase ops |
| Metadata/data separation | ZFS (special vdevs) | IndexedBackend | Tiered blob stores |
| Snapshots | ZFS | IndexedBackend pattern | First-class API, clones |
| Defragmentation | XFS | VACUUM | Blob consolidation |

### High-Value Additions

| Concept | Source | Effort | Impact | Description |
|---|---|---|---|---|
| **SQLite-as-PV spanning** | LVM | High | **Critical** | Overcome SQLite size limits via multi-file pools |
| **Compression middleware** | ZFS | Low | High | Transparent LZ4/ZSTD layer with early-abort |
| **Checksum verification** | ZFS | Low | High | Read-time integrity middleware |
| **Scrubbing** | ZFS | Low | High | Background blob integrity verification |
| **Project quotas** | XFS | Medium | High | Per-directory-tree quota middleware |
| **ARC caching** | ZFS | Medium | High | Replace simple LRU with adaptive cache |
| **Dataset properties** | ZFS | Medium | High | Inheritable per-dataset configuration |
| **Merkle tree hashing** | Git | Medium | Medium | Efficient sync and snapshot diff |
| **Shared blob stores** | Git (alternates/worktrees) | Medium | Medium | Multi-tenant blob dedup |
| **Send/receive protocol** | ZFS | High | Medium | Incremental replication |
| **Bookmarks** | ZFS | Low | Medium | Lightweight send/receive markers |
| **Intent log** | ZFS (ZIL) | Medium | Medium | Crash recovery for two-phase ops |

### Ambitious / Niche

| Concept | Source | Effort | Fit | Notes |
|---|---|---|---|---|
| Pack files (delta compression) | Git | High | Medium | CPU-intensive, good for versioned docs |
| Striping across SQLite PVs | LVM | Medium | Medium | Value only with multiple physical disks |
| Mirroring across SQLite PVs | LVM | Medium | Medium | Redundancy for critical data |
| Thin provisioning | LVM | Medium | Medium | Natural fit for SQLite (grows on demand) |
| pvmove (live migration) | LVM | High | Medium | Zero-downtime PV replacement |
| Allocation groups (sharding) | XFS | High | Low | Cross-shard atomicity is hard |
| RAIDZ | ZFS | Very High | Low | Parity across SQLite files is unnatural |

---

## Recommended Priorities

### Phase 1: Foundation (Multi-SQLite Spanning)

The LVM-inspired multi-file architecture directly solves the SQLite size limit problem:

1. **PhysicalVolume abstraction**: Individual SQLite files as PVs with extent-based storage
2. **VolumeGroup coordinator**: Aggregate PVs into a pool with redundant metadata
3. **LogicalVolume backend**: Implement `Fs` trait over mapped extents
4. **vgextend/vgreduce**: Dynamic add/remove of SQLite PV files

### Phase 2: Integrity & Efficiency

Low-effort, high-impact features from ZFS and Git:

5. **CompressionLayer middleware**: LZ4/ZSTD with early-abort
6. **ChecksumLayer middleware**: Read-time verification
7. **Scrub operation**: Background blob integrity checking
8. **GC grace periods**: Don't delete recently-orphaned blobs

### Phase 3: Advanced Storage Management

9. **Dataset properties with inheritance**: Drive middleware composition from config
10. **Project quotas**: Per-directory-tree limits
11. **ARC cache**: Replace simple LRU
12. **Merkle tree hashing**: Efficient sync and diff

### Phase 4: Replication & Scale

13. **Change log infrastructure**: Transaction numbering for send/receive
14. **Send/receive protocol**: Incremental replication
15. **Bookmarks**: Lightweight send markers
16. **Shared blob stores**: Git alternates pattern for multi-tenant dedup

---

## Sources

### LVM
- [LVM2 Resource Page](https://www.sourceware.org/lvm2/)
- [LVM2 GitLab Repository](https://gitlab.com/lvmteam/lvm2)
- [Device Mapper - Wikipedia](https://en.wikipedia.org/wiki/Device_mapper)
- [LVM - ArchWiki](https://wiki.archlinux.org/title/LVM)
- [LVM Concepts - DigitalOcean](https://www.digitalocean.com/community/tutorials/an-introduction-to-lvm-concepts-terminology-and-operations)
- [dm-thin-provisioning - Kernel Docs](https://docs.kernel.org/admin-guide/device-mapper/thin-provisioning.html)
- [dm-linear - Kernel Docs](https://docs.kernel.org/admin-guide/device-mapper/linear.html)
- [dm-stripe - Kernel Docs](https://docs.kernel.org/admin-guide/device-mapper/striped.html)
- [lvmcache(7) man page](https://man7.org/linux/man-pages/man7/lvmcache.7.html)

### ZFS / OpenZFS
- [OpenZFS Repository](https://github.com/openzfs/zfs)
- `module/zfs/spa.c` - Storage Pool Allocator
- `module/zfs/dsl_dataset.c` - Dataset/Snapshot Layer
- `module/zfs/dmu_send.c` / `dmu_recv.c` - Send/Receive
- `module/zfs/zio_checksum.c` - Checksum framework
- `module/zfs/arc.c` - Adaptive Replacement Cache
- `module/zfs/zil.c` - ZFS Intent Log
- `module/zfs/ddt.c` - Deduplication Table
- `module/zfs/zio_compress.c` - Compression
- `module/zfs/vdev_raidz.c` - RAIDZ
- `module/zfs/dsl_scan.c` - Scrubbing
- `module/zfs/dsl_bookmark.c` - Bookmarks
- `module/zfs/dsl_prop.c` - Dataset Properties

### XFS
- [XFS Allocation Groups](https://xfs.org/docs/xfsdocs-xml-dev/XFS_Filesystem_Structure/tmp/en-US/html/Allocation_Groups.html)
- [XFS Delayed Logging Design - Kernel Docs](https://www.kernel.org/doc/html/v6.1/filesystems/xfs-delayed-logging-design.html)
- [XFS Online Fsck Design](https://www.kernel.org/doc/html/next/filesystems/xfs-online-fsck-design.html)
- [XFS Realtime Rmap and Reflink (Oracle)](https://blogs.oracle.com/linux/xfs-realtime-rmap-and-reflink)
- [XFS Reverse Mapping (LWN)](https://lwn.net/Articles/695290/)
- `fs/xfs/libxfs/xfs_ag.h` - Allocation Group structures
- `fs/xfs/libxfs/xfs_alloc.c` - Allocation logic
- `fs/xfs/libxfs/xfs_bmap.c` - Extent mapping
- `fs/xfs/xfs_reflink.c` - Reflink implementation
- `fs/xfs/libxfs/xfs_rmap_btree.c` - Reverse mapping

### Git
- [Git Internals: Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Git Internals: Packfiles](https://git-scm.com/book/en/v2/Git-Internals-Packfiles)
- [Git's Database Internals (GitHub Blog)](https://github.blog/open-source/git/gits-database-internals-i-packed-object-store/)
- [Git Bitmap Format](https://git-scm.com/docs/bitmap-format)
- [Git Pack Format](https://git-scm.com/docs/gitformat-pack)
- [Git Object Deduplication (GitLab)](https://docs.gitlab.com/18.6/development/git_object_deduplication/)
- `builtin/pack-objects.c` - Pack file creation
- `packfile.c` - Pack file reading
- `pack-bitmap.c` - Bitmap indexes
- `tree.c`, `tree-walk.c` - Tree objects
- `refs.c`, `worktree.c` - References and worktrees

### SQLite
- [SQLite Implementation Limits](https://www.sqlite.org/limits.html)
