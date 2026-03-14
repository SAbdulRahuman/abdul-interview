# C11 Atomics & Memory Ordering

---

## Atomic Types

```c
#include <stdatomic.h>

_Atomic int counter = 0;
// or: atomic_int counter = ATOMIC_VAR_INIT(0);

atomic_fetch_add(&counter, 1);     // atomic increment
atomic_load(&counter);             // atomic read
atomic_store(&counter, 0);         // atomic write
atomic_compare_exchange_strong(&counter, &expected, desired);
```

## Memory Ordering

```c
// Memory ordering (for lock-free programming):
atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);

// Orders (weakest to strongest):
// memory_order_relaxed   – no ordering, just atomicity
// memory_order_consume   – data-dependent ordering (rarely used)
// memory_order_acquire   – subsequent reads/writes cannot move before this
// memory_order_release   – preceding reads/writes cannot move after this
// memory_order_acq_rel   – both acquire and release
// memory_order_seq_cst   – sequential consistency (default, strongest)
```

## C11 Threads (Optional)

```c
#include <threads.h>

int thread_func(void *arg) {
    return 0;
}

thrd_t t;
thrd_create(&t, thread_func, NULL);
thrd_join(t, NULL);

mtx_t m;
mtx_init(&m, mtx_plain);
mtx_lock(&m);
mtx_unlock(&m);
mtx_destroy(&m);
```

---

## Interview Q&A

**Q: Difference between `_Atomic` and `volatile`?**

```
volatile: prevents compiler optimization of reads/writes. No atomicity or ordering.
_Atomic:  guarantees atomic access AND provides memory ordering.
For thread-safe code, use _Atomic (not volatile).
```

**Q: What is a compare-and-swap (CAS) operation?**

```c
// Atomically: if *obj == *expected, set *obj = desired, return true
//             else set *expected = *obj, return false
atomic_compare_exchange_strong(&obj, &expected, desired);
// Foundation of most lock-free data structures
```

---
