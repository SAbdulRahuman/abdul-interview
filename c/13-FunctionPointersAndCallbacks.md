# Function Pointers & Callbacks

---

## Function Pointers

```c
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

// Declaration
int (*fp)(int, int);

// Assignment and call
fp = add;
int result = fp(3, 4);  // 7

// Array of function pointers (dispatch table)
int (*ops[])(int, int) = { add, sub };
ops[0](5, 3);  // 8
```

## Callback Pattern

```c
void apply(int *arr, int n, int (*transform)(int)) {
    for (int i = 0; i < n; i++)
        arr[i] = transform(arr[i]);
}

int double_val(int x) { return x * 2; }
int square(int x) { return x * x; }

int arr[] = {1, 2, 3, 4};
apply(arr, 4, double_val);  // arr = {2, 4, 6, 8}
apply(arr, 4, square);      // arr = {4, 16, 36, 64}
```

## Typedef for Readability

```c
typedef int (*BinaryOp)(int, int);
BinaryOp op = add;
int result = op(5, 3);  // 8

typedef void (*SignalHandler)(int);
```

## qsort Callback Example

```c
int cmp_int(const void *a, const void *b) {
    return (*(int *)a - *(int *)b);
}
qsort(arr, n, sizeof(int), cmp_int);
```

---

## Interview Q&A

**Q: What is a callback function?**

```
A function passed as an argument to another function, to be called later.
Common in C: qsort, bsearch, signal handlers, event-driven programming.
```

**Q: What is a dispatch table?**

```
An array of function pointers indexed by an enum or integer.
Used to replace long switch-case statements:

typedef void (*Handler)(void);
Handler handlers[] = { handle_open, handle_close, handle_read };
handlers[cmd]();  // call appropriate handler
```

---
