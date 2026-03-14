# Arrays

---

## Basics

```c
int arr[5] = {1, 2, 3, 4, 5};
int arr2[5] = {0};              // all elements initialized to 0
int arr3[] = {1, 2, 3};         // size inferred as 3

// 2D array
int matrix[3][4];               // 3 rows, 4 columns (row-major in memory)

// Array decay: array name decays to pointer to first element
int *p = arr;  // p points to arr[0]
```

## Variable-Length Arrays (C99)

```c
void func(int n) {
    int vla[n];  // allocated on stack at runtime
    // VLAs cannot be initialized with { }
    // VLAs cannot have static storage
    // Made optional in C11
}
```

## Multi-Dimensional Arrays

```c
int matrix[3][4];               // stored as 12 contiguous ints (row-major)
matrix[i][j]                     // equivalent to *(*(matrix + i) + j)

// Passing 2D array to function (must specify columns)
void print_matrix(int m[][4], int rows);
void print_matrix(int (*m)[4], int rows);  // equivalent
```

---

## Interview Q&A

**Q: Can you return an array from a function?**

```
No. Arrays cannot be returned by value. Options:
1. Return a pointer to a static/heap-allocated array.
2. Pass the array as a parameter and modify in-place.
3. Wrap the array in a struct and return the struct.
```

**Q: Difference between `arr` and `&arr`?**

```c
int arr[5];
// arr    → type: int*      → points to first element (arr[0])
// &arr   → type: int(*)[5] → points to entire array
// Both have the same address value, but different types.
// arr + 1   advances by sizeof(int)
// &arr + 1  advances by sizeof(int[5]) = 5 * sizeof(int)
```

**Q: What is array decay?**

```
When an array name is used in an expression, it "decays" to a pointer to its
first element. Exceptions:
  - sizeof(arr) – gives size of entire array
  - &arr – gives pointer to entire array
  - String literal used to initialize char[]
```

**Q: How are 2D arrays stored in memory?**

```
Row-major order (C): elements are stored row by row.
matrix[0][0], matrix[0][1], ... matrix[0][3], matrix[1][0], ...
```

---
