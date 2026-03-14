# Variadic Functions

---

## Implementation

```c
#include <stdarg.h>

int sum(int count, ...) {
    va_list args;
    va_start(args, count);
    int total = 0;
    for (int i = 0; i < count; i++)
        total += va_arg(args, int);
    va_end(args);
    return total;
}
// sum(3, 10, 20, 30) → 60
```

## API

```c
va_list args;              // declare argument list
va_start(args, last_named); // initialize (last_named = last fixed parameter)
va_arg(args, type);         // get next argument of specified type
va_end(args);               // cleanup
va_copy(dest, src);         // copy va_list (C99)
```

## Rules and Caveats

```
- Must have at least one named parameter before ...
- No type checking on variadic arguments
- Default argument promotions apply:
    float → double
    char, short → int
- Caller must know the number and types of arguments
  (via count parameter, format string, or sentinel value)
```

---

## Interview Q&A

**Q: How does `printf` know the types of its arguments?**

```
From the format string. Each %d, %s, %f tells printf what type to expect.
If format string and actual types don't match → undefined behavior.
```

**Q: Can you pass a va_list to another function?**

```c
void vlog(const char *fmt, va_list args) {
    vprintf(fmt, args);
}

void log(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    vlog(fmt, args);
    va_end(args);
}
```

---
