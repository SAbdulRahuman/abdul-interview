# Preprocessor

---

## Directives

```c
#include <stdio.h>       // search system include paths
#include "myheader.h"    // search current directory first

#define MAX 100
#define SQUARE(x) ((x) * (x))    // always parenthesize macro arguments!
#undef MAX

// Conditional compilation
#ifdef DEBUG
    printf("Debug mode\n");
#endif

#ifndef HEADER_H         // include guard
#define HEADER_H
// ... declarations ...
#endif

#if defined(__linux__)
    // Linux-specific code
#elif defined(_WIN32)
    // Windows-specific code
#endif

#pragma once             // non-standard but widely supported include guard
```

## Macro Pitfalls

```c
// Bug: SQUARE(1+2) → (1+2) * (1+2) = 9 ✓ (with parens)
//      Without parens: 1+2*1+2 = 5 ✗
#define SQUARE(x) ((x) * (x))

// Bug: double evaluation
#define MAX(a,b) ((a) > (b) ? (a) : (b))
MAX(i++, j++)  // i++ or j++ evaluated TWICE!

// Multiline macro
#define SWAP(a, b) do { \
    typeof(a) _tmp = (a); \
    (a) = (b);            \
    (b) = _tmp;           \
} while (0)

// Stringification: #
#define STR(x) #x     // STR(hello) → "hello"

// Token pasting: ##
#define CONCAT(a, b) a##b   // CONCAT(var, 1) → var1

// __VA_ARGS__ (variadic macros)
#define LOG(fmt, ...) fprintf(stderr, fmt, ##__VA_ARGS__)
```

## Predefined Macros

```c
__FILE__     // current file name
__LINE__     // current line number
__func__     // current function name (C99)
__DATE__     // compilation date
__TIME__     // compilation time
__STDC__     // 1 if standard-conforming compiler
__STDC_VERSION__  // e.g., 201112L for C11
```

---

## Interview Q&A

**Q: Why wrap multi-statement macros in `do { ... } while(0)`?**

```
To make the macro behave as a single statement in if-else:

#define BAD(x) { a(); b(); }
if (cond) BAD(x); else other();  // syntax error!

#define GOOD(x) do { a(); b(); } while(0)
if (cond) GOOD(x); else other();  // works correctly
```

**Q: What is the difference between `#include <>` and `#include ""`?**

```
<header.h>: searches system include directories only.
"header.h": searches current directory first, then system directories.
```

**Q: What does `##` do in macros?**

```
Token pasting: concatenates two tokens into one.
#define CONCAT(a, b) a##b
CONCAT(my, Var) → myVar
```

---
