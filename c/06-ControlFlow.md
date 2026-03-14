# Control Flow

---

## Conditional Statements

```c
// if-else
if (cond) {
    // ...
} else if (cond2) {
    // ...
} else {
    // ...
}

// switch
switch (expr) {
    case val1:
        // ...
        break;
    case val2:
        // fall-through if no break
    default:
        // ...
}
```

## Loops

```c
// for loop
for (init; cond; update) {
    // ...
}

// while loop
while (cond) {
    // ...
}

// do-while loop (executes at least once)
do {
    // ...
} while (cond);
```

## Jump Statements

```c
break;       // exit innermost loop/switch
continue;    // skip to next iteration
goto label;  // unconditional jump (use sparingly)
return expr; // return from function
```

---

## Interview Q&A

**Q: What is the difference between `while` and `do-while`?**

```
while: checks condition before executing the body (may execute 0 times).
do-while: executes body first, then checks condition (always executes at least once).
```

**Q: What happens if you forget `break` in a switch case?**

```
Fall-through: execution continues into the next case.
This is sometimes intentional (e.g., multiple cases with same handler).
```

**Q: When is `goto` acceptable?**

```
Commonly used in C for centralized error cleanup in functions (especially in Linux kernel code):

int func() {
    if (alloc1() < 0) goto err1;
    if (alloc2() < 0) goto err2;
    return 0;
err2: free2();
err1: free1();
    return -1;
}
```

**Q: Can you use a variable in a switch case label?**

```
No. Case labels must be compile-time integer constants.
switch (x) {
    case 1:      // OK
    case 'A':    // OK (char is int)
    case 1+2:    // OK (constant expression)
    case y:      // ERROR – not a constant
}
```

---
