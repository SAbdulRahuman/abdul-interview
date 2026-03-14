# setjmp / longjmp (Non-Local Jumps)

---

## Usage

```c
#include <setjmp.h>

jmp_buf env;

void error_func() {
    longjmp(env, 1);  // jump back to setjmp, returning 1
}

int main() {
    if (setjmp(env) == 0) {
        // Normal execution (first time)
        error_func();
    } else {
        // Jumped back from longjmp
        printf("Error caught\n");
    }
}
```

## How It Works

```
setjmp(env): saves the current execution context (registers, stack pointer, etc.)
             into the jmp_buf. Returns 0 on first call.
longjmp(env, val): restores the saved context. setjmp appears to return 'val'.
```

## Caveats

```
- Does NOT unwind the stack properly
- Does NOT call cleanup/destructors
- Local variables modified between setjmp and longjmp have indeterminate values
  (unless declared volatile)
- Calling longjmp after the function that called setjmp has returned → UB
- Used for primitive exception handling in C
```

---

## Interview Q&A

**Q: When would you use setjmp/longjmp?**

```
1. Error handling / exception simulation in C
2. Implementing coroutines or cooperative multitasking
3. Breaking out of deeply nested function calls

In most cases, return codes and goto cleanup are preferred.
```

**Q: Why are local variables indeterminate after longjmp?**

```
The compiler may have stored them in registers that get overwritten.
Mark variables as 'volatile' if they need to be preserved across longjmp.
```

---
