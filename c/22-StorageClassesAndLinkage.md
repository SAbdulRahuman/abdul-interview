# Storage Classes & Linkage

---

## Storage Class Specifiers

```
auto     – default for local variables (stack). Rarely written explicitly.
register – hint to store in CPU register. Cannot take address. Modern compilers ignore.
static   – different meaning depending on context:
           Local variable: persists across function calls (stored in data segment).
           Global variable/function: internal linkage (visible only within file).
extern   – declares a variable/function defined in another translation unit.
_Thread_local (C11) – one instance per thread.
```

## Linkage

```
External linkage: visible across translation units (default for functions & global variables)
Internal linkage: visible only within translation unit (static globals & static functions)
No linkage:       local variables

// file1.c
int global_var = 10;         // external linkage
static int file_var = 20;   // internal linkage

// file2.c
extern int global_var;       // declaration (no new storage)
// Cannot access file_var from file1.c
```

## Static Local Variables

```c
void counter() {
    static int count = 0;  // initialized once, persists across calls
    count++;
    printf("Called %d times\n", count);
}
// counter() → 1, 2, 3, ...
```

---

## Interview Q&A

**Q: What does `static` mean in different contexts?**

```
1. Static local variable: retains value between function calls.
2. Static global variable: restricts visibility to current file (internal linkage).
3. Static function: restricts visibility to current file (internal linkage).
```

**Q: What is the difference between `extern` declaration and definition?**

```
extern int x;     // declaration – no memory allocated, says "x is defined elsewhere"
int x = 10;       // definition – allocates memory and initializes
extern int x = 10;// definition (extern is redundant when initializer is present)
```

**Q: What is a static function?**

```
A function with internal linkage – visible only within its translation unit (.c file).
Used for: private helper functions, avoiding name collisions across files.
```

---
