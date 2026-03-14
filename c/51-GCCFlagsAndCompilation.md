# GCC Flags & Compilation

---

## Common Flags

```bash
gcc -Wall -Wextra -Werror         # warnings → errors
gcc -std=c11                       # specify C standard
gcc -O0 / -O1 / -O2 / -O3 / -Os  # optimization levels
gcc -g                             # debug info (for gdb/lldb)
gcc -fsanitize=address             # AddressSanitizer (buffer overflow, use-after-free)
gcc -fsanitize=undefined           # UBSan (undefined behavior)
gcc -fsanitize=thread              # ThreadSanitizer (data races)
gcc -pg                            # profiling (gprof)
gcc -E                             # preprocessor output only
gcc -S                             # assembly output
gcc -c                             # compile only (no linking)
gcc -shared -fPIC -o lib.so lib.c  # shared library
gcc -static                        # static linking
gcc -I/path/to/headers             # add include path
gcc -L/path/to/libs -lmylib       # add library path and link library
gcc -D NDEBUG                     # define preprocessor macro
gcc -MMD                           # generate dependency files
```

## Optimization Levels

```
-O0: no optimization (default). Best for debugging.
-O1: basic optimizations. Faster compile than -O2.
-O2: most optimizations without size/speed tradeoffs. Recommended for production.
-O3: aggressive optimizations (loop unrolling, vectorization). May increase binary size.
-Os: optimize for size.
-Og: optimize for debugging experience (C99+).
```

---

## Interview Q&A

**Q: What does `-Wall` enable?**

```
Most common warnings: unused variables, implicit declarations,
format string mismatches, etc. Does NOT enable ALL warnings.
Use -Wextra for additional warnings.
```

**Q: What is `-fPIC`?**

```
Position-Independent Code. Required for shared libraries (.so).
Makes code relocatable – can be loaded at any address by the dynamic linker.
```

---
