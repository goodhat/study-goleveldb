# Code Reading Q&A - 2025-11-12

**Session focus**: Write path, compaction, and batch operations
**Files covered**: batch.go, db.go, db_compaction.go, db_state.go, db_write.go, key.go

---

## Tags

`#go-idioms` `#concurrency` `#performance` `#leveldb-architecture` `#write-path` `#compaction`

---

## Questions & Answers

### 1. Batch Reset Pattern - Why reset like this?

**Location**: [leveldb/batch.go:169](../leveldb/batch.go#L169)

**Question**: 為什麼是這樣 reset？

```go
func (b *Batch) Reset() {
    b.data = b.data[:0]
    b.index = b.index[:0]
    b.internalLen = 0
}
```

**Answer**: This keeps the underlying allocated memory but resets the slice length to 0. It's more efficient than allocating new slices:
- `b.data[:0]` - resets length to 0, but keeps capacity (no reallocation)
- `b.index[:0]` - same, reuses the backing array
- This way the batch can be reused without GC pressure from allocating new slices every time

**Pattern**: Slice reuse for object pooling (relates to Question #9)

**Tags**: `#go-idioms` `#performance`

---

### 2. Table vs Memtable Compaction Channels

**Location**: [leveldb/db.go:79](../leveldb/db.go#L79)

**Question**: tcomp 和 mcomp 的差別是什麼？

```go
tcompCmdC   chan cCmd
mcompCmdC   chan cCmd // QUESTION: tcomp 和 mcomp 的差別是什麼？
```

**Answer**:
- **`tcompCmdC`** = **Table** compaction (compacting SSTables between levels 0→1, 1→2, etc.)
- **`mcompCmdC`** = **Memtable** compaction (flushing memtable to Level 0 SSTable)

Two different goroutines handle these:
- `tcompCmdC` is read by the table compaction goroutine
- `mcompCmdC` is read by the memtable flush goroutine

**Why separated**: Memtable flush is more urgent (need to make space for new writes) while table compaction can be deferred.

**Tags**: `#leveldb-architecture` `#concurrency` `#compaction`

---

### 3. Pause Channel Pattern - Rendezvous Synchronization

**Location**: [leveldb/db_compaction.go:289](../leveldb/db_compaction.go#L289)

**Question**: 這要傳給誰？不會被 block 嗎？

```go
select {
case db.tcompPauseC <- (chan<- struct{})(resumeC): // QUESTION
case <-db.compPerErrC:
    close(resumeC)
    resumeC = nil
```

**Answer**: This sends to the table compaction goroutine's pause handler:
- `tcompPauseC` is a channel of channels: `chan chan<- struct{}`
- The table compaction goroutine reads from `tcompPauseC`, receives the `resumeC` channel, and closes it when it's safe to resume
- Won't block because `tcompPauseC` is buffered OR the compaction goroutine actively reads from it
- The `select` has `case <-db.compPerErrC:` as a fallback if compaction already failed

**Pattern**: **Rendezvous pattern** for synchronizing pause/resume of background goroutines.

**Tags**: `#concurrency` `#go-idioms`

---

### 4. Struct Embedding - Composition Pattern

**Location**: [leveldb/db_state.go:25](../leveldb/db_state.go#L25)

**Question**: 這是什麼寫法？

```go
type memDB struct {
    db *DB
    *memdb.DB  // <- QUESTION: 這是什麼寫法？
    ref int32
}
```

**Answer**: This is **struct embedding** in Go. It embeds `*memdb.DB` (a pointer) into `memDB`:
- `memDB` automatically inherits all methods of `*memdb.DB`
- You can call `mdb.Put()` directly instead of `mdb.DB.Put()`
- It's like inheritance but more flexible (composition over inheritance)
- The wrapper `memDB` adds the `ref` counter for lifecycle management while keeping the skip list functionality

**Tags**: `#go-idioms`

---

### 5. FrozenMem Lock - Concurrent Access Protection

**Location**: [leveldb/db_state.go:191](../leveldb/db_state.go#L191)

**Question**: 為什麼要 lock？

```go
func (db *DB) getFrozenMem() *memDB {
    db.memMu.RLock() // QUESTION: 為什麼要 lock？
    defer db.memMu.RUnlock()
    if db.frozenMem != nil {
        db.frozenMem.incref()
```

**Answer**: Protects against concurrent access to `db.frozenMem`:
- Another goroutine might be freezing the current memtable (which sets `frozenMem`)
- Or the flush goroutine might be clearing `frozenMem` after flushing to disk
- Without the lock, you could get a nil pointer or stale reference

The **RLock** (read lock) allows multiple concurrent readers but blocks writers, which is safe here since we only read `frozenMem`.

**Tags**: `#concurrency`

---

### 6. Level 0 Table Count - Write Throttling Signal

**Location**: [leveldb/db_write.go:82](../leveldb/db_write.go#L82)

**Question**: 這是什麼？

```go
tLen := db.s.tLen(0) // QUESTION: 這是什麼？
mdbFree = mdb.Free()
switch {
case tLen >= slowdownTrigger && !delayed:
    delayed = true
    time.Sleep(time.Millisecond)
```

**Answer**: `db.s.tLen(0)` returns the **number of SSTables at level 0**.

**Big picture**:
- `db.s` is the current DB version/state (tracks all SSTables)
- `tLen(level)` = table length at that level
- Level 0 is special - it's the first level where memtables get flushed
- The code checks if Level 0 has too many tables (`>= slowdownTrigger` or `>= pauseTrigger`)
- Too many L0 tables means reads get slower (have to check each one), so we slow down/pause writes to let compaction catch up

**Connection**: This is the **write throttling mechanism** that balances write throughput vs. compaction pressure.

**Tags**: `#leveldb-architecture` `#write-path` `#performance`

---

### 7. Memtable Free Space Check - Flush Decision

**Location**: [leveldb/db_write.go:88](../leveldb/db_write.go#L88)

**Question**: 還有空間所以不 flush？

```go
case mdbFree >= n: // QUESTION: 還有空間所以不 flush？
    return false
```

**Answer**: Exactly! `mdbFree >= n` means:
- The current memtable still has enough free space for this write
- So return `false` = "no need to flush yet, can write directly"

**Write throttling logic**:
1. If L0 has too many tables → slow down by sleeping or pausing
2. If memtable still has space → use it
3. Only flush memtable when it's full (`mdbFree < n`)

**Tags**: `#leveldb-architecture` `#write-path`

---

### 8. Labeled Loops - Breaking Out of Nested Control Flow

**Location**: [leveldb/db_write.go:184](../leveldb/db_write.go#L184)

**Question**: 這是什麼寫法？

```go
// QUESTION: 這是什麼寫法？
merge:
    for mergeLimit > 0 {
        select {
        // ... cases ...
        }
    }
```

**Answer**: This is a **labeled statement** in Go:
- `merge:` creates a label for the `for` loop
- Allows `break merge` or `continue merge` to control the outer loop from inside the `select`
- Without the label, `break` inside `select` only exits the `select`, not the `for` loop
- It's like a "named loop" you can reference by name

**Use case**: Breaking out of both the `select` and the `for` loop in one go.

**Tags**: `#go-idioms`

---

### 9. Batch Pool - Object Reuse Pattern

**Location**: [leveldb/db_write.go:364](../leveldb/db_write.go#L364)

**Question**: 這裡的 batchPool 是什麼？

```go
// QUESTION: 這裡的 batchPool 是什麼？
batch := db.batchPool.Get().(*Batch)
batch.Reset()
batch.appendRec(kt, key, value)
```

**Answer**: `batchPool` is a `sync.Pool` for reusing `*Batch` objects:
- Pool pattern reduces GC pressure by reusing objects instead of allocating new ones
- `Get()` retrieves a batch from the pool (or creates new if empty)
- After use, batch goes back to pool via `Put()`
- This is why `batch.Reset()` keeps the backing arrays (see Question #1)

**Big picture**: For single operations like `Put()/Delete()`, goleveldb still uses the batch machinery internally, but reuses batch objects from a pool for efficiency.

**Tags**: `#performance` `#go-idioms`

---

### 10. Key Type Constants - Deletion Markers

**Location**: [leveldb/key.go:46](../leveldb/key.go#L46)

**Question**: what is this?

```go
// Value types encoded as the last component of internal keys.
// Don't modify; this value are saved to disk.
// QUESTION: what is this?
const (
    keyTypeDel = keyType(0)
    keyTypeVal = keyType(1)
)
```

**Answer**: These are **deletion markers** in the internal key format:
- LevelDB uses **tombstones** for deletions (doesn't delete immediately)
- Each internal key has format: `[user_key][sequence_number][key_type]`
- `keyTypeDel` (0) = this key is deleted
- `keyTypeVal` (1) = this key has a value

**Why tombstones**:
- Allows deletion to be a fast write operation (just append)
- Actual removal happens during compaction
- Enables MVCC (Multi-Version Concurrency Control) with sequence numbers

**Tags**: `#leveldb-architecture`

---

## Summary by Theme

### Go Idioms
1. Slice reuse pattern (`data[:0]`) - Q1, Q9
2. Struct embedding for composition - Q4
3. Labeled loops for nested control flow - Q8
4. Object pooling with `sync.Pool` - Q9

### Concurrency
1. Channel of channels (rendezvous pattern) - Q3
2. Read/Write locks for shared state - Q5
3. Separate goroutines for different compaction types - Q2

### Performance
1. Avoiding allocations via slice reuse - Q1
2. Object pooling to reduce GC pressure - Q9
3. Write throttling based on L0 table count - Q6, Q7

### LevelDB Architecture
1. Memtable vs table compaction separation - Q2
2. Write throttling mechanism - Q6, Q7
3. Tombstones for deletion - Q10
4. Level 0 as the bottleneck for reads - Q6

---

## Follow-up Study Areas

Based on these questions, consider exploring:
1. **Compaction details**: How does table compaction actually merge SSTables?
2. **Journal format**: How are batches encoded in the WAL?
3. **Internal key format**: How do sequence numbers enable snapshots?
4. **Version management**: What is `db.s` and how does it track table versions?

---

**Related files to study next**:
- [leveldb/version.go](../leveldb/version.go) - Version/state management
- [leveldb/journal/journal.go](../leveldb/journal/journal.go) - WAL format
- [leveldb/db_compaction.go](../leveldb/db_compaction.go) - Compaction logic (full trace)
