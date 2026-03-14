# C99/C11/C17/C23 Features

---

## C99

```
- Variable declarations anywhere in block (not just top)
- Variable-length arrays (VLAs)
- Designated initializers: {.x = 1, .y = 2}
- Compound literals: (int[]){1, 2, 3}
- restrict keyword
- inline functions
- _Bool / <stdbool.h>
- <stdint.h> fixed-width integers
- // single-line comments
- snprintf
- long long int
- Flexible array members
```

## C11

```
- _Atomic types and <stdatomic.h>
- <threads.h> (optional)
- _Generic (type-generic macros)
- _Static_assert (compile-time assertions)
- Anonymous structs and unions
- alignof / alignas / <stdalign.h>
- _Noreturn
- Improved Unicode support
- Bounds-checking interfaces (Annex K, optional)
- VLAs made optional
```

## _Generic Example (C11)

```c
#define print_val(x) _Generic((x), \
    int:    printf("%d\n", (x)), \
    double: printf("%f\n", (x)), \
    char*:  printf("%s\n", (x))  \
)
print_val(42);       // prints: 42
print_val(3.14);     // prints: 3.140000
print_val("hello");  // prints: hello
```

## C17

```
- Bug fix release (no new features)
- Clarifications to C11 wording
```

## C23

```
- typeof / typeof_unqual
- constexpr for objects
- #embed directive (embed binary data)
- nullptr constant (replaces NULL)
- Digit separators: 1'000'000
- Binary literals: 0b1010
- Labels at end of compound statements
- Attributes: [[nodiscard]], [[maybe_unused]], [[deprecated]]
- bool, true, false as keywords (not macros)
- Improved enumerations with fixed underlying type
- auto type inference (like C++11 auto, limited)
```

---

## Interview Q&A

**Q: What is `_Static_assert`?**

```c
_Static_assert(sizeof(int) == 4, "int must be 4 bytes");
// Compile-time check. If condition is false, compilation fails with the message.
// C23 makes the message optional.
```

**Q: What is `_Generic`?**

```
A C11 feature for compile-time type selection.
Enables writing type-generic macros (similar to function overloading).
```

---
