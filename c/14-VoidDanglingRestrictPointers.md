# Void, Dangling & Restrict Pointers

---

## Void Pointers

```c
void *vp;
int x = 10;
vp = &x;           // Any pointer can be assigned to void*
// Cannot dereference void* directly
int val = *(int*)vp;  // must cast first

// Used for generic programming (e.g., qsort, bsearch, malloc return type)
```

## Dangling Pointers

```c
// Points to freed/deallocated memory
int *p = malloc(sizeof(int));
*p = 42;
free(p);
// p is now dangling – set p = NULL after free
p = NULL;

// Also dangling: returning address of local variable
int *bad() {
    int x = 10;
    return &x;  // x is destroyed when function returns → dangling pointer
}
```

## Wild Pointers

```c
// Uninitialized pointer (points to random address)
int *p;  // wild pointer – always initialize pointers

// Fix:
int *p = NULL;
```

## Restrict Pointer (C99)

```c
void add_arrays(int n, int * restrict a, int * restrict b, int * restrict c) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
// 'restrict' tells compiler that pointers don't alias → enables optimizations.
// Violating the restrict contract is undefined behavior.
```

---

## Interview Q&A

**Q: Can you do arithmetic on `void*`?**

```
Not in standard C. sizeof(void) is undefined.
GCC allows it as an extension, treating sizeof(void) as 1.
```

**Q: How to prevent dangling pointers?**

```
1. Set pointer to NULL after free: free(p); p = NULL;
2. Don't return addresses of local variables
3. Be careful with pointer copies – all copies become dangling after free
4. Use tools: Valgrind, AddressSanitizer
```

**Q: What does `restrict` do?**

```
Tells the compiler: "this pointer is the ONLY way to access that memory."
Allows compiler to generate more optimized code by avoiding aliasing checks.
Used in: memcpy (but not memmove, since src/dst may overlap).
```

---
