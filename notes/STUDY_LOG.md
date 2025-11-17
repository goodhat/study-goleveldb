# GolevelDB Study Log

> **Purpose**: Timeline index of learning sessions. For detailed notes, see [topics/](topics/).
>
> **Structure**:
> - STUDY_LOG.md (this file) - Chronological index, session summaries, cross-references
> - topics/*.md - Deep-dive notes on specific components (like SSTables in a database!)

---

## 2025-11-08 - Session 1: MemDB and Skip List

**Duration**: ~2 hours
**Components studied**: MemDB (memtable)
**Detailed notes**: [topics/01-memdb-skiplist.md](topics/01-memdb-skiplist.md)

### Quick Summary
Traced `db.Get()` into memdb and discovered skip list implementation. Understood:
- kvData (append-only KV storage) + nodeData (flattened skip list metadata)
- Put operation handles existing vs new keys differently
- Fragmentation trade-off in append-only design

### Key Insights
1. **Design choice**: Separate kvData/nodeData arrays reduce GC pressure and improve cache locality
2. **Concurrency**: RWMutex works because prevNode array is shared state
3. **Trade-off**: Append-only = fast writes but memory fragmentation

### Questions for Next Session
- How does MemDB get frozen and flushed to SSTable?
- What size threshold triggers freezing?
- How does DB.Put() coordinate journal + memtable writes?

### Status
- [x] Understand skip list data structure
- [x] Trace Get/Put operations
- [ ] Understand freezing mechanism
- [ ] Study write path coordination

---

## 2025-11-17 - Session 2: Compaction Architecture

**Duration**: ~1.5 hours
**Components studied**: Two-level compaction system (mCompaction, tCompaction)
**Detailed notes**: [topics/02-compaction-architecture.md](topics/02-compaction-architecture.md)

### Quick Summary
Discovered LevelDB runs **two independent compaction goroutines**:
- `mCompaction`: Flushes frozen memtable to L0 SSTables (memory → disk)
- `tCompaction`: Merges SSTables across levels (L0 → L1 → L2...)

Traced how they coordinate via pause mechanism and command channels.

### Key Insights
1. **Separation of concerns**: Different goroutines for memory management vs disk organization
2. **Priority management**: mCompaction can pause tCompaction to avoid I/O contention
3. **Channel patterns**: `select` with multiple cases prevents blocking when receiver dies
4. **Write backpressure**: waitQ holds blocked writers when L0 is too full
5. **Connection to memdb study**: Frozen memtable gets flushed via skip list iteration (already sorted!)

### Questions for Next Session
- What's in `sessionRecord` and how is it written to MANIFEST?
- How does `flushMemdb()` convert skip list to SSTable?
- What's the SSTable file format?
- How does `tableAutoCompaction()` pick which levels to merge?
- What happens if mCompaction crashes while tCompaction is paused?

### Status
- [x] Understand two compaction goroutines exist
- [x] Trace command sending and receiving
- [x] Understand pause coordination mechanism
- [x] Map out when each compaction type is triggered
- [ ] Understand sessionRecord and MANIFEST
- [ ] Study flushMemdb() internals
- [ ] Study SSTable format
- [ ] Study table compaction strategy

---

## Session Template

**Duration**:
**Components studied**:
**Detailed notes**: [topics/XX-component-name.md](topics/)

### Quick Summary

### Key Insights

### Questions for Next Session

### Status
- [ ]

---
