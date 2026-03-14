# Inline Functions (C99)

---

## Usage

```c
static inline int max(int a, int b) {
    return (a > b) ? a : b;
}
// Suggests to compiler to expand inline (not guaranteed).
// Must be defined in header if used across translation units.
```

## Rules

```
- 'inline' is a hint, not a command. Compiler may ignore it.
- 'static inline' in header: each translation unit gets its own copy.
- 'extern inline' in one .c file: provides a single external definition.
- Modern compilers auto-inline small functions with optimization enabled (-O2).
```

## Inline vs Macro

```
Inline functions:
  - Type-checked
  - Arguments evaluated once
  - Can be debugged
  - Scoped properly

Macros:
  - No type checking
  - Arguments may be evaluated multiple times
  - Cannot be debugged easily
  - Can cause unexpected side effects
```

---

## Interview Q&A

**Q: Why use `static inline` instead of just `inline`?**

```
Without 'static', inline in a header can cause linker errors
(multiple definitions). 'static inline' gives each translation unit
its own copy, avoiding linker issues.
```

**Q: Does inline guarantee inlining?**

```
No. It's a suggestion. The compiler decides based on:
  - Function size and complexity
  - Optimization level
  - Whether function address is taken
```

---
