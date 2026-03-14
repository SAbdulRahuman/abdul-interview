# Data Types

---

## Primitive Types

```
Type          Size (typical)    Range (signed)
─────────────────────────────────────────────
char          1 byte            -128 to 127
short         2 bytes           -32,768 to 32,767
int           4 bytes           -2^31 to 2^31-1
long          4 or 8 bytes      platform-dependent
long long     8 bytes           -2^63 to 2^63-1
float         4 bytes           ~6-7 decimal digits precision
double        8 bytes           ~15-16 decimal digits precision
long double   10/12/16 bytes    implementation-defined
```

## Modifiers

```
signed    – can hold negative values (default for char depends on platform)
unsigned  – only non-negative values
short     – reduces storage size
long      – increases storage size
```

## Type Qualifiers

```
const      – value cannot be modified after initialization
volatile   – value may change unexpectedly (don't optimize away)
restrict   – pointer is the only way to access that memory (C99)
_Atomic    – atomic access (C11)
```

## Derived Types

```
- Arrays:      int arr[10];
- Pointers:    int *p;
- Structures:  struct Point { int x, y; };
- Unions:      union Data { int i; float f; };
- Enumerations: enum Color { RED, GREEN, BLUE };
- Function types: int (*fp)(int, int);
```

---

## Interview Q&A

**Q: What is the difference between `unsigned int` and `signed int`?**

```
signed int: can hold negative and positive values (e.g., -2^31 to 2^31-1 for 32-bit).
unsigned int: only non-negative values (0 to 2^32-1 for 32-bit), no sign bit.
```

**Q: What does `sizeof` return?**

```
sizeof returns the size (in bytes) of a type or variable at compile time.
sizeof(char) is always 1.
sizeof returns a value of type size_t (unsigned).
```

**Q: Is `sizeof` an operator or a function?**

```
sizeof is a compile-time OPERATOR, not a function.
Parentheses are required for types: sizeof(int)
Optional for variables: sizeof x
```

**Q: What happens when you assign a negative value to an unsigned variable?**

```
The value wraps around (modular arithmetic).
unsigned int x = -1;  // x = UINT_MAX (4294967295 on 32-bit)
```

---
