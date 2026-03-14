# Sequence Points

---

## What Are Sequence Points?

```
A sequence point guarantees all side effects of previous expressions are complete
before evaluating the next expression.
```

## Sequence Points in C

```
- End of full expression (semicolon)
- Before && || ?: , operators evaluate their right operand
- After all function arguments are evaluated (before the function call)
- At the end of the first operand of the comma operator
- After initialization in a declaration
```

## Undefined Behavior Examples

```c
// UNDEFINED: modifying same variable twice between sequence points
i = i++ + ++i;   // UB!
a[i] = i++;      // UB!
printf("%d %d\n", i++, i++);  // UB: order of argument evaluation is unspecified

// DEFINED:
i++; i++;        // OK: semicolon is a sequence point
a && b++;        // OK: && is a sequence point
f(i++);          // OK: one modification, sequence point before call
```

---

## Interview Q&A

**Q: What is the output of `printf("%d %d", i++, ++i)`?**

```
UNDEFINED BEHAVIOR. The order of evaluation of function arguments is unspecified,
and modifying i twice without a sequence point between is undefined.
```

**Q: Is `a[i] = i++` defined?**

```
No. It is undefined behavior. The variable 'i' is both read (for array index)
and modified (i++) without an intervening sequence point.
```

---
