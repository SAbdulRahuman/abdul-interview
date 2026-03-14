# Signals

---

## Signal Handling

```c
#include <signal.h>

volatile sig_atomic_t got_signal = 0;

void handler(int sig) {
    got_signal = 1;  // only async-signal-safe operations!
}

int main() {
    signal(SIGINT, handler);    // or use sigaction() (more portable)
    while (!got_signal) {
        // work
    }
    printf("Caught signal\n");
    return 0;
}
```

## Common Signals

```
SIGINT    – Interrupt (Ctrl+C)
SIGTERM   – Termination request
SIGSEGV   – Segmentation fault
SIGFPE    – Floating-point exception
SIGABRT   – Abort (called by abort())
SIGKILL   – Kill (cannot be caught)
SIGSTOP   – Stop (cannot be caught)
SIGALRM   – Alarm timer
SIGCHLD   – Child process terminated
```

## sigaction (Preferred)

```c
struct sigaction sa;
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
sigaction(SIGINT, &sa, NULL);  // more portable and reliable than signal()
```

---

## Interview Q&A

**Q: What can you safely do inside a signal handler?**

```
Very limited – only async-signal-safe functions:
  - Set sig_atomic_t variables
  - Call _exit(), write(), signal()
  - Cannot call: printf, malloc, free, or any non-reentrant function
```

**Q: Difference between `signal()` and `sigaction()`?**

```
signal(): simpler but behavior varies across platforms. May reset handler after first call.
sigaction(): more portable, reliable, and feature-rich. Preferred in production code.
```

---
