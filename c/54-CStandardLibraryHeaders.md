# C Standard Library Headers

---

## Key Headers Reference

```
<stdio.h>    – I/O: printf, scanf, fopen, fread, fwrite
<stdlib.h>   – General: malloc, free, atoi, qsort, bsearch, exit, rand
<string.h>   – Strings: strlen, strcpy, strcmp, memcpy, memset, strtok
<math.h>     – Math: sqrt, pow, sin, cos, fabs, ceil, floor (link with -lm)
<ctype.h>    – Char classification: isalpha, isdigit, toupper, tolower
<assert.h>   – assert(expr) – aborts if expr is false (disabled with NDEBUG)
<errno.h>    – Error codes: errno, perror, strerror
<signal.h>   – Signal handling: signal, raise, SIGINT, SIGSEGV
<setjmp.h>   – Non-local jumps: setjmp, longjmp
<stdarg.h>   – Variadic functions: va_list, va_start, va_arg, va_end
<stdint.h>   – Fixed-width integers: int8_t, uint32_t, INT_MAX (C99)
<stdbool.h>  – bool, true, false (C99)
<stddef.h>   – NULL, size_t, ptrdiff_t, offsetof
<limits.h>   – INT_MAX, INT_MIN, CHAR_BIT, etc.
<float.h>    – FLT_MAX, DBL_EPSILON, etc.
<time.h>     – time, clock, difftime, strftime
<complex.h>  – Complex number arithmetic (C99)
<threads.h>  – C11 threading: thrd_create, mtx_lock, cnd_wait
<stdatomic.h>– C11 atomics: atomic_int, atomic_fetch_add
<stdalign.h> – alignof, alignas (C11)
<stdnoreturn.h> – _Noreturn (C11)
```

---

## Interview Q&A

**Q: What does `assert()` do?**

```
Evaluates an expression. If false (0), prints an error message and calls abort().
Disabled when NDEBUG is defined: gcc -DNDEBUG program.c
Used for: checking programmer assumptions, catching logic errors during development.
Should NOT be used for runtime error handling in production.
```

**Q: Why link with `-lm` for math functions?**

```
On some systems (especially Linux/glibc), math functions (sin, cos, sqrt, pow)
are in a separate math library (libm). The -lm flag tells the linker to include it.
```

---
