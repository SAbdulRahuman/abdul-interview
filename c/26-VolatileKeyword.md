# Volatile Keyword

---

## Usage

```c
volatile int flag = 0;
// Tells compiler: do NOT optimize away reads/writes to this variable.
// The value may change unexpectedly (hardware register, signal handler, other thread).
// Does NOT provide atomicity or memory ordering guarantees.
```

## Examples

```c
// Without volatile, compiler may optimize this to an infinite loop:
// It reads 'flag' once, sees it's 0, and never re-reads.
while (flag == 0) {
    // wait
}

// With volatile, compiler re-reads 'flag' from memory every iteration:
volatile int flag = 0;
while (flag == 0) {
    // wait – compiler will actually check memory each time
}
```

---

## Interview Q&A

**Q: When to use `volatile`?**

```
1. Memory-mapped I/O registers (embedded systems)
2. Variables modified by signal handlers
3. Variables modified by hardware / DMA
NOT for thread synchronization (use _Atomic or mutexes).
```

**Q: Does `volatile` make a variable thread-safe?**

```
No. volatile does NOT provide:
  - Atomicity (read-modify-write can be interrupted)
  - Memory ordering/fencing
Use _Atomic (C11) or pthread mutexes for thread safety.
```

**Q: Can a variable be both `const` and `volatile`?**

```
Yes. const volatile int *reg = (const volatile int *)0x40021000;
The program cannot modify it (const), but it may change externally (volatile).
Used for read-only hardware registers.
```

---
