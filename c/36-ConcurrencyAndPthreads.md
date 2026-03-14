# Concurrency & Pthreads

---

## POSIX Threads (pthreads)

```c
#include <pthread.h>

void *thread_func(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d\n", id);
    return NULL;
}

int main() {
    pthread_t threads[4];
    int ids[4];
    for (int i = 0; i < 4; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, thread_func, &ids[i]);
    }
    for (int i = 0; i < 4; i++)
        pthread_join(threads[i], NULL);
    return 0;
}
// Compile: gcc -pthread program.c
```

## Thread Lifecycle

```
1. pthread_create()  – create and start a new thread
2. Thread executes its function
3. pthread_join()    – wait for thread to finish (blocks caller)
4. pthread_detach()  – let thread clean up automatically when done
```

## Thread Attributes

```c
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
pthread_attr_setstacksize(&attr, 1024 * 1024);  // 1 MB stack
pthread_create(&thread, &attr, func, arg);
pthread_attr_destroy(&attr);
```

---

## Interview Q&A

**Q: What is the difference between `pthread_join` and `pthread_detach`?**

```
join: calling thread blocks until target thread finishes. Resources freed after join.
detach: thread runs independently. Resources freed automatically on termination.
A thread can be joined OR detached, not both.
```

**Q: Is it safe to pass a local variable's address to a thread?**

```
Only if the variable outlives the thread. Common bug:
for (int i = 0; i < N; i++)
    pthread_create(&t[i], NULL, func, &i);  // BUG: all threads share same 'i'
Fix: use a separate variable per thread, or pass by value (cast to void*).
```

---
