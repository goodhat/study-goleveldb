# Study Workflow Guide

> **How to use Claude Code effectively while studying goleveldb**

---

## Typical Study Session

### 1. Before Studying (Setup)

**Start new Claude Code conversation**:
```
Hey Claude, starting a new study session. Please read CLAUDE.md to understand the context.
```

Claude will understand:
- This is a learning project
- You leave inline comments
- You need help organizing notes

---

### 2. During Study (Active Learning)

**Leave inline comments freely**:
```go
// NOTE: 我的理解
// 這裡用 RWMutex 是因為 prevNode 是 shared state
// Get() 不需要 prevNode 所以可以用 RLock

// QUESTION: 為什麼不用 sync.Map？
```

**Don't worry about**:
- Perfect grammar or formatting
- Mixing languages (Chinese + English is fine!)
- Writing too much (you'll organize later)

**Do capture**:
- Your understanding ("I think this works because...")
- Questions ("Why not use X instead?")
- Connections ("This is like Y component...")
- Aha moments ("Oh! This explains Z!")

---

### 3. After Study (Organizing)

**Ask Claude to help**:

#### Option A: Extract to topic note
```
I finished studying db_write.go and left lots of comments.
Can you:
1. Read my inline comments in db_write.go
2. Create notes/topics/02-write-path.md following the template
3. Answer any questions I raised in comments
4. Update STUDY_LOG.md with today's session
```

#### Option B: Just answer questions
```
I have some questions in my comments at leveldb/db.go:450-500.
Can you explain those parts?
```

#### Option C: Update progress
```
I finished studying the write path. Update STUDY_LOG.md and
suggest what I should study next based on my open questions.
```

---

## Common Patterns

### Pattern: "I don't understand X"

**Bad approach**: 🚫
```
Ask Claude immediately → Get answer → Move on
```
Problem: Passive learning, low retention.

**Better approach**: ✅
```
1. Read code carefully
2. Form hypothesis in inline comment
3. Test hypothesis (add prints, use debugger, check tests)
4. If still confused, ask Claude with context:

   "I think X works like Y because Z. But I'm confused about
   the case when W happens. Here's my comment at line 123..."
```

Claude can then:
- Confirm or correct your hypothesis
- Explain the W case
- Show you where in code to verify

### Pattern: "This design seems weird"

**Leave observation**:
```go
// NOTE: 奇怪的設計？
// 為什麼 Put() 更新 key 時不 reuse 舊的空間，
// 而是 append 到 kvData 後面？會造成 fragmentation！
//
// 可能原因：
// 1. 簡化實作（不用管理 free list）
// 2. 避免移動資料（iterator 可能正在讀）
// 3. Memtable 是短暫的，很快就 flush 到 SSTable
```

**Then ask Claude**:
```
I left notes about the append-only design in memdb.go.
Am I understanding the trade-offs correctly?
What did I miss?
```

Claude will:
- Validate your reasoning
- Add insights you missed
- Show how this connects to compaction/flush

### Pattern: "I want to see the big picture"

**After studying 2-3 related components**:
```
I've now studied:
1. MemDB (skip list)
2. Journal (WAL)
3. DB.Put() (write coordination)

Can you:
1. Create a data flow diagram showing how these connect
2. Explain the overall write path from user's Put() to disk
3. Add this to a topic note (maybe 02-write-path-overview.md)
```

---

## Asking Good Questions

### Question Quality Spectrum

**Level 1** (Basic): ❓
```
"What does this function do?"
```
Better: Read code first, then ask specific parts.

**Level 2** (Specific): ❓❓
```
"Why does findGE() take a 'prev' parameter?"
```
Better: Shows you identified something interesting.

**Level 3** (Analytical): ❓❓❓
```
"findGE() populates prevNode when prev=true. I see this is
needed for Put() to wire up forward pointers. But why is
prevNode a shared array instead of a local variable?"
```
Best: Shows you understand the mechanism, asking about design.

**Level 4** (Synthesis): ❓❓❓❓
```
"I noticed both MemDB and SSTable use separate key/value storage.
MemDB has kvData/nodeData split, and according to the table
format, SSTables also separate data blocks from index blocks.
Is this a common LSM pattern? What's the fundamental reason?"
```
Excellent: Makes connections across components, identifies patterns.

---

## Working with Inline Comments

### When to Use Each Language

**Chinese** (faster for you):
- Quick observations
- Internal reasoning
- Questions to yourself

**English**:
- Technical terms (easier to search)
- When you want Claude to read directly
- Formal documentation

**Mix freely**:
```go
// NOTE:
// Skip list 的 height 是隨機決定的 (randHeight)
// Branching factor = 4, 所以每層有 25% 機會往上
// Max height = 12 (tMaxHeight constant)
//
// QUESTION: Why 4? Why not 2 (like binary tree)?
// Hypothesis: Balance between search speed and space overhead.
```

### Comment Markers to Use

```go
// NOTE: 我的理解 / observation

// QUESTION: 問題 / something unclear

// ANSWERED: 問題已解決 / question resolved (put answer here)

// TODO: 要研究的 / to study later

// CONNECT: 與其他 component 的關係 / relationship to other parts
```

Claude will recognize these markers and organize accordingly.

---

## Example Full Session

### Session: Studying Write Path

**1. Start (5 min)**
```
Tell Claude: "Starting write path study, please read CLAUDE.md"
```

**2. Study db_write.go (60 min)**
- Read code
- Add inline comments with NOTE/QUESTION markers
- Trace execution path
- Maybe use debugger to step through

**3. Organize with Claude (15 min)**
```
Prompt:
"I finished db_write.go. Here's what I want:

1. Create notes/topics/02-write-path.md covering:
   - How DB.Put() coordinates journal + memtable
   - Batch write mechanism
   - Error handling

2. Answer my QUESTIONs from inline comments

3. Update STUDY_LOG.md with:
   - Session summary
   - Key insights about write coordination
   - Questions for next session (like: when does memtable freeze?)

4. Suggest what to study next based on open questions"
```

**4. Review (10 min)**
- Read Claude's organized notes
- Correct any misunderstandings
- Add your own thoughts

**5. Commit (2 min)**
```bash
git add notes/ leveldb/
git commit -m "study: write path and journal coordination

- Understood batch write mechanism
- Learned WAL ensures durability before memtable write
- Questions: memtable freezing threshold?

notes: topics/02-write-path.md
"
```

**Total**: ~90 min focused study + organized notes!

---

## Tips for Effective Collaboration with Claude

### Do:
- ✅ Read CLAUDE.md at start of each conversation
- ✅ Be specific about what you want extracted
- ✅ Ask "why" questions, not just "what"
- ✅ Request connections to previous learning
- ✅ Ask for diagrams when helpful
- ✅ Tell Claude your hypothesis before asking for answer

### Don't:
- 🚫 Expect Claude to read your mind - be explicit
- 🚫 Ask to refactor/optimize goleveldb code (wrong goal)
- 🚫 Accept answers without understanding - push back!
- 🚫 Skip updating CLAUDE.md progress (it helps Claude help you)

---

## Workflow Optimizations

### Using Code References

When asking Claude about specific code:
```
"Explain the logic at db.go:450-480, especially the part
where it checks if memtable is full."
```

Claude will read that section and explain with context.

### Batch Questions

Instead of asking one at a time:
```
"I have 3 questions from studying session.go:
1. [Line 50] Why does session need a mutex?
2. [Line 120] What's the difference between version and manifest?
3. [Line 200] How does file numbering work?

Can you answer all three and show how they relate?"
```

### Request Specific Formats

```
"Create a sequence diagram showing the write path from
DB.Put() → batch → journal → memtable → freeze → compact"

"Make a table comparing Get() vs Put() locking strategy"

"Draw an ASCII diagram of the SSTable block structure"
```

---

## Troubleshooting

### "Claude doesn't understand my context"
→ Make sure you said "read CLAUDE.md" at start of conversation

### "My inline comments are getting messy"
→ That's OK! Leave them messy. Ask Claude to organize later.

### "I'm not sure what to study next"
→ Check STUDY_PLAN.md or ask Claude based on open questions in STUDY_LOG.md

### "Claude's explanation is too abstract"
→ Ask for concrete code examples or specific line references

### "I think Claude's explanation is wrong"
→ Great! Trust your understanding. Ask Claude to justify or show counter-example.

---

## Advanced Techniques

### Active Recall
After Claude creates topic note, close it and try to:
1. Explain component to yourself
2. Draw diagram from memory
3. Compare with notes - what did you miss?

### Spaced Repetition
Revisit old topic notes after 1 week, 1 month:
```
"I studied MemDB 2 weeks ago (topics/01-memdb-skiplist.md).
Quiz me on key concepts to check retention."
```

### Cross-Component Analysis
After finishing Phase 1:
```
"I've now studied memdb, journal, and write path.
Create a comparison table showing:
- What each provides (durability, performance, etc.)
- Trade-offs of each design
- How they work together

Save this as topics/XX-write-path-synthesis.md"
```

---

**Remember**: The goal is deep understanding, not speed. Take your time, think critically, and use Claude as a study partner, not a crutch.

Happy learning! 🚀
