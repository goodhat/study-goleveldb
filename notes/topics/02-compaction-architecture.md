# Compaction Architecture

> **Component**: Two-level compaction system (memdb flush + table merge)
> **Files**:
> - [leveldb/db_compaction.go](../../leveldb/db_compaction.go)
> - [leveldb/db.go](../../leveldb/db.go) (channel definitions)
> **Studied**: 2025-11-17
> **Status**: 🔄 In Progress

---

## Overview

LevelDB has **two independent compaction goroutines** running in parallel:

1. **mCompaction** - Flushes frozen memtable to Level 0 SSTables
2. **tCompaction** - Merges SSTables across levels (L0 → L1 → L2...)

These run concurrently but coordinate to avoid I/O contention.

---

## Architecture

### Two Goroutines, Two Channels

```go
// Defined in db.go:77-79
tcompCmdC chan cCmd  // Table compaction (SSTable merging)
tcompPauseC chan (chan<- struct{})  // Pause mechanism
mcompCmdC chan cCmd  // MemDB compaction (memtable flush)
```

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      DB Write Path                           │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
                ▼                             ▼
          ┌──────────┐                  ┌──────────┐
          │  MemDB   │                  │ Journal  │
          │ (active) │                  │  (WAL)   │
          └────┬─────┘                  └──────────┘
               │ (full)
               ▼
          ┌──────────┐
          │  MemDB   │
          │ (frozen) │
          └────┬─────┘
               │
               │ trigger mcompCmdC
               ▼
      ┌────────────────────┐
      │  mCompaction()     │ ← Goroutine 1
      │  (goroutine)       │
      └────────┬───────────┘
               │
               │ 1. Pause tCompaction
               │ 2. Flush to L0 SSTable
               │ 3. Resume tCompaction
               ▼
      ┌────────────────────┐
      │   Level 0          │
      │  (SSTables)        │
      └────────┬───────────┘
               │
               │ trigger tcompCmdC
               ▼
      ┌────────────────────┐
      │  tCompaction()     │ ← Goroutine 2
      │  (goroutine)       │
      └────────┬───────────┘
               │
               │ Merge L0→L1→L2...
               ▼
      ┌────────────────────┐
      │   Level 1-6        │
      │  (SSTables)        │
      └────────────────────┘
```

---

## Command Types

### Interface Definition

```go
// db_compaction.go:681-712

type cCmd interface {
    ack(err error)  // Send response back to caller
}

type cAuto struct {
    ackC chan<- error  // If nil = fire-and-forget, else = blocking
}

type cRange struct {
    level    int          // Target level (-1 = auto)
    min, max []byte       // Key range to compact
    ackC     chan<- error
}
```

### Trigger Functions

**Fire-and-forget** (L714-720):
```go
func (db *DB) compTrigger(compC chan<- cCmd) {
    select {
    case compC <- cAuto{}:  // Non-blocking send
    default:
    }
}
```

**Blocking** (L723-742):
```go
func (db *DB) compTriggerWait(compC chan<- cCmd) (err error) {
    ch := make(chan error)
    defer close(ch)

    // Send command with response channel
    select {
    case compC <- cAuto{ch}:
    case err = <-db.compErrC:  // Worker died
        return
    case <-db.closeC:          // DB closing
        return ErrClosed
    }

    // Wait for response
    select {
    case err = <-ch:           // Got response
    case err = <-db.compErrC:
    case <-db.closeC:
        return ErrClosed
    }
    return err
}
```

**Key pattern**: Other cases in `select` handle failure scenarios (worker died, DB closing) to avoid blocking forever.

---

## mCompaction Goroutine

### Main Loop

**Location**: [db_compaction.go:766-796](../../leveldb/db_compaction.go#L766-L796)

```go
func (db *DB) mCompaction() {
    var x cCmd
    defer func() { /* cleanup */ }()

    for {
        select {
        case x = <-db.mcompCmdC:  // ← Receive command
            switch x.(type) {
            case cAuto:
                db.memCompaction()  // Do the flush
                x.ack(nil)          // Acknowledge
                x = nil
            default:
                panic("leveldb: unknown command")
            }
        case <-db.closeC:
            return
        }
    }
}
```

### The Flush Process: `memCompaction()`

**Location**: [db_compaction.go:269-351](../../leveldb/db_compaction.go#L269-L351)

#### Step-by-Step Breakdown

**1. Get frozen memdb** (L270-275)
```go
mdb := db.getFrozenMem()
if mdb == nil {
    return  // Nothing to flush
}
defer mdb.decref()  // Reference counting
```

**2. Skip empty memdb** (L278-284)
```go
if mdb.Len() == 0 {
    db.dropFrozenMem()
    return
}
```

**3. Pause table compaction** (L286-295) ⭐
```go
resumeC := make(chan struct{})
select {
case db.tcompPauseC <- (chan<- struct{})(resumeC):
    // Successfully sent pause request to tCompaction
case <-db.compPerErrC:
    close(resumeC)
    resumeC = nil
case <-db.closeC:
    db.compactionExitTransact()
}
```

**Why pause?**
- Prevents I/O contention between flush and table compaction
- Gives memdb flush **priority** (frees memory, unblocks writes)
- tCompaction goroutine receives on `tcompPauseC` and stops working

**Communication protocol**:
```
mCompaction                    tCompaction
     |                              |
     |---- tcompPauseC ------------>| (receives pauseC channel)
     |   (send resumeC)              |
     |                              | pauseCompaction(resumeC)
     |<--- resumeC ----------------| (sends ack)
     |                              |
     | (now safe to flush)          | (PAUSED)
```

**4. Flush to SSTable** (L297-318)
```go
var (
    rec        = &sessionRecord{}  // Metadata about new SSTable
    stats      = &cStatStaging{}   // Performance tracking
    flushLevel int                 // Which level (usually 0)
)

db.compactionTransactFunc("memdb@flush",
    func(cnt *compactionTransactCounter) (err error) {
        stats.startTimer()
        flushLevel, err = db.s.flushMemdb(rec, mdb.DB, db.memdbMaxLevel)
        stats.stopTimer()
        return
    },
    func() error {  // Rollback on failure
        for _, r := range rec.addedTables {
            db.logf("memdb@flush revert @%d", r.num)
            db.s.stor.Remove(storage.FileDesc{
                Type: storage.TypeTable,
                Num: r.num,
            })
        }
        return nil
    })
```

**Key operation**: `flushMemdb()` iterates the skip list (sorted order!) and writes KV pairs to a new SSTable file.

**Transaction pattern**: Second function provides rollback if flush fails mid-way.

**5. Commit to MANIFEST** (L320-326)
```go
rec.setJournalNum(db.journalFd.Num)
rec.setSeqNum(db.frozenSeq)

stats.startTimer()
db.compactionCommit("memdb", rec)  // Write to MANIFEST
stats.stopTimer()
```

**Critical**: MANIFEST records "SSTable X added at level Y". After this, data is durable.

**6. Update stats** (L330-336)
```go
for _, r := range rec.addedTables {
    stats.write += r.size
}
db.compStats.addStat(flushLevel, stats)
atomic.AddUint32(&db.memComp, 1)
```

**7. Drop frozen memdb** (L338-339)
```go
db.dropFrozenMem()  // Free memory!
```

Now safe because data is on disk and recorded in MANIFEST.

**8. Resume table compaction** (L341-348)
```go
if resumeC != nil {
    select {
    case <-resumeC:         // Wait for tCompaction to ack pause
        close(resumeC)      // Signal resume
    case <-db.closeC:
        db.compactionExitTransact()
    }
}
```

**Protocol**:
- Wait for tCompaction to send on `resumeC` (confirms paused)
- Close `resumeC` to signal "resume now"

**9. Trigger table compaction** (L350)
```go
db.compTrigger(db.tcompCmdC)
```

L0 might be full now, so notify tCompaction to check if merge needed.

---

## tCompaction Goroutine

### Main Loop

**Location**: [db_compaction.go:798-874](../../leveldb/db_compaction.go#L798-L874)

```go
func (db *DB) tCompaction() {
    var (
        x     cCmd
        waitQ []cCmd  // Queue for blocked writers
    )
    defer func() { /* cleanup waitQ */ }()

    for {
        if db.tableNeedCompaction() {
            select {
            case x = <-db.tcompCmdC:
            case ch := <-db.tcompPauseC:  // ← Can be paused!
                db.pauseCompaction(ch)
                continue
            case <-db.closeC:
                return
            default:  // Non-blocking when compaction needed
            }
            // Handle waitQ...
        } else {
            // Block when no compaction needed
            select {
            case x = <-db.tcompCmdC:
            case ch := <-db.tcompPauseC:
                db.pauseCompaction(ch)
                continue
            case <-db.closeC:
                return
            }
        }

        if x != nil {
            switch cmd := x.(type) {
            case cAuto:
                if cmd.ackC != nil {
                    if db.resumeWrite() {
                        x.ack(nil)
                    } else {
                        waitQ = append(waitQ, x)  // Wait for L0 to shrink
                    }
                }
            case cRange:
                x.ack(db.tableRangeCompaction(cmd.level, cmd.min, cmd.max))
            }
            x = nil
        }

        db.tableAutoCompaction()  // Actually do compaction!
    }
}
```

### Key Differences from mCompaction

1. **Handles two command types**: `cAuto` and `cRange`
2. **Can be paused**: By mCompaction via `tcompPauseC`
3. **Manages write backpressure**: `waitQ` holds blocked writers waiting for L0 to shrink
4. **Non-blocking when needed**: `default` case when compaction is needed

### Pause Handling

**Location**: [db_compaction.go:673-679](../../leveldb/db_compaction.go#L673-L679)

```go
func (db *DB) pauseCompaction(ch chan<- struct{}) {
    select {
    case ch <- struct{}{}:  // Acknowledge pause request
    case <-db.closeC:
        db.compactionExitTransact()
    }
}
```

This is the receiving side of the pause protocol initiated by mCompaction.

---

## When Are Compactions Triggered?

### mcompCmdC Triggers (MemDB Flush)

From **write path** when memtable is full:

1. **Blocking** - [db_write.go:39](../../leveldb/db_write.go#L39)
   ```go
   err = db.compTriggerWait(db.mcompCmdC)
   ```
   Wait for flush to complete before proceeding.

2. **Blocking** - [db_write.go:59](../../leveldb/db_write.go#L59)
   ```go
   err = db.compTriggerWait(db.mcompCmdC)
   ```
   After rotating memdb (old → frozen, new → active).

3. **Async** - [db_write.go:61](../../leveldb/db_write.go#L61)
   ```go
   db.compTrigger(db.mcompCmdC)
   ```
   Fire-and-forget trigger.

### tcompCmdC Triggers (Table Compaction)

From **multiple places**:

1. **After memdb flush** - [db_compaction.go:350](../../leveldb/db_compaction.go#L350)
   ```go
   db.compTrigger(db.tcompCmdC)
   ```
   L0 might have too many files now.

2. **Write path backpressure** - [db_write.go:94](../../leveldb/db_write.go#L94)
   ```go
   err = db.compTriggerWait(db.tcompCmdC)
   ```
   Too many L0 files - block writes until compaction shrinks L0.

3. **After DB open** - [db_state.go:70](../../leveldb/db_state.go#L70)
   ```go
   db.compTrigger(db.tcompCmdC)
   ```
   Check if compaction needed after opening.

4. **After recovery** - [db.go:814, 852](../../leveldb/db.go#L814)
   ```go
   db.compTrigger(db.tcompCmdC)
   ```
   Post-recovery cleanup.

---

## Comparison Table

| Aspect | mCompaction | tCompaction |
|--------|-------------|-------------|
| **Channel** | `mcompCmdC` | `tcompCmdC` |
| **Goroutine** | `mCompaction()` L766 | `tCompaction()` L798 |
| **Main function** | `memCompaction()` L269 | `tableAutoCompaction()` L654 |
| **Purpose** | Flush frozen memdb → L0 | Merge SSTables L0→L1→... |
| **Priority** | High (blocks writes) | Low (background) |
| **Can pause?** | No | Yes (by mCompaction) |
| **Commands** | `cAuto` only | `cAuto`, `cRange` |
| **Triggered by** | Write path (memdb full) | L0 full, level exceeded, etc. |
| **Blocking?** | Usually yes | Sometimes (backpressure) |

---

## Key Design Insights

### 1. Separation of Concerns

Two goroutines handle different problems:
- **mCompaction**: Memory management (free frozen memdb)
- **tCompaction**: Disk organization (keep levels balanced)

### 2. Priority Management

mCompaction can **pause** tCompaction because:
- Memdb flush is urgent (frees memory, unblocks writes)
- Table compaction is optimization (can wait)
- Prevents I/O thrashing

### 3. Coordinated I/O

Pause mechanism prevents both goroutines from:
- Competing for disk I/O
- Thrashing the filesystem cache
- Degrading write performance

### 4. Write Backpressure

When L0 has too many files:
- Writers get blocked on `compTriggerWait(tcompCmdC)`
- tCompaction shrinks L0 via merging
- `waitQ` holds blocked writers
- Writers resume when L0 drops below threshold

### 5. Failure Handling

The `select` pattern with multiple cases handles:
- Worker goroutine crashes (`compErrC`)
- DB shutdown (`closeC`)
- Normal operation (compaction channel)

Prevents indefinite blocking when receiver is dead.

---

## Connection to Previous Study

### From MemDB Study (Session 1)

Remember the **frozen memtable** question? Now answered!

1. **When frozen?** When active memdb fills up (write path)
2. **What happens?** `mCompaction` goroutine flushes it to L0 SSTable
3. **How flushed?** Iterate skip list in sorted order → write to SSTable
4. **Why sorted matters?** Skip list is already sorted, so no sorting needed!

### Skip List → SSTable Connection

```
MemDB (in memory)              SSTable (on disk)
┌────────────────┐            ┌────────────────┐
│ Skip List      │            │ Sorted KV file │
│ (sorted)       │  flush     │ (sorted)       │
│                │ ────────>  │                │
│ k1:v1          │            │ k1:v1          │
│ k2:v2          │            │ k2:v2          │
│ k3:v3          │            │ k3:v3          │
└────────────────┘            └────────────────┘
```

Direct serialization - no sorting needed because skip list maintains order!

---

## Open Questions

### About sessionRecord (`rec`)

- What exactly is stored in `sessionRecord`?
- How is it written to MANIFEST file?
- What's the format?
- → **Need to study**: MANIFEST and version management

### About flushMemdb()

- How does it iterate the skip list?
- What's the SSTable file format?
- How are keys/values encoded?
- → **Need to study**: SSTable format (Phase 2)

### About flushLevel

- Why not always L0?
- What determines higher levels?
- What's `memdbMaxLevel`?
- → **Need to study**: Level selection logic

### About tableAutoCompaction()

- How are levels picked for compaction?
- What's the merging algorithm?
- How does it avoid overlapping files?
- → **Need to study**: Compaction strategy (Phase 4)

### About Pause Protocol

- What if mCompaction crashes while tCompaction is paused?
- Is there a timeout?
- Can multiple pauses queue up?
- → **Need to trace**: Error handling paths

### About waitQ

- What conditions add to waitQ?
- When are they released?
- What if DB closes with pending waitQ?
- → **Need to study**: Write backpressure logic

---

## Next Study Targets

Based on this session:

1. **sessionRecord and MANIFEST** - How metadata is persisted
2. **flushMemdb() internals** - Skip list → SSTable conversion
3. **SSTable format** - File structure, encoding
4. **tableAutoCompaction()** - Level selection and merging
5. **Write path coordination** - How Put() ties into this

---

## Code References

### Main Functions
- `mCompaction()` - [db_compaction.go:766-796](../../leveldb/db_compaction.go#L766-L796)
- `tCompaction()` - [db_compaction.go:798-874](../../leveldb/db_compaction.go#L798-L874)
- `memCompaction()` - [db_compaction.go:269-351](../../leveldb/db_compaction.go#L269-L351)
- `tableAutoCompaction()` - [db_compaction.go:654-658](../../leveldb/db_compaction.go#L654-L658)

### Trigger Functions
- `compTrigger()` - [db_compaction.go:714-720](../../leveldb/db_compaction.go#L714-L720)
- `compTriggerWait()` - [db_compaction.go:723-742](../../leveldb/db_compaction.go#L723-L742)
- `compTriggerRange()` - [db_compaction.go:745-764](../../leveldb/db_compaction.go#L745-L764)

### Command Types
- `cCmd` interface - [db_compaction.go:681-683](../../leveldb/db_compaction.go#L681-L683)
- `cAuto` struct - [db_compaction.go:685-697](../../leveldb/db_compaction.go#L685-L697)
- `cRange` struct - [db_compaction.go:699-712](../../leveldb/db_compaction.go#L699-L712)

### Channel Definitions
- Channel declarations - [db.go:77-79](../../leveldb/db.go#L77-L79)
- Channel initialization - [db.go:112-114](../../leveldb/db.go#L112-L114)

---

## Inline Comments Found

From `db.go:79`:
```go
mcompCmdC chan cCmd // QUESTION: tcomp 和 mcomp 的差別是什麼？
```
**Answer**: Now documented in this note! Two separate compaction systems.

From `db_compaction.go:289`:
```go
case db.tcompPauseC <- (chan<- struct{})(resumeC): // QUESTION: 這要傳給誰？不會被 block 嗎？
```
**Answer**: Sends to tCompaction goroutine. Won't block because tCompaction always listens on `tcompPauseC` in its main loop.

---

## Study Session Notes

**Date**: 2025-11-17
**Duration**: ~1.5 hours
**Approach**: Bottom-up - started with `compTriggerWait()`, traced to goroutine loops
**Discoveries**:
- Two independent compaction systems
- Pause coordination mechanism
- Channel-based command pattern with acknowledgment
- Write backpressure via waitQ

**Feeling**: Exciting but overwhelming - many new questions emerged. Decided to capture current understanding before diving deeper.
