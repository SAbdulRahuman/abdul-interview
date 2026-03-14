# Enums & Typedef

---

## Enums

```c
enum Color { RED, GREEN, BLUE };          // 0, 1, 2
enum Color { RED = 1, GREEN = 5, BLUE };  // 1, 5, 6

// Enums are ints in C (unlike C++ enum class)
enum Color c = 42;  // valid in C (no type safety)

// Use enum for named constants
enum { BUFFER_SIZE = 1024 };  // anonymous enum as named constant
```

## Typedef

```c
typedef unsigned long ulong;
typedef struct Node {
    int data;
    struct Node *next;   // must use 'struct Node' here (not 'Node')
} Node;

// Now can use: Node n; instead of struct Node n;

// Function pointer typedef
typedef void (*SignalHandler)(int);
typedef int (*Comparator)(const void *, const void *);
```

## Typedef vs #define

```
typedef: creates an alias for a type. Scoped, type-checked.
#define: text substitution. No scope, no type checking.

typedef int *IntPtr;
IntPtr a, b;     // both are int*

#define INTPTR int*
INTPTR a, b;     // a is int*, b is int (!)
```

---

## Interview Q&A

**Q: Are enums type-safe in C?**

```
No. Enums are just ints in C. You can assign any integer value:
enum Color c = 42;  // compiles without warning in many compilers
C23 adds enums with fixed underlying types for better safety.
```

**Q: When to use typedef?**

```
- Simplify complex types: function pointers, struct names
- Improve portability: typedef int int32_t;
- Improve readability: typedef unsigned char byte;
Don't overuse – hiding pointer types (typedef int* intptr) can be confusing.
```

---
