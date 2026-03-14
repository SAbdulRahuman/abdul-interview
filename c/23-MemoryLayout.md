# Memory Layout of a C Program

---

## Segments

```
High Address ┌─────────────┐
             │   Stack     │  ← local vars, function frames (grows ↓)
             ├─────────────┤
             │     ↓       │
             │   (free)    │
             │     ↑       │
             ├─────────────┤
             │   Heap      │  ← malloc/calloc (grows ↑)
             ├─────────────┤
             │   BSS       │  ← uninitialized globals/statics (zero-filled)
             ├─────────────┤
             │   Data      │  ← initialized globals/statics
             ├─────────────┤
             │   Text      │  ← machine code (read-only)
Low Address  └─────────────┘
```

## Segment Details

```
Text (Code):  Read-only, contains compiled machine instructions.
Data:         Initialized global and static variables.
BSS:          Uninitialized global and static variables (zero-filled by OS).
Heap:         Dynamically allocated memory (malloc/calloc/realloc).
Stack:        Function call frames, local variables, return addresses.
```

---

## Interview Q&A

**Q: What is the difference between stack and heap?**

```
Stack:                           Heap:
- Automatic management          - Manual management (malloc/free)
- LIFO order                    - No order
- Fast allocation (pointer move)- Slower (fragmentation, bookkeeping)
- Limited size (1-8 MB)         - Large (limited by virtual memory)
- Thread-local                  - Shared across threads
- No fragmentation              - Can fragment
```

**Q: What is the BSS segment?**

```
Block Started by Symbol. Stores uninitialized global and static variables.
They are guaranteed to be zero-initialized by the OS/runtime.
BSS doesn't occupy space in the executable file (only in memory).
```

**Q: What is a stack frame?**

```
Memory allocated on the stack for each function call. Contains:
- Return address
- Saved registers
- Local variables
- Function arguments (sometimes)
Created on function entry, destroyed on return.
```

---
