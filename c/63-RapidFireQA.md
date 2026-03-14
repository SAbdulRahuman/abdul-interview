# Rapid-Fire Interview Q&A

---

| Question | Answer |
|----------|--------|
| Is C pass-by-value or pass-by-reference? | Always pass-by-value. Pointers simulate pass-by-reference. |
| Can `main()` call itself? | Yes, recursion of `main()` is allowed in C. |
| What is a segmentation fault? | Accessing memory the process doesn't own (NULL deref, stack overflow, etc.). |
| Difference between `malloc` and `calloc`? | `calloc` zero-initializes and takes two args (count, size). |
| What is a memory leak? | Allocated memory that is never freed and has no remaining reference. |
| What is `size_t`? | Unsigned integer type for sizes, returned by `sizeof`. |
| What is a translation unit? | A source file after preprocessing (all `#include`s expanded). |
| Can `sizeof` be used on `void`? | `sizeof(void)` is undefined in standard C. GCC allows it (= 1). |
| What is the value of uninitialized local variable? | Indeterminate (garbage). Reading it may be UB. |
| Difference between `struct` and `union`? | Struct: members have separate memory. Union: members share memory. |
| What is `restrict`? | Promise to compiler that pointer is the only way to access that memory. |
| Can you `free(NULL)`? | Yes, `free(NULL)` is safe (no operation). |
| What does `exit()` vs `_exit()`? | `exit()` flushes buffers, calls `atexit` handlers. `_exit()` immediately terminates. |
| Header guard vs `#pragma once`? | Header guard is standard. `#pragma once` is non-standard but widely supported. |
| What is a static function? | A function with internal linkage – visible only within its translation unit. |
| What is the null pointer constant in C? | `((void*)0)` or `0` or `NULL`. C23 adds `nullptr`. |
| Can arrays have zero length? | Not in standard C. GCC allows zero-length arrays as extension. |
| What is `CHAR_BIT`? | Number of bits in a char. Always at least 8. Defined in `<limits.h>`. |
| What does `auto` keyword do? | Default storage class for local variables (stack). Rarely written explicitly. |
| Difference between `#include ""` and `#include <>`? | `""` searches current directory first, `<>` searches system paths only. |
| What is the return type of `sizeof`? | `size_t` (unsigned integer type). |
| Can you compare structs with `==`? | No. Must compare member by member. |
| What is `EOF`? | A negative integer constant (typically -1) indicating end-of-file. |
| Difference between `fgets` and `gets`? | `fgets` is safe (takes size), `gets` is unsafe (removed in C11). |
| What is a forward declaration? | Declaring a struct/function before its full definition. |

---
