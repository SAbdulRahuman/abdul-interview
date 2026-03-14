# Static Analysis & Tools

---

## Tools Overview

```
Valgrind:       Memory error detector (leaks, invalid reads/writes)
cppcheck:       Static analysis for C/C++
Clang Static Analyzer: scan-build make
AddressSanitizer: -fsanitize=address (compile-time instrumentation)
gcov/lcov:      Code coverage
strace:         Trace system calls
ltrace:         Trace library calls
objdump:        Disassemble object files
nm:             List symbols in object file
ldd:            List shared library dependencies
```

## Valgrind Usage

```bash
gcc -g program.c -o program
valgrind --leak-check=full ./program
valgrind --tool=helgrind ./program   # detect data races
valgrind --tool=cachegrind ./program # cache profiling
```

## Sanitizers (Compiler-Based)

```bash
gcc -fsanitize=address -g prog.c    # buffer overflows, use-after-free, leaks
gcc -fsanitize=undefined -g prog.c  # signed overflow, null deref, alignment
gcc -fsanitize=thread -g prog.c     # data races (cannot combine with ASan)
gcc -fsanitize=memory -g prog.c     # uninitialized reads (Clang only)
```

## Binary Inspection

```bash
nm program          # list symbols (T=text, U=undefined, D=data)
objdump -d program  # disassemble
readelf -h program  # ELF header info
ldd program         # shared library dependencies
strings program     # printable strings in binary
```

---

## Interview Q&A

**Q: How to detect memory leaks?**

```
1. Valgrind: valgrind --leak-check=full ./program
2. ASan: gcc -fsanitize=address -g prog.c && ./a.out
3. LeakSanitizer (part of ASan or standalone)
```

**Q: What is the difference between static and dynamic analysis?**

```
Static analysis: examines source code without running it (cppcheck, Clang SA).
Dynamic analysis: instruments running program (Valgrind, ASan, TSan).
Both catch different classes of bugs – use both.
```

---
