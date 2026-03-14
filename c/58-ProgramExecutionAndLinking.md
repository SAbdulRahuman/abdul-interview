# Program Execution & Linking

---

## What Happens When You Type `./a.out`?

```
1. Shell calls fork() to create child process
2. Child calls execve("./a.out", ...)
3. Kernel loads ELF binary, maps segments (text, data, bss)
4. Dynamic linker (ld-linux.so) resolves shared libraries
5. Calls __libc_start_main → sets up environment → calls main(argc, argv)
6. main() returns → exit() → _exit() syscall → process terminates
```

## Static vs Dynamic Linking

```
Static Linking:
  - Libraries copied into executable at compile time
  - Larger binary, no runtime dependencies
  - gcc -static program.c

Dynamic Linking:
  - Libraries loaded at runtime (.so / .dll)
  - Smaller binary, shared across processes
  - Requires libraries present at runtime
  - nm, ldd to inspect
```

## Shared Libraries

```bash
# Create shared library
gcc -shared -fPIC -o libmylib.so mylib.c

# Link against it
gcc program.c -L. -lmylib -o program

# Run (must find library at runtime)
LD_LIBRARY_PATH=. ./program

# Or install to system path and run ldconfig
```

---

## Interview Q&A

**Q: What is the difference between `exit()` and `_exit()`?**

```
exit():  flushes stdio buffers, calls atexit() handlers, then calls _exit().
_exit(): immediately terminates the process (no cleanup).
Use _exit() in child process after fork() to avoid flushing parent's buffers.
```

**Q: What is Position-Independent Code (PIC)?**

```
Code that can execute at any memory address without modification.
Required for shared libraries. Generated with -fPIC flag.
Uses relative addressing instead of absolute addresses.
```

---
