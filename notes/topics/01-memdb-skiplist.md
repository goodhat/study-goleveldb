# MemDB - Skip List Implementation

> **Component**: In-memory database (memtable)
> **File**: [leveldb/memdb/memdb.go](../../leveldb/memdb/memdb.go)
> **Studied**: 2025-11-08
> **Status**: ✓ Completed

---

## Overview

MemDB is goleveldb's in-memory write buffer (memtable) implemented as a skip list. It provides fast O(log n) reads and writes while maintaining sorted order.

## Architecture

### Data Structures

**Two parallel arrays store all data:**

```
kvData []byte      - Append-only storage for key-value pairs
nodeData []int     - Flattened skip list node metadata
```

### Node Layout in nodeData

Each node occupies `4 + height` integers:

```
[0] nKV     : Offset into kvData where this KV starts
[1] nKey    : Length of key
[2] nVal    : Length of value
[3] nHeight : Skip list height (1-12)
[4..h]      : Forward pointers at each level (nNext+0, nNext+1, ...)
```

**Example**: A height-3 node takes 7 integers total (4 + 3).

### Skip List Properties

| Property | Value | Notes |
|----------|-------|-------|
| Max height | 12 | Defined as `tMaxHeight` |
| Branching factor | 4 | 25% probability to increase height |
| Height distribution | Geometric | `randHeight()` uses random generator |

---

## Key Operations

### 1. Put(key, value)

**Two cases:**

#### Case A: Existing Key ([memdb.go:296-304](../../leveldb/memdb/memdb.go#L296-L304))
1. Append new KV to end of `kvData` (old data not reclaimed)
2. Update node's `nKV` offset to point to new location
3. Update `nVal` length
4. Adjust `kvSize` accounting

**Fragmentation note**: Old KV data remains in buffer, causing memory waste over time.

#### Case B: New Key ([memdb.go:312-343](../../leveldb/memdb/memdb.go#L312-L343))
1. Randomly determine node height via `randHeight()`
2. If taller than current max, update `maxHeight` and initialize new level prevNodes
3. Append KV data to `kvData`
4. Create new node in `nodeData`:
   - Store KV offset, key length, value length, height
   - Wire up forward pointers using `prevNode` array
5. Update predecessor nodes' forward pointers

**Why lock is required**: `prevNode[tMaxHeight]` is shared across entire DB, so writes must be exclusive.

### 2. Get(key) → value

**Process** ([memdb.go:385-395](../../leveldb/memdb/memdb.go#L385-L395)):
1. Call `findGE(key, false)` - binary search through skip list
2. If exact match, extract value from `kvData` using offset + lengths
3. Return slice pointing directly into `kvData` (zero-copy)

**Locking**: Only requires read lock since `prevNode` not needed.

### 3. findGE(key, prev) → (node, exact)

**Classic skip list search** ([memdb.go:217-242](../../leveldb/memdb/memdb.go#L217-L242)):

```
Start at head node, highest level
while true:
    next = current.forward[level]
    if next.key < target:
        current = next          // Move forward
    else:
        if prev:
            prevNode[level] = current  // Record predecessor
        if level == 0:
            return next
        level--                 // Drop down
```

**When `prev=true`**: Populates `prevNode` array with predecessors at each level (needed for insertion/deletion).

---

## Design Insights

### Why Append-Only kvData?

**Advantages**:
- Simple, fast writes (no shifting)
- Lock-free reads of KV data
- Cache-friendly sequential writes

**Disadvantages**:
- Memory fragmentation on updates
- Cannot reclaim space for deleted/updated keys
- Mitigation: Memtable has size limit, gets frozen and flushed periodically

### Why Separate kvData and nodeData?

**Separation benefits**:
1. Pointers in nodeData are simple integer offsets (no garbage collection pressure)
2. Can iterate through keys without loading values
3. Node metadata stays hot in cache during searches

### Concurrency Model

| Operation | Lock Type | Why |
|-----------|-----------|-----|
| Get, Contains, Find | RLock | Read-only, no shared state modification |
| Put, Delete | Lock | Modifies shared `prevNode` array |
| Iterators | RLock per operation | Snapshot consistency not guaranteed across calls |

**Note**: Iterators can see inconsistent state if DB is modified concurrently. This is acceptable for memtable since it's ephemeral.

---

## Open Questions

- [x] How does skip list search work? → Answered: Classic algorithm, start high and drop down
- [ ] When does memtable get frozen and flushed?
- [ ] What size threshold triggers freezing?
- [ ] How much fragmentation is acceptable before compaction?
- [ ] How does iterator maintain consistency during concurrent writes?

---

## Connections to Other Components

```
DB.Put()
   ↓
Journal (WAL) → [write durability]
   ↓
MemDB.Put() → [in-memory write]
   ↓
(when full) → Frozen Memtable
   ↓
Compaction → SSTable files
```

**Next study targets**:
- Write path: How does `DB.Put()` coordinate journal + memtable?
- Flush mechanism: When/how does memtable freeze?

---

## Performance Characteristics

| Operation | Time Complexity | Space Complexity |
|-----------|----------------|------------------|
| Get | O(log n) average | O(1) |
| Put | O(log n) average | O(n) amortized |
| Delete | O(log n) average | O(1)* |
| Scan (iterator) | O(1) per key | O(1) |

*Delete doesn't free memory, just updates metadata.

---

## Code Quality Observations

**Strengths**:
- Clean separation of concerns (data vs metadata)
- Well-commented constants and node layout
- Proper use of RWMutex for concurrency

**Potential improvements**:
- Could document fragmentation behavior more explicitly
- Helper functions for node offset calculation would improve readability

---

## Further Reading

- [Skip List Paper](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf) - Original Pugh paper
- [LevelDB Memtable](https://github.com/google/leveldb/blob/main/db/skiplist.h) - C++ implementation for comparison
