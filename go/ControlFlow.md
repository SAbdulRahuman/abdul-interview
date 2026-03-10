# Chapter 3 — Control Flow

Go's control flow is minimal by design: one loop keyword (`for`), no parentheses around conditions, no ternary operator, and `switch` doesn't fall through by default. This simplicity makes Go code easy to read and reason about.

```
┌──────────────────────────────────────────────────────────┐
│              Go Control Flow Constructs                  │
│                                                          │
│  Branching:    if / else if / else                       │
│  Looping:      for (the ONLY loop keyword)               │
│  Selection:    switch (expression, tagless, type)        │
│  Concurrency:  select (channel multiplexing)             │
│  Jump:         goto, break, continue (with labels)       │
│  Deferral:     defer (LIFO cleanup on function exit)     │
│                                                          │
│  Notable MISSING constructs:                             │
│  • No while, do-while  → use for                         │
│  • No ternary (?:)     → use if/else                     │
│  • No try/catch        → use error returns               │
└──────────────────────────────────────────────────────────┘
```

## if / else if / else

**Tutorial: Basic Conditional Branching**

Go's `if` statement requires no parentheses around the condition — the braces `{}` are always mandatory. The conditions are evaluated top-to-bottom; the first truthy branch executes and the rest are skipped. Since `age` is `25`, it fails `< 13`, fails `< 20`, passes `< 65`, so `"Adult"` prints. There is no ternary operator in Go — use `if/else` instead.

```
┌──────────────────────────────────────────────────────────┐
│        if/else if — Evaluation Flow (age=25)             │
│                                                          │
│  age < 13?  ──► NO (25 ≥ 13)                            │
│       │                                                  │
│  age < 20?  ──► NO (25 ≥ 20)                            │
│       │                                                  │
│  age < 65?  ──► YES ──► print "Adult" ──► DONE          │
│       │                                                  │
│  (else never reached)                                    │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    age := 25

    if age < 13 {
        fmt.Println("Child")
    } else if age < 20 {
        fmt.Println("Teenager")
    } else if age < 65 {
        fmt.Println("Adult")
    } else {
        fmt.Println("Senior")
    }
    // Output: Adult
}
```

---

## if with Init Statement

**Tutorial: Scoped Initialization in if**

Go allows a short statement before the `if` condition, separated by `;`. The variable declared in the init statement is scoped to the entire `if/else` block — it doesn't leak into the surrounding function. This is heavily used with error checks (`if err := ...; err != nil`) and map lookups (`if val, ok := m[key]; ok`). The comma-ok pattern is idiomatic Go for distinguishing "key not found" from "key has zero value."

```
┌──────────────────────────────────────────────────────────┐
│     if-init Scope Visualization                          │
│                                                          │
│  func main() {                                           │
│  │                                                       │
│  │  if err := os.Setenv(...); err != nil {               │
│  │  │  ┌─────────────────────────────────┐               │
│  │  │  │ err is accessible here          │               │
│  │  │  └─────────────────────────────────┘               │
│  │  } else {                                             │
│  │  │  ┌─────────────────────────────────┐               │
│  │  │  │ err is still accessible here    │               │
│  │  │  └─────────────────────────────────┘               │
│  │  }                                                    │
│  │                                                       │
│  │  // err is NOT accessible here ← out of scope         │
│  }                                                       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Variable 'err' is scoped to this if/else block
    if err := os.Setenv("APP_MODE", "production"); err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Environment variable set successfully")
    }

    // err is NOT accessible here
    // fmt.Println(err) // ERROR: undefined: err

    // Common pattern with map lookup
    m := map[string]int{"alice": 90, "bob": 85}

    if score, ok := m["alice"]; ok {
        fmt.Println("Alice's score:", score)
    } else {
        fmt.Println("Alice not found")
    }
    // Output: Alice's score: 90
}
```

---

## for Loop — All Forms

Go has only one looping construct: `for`. It covers all cases that other languages handle with `for`, `while`, `do-while`, and `foreach`.

```
┌──────────────────────────────────────────────────────────┐
│                 for Loop — All 5 Forms                   │
│                                                          │
│  1. Classic:     for i := 0; i < n; i++ { }              │
│                  ┌─init─►condition─►body─►post──┐        │
│                  │         ▲                     │        │
│                  │         └─────────────────────┘        │
│                                                          │
│  2. While:       for condition { }                       │
│                                                          │
│  3. Infinite:    for { }  (must break/return)            │
│                                                          │
│  4. Range:       for i, v := range collection { }        │
│                  Works on: slice, array, map, string,    │
│                            channel, integer (Go 1.22+)   │
│                                                          │
│  5. Integer:     for i := range 5 { }  (Go 1.22+)       │
│                  Iterates 0, 1, 2, 3, 4                  │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: All 5 Forms of the for Loop**

This example demonstrates every looping pattern Go supports — all using the single `for` keyword. Form 1 (classic) is for counted iteration. Form 2 (while-style) loops while a condition holds. Form 3 (infinite) runs forever until `break` or `return`. Form 4 (range) iterates collections — yielding both index and value. Form 5 (skip index/value) uses `_` to discard the unwanted variable.

```go
        fmt.Print(count, " ")
        count++
    }
    fmt.Println() // 0 1 2

    // 3. Infinite loop
    sum := 0
    for {
        sum++
        if sum >= 5 {
            break // must break out manually
        }
    }
    fmt.Println("Sum:", sum) // Sum: 5

    // 4. Range-based — over slices
    fruits := []string{"apple", "banana", "cherry"}
    for index, value := range fruits {
        fmt.Printf("  %d: %s\n", index, value)
    }
    // 0: apple, 1: banana, 2: cherry

    // 5. Range — skip index or value
    for _, fruit := range fruits {
        fmt.Print(fruit, " ") // apple banana cherry
    }
    fmt.Println()

    for i := range fruits {
        fmt.Print(i, " ") // 0 1 2
    }
    fmt.Println()
}
```

---

## for range Over Various Types

The `range` keyword iterates over different data structures. What it yields depends on the type:

```
┌────────────────────────────────────────────────────────┐
│            range Yields by Type                        │
│                                                        │
│  Type       │ 1st value      │ 2nd value               │
│  ───────────┼────────────────┼─────────────────        │
│  []T, [N]T  │ index (int)    │ value (T)               │
│  map[K]V    │ key (K)        │ value (V)               │
│  string     │ byte index     │ rune (int32)            │
│  chan T      │ value (T)      │ (none)                  │
│  int (1.22+)│ value (0..n-1) │ (none)                  │
│                                                        │
│  ⚠ Map iteration order is RANDOMIZED                  │
│  ⚠ String ranges over RUNES, byte index may skip      │
│  ⚠ Channel range blocks until channel is closed        │
└────────────────────────────────────────────────────────┘
```

**Tutorial: range Over Different Data Structures**

The `range` keyword adapts its behavior based on the collection type. For slices and arrays, it yields (index, value). For maps, it yields (key, value) in random order — this randomization is intentional to prevent code from depending on iteration order. For strings, it iterates over **runes** (Unicode code points), not bytes — the byte index may skip for multi-byte characters. For channels, it blocks until the channel is closed.

```go

    // Range over map
    capitals := map[string]string{
        "France": "Paris",
        "Japan":  "Tokyo",
        "India":  "Delhi",
    }
    for country, capital := range capitals {
        fmt.Printf("%s -> %s\n", country, capital)
    }
    // Order is randomized!

    // Range over string (iterates runes, not bytes)
    for i, r := range "Go世界" {
        fmt.Printf("byte index %d: %c (U+%04X)\n", i, r, r)
    }
    // byte index 0: G (U+0047)
    // byte index 1: o (U+006F)
    // byte index 2: 世 (U+4E16)
    // byte index 5: 界 (U+754C)

    // Range over channel
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch) // must close, or range will block forever

    for v := range ch {
        fmt.Print(v, " ") // 1 2 3
    }
    fmt.Println()

    // Range over integers (Go 1.22+)
    for i := range 5 {
        fmt.Print(i, " ") // 0 1 2 3 4
    }
    fmt.Println()
}
```

---

## switch — Expression, Tagless, fallthrough

Go's `switch` is more powerful than in C/Java: cases don't fall through by default, cases can be expressions (not just constants), and the tagless form replaces long `if-else` chains.

```
┌──────────────────────────────────────────────────────────┐
│              switch vs C/Java switch                     │
│                                                          │
│  Go switch:                 C/Java switch:               │
│  ┌────────────────┐         ┌────────────────┐           │
│  │ case "a":      │         │ case "a":      │           │
│  │   doA()        │         │   doA();       │           │
│  │   // auto-break│         │   break; ← need│           │
│  │ case "b":      │         │ case "b":      │           │
│  │   doB()        │         │   doB();       │           │
│  │   fallthrough←explicit   │   // falls thru│← implicit │
│  │ case "c":      │         │ case "c":      │           │
│  │   doC()        │         │   doC();       │           │
│  └────────────────┘         └────────────────┘           │
│                                                          │
│  Tagless switch (no expression, each case is bool cond): │
│  switch {                                                │
│      case x > 0:  "positive"                             │
│      case x < 0:  "negative"                             │
│      default:     "zero"                                 │
│  }                                                       │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: Switch Statement — Three Variants**

Go's switch has three forms. The expression switch matches a value against cases. The tagless switch (no expression after `switch`) evaluates each case as a boolean — replacing long `if-else` chains. Switch with `fallthrough` explicitly continues into the next case body regardless of that case's condition — this is the opposite of C where fallthrough is the default.

```go
    case "Monday":
        fmt.Println("Start of work week")
    case "Tuesday", "Wednesday", "Thursday": // multiple values in one case
        fmt.Println("Midweek")
    case "Friday":
        fmt.Println("TGIF!")
    default:
        fmt.Println("Weekend!")
    }
    // Output: Midweek

    // 2. Tagless switch (like if-else chain, more readable)
    score := 85
    switch {
    case score >= 90:
        fmt.Println("Grade: A")
    case score >= 80:
        fmt.Println("Grade: B")
    case score >= 70:
        fmt.Println("Grade: C")
    default:
        fmt.Println("Grade: F")
    }
    // Output: Grade: B

    // 3. Switch with init statement
    switch lang := "Go"; lang {
    case "Go":
        fmt.Println("Golang!")
    case "Rust":
        fmt.Println("Rustacean!")
    }

    // 4. fallthrough — explicitly falls to next case (rarely used)
    switch num := 5; {
    case num < 10:
        fmt.Println("Less than 10")
        fallthrough // continues to next case regardless of condition
    case num < 20:
        fmt.Println("Less than 20")
        fallthrough
    case num < 30:
        fmt.Println("Less than 30")
    }
    // Output:
    // Less than 10
    // Less than 20
    // Less than 30
}
```

---

## Type Switch

**Tutorial: Dynamic Type Inspection**

A type switch inspects the concrete type stored inside an interface. The syntax `v := i.(type)` extracts the value with its real type. Each case branch has `v` already typed correctly — inside `case int:`, `v` is an `int` and you can do `v*2` without any casting. This is critical for handling generic data (e.g., JSON parsing, plugin systems, event handlers).

```
┌──────────────────────────────────────────────────────────┐
│       Type Switch — Runtime Type Extraction              │
│                                                          │
│  interface{} value = 42                                  │
│  ┌──────────────────────────────────┐                    │
│  │ type info: *int                  │                    │
│  │ data:      42                    │                    │
│  └──────────────────────────────────┘                    │
│                                                          │
│  switch v := i.(type) {                                  │
│    case int:    → v is int, v=42        ← MATCHES        │
│    case string: → v is string           ← skipped        │
│    case bool:   → v is bool             ← skipped        │
│    default:     → v is interface{}      ← skipped        │
│  }                                                       │
│                                                          │
│  Inside the matching case, v has the concrete type!      │
│  v*2 = 84 (int multiplication, not interface)            │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d (doubled: %d)\n", v, v*2)
    case string:
        fmt.Printf("String: %q (length: %d)\n", v, len(v))
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    case []int:
        fmt.Printf("Int slice with %d elements\n", len(v))
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

func main() {
    describe(42)          // Integer: 42 (doubled: 84)
    describe("hello")     // String: "hello" (length: 5)
    describe(true)        // Boolean: true
    describe([]int{1, 2}) // Int slice with 2 elements
    describe(3.14)        // Unknown type: float64
}
```

---

## select Statement (Channel Multiplexing)

`select` is like `switch` but for channel operations. It waits until one of its channel cases can proceed — if multiple are ready, one is chosen **at random**. This is fundamental for Go's concurrency model.

```
┌──────────────────────────────────────────────────────────┐
│                 select Behavior                          │
│                                                          │
│  select {                                                │
│      case msg := <-ch1:   ← waits for data from ch1     │
│      case ch2 <- value:   ← waits to send to ch2        │
│      case <-time.After(d):← timeout after duration d    │
│      default:             ← non-blocking (optional)     │
│  }                                                       │
│                                                          │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐               │
│  │  ch1    │   │  ch2    │   │ timeout │               │
│  │ waiting │   │ waiting │   │ waiting │               │
│  └────┬────┘   └────┬────┘   └────┬────┘               │
│       └──────┬──────┘             │                     │
│              ▼                    │                     │
│        select picks               │                     │
│        whichever is               │                     │
│        ready first ◄──────────────┘                     │
│                                                          │
│  With default: returns immediately if no channel ready   │
│  Without default: blocks until a channel is ready        │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: select for Channel Multiplexing**

This example creates two channels and two goroutines that send messages at different delays. The `select` statement waits until one of its channel operations can proceed — whichever channel receives data first gets handled. The `time.After` pattern provides timeouts. Adding a `default` case makes `select` non-blocking — it immediately returns if no channel is ready.

```go

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "message from channel 2"
    }()

    // select waits on multiple channel operations
    // whichever is ready first gets executed
    for i := 0; i < 2; i++ {
        select {
        case msg := <-ch1:
            fmt.Println(msg)
        case msg := <-ch2:
            fmt.Println(msg)
        }
    }

    // Timeout with select
    ch3 := make(chan string)
    select {
    case msg := <-ch3:
        fmt.Println(msg)
    case <-time.After(50 * time.Millisecond):
        fmt.Println("Timeout! No message received")
    }

    // Non-blocking select with default
    ch4 := make(chan int)
    select {
    case v := <-ch4:
        fmt.Println("Received:", v)
    default:
        fmt.Println("No value ready, moving on")
    }
}
```

---

## goto and Labels

**Tutorial: goto — The Rarely Used Jump**

`goto` unconditionally jumps to a labeled statement within the same function. Go restricts `goto` to prevent jumping over variable declarations or into different scopes. In modern Go, `defer` replaces most use cases where `goto` was historically used for cleanup. This example simulates a while loop using `goto` — purely for illustration.

```
┌──────────────────────────────────────────────────────────┐
│        goto Control Flow                                 │
│                                                          │
│  i := 0                                                  │
│  loop: ◄─────────┐   ← label (target of goto)          │
│    if i < 5       │                                      │
│      print i      │                                      │
│      i++          │                                      │
│      goto loop ───┘   ← jumps back to label             │
│    (end if)                                              │
│  Println()   ← reached when i >= 5                       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // goto jumps to a label (rarely used, avoid in most cases)
    i := 0
loop:
    if i < 5 {
        fmt.Print(i, " ")
        i++
        goto loop
    }
    fmt.Println() // 0 1 2 3 4

    // More practical: error handling cleanup (before defer was common)
    // Prefer defer over goto in modern Go
}
```

---

## break / continue with Labels

**Tutorial: Escaping Nested Loops**

Labels are essential when you need to break out of or continue and outer loop from within an inner loop. Without a label, `break` only exits the innermost loop. `break outer` exits both loops entirely. `continue outerLoop` skips the rest of the inner loop AND the rest of the current outer iteration, jumping directly to the next iteration of the outer loop.

```
┌──────────────────────────────────────────────────────────┐
│     break outer vs continue outerLoop                    │
│                                                          │
│  break outer (at i=1, j=1):                              │
│  i=0: j=0 ✓  j=1 ✓  j=2 ✓                              │
│  i=1: j=0 ✓  j=1 ✗ BREAK → exits BOTH loops            │
│  i=2: (never reached)                                    │
│                                                          │
│  continue outerLoop (at j=1):                            │
│  i=0: j=0 ✓  j=1 → SKIP rest, go to i=1                │
│  i=1: j=0 ✓  j=1 → SKIP rest, go to i=2                │
│  i=2: j=0 ✓  j=1 → SKIP rest, loop ends                │
└──────────────────────────────────────────────────────────┘
```

Labels are useful for **exiting nested loops**.

```go
package main

import "fmt"

func main() {
    // break with label — exits the outer loop
    fmt.Println("break with label:")
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break outer // breaks out of BOTH loops
            }
            fmt.Printf("  i=%d j=%d\n", i, j)
        }
    }
    // i=0 j=0, i=0 j=1, i=0 j=2, i=1 j=0

    // continue with label — skips to next iteration of outer loop
    fmt.Println("\ncontinue with label:")
outerLoop:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if j == 1 {
                continue outerLoop // skips rest of inner loop, goes to next i
            }
            fmt.Printf("  i=%d j=%d\n", i, j)
        }
    }
    // i=0 j=0, i=1 j=0, i=2 j=0
}
```

---

## defer — LIFO Execution

## defer — LIFO Execution

`defer` schedules a function call to run **after** the surrounding function returns. Multiple defers execute in LIFO (Last In, First Out) order — like a stack. Arguments are evaluated at the defer site, not when the deferred call runs.

```
┌──────────────────────────────────────────────────────────┐
│              defer Execution (LIFO Stack)                │
│                                                          │
│  func example() {                                        │
│      defer fmt.Println("A")    ← pushed 1st             │
│      defer fmt.Println("B")    ← pushed 2nd             │
│      defer fmt.Println("C")    ← pushed 3rd             │
│      fmt.Println("body")                                 │
│  }                                                       │
│                                                          │
│  Execution:                     Defer Stack:             │
│  1. Print "body"                ┌───┐                    │
│  2. func returns                │ C │ ← top (last in)   │
│  3. Pop & run C                 │ B │                    │
│  4. Pop & run B                 │ A │ ← bottom (first in)│
│  5. Pop & run A                 └───┘                    │
│                                                          │
│  Output: body → C → B → A                               │
│                                                          │
│  ⚠ Arguments evaluated at defer site:                   │
│    x := 10                                               │
│    defer fmt.Println(x)  ← captures 10, not later value │
│    x = 20                                                │
│    // prints 10                                          │
└──────────────────────────────────────────────────────────┘
```

**Tutorial: LIFO Defer Order**

This example shows that three `defer` calls stack up in LIFO order. `"Start"` and `"End"` print normally during execution. When `main()` returns, the defer stack unwinds: the last deferred call (`"Deferred 3"`) runs first, then `"Deferred 2"`, then `"Deferred 1"`.

```go

    // Output:
    // Start
    // End
    // Deferred 3   (LIFO order)
    // Deferred 2
    // Deferred 1
}
```

### Arguments evaluated immediately at defer site

**Tutorial: Defer Argument Capture**

This is a critical subtlety: `defer fmt.Println("Deferred x =", x)` evaluates `x` immediately when the `defer` statement is reached — capturing the value `10`. Even though `x` is later changed to `20`, the deferred call still prints `10`. This is because function arguments are evaluated eagerly, not lazily. To capture the final value, use a closure: `defer func() { fmt.Println(x) }()`.

```
┌──────────────────────────────────────────────────────────┐
│        defer Argument Evaluation Timeline                │
│                                                          │
│  Time ──────────────────────────────────────────►        │
│                                                          │
│  x := 10                                                 │
│  ──► defer Println(x)  ← x=10 captured NOW              │
│       ┌──────────┐                                       │
│       │ saved: 10│  (stored on defer stack)               │
│       └──────────┘                                       │
│  x = 20                                                  │
│  ──► Println("Current x =", 20)  ← prints 20            │
│  ──► function returns                                    │
│  ──► deferred call runs: Println("Deferred x =", 10)    │
│       └── uses the SAVED value 10, not current 20        │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    x := 10
    defer fmt.Println("Deferred x =", x) // x is captured as 10 NOW
    x = 20
    fmt.Println("Current x =", x)

    // Output:
    // Current x = 20
    // Deferred x = 10   (NOT 20! Arguments evaluated at defer site)
}
```

### Common cleanup pattern

**Tutorial: defer for Resource Cleanup**

The `defer file.Close()` pattern is the most common use of `defer` in Go. By placing `defer` immediately after a successful `os.Open()`, you guarantee the file handle is released when the function returns — regardless of whether it returns normally, early via `return`, or even during a panic. This prevents resource leaks.

```
┌──────────────────────────────────────────────────────────┐
│       Resource Cleanup Pattern                           │
│                                                          │
│  file, err := os.Open(filename)                          │
│  if err != nil { return }     ← early return, no leak   │
│  defer file.Close()           ← registered immediately  │
│                                                          │
│  // ... use file ...                                     │
│  // ... maybe return early ...                           │
│  // ... maybe panic ...                                  │
│                                                          │
│  } ← function exits here                                 │
│    └─► defer fires: file.Close() ← ALWAYS runs          │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "os"
)

func readFile(filename string) {
    file, err := os.Open(filename)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer file.Close() // Guaranteed to run when function exits

    // Read and process file...
    buf := make([]byte, 100)
    n, _ := file.Read(buf)
    fmt.Println("Read", n, "bytes")
}
```

---

## Interview Questions

1. **Does Go have a `while` loop?**
   - No. Go only has `for`. A while-style loop is written as `for condition { }`. An infinite loop is `for { }`.

2. **What is a type switch in Go?**
   - A type switch checks the dynamic type of an interface value: `switch v := x.(type) { case int: ... case string: ... }`. It's used to handle different concrete types stored in an interface.

3. **How does `switch` in Go differ from C/Java?**
   - Go's `switch` does not fall through by default (no `break` needed). Each case body breaks automatically. Use `fallthrough` to explicitly fall into the next case. Cases can be expressions, not just constants.

4. **What is the `select` statement used for?**
   - `select` multiplexes channel operations. It blocks until one of its cases can proceed. If multiple cases are ready, one is chosen randomly. A `default` case makes it non-blocking.

5. **Explain `defer` in Go. When are deferred calls executed?**
   - `defer` schedules a function call to execute when the surrounding function returns. Deferred calls are executed in LIFO (Last In, First Out) order. Arguments are evaluated at the `defer` site, not at execution time.

6. **Can `if` statements in Go have an init statement?**
   - Yes: `if err := doSomething(); err != nil { ... }`. The variable `err` is scoped to the `if`/`else` block only.

7. **What happens if you use `goto` in Go?**
   - `goto` jumps to a label in the same function. It cannot jump over variable declarations or into a different scope (block). It's rarely used in practice.

8. **How do labeled `break` and `continue` work?**
   - Labels let you break/continue outer loops from inner loops: `outer: for { for { break outer } }`. Without labels, `break`/`continue` only affect the innermost loop.

9. **What is a tagless switch in Go?**
   - A switch without an expression: `switch { case x > 0: ... case x < 0: ... }`. Each case is a boolean expression. It's a cleaner alternative to `if-else` chains.

10. **Can you `range` over an integer in Go?**
    - Yes, since Go 1.22: `for i := range 5 { }` iterates i from 0 to 4. Before 1.22, only slices, maps, strings, channels, and arrays were supported.
