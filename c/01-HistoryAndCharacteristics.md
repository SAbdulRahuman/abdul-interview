# History & Characteristics of C

---

## Overview

```
- Created by Dennis Ritchie at Bell Labs (1972)
- Procedural, compiled, statically typed, low-level access
- ANSI C (C89/C90), C99, C11, C17, C23
- Used in OS kernels, embedded systems, compilers, databases
```

## Key Characteristics

```
- General-purpose, portable
- Close to hardware (pointers, bitwise ops, memory manipulation)
- Small set of keywords (~32 in C89)
- Rich set of operators
- Supports structured programming
- Modular via functions and separate compilation
- Minimal runtime – no garbage collector, no built-in exception handling
```

## Why C?

```
- Linux kernel, Windows kernel, macOS kernel (XNU) – all written in C
- Most embedded systems and microcontrollers use C
- Python, Ruby, PHP interpreters are written in C
- Databases (SQLite, PostgreSQL, MySQL) use C
- Provides foundation for C++, Objective-C, and many modern languages
```

---

## Interview Q&A

**Q: Why is C called a "middle-level" language?**

```
It combines high-level constructs (functions, loops, structs) with low-level capabilities
(pointer arithmetic, bitwise operations, direct memory access).
```

**Q: What are the advantages of C over other languages?**

```
1. Speed – compiles to native machine code, minimal runtime overhead
2. Portability – C programs can be compiled on virtually any platform
3. Low-level access – direct memory manipulation via pointers
4. Small footprint – suitable for resource-constrained environments
5. Mature ecosystem – extensive libraries, tools, and community
```

**Q: What are the different C standards?**

```
C89/C90  – First ANSI/ISO standard
C99      – VLAs, inline, restrict, _Bool, designated initializers, // comments
C11      – _Atomic, _Generic, _Static_assert, threads.h, anonymous structs
C17      – Bug fixes only, no new features
C23      – typeof, constexpr, #embed, nullptr, binary literals, attributes
```

---
