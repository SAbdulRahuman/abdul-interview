# Functions

---

## Function Basics

```c
// Declaration (prototype)
int add(int a, int b);

// Definition
int add(int a, int b) {
    return a + b;
}

// Call by value – C passes ALL arguments by value
void swap(int a, int b) {
    int t = a; a = b; b = t;  // does NOT affect caller
}

// Simulating call by reference using pointers
void swap(int *a, int *b) {
    int t = *a; *a = *b; *b = t;
}
```

## Function Prototypes

```c
// Why prototypes matter:
// Without a prototype, the compiler assumes the function returns int
// and performs default argument promotions (float→double, char→int).

// Best practice: declare prototypes in header files
// mylib.h
int compute(int x, int y);

// mylib.c
#include "mylib.h"
int compute(int x, int y) {
    return x * y + x;
}
```

## Passing Arrays to Functions

```c
// Array decays to pointer when passed to function
void process(int arr[], int size);   // arr is actually int*
void process(int *arr, int size);    // equivalent

// sizeof(arr) inside function gives pointer size, NOT array size!
```

---

## Interview Q&A

**Q: Does C support function overloading?**

```
No. C does not support function overloading. Each function must have a unique name.
C11 added _Generic for type-generic macros, but not true overloading.
```

**Q: Does C support pass-by-reference?**

```
No. C only has pass-by-value. You simulate pass-by-reference by passing
pointers (address of variable). The pointer itself is passed by value.
```

**Q: What is the difference between declaration and definition of a function?**

```
Declaration: specifies return type, name, and parameters. No body.
  int add(int, int);

Definition: includes the function body.
  int add(int a, int b) { return a + b; }
```

**Q: Can `main()` be called recursively?**

```
Yes, main() can call itself in C. It is just a regular function.
(Not allowed in C++.)
```

---
