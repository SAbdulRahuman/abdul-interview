# Chapter 14 — Context Package

The `context` package provides a way to carry **deadlines, cancellation signals, and request-scoped values** through the call chain. It's essential for managing the lifecycle of operations in servers, where you need to cancel work when a client disconnects or a timeout expires.

```
┌──────────────────────────────────────────────────────────┐
│              Context Hierarchy                          │
│                                                          │
│  context.Background()     ← root (never cancelled)      │
│    │                                                     │
│    ├── WithCancel(parent)                                 │
│    │     │  → Cancel manually with cancel()              │
│    │     │                                               │
│    │     ├── WithTimeout(parent, 5s)                     │
│    │     │     → Auto-cancels after 5 seconds            │
│    │     │                                               │
│    │     └── WithValue(parent, key, val)                 │
│    │           → Carries request-scoped data             │
│    │                                                     │
│    └── WithDeadline(parent, time)                        │
│          → Cancels at specific wall-clock time           │
│                                                          │
│  Cancellation propagates DOWN:                          │
│  Parent cancelled → ALL children cancelled              │
│                                                          │
│  Usage Pattern:                                          │
│  func handler(ctx context.Context) {                     │
│      select {                                            │
│      case <-ctx.Done():     ← react to cancellation     │
│          return ctx.Err()   ← Canceled or DeadlineExceeded│
│      case result := <-work():                            │
│          return result                                   │
│      }                                                   │
│  }                                                       │
│                                                          │
│  ⚠ ALWAYS pass context as the first parameter           │
│  ⚠ NEVER store context in a struct                      │
│  ⚠ ALWAYS call cancel() (even if timeout fires)         │
└──────────────────────────────────────────────────────────┘
```

## context.Background() and context.TODO()

```go
package main

import (
    "context"
    "fmt"
)

func main() {
    // context.Background() — the root context
    // Use at the top level: main, init, or incoming requests
    ctx := context.Background()
    fmt.Println(ctx) // context.Background

    // context.TODO() — placeholder when you're not sure which context to use
    // Use during refactoring when context isn't available yet
    todoCtx := context.TODO()
    fmt.Println(todoCtx) // context.TODO
}
```

---

## context.WithCancel

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: cancelled (%v)\n", id, ctx.Err())
            return
        default:
            fmt.Printf("Worker %d: working...\n", id)
            time.Sleep(100 * time.Millisecond)
        }
    }
}

func main() {
    // WithCancel returns a new context and a cancel function
    ctx, cancel := context.WithCancel(context.Background())

    go worker(ctx, 1)
    go worker(ctx, 2)

    time.Sleep(350 * time.Millisecond)

    // Calling cancel() closes ctx.Done() channel, signaling all workers
    cancel()
    time.Sleep(50 * time.Millisecond) // let workers print their cancellation message

    fmt.Println("Main: all workers cancelled")
}
```

---

## context.WithTimeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func slowOperation(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second):
        fmt.Println("Operation completed")
        return nil
    case <-ctx.Done():
        return fmt.Errorf("operation cancelled: %w", ctx.Err())
    }
}

func main() {
    // WithTimeout auto-cancels after the specified duration
    ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
    defer cancel() // always call cancel to release resources

    err := slowOperation(ctx)
    if err != nil {
        fmt.Println("Error:", err)
        // Error: operation cancelled: context deadline exceeded
    }

    // Check what caused the cancellation
    fmt.Println("Context error:", ctx.Err())
    // context.DeadlineExceeded
}
```

---

## context.WithDeadline

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // WithDeadline sets an absolute time for cancellation
    deadline := time.Now().Add(300 * time.Millisecond)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()

    select {
    case <-time.After(1 * time.Second):
        fmt.Println("Completed")
    case <-ctx.Done():
        fmt.Println("Deadline exceeded:", ctx.Err())
        // Deadline exceeded: context deadline exceeded
    }

    // Check deadline
    dl, ok := ctx.Deadline()
    fmt.Printf("Deadline: %v, Has deadline: %v\n", dl.Format(time.RFC3339), ok)
}
```

---

## context.WithValue

```go
package main

import (
    "context"
    "fmt"
)

// Use custom types for context keys to avoid collisions
type contextKey string

const (
    userIDKey    contextKey = "userID"
    requestIDKey contextKey = "requestID"
)

func processRequest(ctx context.Context) {
    userID := ctx.Value(userIDKey).(string)
    requestID := ctx.Value(requestIDKey).(string)
    fmt.Printf("Processing request %s for user %s\n", requestID, userID)
}

func middleware(ctx context.Context, userID, requestID string) context.Context {
    ctx = context.WithValue(ctx, userIDKey, userID)
    ctx = context.WithValue(ctx, requestIDKey, requestID)
    return ctx
}

func main() {
    ctx := context.Background()

    // Add values to context
    ctx = middleware(ctx, "user-123", "req-456")

    processRequest(ctx)
    // Processing request req-456 for user user-123

    // Missing key returns nil
    val := ctx.Value(contextKey("missing"))
    fmt.Println("Missing key:", val) // Missing key: <nil>
}
```

---

## Context Propagation — Best Practices

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// ✅ Always pass context as FIRST parameter
func fetchUser(ctx context.Context, userID string) (string, error) {
    select {
    case <-time.After(50 * time.Millisecond):
        return "Alice", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func fetchOrders(ctx context.Context, userID string) ([]string, error) {
    select {
    case <-time.After(50 * time.Millisecond):
        return []string{"order-1", "order-2"}, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}

// ✅ Propagate context through call chain
func handleUserRequest(ctx context.Context, userID string) error {
    user, err := fetchUser(ctx, userID)
    if err != nil {
        return fmt.Errorf("fetch user: %w", err)
    }

    orders, err := fetchOrders(ctx, userID)
    if err != nil {
        return fmt.Errorf("fetch orders: %w", err)
    }

    fmt.Printf("User: %s, Orders: %v\n", user, orders)
    return nil
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
    defer cancel() // ✅ Always call cancel (use defer)

    err := handleUserRequest(ctx, "user-123")
    if err != nil {
        fmt.Println("Error:", err)
    }
    // User: Alice, Orders: [order-1 order-2]

    // Best practices:
    // 1. Never store context in a struct
    // 2. Always pass as first parameter named 'ctx'
    // 3. Use WithValue sparingly — prefer explicit function parameters
    // 4. Always call cancel() to release resources (defer cancel())
}
```

---

## context.WithoutCancel and context.AfterFunc (Go 1.21+)

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // context.WithoutCancel (Go 1.21+)
    // Creates a child context that is NOT cancelled when parent is cancelled
    // Useful for cleanup operations that should complete even after cancellation

    parentCtx, parentCancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer parentCancel()

    // This context inherits values but NOT cancellation
    cleanupCtx := context.WithoutCancel(parentCtx)

    time.Sleep(150 * time.Millisecond) // parent times out

    fmt.Println("Parent cancelled:", parentCtx.Err())   // context deadline exceeded
    fmt.Println("Cleanup cancelled:", cleanupCtx.Err())  // <nil> — not cancelled!

    // context.AfterFunc (Go 1.21+)
    // Runs a function after context is done (in its own goroutine)

    ctx2, cancel2 := context.WithCancel(context.Background())

    context.AfterFunc(ctx2, func() {
        fmt.Println("AfterFunc: context was cancelled, performing cleanup")
    })

    cancel2() // triggers the AfterFunc
    time.Sleep(50 * time.Millisecond)
}
```

---

## Interview Questions

1. **What is `context.Context` used for in Go?**
   - It carries deadlines, cancellation signals, and request-scoped values across API boundaries and goroutines. It's the standard mechanism for cancellation propagation.

2. **What is the difference between `context.Background()` and `context.TODO()`?**
   - Both return empty, non-nil contexts. `Background()` is the top-level default for main, init, and tests. `TODO()` is a placeholder when you're unsure which context to use — it signals intent to add one later.

3. **How does `context.WithCancel` work?**
   - Returns a derived context and a `cancel` function. Calling `cancel()` closes the context's `Done()` channel, notifying all goroutines watching it. Always `defer cancel()` to avoid leaks.

4. **What is the difference between `WithTimeout` and `WithDeadline`?**
   - `WithTimeout(ctx, 5*time.Second)` cancels after a relative duration. `WithDeadline(ctx, time.Now().Add(5*time.Second))` cancels at an absolute time. They're functionally equivalent.

5. **Should you store context in a struct?**
   - No. Always pass context as the first parameter: `func DoWork(ctx context.Context, ...)`. Storing it in a struct makes the lifecycle unclear and prevents proper propagation.

6. **What happens when a parent context is cancelled?**
   - All child contexts derived from it are also cancelled. Cancellation propagates down the context tree automatically.

7. **When should you use `context.WithValue`?**
   - Sparingly — only for request-scoped data that transit processes and APIs (trace IDs, auth tokens). Never for function parameters. Use unexported key types to avoid collisions.

8. **What does `ctx.Done()` return?**
   - A channel (`<-chan struct{}`) that is closed when the context is cancelled or times out. Select on it to detect cancellation: `case <-ctx.Done():`.

9. **What is `ctx.Err()` and what values can it return?**
   - Returns `nil` (not cancelled), `context.Canceled` (explicitly cancelled), or `context.DeadlineExceeded` (timeout/deadline hit).

10. **What is `context.WithoutCancel` (Go 1.21+)?**
    - Creates a child context that is NOT cancelled when the parent is cancelled. Useful when you need the parent's values but want to continue work beyond the parent's lifetime.
