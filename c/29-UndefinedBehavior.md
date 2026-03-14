# Undefined, Unspecified & Implementation-Defined Behavior

---

## Undefined Behavior (UB)

```
Anything can happen – crash, wrong results, or appear to "work":
  - Null pointer dereference
  - Signed integer overflow
  - Array out of bounds
  - Use after free / double free
  - Modifying string literals
  - Sequence point violations (i++ + i++)
  - Shifting by negative or >= bit-width
  - Accessing uninitialized variables
  - Violating strict aliasing rules
  - Buffer overflows
```

## Unspecified Behavior

```
Valid but unpredictable – the standard allows multiple outcomes:
  - Evaluation order of function arguments: f(a(), b()) – a() or b() first?
  - Order of evaluation of operands of most operators
  - Which of two equal elements comes first after qsort

The program is still well-formed, just non-deterministic.
```

## Implementation-Defined Behavior

```
Documented by the compiler/platform:
  - sizeof(int), sizeof(long)
  - Right shift of negative signed integers
  - Whether char is signed or unsigned by default
  - Behavior of casting out-of-range values
  - Alignment requirements
```

---

## Interview Q&A

**Q: Why does undefined behavior exist in C?**

```
To give the compiler maximum freedom for optimization.
If the compiler can assume UB never occurs, it can:
  - Eliminate unreachable code paths
  - Reorder operations
  - Omit bounds checks
This makes C fast, but puts responsibility on the programmer.
```

**Q: Give 5 common examples of undefined behavior.**

```
1. int x = INT_MAX + 1;          // signed overflow
2. int *p = NULL; *p = 10;       // null dereference
3. int a[5]; a[10] = 1;          // out of bounds
4. free(p); free(p);             // double free
5. char *s = "hello"; s[0]='H';  // modifying string literal
```

---
