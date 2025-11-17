# Study Notes - Quick Start

> **New to this study system?** Read [CLAUDE.md](../CLAUDE.md) first!

---

## How to Use These Notes

### For Me (the Student)

**Daily workflow**:
1. Study code, leave inline comments (Chinese/English, whatever is fast)
2. When done with a session, ask Claude to:
   - Extract notes into `topics/XX-name.md`
   - Update `STUDY_LOG.md` with session summary
   - Answer questions from comments
3. Review organized notes, adjust as needed
4. Plan next session using open questions

### For Claude

**Your job**:
- Read my inline comments (both languages)
- Extract into structured topic notes
- Answer questions I raise
- Make connections to previous learning
- Suggest next steps

See [CLAUDE.md](../CLAUDE.md) for detailed instructions.

---

## File Organization

### STUDY_LOG.md (Timeline / WAL)
- Chronological session entries
- Quick summaries and cross-references
- Like a write-ahead log: append-only, shows history

### topics/*.md (Knowledge Base / SSTables)
- One file per major component or concept
- Deep, structured explanations
- Like SSTables: organized, immutable snapshots

### STUDY_PLAN.md (Roadmap)
- Phases and learning objectives
- Progress tracking
- Next targets

---

## Naming Convention for Topics

```
topics/
├── 01-memdb-skiplist.md          # Component deep-dive
├── 02-write-path-journal.md      # Data flow / pathway
├── 03-sstable-format.md          # File format
├── 04-compaction-levels.md       # Process/mechanism
└── ...
```

**Pattern**: `NN-short-descriptive-name.md`
- NN = rough learning order (01, 02, 03...)
- Name should be searchable and self-explanatory

---

## Templates

### Topic Note Template

See [topics/01-memdb-skiplist.md](topics/01-memdb-skiplist.md) as reference.

**Sections**:
1. Overview (what & why)
2. Architecture (data structures)
3. Key Operations (detailed traces)
4. Design Insights (trade-offs)
5. Open Questions
6. Connections to other components
7. Performance characteristics

### Session Log Template

See [STUDY_LOG.md](STUDY_LOG.md) template section.

**Fields**:
- Duration, components studied, detailed notes link
- Quick summary (3-5 bullets)
- Key insights (design lessons)
- Questions for next session
- Status checklist

---

## Tips for Effective Study Notes

### Good inline comments
```go
// NOTE:
// 為什麼要用兩個 array 分開存？
// - kvData: 減少 pointer，降低 GC 壓力
// - nodeData: 搜尋時不用載入 value，cache 友善
```

**Why good**: Asks "why", shows thinking, bilingual OK

### Good questions for Claude
- "Why does Put() need exclusive lock but Get() only needs read lock?"
- "How does this connect to what I learned about skip list?"
- "What are the trade-offs of append-only design?"

### Less useful
- "What does this code do?" (read the code first!)
- "Summarize this file" (too vague)

---

## Progress Tracking

| Phase | Component | Status | Notes |
|-------|-----------|--------|-------|
| 1.1 | MemDB Skip List | ✓ Done | topics/01-memdb-skiplist.md |
| 1.2 | Write Path | 🎯 Next | - |
| 1.3 | Read Path | ⏳ Later | - |
| 2.1 | SSTable Format | ⏳ Later | - |
| ... | ... | ... | ... |

See [STUDY_PLAN.md](STUDY_PLAN.md) for complete roadmap.

---

## FAQ

**Q: Should I write notes in English or Chinese?**
A: Use whatever helps you think faster! Claude understands both. Be consistent within each note for readability.

**Q: How detailed should inline comments be?**
A: Capture key insights and questions. Save deep analysis for topic notes (Claude will help organize).

**Q: What if I realize my earlier understanding was wrong?**
A: Great! Update the topic note with a "Corrections" section. Learning is iterative.

**Q: Should I update STUDY_LOG myself or ask Claude?**
A: Either works. Claude can save time by extracting from your inline comments.

---

**Happy studying!** 🚀
