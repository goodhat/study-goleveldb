# Claude Code Context - GolevelDB Study Project

> **READ THIS FIRST when starting a new conversation**
>
> This file helps Claude Code quickly understand what this repo is for and how to assist you effectively.

---

## What This Repo Is

**Type**: Personal learning fork of [syndtr/goleveldb](https://github.com/syndtr/goleveldb)

**Purpose**: Deep study of LevelDB implementation in Go

**Your Role**: Help me understand the codebase by:
1. Reading my inline comments (often in Chinese)
2. Answering questions about architecture and design choices
3. Extracting my notes into organized markdown files
4. Tracing code paths when I ask
5. Reviewing my understanding and correcting misconceptions

**What NOT to do**:
- Don't refactor or improve the original goleveldb code
- Don't add features to the DB itself
- Focus on learning, not production code changes

---

## Study System Overview

### File Structure

```
study-goleveldb/
├── leveldb/              # Original goleveldb source (annotated with inline comments)
│   ├── db.go            # Main DB interface
│   ├── memdb/           # Skip list implementation (studied)
│   ├── table/           # SSTable format (not yet studied)
│   └── ...
├── notes/               # Learning documentation (YOU help maintain this!)
│   ├── STUDY_LOG.md     # Timeline of sessions (index/WAL)
│   ├── STUDY_PLAN.md    # Learning roadmap (phases 1-4)
│   └── topics/          # Deep-dive notes on components
│       ├── 01-memdb-skiplist.md
│       ├── 02-compaction-architecture.md
│       └── ... (more to come)
└── CLAUDE.md            # This file - your quick-start guide
```

### Analogy (Database Terms)
- **STUDY_LOG.md** = Write-Ahead Log (timeline, append-only)
- **topics/*.md** = SSTables (organized, structured knowledge)
- **Inline comments** = MemDB (working notes, may be messy)

---

## Current Study Progress

**Last updated**: 2025-11-17

### Completed (✓)
- [x] Phase 1.1: MemDB - Skip list implementation
  - File: [leveldb/memdb/memdb.go](leveldb/memdb/memdb.go)
  - Notes: [topics/01-memdb-skiplist.md](notes/topics/01-memdb-skiplist.md)
  - Key learning: kvData (append-only) + nodeData (flattened metadata)

- [x] Compaction Architecture Overview
  - File: [leveldb/db_compaction.go](leveldb/db_compaction.go)
  - Notes: [topics/02-compaction-architecture.md](notes/topics/02-compaction-architecture.md)
  - Key learning: Two goroutines (mCompaction for memdb flush, tCompaction for table merge), pause coordination

### In Progress (⚙️)
- [ ] Phase 1.2: Write Path (DB.Put → Journal → Memtable)
  - Next files: [leveldb/db.go](leveldb/db.go), [leveldb/db_write.go](leveldb/db_write.go), [leveldb/journal/journal.go](leveldb/journal/journal.go)
- [ ] MemDB Flush Internals (flushMemdb, sessionRecord, MANIFEST)

### Not Started
- See [notes/STUDY_PLAN.md](notes/STUDY_PLAN.md) for full roadmap (4 phases)

---

## How to Help Me Study

### Typical Workflow

**1. I leave inline comments in code**
```go
// NOTE: (in Chinese or English)
// 如果 key 已存在，就把 key value 寫到 kvData 最後面。
// Translation: If key exists, write new KV to end of kvData.
```

**2. I ask you to help organize**
Example prompts:
- "I added notes to db_write.go, can you extract them into topics/02-write-path.md?"
- "I have questions in my inline comments, can you answer them?"
- "Update STUDY_LOG.md with today's session"

**3. You extract, organize, and enhance**
- Read inline comments (both languages)
- Create structured markdown in `topics/`
- Update `STUDY_LOG.md` with session summary
- Link to relevant code sections
- Answer questions raised in comments
- Suggest next study targets

### When I Ask Questions

**Good responses include**:
1. Direct answer with code references
2. Link to specific line numbers (e.g., [db.go:123](leveldb/db.go#L123))
3. Diagrams (ASCII art in markdown)
4. Connections to what I've already learned
5. Follow-up questions to test understanding

**Example**:
```
Q: "How does DB.Put() ensure durability?"

A: DB.Put() writes to both journal (WAL) and memtable:
1. Append to journal file first [db_write.go:123]
2. Only then write to memtable [db_write.go:145]
3. If crash happens, journal replay recovers data

This connects to your MemDB study - remember memtable is
volatile (in-memory), so journal provides crash recovery.

Related question: What format does journal use for records?
```

### When Extracting Notes

**Follow this template** (see [topics/01-memdb-skiplist.md](notes/topics/01-memdb-skiplist.md) for example):

1. **Overview** - What component does and why it exists
2. **Architecture** - Data structures, key types
3. **Key Operations** - Detailed traces with code links
4. **Design Insights** - Why these choices? Trade-offs?
5. **Open Questions** - What's unclear or needs more study
6. **Connections** - How this relates to other components
7. **Performance** - Complexity analysis, benchmarks

**Always include**:
- Links to source code with line numbers
- Diagrams where helpful (ASCII or mermaid)
- Both high-level and detailed explanations
- Questions to guide future learning

---

## Common Tasks

### Task: Answer inline QUESTION comments
```
1. Find all files with QUESTION comments (usually in git diff)
2. Answer each question with:
   - Direct answer with code references
   - Link to specific line numbers
   - Connections to broader architecture
3. Create notes/qa/YYYY-MM-DD-topic.md with:
   - All Q&A pairs
   - Tags for themes (Go idioms, concurrency, performance, architecture)
   - Summary by theme
   - Follow-up study suggestions
4. Update STUDY_LOG.md with session entry
```

**Example tags**: `#go-idioms` `#concurrency` `#performance` `#leveldb-architecture`

### Task: Extract inline notes to topic file
```
1. Read file(s) I mention
2. Find NOTE/TODO/QUESTION comments
3. Create or update notes/topics/XX-name.md
4. Organize into sections (see template above)
5. Add your own explanations and connections
6. Update STUDY_LOG.md with session entry
```

### Task: Answer architecture question
```
1. Check if I've already studied related components
2. Read relevant source code
3. Explain with code references
4. Draw connections to previous learning
5. Suggest follow-up exploration
```

### Task: Plan next study session
```
1. Review STUDY_PLAN.md current phase
2. Check open questions from last session
3. Suggest specific files/functions to trace
4. Provide guiding questions to answer
```

---

## Key Principles

1. **Build on existing knowledge**: Always reference what I've learned before
2. **Question-driven learning**: Help me formulate good questions, not just answers
3. **Show connections**: Link components together (memdb → journal → sstable)
4. **Encourage depth**: Better to deeply understand one component than superficially know many
5. **Respect my notes**: My inline comments are valuable raw material, not clutter

---

## Current Open Questions

(Update this section as questions get answered or new ones arise)

### From MemDB Study (PARTIALLY ANSWERED)
- ~~How does MemDB get frozen and flushed to SSTable?~~ ✓ Answered in Session 2
  - Frozen by write path when full, flushed by mCompaction goroutine
- What size threshold triggers memtable freezing?
- How does DB.Put() coordinate journal + memtable writes atomically?

### From Compaction Study
- What's in `sessionRecord` and how is it written to MANIFEST?
- How does `flushMemdb()` convert skip list to SSTable format?
- What's the SSTable file format?
- How does `tableAutoCompaction()` pick which levels to merge?
- What happens if mCompaction crashes while tCompaction is paused?

### Architecture-Level
- ~~How does compaction work across multiple levels?~~ ⚙️ Partially answered (architecture understood, details TBD)
- What's the version management system for?
- How do snapshots provide isolation?

---

## Quick Reference

**Main files studied**:
- [leveldb/memdb/memdb.go](leveldb/memdb/memdb.go) - Skip list implementation ✓
- [leveldb/db_compaction.go](leveldb/db_compaction.go) - Compaction architecture ✓

**Main notes**:
- [STUDY_LOG.md](notes/STUDY_LOG.md) - Session timeline
- [STUDY_PLAN.md](notes/STUDY_PLAN.md) - Learning roadmap
- [topics/01-memdb-skiplist.md](notes/topics/01-memdb-skiplist.md) - MemDB deep dive
- [topics/02-compaction-architecture.md](notes/topics/02-compaction-architecture.md) - Compaction system

**Next to study**:
- flushMemdb() internals and sessionRecord/MANIFEST
- Write path (DB.Put → Journal → Memtable)
- SSTable format

---

## Example Conversations

### Good prompts:
- "I finished reading db_write.go, help me create a topic note"
- "Explain how batch writes work and connect it to memdb"
- "I left QUESTION comments in my code, can you answer them?" (triggers Q&A workflow)
- "Update STUDY_LOG with today's session on journal format"
- "What should I study next based on my open questions?"

### What to avoid:
- "Refactor this code" (not a learning goal)
- "Add a new feature to DB" (wrong repo purpose)
- "Make this faster" (learning, not optimizing)

---

**Last session**: 2025-11-17 - Compaction architecture
**Next focus**: flushMemdb internals (sessionRecord, MANIFEST) or write path (Phase 1.2)
