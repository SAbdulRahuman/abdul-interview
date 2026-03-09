# Chapter 10 — Error Handling

Go handles errors as **values**, not exceptions. Functions that might fail return an `error` as the last return value. This makes error handling explicit, not hidden in call stacks.

```
┌──────────────────────────────────────────────────────────┐
│              Go Error Handling Strategy                  │
│                                                          │
│  type error interface {                                  │
│      Error() string                                      │
│  }                                                       │
│                                                          │
│  Error Strategies:                                       │
│  ┌────────────────────────────────────────────────┐      │
│  │ Sentinel Errors    → var ErrNotFound = errors.New()│  │
│  │                      Check: errors.Is(err, target) │  │
│  ├────────────────────────────────────────────────┤      │
│  │ Custom Error Types → type MyError struct { ... }   │  │
│  │                      Check: errors.As(err, &target)│  │
│  ├────────────────────────────────────────────────┤      │
│  │ Error Wrapping     → fmt.Errorf("ctx: %w", err)    │  │
│  │                      Adds context, preserves chain │  │
│  ├────────────────────────────────────────────────┤      │
│  │ panic/recover      → For truly unrecoverable errors│  │
│  │                      (programmer bugs, not normal) │  │
│  └────────────────────────────────────────────────┘      │
│                                                          │
│  Error Chain:                                            │
│  initApp: readConfig: open /cfg: no such file or dir    │
│  ◄─────── wrapped ────────── wrapped ──── original ──►  │
│                                                          │
│  errors.Is()  → traverses chain, matches by VALUE       │
│  errors.As()  → traverses chain, matches by TYPE        │
│  errors.Unwrap() → peels one layer                      │
└──────────────────────────────────────────────────────────┘
```

## The error Interface

```go
package main

import (
    "errors"
    "fmt"
)

// The error interface is built-in:
// type error interface {
//     Error() string
// }

func main() {
    // Creating errors
    err1 := errors.New("something went wrong")
    fmt.Println(err1) // something went wrong

    err2 := fmt.Errorf("failed to process item %d: %s", 42, "invalid format")
    fmt.Println(err2) // failed to process item 42: invalid format
}
```

---

## Error Wrapping (Go 1.13+)

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func readConfig(path string) error {
    _, err := os.Open(path)
    if err != nil {
        // Wrap error with context using %w
        return fmt.Errorf("readConfig: %w", err)
    }
    return nil
}

func initApp() error {
    err := readConfig("/nonexistent/config.yaml")
    if err != nil {
        return fmt.Errorf("initApp: %w", err) // wrap again with more context
    }
    return nil
}

func main() {
    err := initApp()
    if err != nil {
        fmt.Println(err)
        // initApp: readConfig: open /nonexistent/config.yaml: no such file or directory

        // Unwrap one layer
        unwrapped := errors.Unwrap(err)
        fmt.Println("Unwrapped:", unwrapped)
        // Unwrapped: readConfig: open /nonexistent/config.yaml: no such file or directory
    }
}
```

---

## errors.Is — Check Error Chain

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

var ErrPermission = errors.New("permission denied")

func openFile(path string) error {
    return fmt.Errorf("openFile %s: %w", path, os.ErrNotExist)
}

func processFile() error {
    err := openFile("/data/secrets.txt")
    if err != nil {
        return fmt.Errorf("processFile: %w", err)
    }
    return nil
}

func main() {
    err := processFile()

    // errors.Is checks the ENTIRE error chain
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("File does not exist!")
        // File does not exist!
    }

    // Regular == comparison only checks the top level
    fmt.Println(err == os.ErrNotExist) // false — they're different error values

    // errors.Is traverses the chain:
    // processFile: openFile /data/secrets.txt: file does not exist
    //                                          ^^^^^^^^^^^^^^^^^^^
    //                                          Found os.ErrNotExist!
}
```

---

## errors.As — Find Specific Error Type

```go
package main

import (
    "errors"
    "fmt"
    "net"
)

type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s - %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 || age > 150 {
        return fmt.Errorf("validateAge: %w", &ValidationError{
            Field:   "age",
            Message: fmt.Sprintf("invalid age: %d", age),
        })
    }
    return nil
}

func validateUser() error {
    return fmt.Errorf("validateUser: %w", validateAge(-5))
}

func main() {
    err := validateUser()

    // errors.As finds a specific error TYPE in the chain
    var ve *ValidationError
    if errors.As(err, &ve) {
        fmt.Println("Validation error!")
        fmt.Println("Field:", ve.Field)     // Field: age
        fmt.Println("Message:", ve.Message) // Message: invalid age: -5
    }

    // Works with standard library error types too
    _, netErr := net.Dial("tcp", "invalid:99999")
    var dnsErr *net.DNSError
    if errors.As(netErr, &dnsErr) {
        fmt.Println("DNS error:", dnsErr.Name)
    }
}
```

---

## Custom Error Types

```go
package main

import "fmt"

// Custom error with additional context
type HTTPError struct {
    StatusCode int
    Message    string
    URL        string
}

func (e *HTTPError) Error() string {
    return fmt.Sprintf("HTTP %d: %s (url: %s)", e.StatusCode, e.Message, e.URL)
}

// Custom error wrapping another error
type DatabaseError struct {
    Operation string
    Table     string
    Err       error // wrapped error
}

func (e *DatabaseError) Error() string {
    return fmt.Sprintf("db error during %s on %s: %v", e.Operation, e.Table, e.Err)
}

func (e *DatabaseError) Unwrap() error {
    return e.Err // allows errors.Is and errors.As to traverse this
}

func main() {
    err := &HTTPError{
        StatusCode: 404,
        Message:    "Not Found",
        URL:        "/api/users/123",
    }
    fmt.Println(err) // HTTP 404: Not Found (url: /api/users/123)

    // Check status code
    if err.StatusCode == 404 {
        fmt.Println("Resource not found")
    }
}
```

---

## Sentinel Errors

```go
package main

import (
    "errors"
    "fmt"
)

// Sentinel errors — package-level variables for known error conditions
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
    ErrConflict     = errors.New("conflict")
)

type UserStore struct {
    users map[int]string
}

func NewUserStore() *UserStore {
    return &UserStore{
        users: map[int]string{1: "Alice", 2: "Bob"},
    }
}

func (s *UserStore) GetUser(id int) (string, error) {
    user, ok := s.users[id]
    if !ok {
        return "", fmt.Errorf("GetUser(%d): %w", id, ErrNotFound)
    }
    return user, nil
}

func main() {
    store := NewUserStore()

    user, err := store.GetUser(1)
    fmt.Println(user, err) // Alice <nil>

    _, err = store.GetUser(999)
    if errors.Is(err, ErrNotFound) {
        fmt.Println("User not found!") // User not found!
    }
}
```

---

## Error Handling Patterns

```go
package main

import (
    "errors"
    "fmt"
)

// Pattern 1: Early return on error
func processOrder(orderID string) error {
    if orderID == "" {
        return errors.New("order ID cannot be empty")
    }

    items, err := fetchItems(orderID)
    if err != nil {
        return fmt.Errorf("processOrder: %w", err) // wrap and return early
    }

    total, err := calculateTotal(items)
    if err != nil {
        return fmt.Errorf("processOrder: %w", err)
    }

    fmt.Printf("Order %s total: $%.2f\n", orderID, total)
    return nil
}

func fetchItems(orderID string) ([]string, error) {
    if orderID == "bad" {
        return nil, errors.New("items not found")
    }
    return []string{"item1", "item2"}, nil
}

func calculateTotal(items []string) (float64, error) {
    return float64(len(items)) * 9.99, nil
}

// Pattern 2: errors.Join (Go 1.20+) — combine multiple errors
func validateForm(name, email string) error {
    var errs []error

    if name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    if email == "" {
        errs = append(errs, errors.New("email is required"))
    }

    return errors.Join(errs...) // returns nil if no errors
}

func main() {
    // Early return
    err := processOrder("ORD-123")
    fmt.Println(err) // <nil> (success)

    err = processOrder("bad")
    fmt.Println(err) // processOrder: items not found

    // errors.Join
    err = validateForm("", "")
    if err != nil {
        fmt.Println("Validation errors:")
        fmt.Println(err)
        // name is required
        // email is required
    }

    err = validateForm("Alice", "alice@test.com")
    fmt.Println(err) // <nil>
}
```

---

## panic and recover — When to Use

```go
package main

import "fmt"

// panic: use ONLY for truly unrecoverable situations
// - Programmer bugs (index out of range, nil pointer)
// - Initialization failures (can't connect to essential database)
// - Impossible states

// recover: catches panics in deferred functions
// - Useful in HTTP servers (don't crash server for one bad request)
// - Library boundaries (convert panic to error)

func safeHandler(handlerName string, fn func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Handler %s panicked: %v\n", handlerName, r)
        }
    }()
    fn()
}

func main() {
    safeHandler("order", func() {
        fmt.Println("Processing order...")
        panic("database connection lost!")
    })

    // Program continues after recover
    safeHandler("payment", func() {
        fmt.Println("Processing payment...")
        fmt.Println("Payment successful!")
    })

    fmt.Println("Server still running!")

    // Output:
    // Processing order...
    // Handler order panicked: database connection lost!
    // Processing payment...
    // Payment successful!
    // Server still running!
}
```

---

## Interview Questions

1. **How does error handling work in Go?**
   - Go uses explicit error return values instead of exceptions. Functions return `error` as the last return value. Callers check `if err != nil` and handle or propagate errors.

2. **What is error wrapping and why is it important?**
   - `fmt.Errorf("context: %w", err)` wraps an error with additional context while preserving the original error chain. This enables `errors.Is` and `errors.As` to match errors deeper in the chain.

3. **What is the difference between `errors.Is` and `errors.As`?**
   - `errors.Is(err, target)` checks if any error in the chain matches a specific value (for sentinel errors). `errors.As(err, &target)` finds the first error in the chain matching a specific type and assigns it.

4. **What are sentinel errors?**
   - Package-level error variables: `var ErrNotFound = errors.New("not found")`. They represent specific, well-known error conditions that callers can check with `errors.Is`.

5. **When should you use `panic` vs returning an error?**
   - Use `error` for expected failure conditions (file not found, invalid input). Use `panic` only for truly unrecoverable situations (programming bugs, impossible states). Libraries should almost never panic.

6. **How does `recover` work?**
   - `recover()` must be called inside a deferred function. It catches a panic, returns the panic value, and allows the program to continue. Outside a deferred function, it returns `nil`.

7. **What is `errors.Join` (Go 1.20+)?**
   - Combines multiple errors into one: `errors.Join(err1, err2, err3)`. The result works with `errors.Is` and `errors.As` for all wrapped errors. Useful for aggregating validation errors.

8. **What is the idiomatic error handling pattern in Go?**
   - Early return on error, wrap with context at each layer: `if err != nil { return fmt.Errorf("operation X: %w", err) }`. Never ignore errors silently.

9. **How do you create a custom error type?**
   - Implement the `error` interface by defining an `Error() string` method on your type. Custom errors can carry structured data (status codes, field names, etc.).

10. **What happens if you return a typed nil pointer as an error interface?**
    - The interface will be non-nil (it holds type info), so `err != nil` is `true` even though the underlying value is nil. This is the nil interface gotcha — always return `nil` explicitly for no-error.
