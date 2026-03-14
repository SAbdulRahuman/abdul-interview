# Variables & Constants

---

## Variables

```c
// Variable declaration and initialization
int x;          // declaration (indeterminate value if local)
int y = 10;     // definition with initialization

// Multiple declarations
int a, b, c;
int d = 1, e = 2;

// Scope
{
    int local = 5;   // block scope – visible only within { }
}
// local is not accessible here
```

## Constants

```c
// const qualifier
const int MAX = 100;       // read-only variable
// const variables are NOT true compile-time constants in C (unlike C++)

// Preprocessor macro
#define PI 3.14159         // text substitution, no type checking

// Enumeration constants
enum { RED = 0, GREEN, BLUE };  // compile-time integer constants

// C99 compound literals
int *p = (int[]){1, 2, 3};
struct Point origin = (struct Point){0, 0};
```

---

## Interview Q&A

**Q: Difference between `const` and `#define`?**

```
#define: preprocessor text substitution, no type checking, no scope.
const: typed, scoped, debugger-visible, respects block scope.
const variables are NOT true compile-time constants in C (unlike C++).
  → Cannot be used for array sizes in C89 (allowed in C99 with VLAs).
```

**Q: Where are variables stored in memory?**

```
Local variables       → Stack
Global/static vars    → Data segment (initialized) or BSS (uninitialized)
Dynamic allocations   → Heap
String literals       → Text/Read-only data segment
```

**Q: What is the default value of uninitialized variables?**

```
Global/static variables: zero-initialized by the C standard.
Local variables: indeterminate (garbage). Reading may be undefined behavior.
```

---
