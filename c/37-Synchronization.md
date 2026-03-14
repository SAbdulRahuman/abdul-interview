# Synchronization (Mutex, Condvar, RWLock)

---

## Mutex

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&lock);
// critical section
pthread_mutex_unlock(&lock);

// Dynamic initialization
pthread_mutex_t lock;
pthread_mutex_init(&lock, NULL);
// ... use ...
pthread_mutex_destroy(&lock);
```

## Condition Variable

```c
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

// Wait pattern (always use while loop, not if)
pthread_mutex_lock(&lock);
while (!ready)
    pthread_cond_wait(&cond, &lock);  // atomically unlocks mutex + waits
// ... process ...
pthread_mutex_unlock(&lock);

// Signal
pthread_mutex_lock(&lock);
ready = 1;
pthread_cond_signal(&cond);     // wake one waiting thread
pthread_mutex_unlock(&lock);

pthread_cond_broadcast(&cond);  // wake ALL waiting threads
```

## Read-Write Lock

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// Multiple readers can hold the lock simultaneously
pthread_rwlock_rdlock(&rwlock);  // shared read
// ... read data ...
pthread_rwlock_unlock(&rwlock);

// Only one writer at a time, excludes all readers
pthread_rwlock_wrlock(&rwlock);  // exclusive write
// ... write data ...
pthread_rwlock_unlock(&rwlock);
```

---

## Interview Q&A

**Q: Why use `while` instead of `if` with `pthread_cond_wait`?**

```
Spurious wakeups: the thread may wake up even without a signal.
The while loop re-checks the condition to handle this correctly.
```

**Q: What is a deadlock? How to prevent it?**

```
Two or more threads waiting for each other's locks.
Prevention:
  1. Lock ordering: always acquire locks in the same order
  2. Use pthread_mutex_trylock() with timeout
  3. Lock hierarchy / leveling
  4. Avoid holding multiple locks
```

---
