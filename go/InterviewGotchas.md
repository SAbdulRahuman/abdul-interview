# Chapter 27 — Common Interview Gotchas & Tricky Questions

These are the traps and subtleties that frequently appear in Go interviews. Understanding them shows deep knowledge of Go's internals.

```
┌──────────────────────────────────────────────────────────┐
│             Top Go Interview Gotchas                    │
│                                                          │
│  1. Slice append may share underlying array             │
│     → Use full slice expr a[low:high:max] to protect   │
│                                                          │
│  2. Map iteration order is random                       │
│     → Sort keys if you need deterministic order         │
│                                                          │
│  3. Nil interface ≠ interface holding nil               │
│     → Return nil directly, not typed nil pointer        │
│                                                          │
│  4. Loop variable capture (pre-Go 1.22)                 │
│     → All closures captured same variable               │
│     → Fixed in Go 1.22 (each iteration = new var)      │
│                                                          │
│  5. defer args evaluated immediately                    │
│     → defer f(x) captures x's current value            │
│                                                          │
│  6. Writing to nil map panics                           │
│     → Reading from nil map returns zero value           │
│                                                          │
│  7. string(65) = "A", not "65"                          │
│     → Use strconv.Itoa(65) for "65"                    │
│                                                          │
│  8. Goroutine leak: blocked goroutines = memory leak    │
│     → Always ensure goroutines can exit                 │
│                                                          │
│  9. sync.Mutex is not reentrant                         │
│     → Same goroutine locking twice = deadlock           │
│                                                          │
│  10. Copying sync types (Mutex, WaitGroup) = bug       │
│      → Pass by pointer, never by value                 │
└──────────────────────────────────────────────────────────┘
```

## 1. Slice Append & Capacity

```go
package main

import "fmt"

func main() {
    // append may return a NEW underlying array
    a := make([]int, 3, 5)
    a[0], a[1], a[2] = 1, 2, 3

    b := append(a, 4) // capacity 5, still fits — shares underlying array
    b[0] = 99

    fmt.Println("a:", a) // a: [99 2 3]  ← MODIFIED because same array!
    fmt.Println("b:", b) // b: [99 2 3 4]

    // Now trigger reallocation
    c := append(a, 10, 20, 30) // exceeds capacity → new array
    c[0] = 0

    fmt.Println("a:", a) // a: [99 2 3]  ← NOT affected
    fmt.Println("c:", c) // c: [0 2 3 10 20 30]

    // FIX: use full slice expression to limit capacity
    d := a[:2:2] // len=2, cap=2 — forces new array on append
    e := append(d, 100)
    e[0] = -1

    fmt.Println("a:", a) // a: [99 2 3]  ← NOT affected
    fmt.Println("e:", e) // e: [-1 2 100]
}
```

---

## 2. Map Iteration Order

```go
package main

import "fmt"

func main() {
    m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4}

    // Order is RANDOMIZED — different every run!
    for k, v := range m {
        fmt.Println(k, v)
    }
    // Could print: b 2, d 4, a 1, c 3  — or any order

    // If you need sorted keys:
    // import "slices"
    // keys := slices.Sorted(maps.Keys(m))
}
```

---

## 3. Goroutine Loop Variable Capture

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // Pre-Go 1.22: classic bug — all goroutines see LAST value of v
    values := []int{1, 2, 3, 4, 5}
    var wg sync.WaitGroup

    // BUG (pre-Go 1.22):
    // for _, v := range values {
    //     wg.Add(1)
    //     go func() {
    //         defer wg.Done()
    //         fmt.Println(v) // all print 5!
    //     }()
    // }

    // FIX 1: pass as argument
    for _, v := range values {
        wg.Add(1)
        go func(val int) {
            defer wg.Done()
            fmt.Println(val) // correct
        }(v)
    }
    wg.Wait()

    // FIX 2: shadow variable
    for _, v := range values {
        v := v // shadow
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(v) // correct
        }()
    }
    wg.Wait()

    // Go 1.22+: FIXED — each iteration gets its own variable
    // No workaround needed anymore
}
```

---

## 4. Defer with Closures

```go
package main

import "fmt"

func main() {
    // Deferred CLOSURE sees variable's FINAL value
    x := 0
    defer func() {
        fmt.Println("Closure x:", x) // prints 3 (final value)
    }()
    x = 1
    x = 2
    x = 3

    // Deferred METHOD CALL evaluates args at DEFER site
    y := 10
    defer fmt.Println("Direct y:", y) // prints 10 (captured at defer)
    y = 20
    y = 30
}

// Output:
// Direct y: 10     ← evaluated when defer was called
// Closure x: 3     ← evaluated when function returns
```

---

## 5. Nil Interface vs Nil Pointer in Interface

```go
package main

import "fmt"

type MyError struct {
    Msg string
}

func (e *MyError) Error() string {
    return e.Msg
}

func getError(fail bool) error {
    var err *MyError // nil pointer
    if fail {
        err = &MyError{Msg: "something failed"}
    }
    return err // GOTCHA: returns non-nil interface holding nil pointer!
}

func main() {
    err := getError(false)

    // err is NOT nil! The interface has type *MyError with nil value
    if err != nil {
        fmt.Println("Error:", err) // "Error: <nil>" — confusing!
    }

    // FIX: return nil explicitly
    // func getError(fail bool) error {
    //     if fail {
    //         return &MyError{Msg: "something failed"}
    //     }
    //     return nil  // ← explicit nil interface
    // }

    // Why? An interface is (type, value). A nil pointer inside an
    // interface makes (type=*MyError, value=nil) which is != nil.
    // Only (type=nil, value=nil) is a nil interface.
}
```

---

## 6. String Immutability

```go
package main

import "fmt"

func main() {
    s := "hello"
    // s[0] = 'H' // COMPILE ERROR: strings are immutable

    // Convert to []byte for mutation
    b := []byte(s)
    b[0] = 'H'
    fmt.Println(string(b)) // Hello

    // Convert to []rune for Unicode mutation
    r := []rune("café")
    r[3] = 'E'
    fmt.Println(string(r)) // cafE
}
```

---

## 7. String Concatenation in Loops — O(n²)

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // BAD — O(n²): each += allocates a new string
    s := ""
    for i := 0; i < 10000; i++ {
        s += "x" // new allocation every iteration!
    }

    // GOOD — O(n): use strings.Builder
    var sb strings.Builder
    for i := 0; i < 10000; i++ {
        sb.WriteString("x")
    }
    result := sb.String()

    fmt.Println(len(s), len(result)) // 10000 10000
}
```

---

## 8. Channel Deadlocks

```go
package main

import "fmt"

func main() {
    // DEADLOCK: unbuffered channel with no receiver
    // ch := make(chan int)
    // ch <- 42  // blocks forever — no goroutine to receive
    // fmt.Println(<-ch)

    // FIX: use goroutine or buffered channel
    ch := make(chan int, 1) // buffered — won't block on send
    ch <- 42
    fmt.Println(<-ch) // 42

    // Or use a goroutine
    ch2 := make(chan int)
    go func() { ch2 <- 42 }()
    fmt.Println(<-ch2) // 42
}
```

---

## 9. Value vs Pointer Receiver Method Sets

```go
package main

import "fmt"

type Counter struct {
    n int
}

func (c Counter) Value() int { return c.n }   // value receiver
func (c *Counter) Increment() { c.n++ }        // pointer receiver

type Incrementer interface {
    Increment()
}

func main() {
    c := Counter{n: 0}

    // Direct calls work fine
    c.Increment() // Go auto-takes address: (&c).Increment()
    fmt.Println(c.Value()) // 1

    // But for INTERFACES:
    // var inc Incrementer = c   // COMPILE ERROR!
    //   Counter does not implement Incrementer
    //   (Increment method has pointer receiver)

    var inc Incrementer = &c // OK — *Counter has both value AND pointer methods
    inc.Increment()
    fmt.Println(c.Value()) // 2
}
```

**Rule:** A value type's method set includes only value-receiver methods. A pointer type includes both.

---

## 10. init() Execution Order

```go
package main

import "fmt"

// Multiple init() functions per package are allowed
func init() { fmt.Println("init 1") }
func init() { fmt.Println("init 2") }
func init() { fmt.Println("init 3") }

func main() {
    fmt.Println("main")
}

// Output:
// init 1
// init 2
// init 3
// main

// Order: imports (depth-first, alphabetical) → package-level vars → init() → main()
```

---

## 11. Named Return + Defer

```go
package main

import "fmt"

func compute() (result int) {
    defer func() {
        result *= 2 // modifies the NAMED return value!
    }()
    return 10 // sets result = 10, then defer runs
}

func main() {
    fmt.Println(compute()) // 20 (not 10!)
}
```

---

## 12. Map Value Not Addressable

```go
package main

import "fmt"

type Point struct {
    X, Y int
}

func main() {
    m := map[string]Point{
        "origin": {0, 0},
    }

    // m["origin"].X = 5  // COMPILE ERROR: cannot assign to struct field in map

    // FIX 1: reassign the whole value
    p := m["origin"]
    p.X = 5
    m["origin"] = p

    // FIX 2: use map of pointers
    m2 := map[string]*Point{
        "origin": {0, 0},
    }
    m2["origin"].X = 5 // OK — pointer is addressable

    fmt.Println(m["origin"])  // {5 0}
    fmt.Println(m2["origin"]) // &{5 0}
}
```

---

## 13. range Copies Values

```go
package main

import "fmt"

type Item struct {
    Name  string
    Price int
}

func main() {
    items := []Item{
        {"Apple", 100},
        {"Banana", 50},
    }

    // BUG: v is a COPY — modifying it doesn't affect the slice
    for _, v := range items {
        v.Price *= 2 // modifies copy only
    }
    fmt.Println(items) // [{Apple 100} {Banana 50}] — unchanged!

    // FIX: use index
    for i := range items {
        items[i].Price *= 2
    }
    fmt.Println(items) // [{Apple 200} {Banana 100}]
}
```

---

## 14. select Randomness

```go
package main

import "fmt"

func main() {
    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)

    ch1 <- "one"
    ch2 <- "two"

    // When BOTH cases are ready, Go picks one RANDOMLY
    select {
    case msg := <-ch1:
        fmt.Println("From ch1:", msg)
    case msg := <-ch2:
        fmt.Println("From ch2:", msg)
    }
    // Could print either "From ch1: one" or "From ch2: two"
}
```

---

## 15. Struct Comparison

```go
package main

import "fmt"

type Comparable struct {
    Name string
    Age  int
}

type NotComparable struct {
    Name  string
    Tags  []string // slices are NOT comparable
}

func main() {
    a := Comparable{"Alice", 30}
    b := Comparable{"Alice", 30}
    fmt.Println(a == b) // true — all fields are comparable

    // c := NotComparable{"Alice", []string{"go"}}
    // d := NotComparable{"Alice", []string{"go"}}
    // fmt.Println(c == d) // COMPILE ERROR: struct containing []string cannot be compared
}
```

---

## 16. Zero-Size Struct — struct{}

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    // struct{} uses ZERO memory
    fmt.Println(unsafe.Sizeof(struct{}{})) // 0

    // Use case 1: Set (map with no value overhead)
    set := map[string]struct{}{
        "apple":  {},
        "banana": {},
    }
    if _, ok := set["apple"]; ok {
        fmt.Println("apple is in set")
    }

    // Use case 2: Signal channel (no data, just notification)
    done := make(chan struct{})
    go func() {
        fmt.Println("Working...")
        close(done) // signal completion
    }()
    <-done // wait for signal
}
```

---

## Interview Questions

1. **What happens when you range over a slice and modify it?**
   - The range expression is evaluated once. Appending during iteration may not process new elements. Modifying existing elements via index works. Using the loop variable's address captures the same variable (pre-Go 1.22).

2. **What is the nil interface gotcha?**
   - An interface holding a nil pointer is NOT equal to nil. `var p *MyType = nil; var i interface{} = p; i == nil` is false. The interface has a type but nil value. Always return `nil` explicitly for interface returns.

3. **What happens when you close a nil channel?**
   - It panics. Always check if a channel is non-nil before closing. Closing an already-closed channel also panics. Only the sender should close a channel.

4. **What is the loop variable capture gotcha?**
   - Pre-Go 1.22: goroutines in a loop capture the same variable. Fix with `v := v` or pass as parameter. Go 1.22+ creates a new variable per iteration (`GOEXPERIMENT=loopvar`, default in 1.22).

5. **What happens with slice append and capacity?**
   - If `len < cap`, append modifies the underlying array (affecting other slices sharing it). If `len == cap`, append allocates a new array. Use full slice expression `s[:n:n]` to prevent aliasing.

6. **Can you compare slices and maps with `==`?**
   - No. Slices and maps can only be compared to `nil`. Use `slices.Equal()` (Go 1.21+) or `reflect.DeepEqual()` for comparison. Structs containing slices/maps also cannot use `==`.

7. **What is the shadowing gotcha with `:=`?**
   - `:=` in an inner scope creates a new variable shadowing the outer one. The outer variable remains unchanged. Common with `err` in if-else chains. Use explicit declaration to avoid.

8. **What happens when you send to a closed channel?**
   - It panics. Receiving from a closed channel returns the zero value immediately. Use `val, ok := <-ch` to detect closed channels. Design so only producers close channels.

9. **Why does `defer` in a loop not execute until function return?**
   - `defer` is function-scoped, not block-scoped. In a loop, all deferred calls stack up and execute at function return (LIFO). Wrap in a closure or use explicit cleanup for per-iteration resources.

10. **What is the map concurrent access gotcha?**
    - Maps are not safe for concurrent use. Concurrent read+write causes a fatal runtime panic (not a data race—a hard crash). Use `sync.Mutex`, `sync.RWMutex`, or `sync.Map` for concurrent access.
