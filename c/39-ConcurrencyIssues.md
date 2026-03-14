# Common Concurrency Issues

---

## Issues

```
Race condition:     Two threads access shared data, at least one writes, no synchronization.
Deadlock:           Two threads each hold a lock the other needs.
Livelock:           Threads keep changing state but make no progress.
Starvation:         A thread can never acquire the resource it needs.
Priority inversion: Low-priority thread holds lock needed by high-priority thread.
```

## Race Condition Example

```c
// BUG: race condition
int counter = 0;
void *increment(void *arg) {
    for (int i = 0; i < 1000000; i++)
        counter++;  // read-modify-write is NOT atomic
    return NULL;
}
// Result: counter < 2000000 due to lost updates

// FIX: use mutex or atomic
_Atomic int counter = 0;
atomic_fetch_add(&counter, 1);
```

## Deadlock Example

```c
// Thread 1: lock(A); lock(B);
// Thread 2: lock(B); lock(A);
// → Deadlock! Each holds what the other needs.

// Prevention: always lock in same order
// Thread 1: lock(A); lock(B);
// Thread 2: lock(A); lock(B);  // same order → no deadlock
```

---

## Interview Q&A

**Q: How to detect data races?**

```
Tools:
  gcc -fsanitize=thread program.c  (ThreadSanitizer)
  Helgrind (Valgrind tool)
  Intel Inspector
```

**Q: Four conditions for deadlock (Coffman conditions)?**

```
All four must hold simultaneously:
1. Mutual exclusion – resource held exclusively
2. Hold and wait – thread holds one resource, waits for another
3. No preemption – resources cannot be forcibly taken
4. Circular wait – circular chain of threads waiting for each other
Break any one to prevent deadlock.
```

---
