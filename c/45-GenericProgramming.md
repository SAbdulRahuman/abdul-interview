# Generic Programming with void*

---

## Generic Swap

```c
void generic_swap(void *a, void *b, size_t size) {
    char tmp[size];  // VLA
    memcpy(tmp, a, size);
    memcpy(a, b, size);
    memcpy(b, tmp, size);
}

int x = 1, y = 2;
generic_swap(&x, &y, sizeof(int));

double a = 3.14, b = 2.71;
generic_swap(&a, &b, sizeof(double));
```

## qsort (Generic Sort)

```c
int cmp_int(const void *a, const void *b) {
    return (*(int *)a - *(int *)b);
}

int cmp_str(const void *a, const void *b) {
    return strcmp(*(const char **)a, *(const char **)b);
}

int arr[] = {5, 2, 8, 1, 4};
qsort(arr, 5, sizeof(int), cmp_int);

const char *names[] = {"Charlie", "Alice", "Bob"};
qsort(names, 3, sizeof(char*), cmp_str);
```

## bsearch (Generic Binary Search)

```c
int key = 4;
int *result = bsearch(&key, arr, 5, sizeof(int), cmp_int);
// Returns pointer to found element, or NULL
// Array must be sorted!
```

---

## Interview Q&A

**Q: How does qsort achieve generic sorting?**

```
- Takes void* array (any type)
- Takes element size (to calculate offsets)
- Takes comparator function pointer (user defines comparison logic)
- Internally uses byte-level operations (memcpy, pointer arithmetic)
```

**Q: Why does the comparator use `const void*`?**

```
- void* to accept any type
- const because the comparison should not modify elements
- User casts to the actual type inside the comparator
```

---
