# Const Correctness

---

## Const with Variables and Pointers

```c
const int x = 10;        // x cannot be modified

const int *p;            // pointer to const int (data is read-only)
int const *p;            // same as above
int * const p;           // const pointer to int (pointer is read-only)
const int * const p;     // both are read-only
```

## Const in Function Parameters

```c
// Documents intent: function will not modify the array
void print_array(const int *arr, int size);

// Compiler enforces:
void bad(const int *arr) {
    arr[0] = 42;  // ERROR: assignment to read-only location
}

// String functions use const correctly:
size_t strlen(const char *s);
char *strcpy(char *dst, const char *src);
```

---

## Interview Q&A

**Q: What is const correctness?**

```
Using 'const' wherever possible to indicate read-only intent.
- Prevents accidental modification
- Documents API contracts
- Enables compiler optimizations
```

**Q: Can you modify a const variable through a pointer?**

```c
const int x = 10;
int *p = (int *)&x;
*p = 20;  // Compiles, but UNDEFINED BEHAVIOR!
The compiler may have placed x in read-only memory.
```

**Q: Difference between `const` in C and C++?**

```
C:   const variables are not true compile-time constants.
     Cannot be used for array sizes (pre-C99).
     Have external linkage by default.
C++: const variables are compile-time constants.
     Can be used for array sizes.
     Have internal linkage by default.
```

---
