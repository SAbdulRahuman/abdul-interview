# Threading & Synchronization

> POSIX threads, synchronization primitives, concurrency bugs, and lock-free programming.

---

## Thread vs Process

| Aspect | Process | Thread |
|---|---|---|
| Address space | Separate (isolated) | Shared (same process) |
| Creation cost | High (page tables, COW) | Low (shared memory) |
| Context switch | Expensive (TLB flush, cache) | Cheaper (shared address space) |
| Communication | IPC needed (pipes, sockets, shmem) | Direct memory access |
| Crash isolation | One crash doesn't affect others | One crash kills all threads |
| Scheduling | Independent | Kernel schedules individually |

### Linux Threading Model
- Linux uses **1:1 threading** (each user thread = one kernel thread)
- Threads created via `clone()` syscall with `CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_THREAD`
- `pthread_create()` wraps `clone()` internally (via NPTL — Native POSIX Threads Library)
- Thread IDs are kernel task IDs (visible in `/proc/<pid>/task/`)

---

## POSIX Threads (pthreads)

### Thread Lifecycle

```c
#include <pthread.h>
#include <stdio.h>

void *worker(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d running\n", id);
    // Do work...
    return (void *)(long)id;  // Return value
}

int main() {
    pthread_t threads[4];
    int ids[4] = {0, 1, 2, 3};

    // Create threads
    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }

    // Join threads (wait for completion)
    for (int i = 0; i < 4; i++) {
        void *ret;
        pthread_join(threads[i], &ret);
        printf("Thread %d returned %ld\n", i, (long)ret);
    }
    return 0;
}
// Compile: gcc -pthread -o mythreads mythreads.c
```

### Thread Attributes

```c
pthread_attr_t attr;
pthread_attr_init(&attr);

// Set stack size (default: typically 2MB or 8MB)
pthread_attr_setstacksize(&attr, 1024 * 1024);  // 1MB

// Detached thread (auto-cleanup, cannot join)
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

pthread_create(&thread, &attr, worker, arg);
pthread_attr_destroy(&attr);
```

### Detached Threads

```c
// Option 1: Create detached
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

// Option 2: Detach after creation
pthread_detach(thread);

// Detached threads:
// - Cannot be joined (pthread_join returns EINVAL)
// - Resources auto-released on exit
// - Use for "fire and forget" tasks
```

---

## Thread-Local Storage (TLS)

### Compiler-Level TLS

```c
// GCC/Clang __thread (fastest)
static __thread int per_thread_counter = 0;

// C11 _Thread_local
_Thread_local int counter = 0;

void *worker(void *arg) {
    per_thread_counter++;  // Each thread has its own copy
    return NULL;
}
```

### POSIX TLS (pthread_key)

```c
pthread_key_t key;

void destructor(void *value) {
    free(value);  // Called when thread exits
}

void init() {
    pthread_key_create(&key, destructor);
}

void *worker(void *arg) {
    int *data = malloc(sizeof(int));
    *data = 42;
    pthread_setspecific(key, data);

    // Later...
    int *val = pthread_getspecific(key);
    printf("Thread-local value: %d\n", *val);
    return NULL;
}
```

---

## Synchronization Primitives

### Mutex (Mutual Exclusion)

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Or dynamic initialization with attributes
pthread_mutex_t mutex;
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);  // Detect errors
pthread_mutex_init(&mutex, &attr);

void *worker(void *arg) {
    pthread_mutex_lock(&mutex);
    // Critical section — only one thread at a time
    shared_counter++;
    pthread_mutex_unlock(&mutex);
    return NULL;
}

// Try lock (non-blocking)
if (pthread_mutex_trylock(&mutex) == 0) {
    // Got the lock
    pthread_mutex_unlock(&mutex);
} else {
    // Lock is held by another thread
}

// Timed lock
struct timespec ts;
clock_gettime(CLOCK_REALTIME, &ts);
ts.tv_sec += 5;  // 5 second timeout
if (pthread_mutex_timedlock(&mutex, &ts) == 0) {
    // Got the lock within timeout
    pthread_mutex_unlock(&mutex);
}
```

### Mutex Types

| Type | Behavior on Double Lock | Use Case |
|---|---|---|
| `PTHREAD_MUTEX_NORMAL` | **Deadlock** | Default, fastest |
| `PTHREAD_MUTEX_ERRORCHECK` | Returns `EDEADLK` | Debugging |
| `PTHREAD_MUTEX_RECURSIVE` | Allows same thread to lock multiple times | Recursive functions |
| `PTHREAD_MUTEX_ADAPTIVE_NP` | Spins briefly before blocking | Short critical sections |

---

### Read-Write Lock (RWLock)

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

void *reader(void *arg) {
    pthread_rwlock_rdlock(&rwlock);  // Multiple readers allowed
    // Read shared data
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}

void *writer(void *arg) {
    pthread_rwlock_wrlock(&rwlock);  // Exclusive access
    // Modify shared data
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}
```

**Semantics:**
- Multiple threads can hold read lock simultaneously
- Write lock is exclusive — blocks all readers and writers
- Write-preferring vs read-preferring (configurable)

---

### Condition Variable

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

// Producer
void *producer(void *arg) {
    pthread_mutex_lock(&mutex);
    ready = 1;
    pthread_cond_signal(&cond);    // Wake one waiter
    // pthread_cond_broadcast(&cond);  // Wake ALL waiters
    pthread_mutex_unlock(&mutex);
    return NULL;
}

// Consumer
void *consumer(void *arg) {
    pthread_mutex_lock(&mutex);
    while (!ready) {                // MUST use while loop (spurious wakeups!)
        pthread_cond_wait(&cond, &mutex);
        // wait() atomically: releases mutex → sleeps → reacquires mutex
    }
    // Process data
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

**Critical Pattern**: Always use `while (!condition)` not `if (!condition)` to guard against **spurious wakeups**.

---

### Semaphore

```c
#include <semaphore.h>

sem_t sem;

// Initialize semaphore with count 3 (allows 3 concurrent accesses)
sem_init(&sem, 0, 3);   // 0 = shared between threads (1 = processes)

void *worker(void *arg) {
    sem_wait(&sem);      // Decrement (blocks if 0)
    // Access shared resource (up to 3 threads concurrently)
    sem_post(&sem);      // Increment
    return NULL;
}

// Named semaphore (works across processes)
sem_t *sem = sem_open("/my_sem", O_CREAT, 0644, 1);
sem_wait(sem);
// Critical section
sem_post(sem);
sem_close(sem);
sem_unlink("/my_sem");
```

**Mutex vs Semaphore:**
- Mutex: Binary (locked/unlocked), has **ownership** (only locker can unlock)
- Semaphore: Counting, **no ownership** (any thread can post), can be used for signaling

---

### Spinlock

```c
pthread_spinlock_t spinlock;
pthread_spin_init(&spinlock, PTHREAD_PROCESS_PRIVATE);

void *worker(void *arg) {
    pthread_spin_lock(&spinlock);   // Busy-wait (burns CPU)
    // Very short critical section
    pthread_spin_unlock(&spinlock);
    return NULL;
}
```

**When to use spinlocks:**
- Critical section is **very short** (< 1μs)
- Running on **multiprocessor** system (useless on single CPU)
- Cannot afford the overhead of sleeping (context switch ≈ 1-10μs)
- Common in kernel code, less common in user space

---

### Barrier

```c
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 4);  // Wait for 4 threads

void *worker(void *arg) {
    // Phase 1: each thread does setup
    setup();

    // All threads wait here until all 4 arrive
    pthread_barrier_wait(&barrier);

    // Phase 2: all threads proceed together
    process();
    return NULL;
}
```

---

## Common Concurrency Bugs

### 1. Data Race

```
Thread 1:  read(x)  →  compute  →  write(x)
Thread 2:       read(x)  →  compute  →  write(x)
                    ↑ interleaved → lost update!
```

- Two threads access same memory, at least one writes, no synchronization
- **Undefined behavior** in C/C++ (nasal demons)
- Detection: ThreadSanitizer (`-fsanitize=thread`), Helgrind (valgrind)

### 2. Deadlock

```
Thread 1: lock(A) → lock(B) → ...
Thread 2: lock(B) → lock(A) → ...
           ↑ Both waiting for each other forever
```

**Coffman Conditions** (all 4 required for deadlock):
1. **Mutual exclusion** — resources are non-shareable
2. **Hold and wait** — hold one resource while waiting for another
3. **No preemption** — resources can't be forcibly taken
4. **Circular wait** — circular chain of waiting

**Prevention:**
- Always acquire locks in a **consistent global order**
- Use `trylock()` with backoff
- Use lock hierarchies / lock ordering
- Detect with `pthread_mutex_timedlock()` timeouts

### 3. Livelock

```
Thread 1: tries lock A, fails → backs off → retries
Thread 2: tries lock A, fails → backs off → retries
           ↑ Both actively running but making no progress
```

- Threads keep changing state in response to each other but no progress
- Fix: Add **random jitter** to backoff

### 4. Priority Inversion

```
High-priority:   ────wait(lock)──────────────────run──
Medium-priority:  ──────────run──run──run──────────────
Low-priority:    ──lock──────────────────unlock────────
                        ↑ High blocked by low, 
                          while medium runs freely!
```

- High-priority task blocked by low-priority task holding a lock
- Medium-priority task preempts low-priority, further delaying high
- **Fix: Priority inheritance** — low-priority thread temporarily gets high priority while holding the lock
- Linux: `PTHREAD_PRIO_INHERIT` mutex attribute

### 5. ABA Problem

```
Thread 1: reads A, preempted
Thread 2: changes A → B → A
Thread 1: resumes, CAS(A, new) succeeds!
           ↑ But the world changed while Thread 1 was away
```

- CAS succeeds because value matches, but object was modified/freed/reallocated
- Fix: **Tagged pointers** (version counter + pointer) or **hazard pointers**

### 6. False Sharing

```
┌─────────────────── Cache Line (64 bytes) ───────────────────┐
│  [Thread 1's counter]  [Thread 2's counter]  [padding...]  │
└─────────────────────────────────────────────────────────────┘
          ↑                      ↑
       Thread 1 writes      Thread 2 writes
       → invalidates         → invalidates
         Thread 2's cache      Thread 1's cache
         
Fix: Pad each counter to 64 bytes (separate cache lines)
```

- Different variables on **same cache line** → cache bouncing between cores
- Severe performance degradation (10-100x slower)
- Detection: `perf stat -e cache-misses` and `perf c2c`

```c
// Problem: false sharing
struct counters {
    volatile long thread1_count;  // Same cache line!
    volatile long thread2_count;
};

// Fix: cache line padding
struct counters {
    volatile long thread1_count;
    char pad[64 - sizeof(long)];   // Pad to cache line
    volatile long thread2_count;
};

// C11/GCC way
struct counters {
    _Alignas(64) volatile long thread1_count;
    _Alignas(64) volatile long thread2_count;
};
```

---

## Lock-Free Programming

### Atomic Operations

```c
#include <stdatomic.h>  // C11

atomic_int counter = 0;

// Atomic load/store
int val = atomic_load(&counter);
atomic_store(&counter, 42);

// Atomic read-modify-write
atomic_fetch_add(&counter, 1);   // counter++ atomically
atomic_fetch_sub(&counter, 1);   // counter-- atomically

// Compare-And-Swap (CAS) — foundation of lock-free
int expected = 10;
int desired = 20;
bool success = atomic_compare_exchange_strong(&counter, &expected, desired);
// If counter == 10: sets counter = 20, returns true
// If counter != 10: sets expected = current value, returns false
```

### GCC Built-in Atomics

```c
// Legacy GCC __sync builtins
__sync_fetch_and_add(&counter, 1);
__sync_val_compare_and_swap(&counter, old, new);
__sync_synchronize();  // Full memory barrier

// Modern GCC __atomic builtins
__atomic_load_n(&counter, __ATOMIC_SEQ_CST);
__atomic_store_n(&counter, 42, __ATOMIC_SEQ_CST);
__atomic_compare_exchange_n(&counter, &expected, desired,
                            false, __ATOMIC_SEQ_CST, __ATOMIC_SEQ_CST);
```

### Memory Ordering

| Order | Guarantee | Performance |
|---|---|---|
| `memory_order_relaxed` | Only atomicity, no ordering | Fastest |
| `memory_order_acquire` | Reads after this see writes before the paired release | |
| `memory_order_release` | Writes before this are visible to the paired acquire | |
| `memory_order_acq_rel` | Both acquire and release | |
| `memory_order_seq_cst` | Total global ordering (default) | Slowest |

```c
// Producer-consumer with acquire/release
atomic_int data_ready = 0;
int data = 0;

// Producer
void produce() {
    data = 42;                                              // Store data first
    atomic_store_explicit(&data_ready, 1, memory_order_release);  // Release
}

// Consumer
void consume() {
    while (!atomic_load_explicit(&data_ready, memory_order_acquire));  // Acquire
    printf("data = %d\n", data);  // Guaranteed to see data = 42
}
```

### Memory Barriers

```c
// Compiler barrier — prevent compiler reordering (not CPU)
asm volatile("" ::: "memory");

// CPU memory barrier — prevent hardware reordering
__sync_synchronize();           // Full barrier (GCC)

// Linux kernel barriers
// smp_mb()    — full memory barrier
// smp_rmb()   — read barrier
// smp_wmb()   — write barrier
```

### Lock-Free Queue (Michael-Scott Queue)

```
Enqueue:                              Dequeue:
1. Create new node                    1. Read head
2. CAS: tail->next = new_node        2. Read head->next (actual first element)
3. CAS: tail = new_node              3. CAS: head = head->next
                                      4. Return old head->next value

┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
│Sentinel│──►│ Item1│──►│ Item2│──►│ NULL │
└──────┘    └──────┘    └──────┘    └──────┘
  ↑ head                              ↑ tail
```

Key properties:
- Uses CAS for thread-safe modification without locks
- Sentinel (dummy) node simplifies empty queue handling
- Must handle ABA problem (usually with tagged pointers)

---

## Go Concurrency Comparison

| C/POSIX | Go | Notes |
|---|---|---|
| `pthread_create` | `go func()` | Goroutine is much cheaper (2KB stack) |
| `pthread_mutex` | `sync.Mutex` | Same concept |
| `pthread_rwlock` | `sync.RWMutex` | Same concept |
| `pthread_cond` | `chan` / `sync.Cond` | Channels preferred in Go |
| `sem_t` | `chan struct{}` with buffer | Buffered channel as semaphore |
| `pthread_once` | `sync.Once` | Same concept |
| `atomic_*` | `sync/atomic` | Same concept |
| Thread-local | No direct equivalent | Use goroutine-local via context |

---

## Debugging Concurrency Issues

```bash
# ThreadSanitizer — detect data races
gcc -fsanitize=thread -g -O1 -o myapp myapp.c
./myapp   # Reports data races with stack traces

# Helgrind (Valgrind) — detect lock ordering issues
valgrind --tool=helgrind ./myapp

# Go race detector
go run -race main.go
go test -race ./...

# GDB thread debugging
(gdb) info threads
(gdb) thread apply all bt     # Stack traces for all threads
(gdb) thread 3                # Switch to thread 3

# Check CPU cache behavior (false sharing)
perf stat -e cache-misses,cache-references ./myapp
perf c2c record ./myapp       # Detect cache contention
perf c2c report
```

---

## Interview Tips

1. **When to use mutex vs semaphore vs spinlock?**
   → Mutex: simple mutual exclusion. Semaphore: counting/signaling. Spinlock: very short critical sections on multiprocessor.

2. **How to prevent deadlock?**
   → Lock ordering, trylock with backoff, timeout, lock hierarchies.

3. **What is false sharing and how to fix it?**
   → Different variables on same cache line cause cache bouncing. Fix: pad to cache line size (64 bytes).

4. **Explain memory ordering (acquire/release).**
   → Acquire: subsequent reads see writes before paired release. Release: prior writes visible after paired acquire.

5. **Lock-free vs wait-free?**
   → Lock-free: at least one thread makes progress (others may retry). Wait-free: every thread completes in bounded steps.

---

*Last updated: March 13, 2026*
