# Error Handling Patterns

---

## Pattern 1: Return Error Codes

```c
#define SUCCESS  0
#define ERR_NULL -1
#define ERR_OOM  -2

int do_something(int *result) {
    int *p = malloc(100);
    if (!p) return ERR_OOM;
    *result = compute(p);
    free(p);
    return SUCCESS;
}

// Caller:
int val;
int err = do_something(&val);
if (err != SUCCESS) { /* handle error */ }
```

## Pattern 2: errno (POSIX)

```c
#include <errno.h>
FILE *fp = fopen("nofile.txt", "r");
if (!fp) {
    fprintf(stderr, "Error: %s\n", strerror(errno));
    // or: perror("fopen");
}
```

## Pattern 3: Goto Cleanup

```c
int func() {
    int ret = -1;
    int *a = malloc(100); if (!a) goto out;
    int *b = malloc(200); if (!b) goto free_a;

    // ... work ...
    ret = 0;

free_b: free(b);
free_a: free(a);
out:    return ret;
}
```

---

## Interview Q&A

**Q: What is `errno`?**

```
A thread-local integer set by system calls and library functions on error.
Must check immediately after the failing call (before any other call).
Common values: ENOMEM, EINVAL, ENOENT, EACCES, EAGAIN
```

**Q: Why is goto cleanup considered acceptable in C?**

```
C lacks exceptions, destructors, and defer statements.
Goto cleanup is the cleanest way to handle multiple resource allocations
that need cleanup on failure. Used extensively in the Linux kernel.
```

---
