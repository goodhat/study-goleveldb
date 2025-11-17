# GolevelDB Study Plan

> This roadmap guides your exploration of goleveldb, organized by major components and increasing complexity.

---

## Progress Tracker

- [x] Phase 1.1: MemDB (Skip List)
- [ ] Phase 1.2: Write Path (Put → Journal → Memtable)
- [ ] Phase 1.3: Read Path (Get from Memtable & SSTables)
- [ ] Phase 2.1: SSTable Format
- [ ] Phase 2.2: Iterator Framework
- [ ] Phase 2.3: Compaction System
- [ ] Phase 3.1: Version Management
- [ ] Phase 3.2: Manifest & Recovery
- [ ] Phase 3.3: Snapshots & Transactions
- [ ] Phase 4.1: Cache (LRU)
- [ ] Phase 4.2: Bloom Filters
- [ ] Phase 4.3: Storage Abstraction

---

## Phase 1: Core Read/Write Path (CURRENT)

**Goal**: Understand how data flows from user API to persistent storage.

### 1.1 MemDB (Skip List) ✓ COMPLETED
**Files**: [leveldb/memdb/memdb.go](../leveldb/memdb/memdb.go)

**What you learned**:
- Skip list implementation with `kvData` and `nodeData` arrays
- Put/Get operations and locking strategy
- Append-only design and fragmentation trade-offs

---

### 1.2 Write Path (Put → Journal → Memtable) 🎯 NEXT

**Files to explore**:
- [leveldb/db.go](../leveldb/db.go) - Main DB.Put() implementation
- [leveldb/db_write.go](../leveldb/db_write.go) - Write coordination
- [leveldb/batch.go](../leveldb/batch.go) - Batch operations
- [leveldb/journal/journal.go](../leveldb/journal/journal.go) - Write-ahead log

**Key questions to answer**:
1. How does `DB.Put()` coordinate journal writes and memtable updates?
2. What is the batch write mechanism?
3. How does the write-ahead log (journal) ensure durability?
4. When does memtable get frozen?
5. What triggers a new journal file creation?

**Learning objectives**:
- [ ] Trace `DB.Put(key, value)` end-to-end
- [ ] Understand journal record format
- [ ] Learn about write batching and atomicity
- [ ] Discover memtable freezing conditions (size threshold?)
- [ ] See how errors are handled in write path

**Suggested approach**:
1. Start with simple `DB.Put()` in [db.go](../leveldb/db.go)
2. Follow it into `db_write.go` to see batch coordination
3. Look at journal writer to understand WAL format
4. Check when `mdb` (memtable) gets swapped with `frozenMem`

---

### 1.3 Read Path (Get from Memtable & SSTables)

**Files to explore**:
- [leveldb/db.go](../leveldb/db.go) - `DB.Get()` implementation
- [leveldb/version.go](../leveldb/version.go) - Multi-level file search
- [leveldb/table/reader.go](../leveldb/table/reader.go) - SSTable reading

**Key questions to answer**:
1. What is the search order? (memtable → frozen → L0 → L1+)
2. How does `DB.Get()` handle the "not found" case efficiently?
3. How are SSTables opened and cached?
4. What role does the version play?

**Learning objectives**:
- [ ] Trace `DB.Get(key)` through all levels
- [ ] Understand level ordering and search optimization
- [ ] Learn how table cache works
- [ ] See how bloom filters short-circuit reads

---

## Phase 2: Storage Layer

**Goal**: Understand persistent data structures and compaction.

### 2.1 SSTable Format

**Files to explore**:
- [leveldb/table/table.go](../leveldb/table/table.go) - Table structure
- [leveldb/table/reader.go](../leveldb/table/reader.go) - Reading tables
- [leveldb/table/writer.go](../leveldb/table/writer.go) - Writing tables

**Key concepts**:
- Block structure (data, filter, index, metaindex, footer)
- Compression (Snappy)
- Checksums (CRC32-C)
- Two-level indexing

---

### 2.2 Iterator Framework

**Files to explore**:
- [leveldb/iterator/iter.go](../leveldb/iterator/iter.go) - Iterator interfaces
- [leveldb/iterator/merged_iter.go](../leveldb/iterator/merged_iter.go) - Merge iteration
- [leveldb/iterator/indexed_iter.go](../leveldb/iterator/indexed_iter.go) - Two-level iteration
- [leveldb/db_iter.go](../leveldb/db_iter.go) - DB-level iterator

**Key concepts**:
- Iterator interface (Seek, Next, Prev, Valid, Key, Value)
- Merging sorted streams
- Snapshot consistency

---

### 2.3 Compaction System

**Files to explore**:
- [leveldb/db_compaction.go](../leveldb/db_compaction.go) - Compaction execution
- [leveldb/session_compaction.go](../leveldb/session_compaction.go) - Compaction picking
- [leveldb/version.go](../leveldb/version.go) - Level management

**Key questions**:
- When does compaction trigger?
- How are files selected for compaction?
- What's the difference between L0→L1 and L1→L2 compaction?
- How does compaction affect read/write performance?

---

## Phase 3: Consistency & Recovery

**Goal**: Understand crash recovery, versioning, and transactions.

### 3.1 Version Management

**Files**: [leveldb/version.go](../leveldb/version.go)

**Key concepts**:
- What is a "version"?
- How do versions enable snapshot isolation?
- Version reference counting

---

### 3.2 Manifest & Recovery

**Files**:
- [leveldb/session.go](../leveldb/session.go) - Session management
- [leveldb/session_record.go](../leveldb/session_record.go) - Version edits

**Key questions**:
- How does the manifest track file state?
- What happens during DB open/recovery?
- How are version edits applied?

---

### 3.3 Snapshots & Transactions

**Files**:
- [leveldb/db_snapshot.go](../leveldb/db_snapshot.go)
- [leveldb/db_transaction.go](../leveldb/db_transaction.go)

**Key concepts**:
- Snapshot isolation using sequence numbers
- Transaction semantics
- MVCC (Multi-Version Concurrency Control)

---

## Phase 4: Optimizations

**Goal**: Understand performance optimizations.

### 4.1 Cache (LRU)

**Files**: [leveldb/cache/cache.go](../leveldb/cache/cache.go), [leveldb/cache/lru.go](../leveldb/cache/lru.go)

**Key concepts**:
- Table cache (open file handles)
- Block cache (data blocks)
- LRU eviction policy

---

### 4.2 Bloom Filters

**Files**: [leveldb/filter/bloom.go](../leveldb/filter/bloom.go)

**Key concepts**:
- Bloom filter generation
- False positive rate tuning
- Integration with SSTable reads

---

### 4.3 Storage Abstraction

**Files**: [leveldb/storage/storage.go](../leveldb/storage/storage.go), [leveldb/storage/file_storage.go](../leveldb/storage/file_storage.go)

**Key concepts**:
- Storage interface
- File locking
- Platform-specific implementations

---

## Study Tips

### Effective Tracing
1. **Start with a use case**: Pick a simple operation (Get, Put, Scan)
2. **Use the debugger**: Set breakpoints and step through
3. **Draw diagrams**: Visualize data structures and flow
4. **Leave breadcrumbs**: Add comments explaining "why" not "what"

### Documentation Strategy
1. **Inline comments**: Use your native language (Chinese?) if it helps you think clearly
2. **Session logs**: After each study session, update [STUDY_LOG.md](./STUDY_LOG.md)
3. **Component notes**: Create detailed notes for complex components (e.g., `notes/compaction.md`)
4. **Questions list**: Keep a running list of questions; answer them as you learn

### Testing Your Understanding
- Write test cases for edge cases you discover
- Implement mini-versions of data structures (e.g., simple skip list)
- Explain concepts to yourself or others (Feynman technique)
- Draw state diagrams for concurrency scenarios

---

## Advanced Topics (Optional)

- Concurrency model (goroutines, channels, locks)
- Performance profiling and benchmarking
- Comparison with RocksDB, Badger, Pebble
- Using goleveldb in production systems
- Contributing fixes or features upstream

---

## Resources

- [LevelDB Documentation](https://github.com/google/leveldb/blob/main/doc/index.md)
- [LSM Tree Paper](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
- [Skip List Paper](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf)

---

**Remember**: Deep learning takes time. It's okay to revisit topics multiple times as you build intuition. The goal is understanding, not speed.
