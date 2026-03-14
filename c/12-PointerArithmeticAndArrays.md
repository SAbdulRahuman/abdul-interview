# Pointer Arithmetic & Arrays

---

## Pointer Arithmetic

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;

p + 1     // points to arr[1] (advances by sizeof(int) bytes)
p[2]      // equivalent to *(p + 2)
p - arr   // number of elements between p and arr (ptrdiff_t)

// Pointer arithmetic is only valid within the same array (or one past the end).
```

## Pointers and Arrays

```c
// Equivalent notations:
arr[i]   ≡  *(arr + i)  ≡  *(i + arr)  ≡  i[arr]

// Array of pointers vs Pointer to array
int *arr_of_ptrs[5];      // array of 5 int pointers
int (*ptr_to_arr)[5];     // pointer to array of 5 ints
```

## Pointer Subtraction

```c
int arr[5] = {10, 20, 30, 40, 50};
int *p1 = &arr[1];
int *p2 = &arr[4];
ptrdiff_t diff = p2 - p1;  // 3 (number of elements, not bytes)
```

## Pointer Comparison

```c
// Pointers can be compared with ==, !=, <, >, <=, >=
// Only meaningful when both point to elements of the same array
if (p1 < p2) { /* p1 points to earlier element */ }
```

---

## Interview Q&A

**Q: Why does `i[arr]` work in C?**

```
arr[i] is defined as *(arr + i).
Since addition is commutative: *(arr + i) == *(i + arr) == i[arr].
This is valid but never used in practice.
```

**Q: What is the difference between `int *p[5]` and `int (*p)[5]`?**

```
int *p[5]   – array of 5 pointers to int
int (*p)[5] – pointer to an array of 5 ints

Use the "clockwise/spiral rule" to parse complex declarations.
```

---
