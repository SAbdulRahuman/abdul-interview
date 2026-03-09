# Chapter 29 — Quick Reference: Keywords & Built-in Functions

Go has exactly **25 keywords** — intentionally minimal compared to Java (67) or C++ (95+). This simplicity is a core design principle.

```
┌──────────────────────────────────────────────────────────┐
│              Go's 25 Keywords                           │
│                                                          │
│  Declarations:   const  func  import  package           │
│                  type   var                              │
│                                                          │
│  Composite:      struct  interface  map                 │
│                                                          │
│  Control flow:   if  else  for  switch  case  default   │
│                  break  continue  goto  fallthrough      │
│                  return                                  │
│                                                          │
│  Concurrency:    go  chan  select                        │
│                                                          │
│  Other:          defer  range                            │
│                                                          │
│  Built-in Functions (predeclared, not keywords):        │
│  ┌──────────────────────────────────────────────────┐    │
│  │ make(T, ...)     → create slice/map/chan         │   │
│  │ new(T)           → allocate zeroed *T            │   │
│  │ len(x)           → length of string/slice/map/chan│  │
│  │ cap(x)           → capacity of slice/chan         │   │
│  │ append(s, ...)   → append to slice               │   │
│  │ copy(dst, src)   → copy slice elements           │   │
│  │ delete(m, key)   → delete map entry              │   │
│  │ close(ch)        → close channel                 │   │
│  │ panic(v)         → trigger runtime panic         │   │
│  │ recover()        → catch panic in deferred func  │   │
│  │ print/println    → low-level output (use fmt)    │   │
│  │ complex/real/imag→ complex number operations     │   │
│  │ clear(x)         → clear map/slice (Go 1.21+)   │   │
│  │ min(a,b)/max(a,b)→ built-in min/max (Go 1.21+)  │   │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

## 25 Go Keywords

### Flow Control

```go
package main

import "fmt"

func main() {
    // break — exit loop or switch
    for i := 0; i < 10; i++ {
        if i == 5 {
            break
        }
        fmt.Print(i, " ") // 0 1 2 3 4
    }
    fmt.Println()

    // continue — skip to next iteration
    for i := 0; i < 10; i++ {
        if i%2 == 0 {
            continue
        }
        fmt.Print(i, " ") // 1 3 5 7 9
    }
    fmt.Println()

    // goto — jump to a label (rarely used)
    i := 0
loop:
    if i < 3 {
        fmt.Print(i, " ") // 0 1 2
        i++
        goto loop
    }
    fmt.Println()

    // return — exit function (with optional values)
    result := add(3, 4)
    fmt.Println(result) // 7

    // fallthrough — fall into next case in switch
    switch 1 {
    case 1:
        fmt.Println("case 1")
        fallthrough
    case 2:
        fmt.Println("case 2 (via fallthrough)")
    case 3:
        fmt.Println("case 3 (not reached)")
    }
}

func add(a, b int) int {
    return a + b
}
```

### Conditionals

```go
package main

import "fmt"

func main() {
    x := 42

    // if / else
    if x > 50 {
        fmt.Println("big")
    } else if x > 20 {
        fmt.Println("medium") // ← this one
    } else {
        fmt.Println("small")
    }

    // switch / case / default
    switch x {
    case 1, 2, 3:
        fmt.Println("tiny")
    case 42:
        fmt.Println("the answer") // ← this one
    default:
        fmt.Println("something else")
    }
}
```

### Loops

```go
package main

import "fmt"

func main() {
    // for — the only loop keyword in Go
    for i := 0; i < 5; i++ {
        fmt.Print(i, " ")
    }
    fmt.Println()

    // range — iterate over slices, maps, strings, channels
    fruits := []string{"apple", "banana", "cherry"}
    for i, fruit := range fruits {
        fmt.Printf("%d: %s\n", i, fruit)
    }

    // Range over map
    m := map[string]int{"a": 1, "b": 2}
    for key, value := range m {
        fmt.Printf("%s=%d\n", key, value)
    }

    // Range over integer (Go 1.22+)
    for i := range 5 {
        fmt.Print(i, " ") // 0 1 2 3 4
    }
    fmt.Println()
}
```

### Functions

```go
package main

import "fmt"

// func — declare a function
func greet(name string) string {
    return "Hello, " + name + "!"
}

// Multiple return values
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

func main() {
    fmt.Println(greet("Go"))
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
    }
    fmt.Println(result)
}
```

### Declarations

```go
package main

import "fmt"

// var — variable declaration
var globalVar = "I'm global"

// const — constant declaration
const Pi = 3.14159

// type — define a new type
type Celsius float64
type Fahrenheit float64

func (c Celsius) ToF() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func main() {
    var x int = 10
    y := 20 // short declaration (inside functions only)
    fmt.Println(x, y, globalVar, Pi)

    temp := Celsius(100)
    fmt.Printf("%.1f°C = %.1f°F\n", temp, temp.ToF())
}
```

### Organization

```go
// package — every Go file starts with a package declaration
package main

// import — bring in other packages
import (
    "fmt"
    "strings"
)

func main() {
    fmt.Println(strings.ToUpper("hello"))
}
```

### Types

```go
package main

import "fmt"

// struct — composite type with named fields
type Person struct {
    Name string
    Age  int
}

// interface — set of method signatures
type Greeter interface {
    Greet() string
}

func (p Person) Greet() string {
    return fmt.Sprintf("Hi, I'm %s!", p.Name)
}

func main() {
    p := Person{Name: "Alice", Age: 30}
    var g Greeter = p // Person satisfies Greeter
    fmt.Println(g.Greet())
}
```

### Data Structure

```go
package main

import "fmt"

func main() {
    // map — hash table (key-value pairs)
    scores := map[string]int{
        "Alice": 95,
        "Bob":   87,
    }
    scores["Charlie"] = 92
    fmt.Println(scores)
}
```

### Concurrency

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // chan — typed communication channel
    ch := make(chan string)

    // go — launch a goroutine
    go func() {
        ch <- "hello from goroutine"
    }()

    // select — multiplex channel operations
    select {
    case msg := <-ch:
        fmt.Println(msg)
    case <-time.After(1 * time.Second):
        fmt.Println("timeout")
    }
}
```

### Deferred Execution

```go
package main

import "fmt"

func main() {
    // defer — schedule function call to run when enclosing function returns
    // LIFO order (Last In, First Out)
    defer fmt.Println("third")
    defer fmt.Println("second")
    defer fmt.Println("first")
    fmt.Println("start")
}
// Output: start, first, second, third
```

---

## Built-in Functions

```go
package main

import "fmt"

func main() {
    // make() — create slice, map, or channel
    s := make([]int, 5, 10)    // len=5, cap=10
    m := make(map[string]int)  // empty map
    ch := make(chan int, 3)     // buffered channel

    // new() — allocate zeroed memory, return pointer
    p := new(int)    // *int, points to 0
    *p = 42

    // len() — length
    fmt.Println(len(s))        // 5
    fmt.Println(len(m))        // 0
    fmt.Println(len("hello"))  // 5

    // cap() — capacity
    fmt.Println(cap(s))  // 10
    fmt.Println(cap(ch)) // 3

    // append() — add elements to slice
    s = append(s, 1, 2, 3)
    fmt.Println(s)

    // copy() — copy slice elements
    dst := make([]int, 3)
    copied := copy(dst, s)
    fmt.Println(dst, "copied:", copied)

    // delete() — remove map entry
    m["key"] = 100
    delete(m, "key")
    fmt.Println(m) // map[]

    // close() — close a channel
    close(ch)

    // panic() / recover() — emergency error handling
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    // panic("something went wrong")

    // complex(), real(), imag() — complex numbers
    c := complex(3, 4) // 3+4i
    fmt.Println(real(c), imag(c)) // 3 4

    // clear() (Go 1.21+) — clear map or zero-out slice
    s2 := []int{1, 2, 3}
    clear(s2)
    fmt.Println(s2) // [0 0 0]

    m2 := map[string]int{"a": 1, "b": 2}
    clear(m2)
    fmt.Println(m2) // map[]

    // min(), max() (Go 1.21+)
    fmt.Println(min(1, 2, 3))    // 1
    fmt.Println(max(1, 2, 3))    // 3
    fmt.Println(min("a", "b"))   // a

    _ = p
}
```

---

## Predeclared Types

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    // Boolean
    var b bool = true

    // Integers
    var i int = 42          // platform-dependent (32 or 64 bit)
    var i8 int8 = 127       // -128 to 127
    var i16 int16 = 32767
    var i32 int32 = 2147483647
    var i64 int64 = 9223372036854775807
    var u uint = 42         // unsigned
    var u8 uint8 = 255
    var u16 uint16 = 65535
    var u32 uint32 = 4294967295
    var u64 uint64 = 18446744073709551615
    var uptr uintptr = 0   // holds a pointer value as integer

    // Float
    var f32 float32 = 3.14
    var f64 float64 = 3.141592653589793

    // Complex
    var c64 complex64 = 1 + 2i
    var c128 complex128 = 1 + 2i

    // String
    var s string = "Hello, 世界"

    // Aliases
    var by byte = 'A'            // alias for uint8
    var r rune = '世'             // alias for int32
    var a any = "anything"        // alias for interface{} (Go 1.18+)
    // comparable — constraint for generics (== and != support)

    // Error
    var err error = fmt.Errorf("an error")

    // Print sizes
    fmt.Printf("bool: %d bytes\n", unsafe.Sizeof(b))
    fmt.Printf("int: %d bytes\n", unsafe.Sizeof(i))
    fmt.Printf("int8: %d bytes\n", unsafe.Sizeof(i8))
    fmt.Printf("int64: %d bytes\n", unsafe.Sizeof(i64))
    fmt.Printf("float32: %d bytes\n", unsafe.Sizeof(f32))
    fmt.Printf("float64: %d bytes\n", unsafe.Sizeof(f64))
    fmt.Printf("complex128: %d bytes\n", unsafe.Sizeof(c128))
    fmt.Printf("string: %d bytes (header)\n", unsafe.Sizeof(s))

    _, _, _, _, _, _, _, _, _, _, _, _ = i16, i32, u, u8, u16, u32, u64, uptr, c64, by, r, a
    _ = err
}
```

---

## Go Version Quick Reference

| Version | Key Features |
|---------|-------------|
| Go 1.11 | Modules (experimental) |
| Go 1.13 | Error wrapping (`%w`, `errors.Is`, `errors.As`) |
| Go 1.14 | Testing cleanup (`t.Cleanup`) |
| Go 1.16 | `embed`, `io/fs`, `os.ReadFile` / `os.WriteFile` |
| Go 1.17 | Build constraints `//go:build`, `unsafe.Slice` |
| Go 1.18 | **Generics**, Fuzzing, Workspaces |
| Go 1.19 | `GOMEMLIMIT`, atomic types |
| Go 1.20 | `errors.Join`, `context.Cause` |
| Go 1.21 | `log/slog`, `slices`/`maps`/`cmp`, `min`/`max`, `clear` |
| Go 1.22 | Range over integers, loop variable fix, enhanced `ServeMux` |
| Go 1.23 | Iterator functions (`iter` package), `range over func` |

---

## Interview Questions

1. **How many keywords does Go have? Name them.**
   - 25 keywords: `break`, `case`, `chan`, `const`, `continue`, `default`, `defer`, `else`, `fallthrough`, `for`, `func`, `go`, `goto`, `if`, `import`, `interface`, `map`, `package`, `range`, `return`, `select`, `struct`, `switch`, `type`, `var`.

2. **What is the difference between `make` and `new`?**
   - `new(T)` allocates zeroed memory and returns `*T`. `make(T, args)` initializes slices, maps, and channels (returns T, not pointer). `make` is required for these types to be usable.

3. **What does `fallthrough` do in a switch statement?**
   - Forces execution to fall through to the next case body (unconditionally). Go switch cases don't fall through by default (unlike C). `fallthrough` must be the last statement in a case.

4. **What are the built-in functions in Go?**
   - `append`, `cap`, `close`, `complex`, `copy`, `delete`, `imag`, `len`, `make`, `new`, `panic`, `print`, `println`, `real`, `recover`. Plus Go 1.21+: `min`, `max`, `clear`.

5. **What is the `select` keyword used for?**
   - Waits on multiple channel operations simultaneously. Picks a random ready case. `default` makes it non-blocking. Essential for multiplexing channels and implementing timeouts.

6. **What does `defer` do and in what order do deferred calls execute?**
   - Schedules a function call to execute when the surrounding function returns. Multiple defers execute in LIFO (stack) order. Arguments are evaluated at the defer statement, not at execution.

7. **What is the `go` keyword?**
   - Launches a goroutine: `go myFunc()`. The function runs concurrently. No return value can be captured directly—use channels or shared state with synchronization.

8. **What does `chan` keyword do and what are directional channels?**
   - Declares a channel type. `chan T` is bidirectional. `chan<- T` is send-only. `<-chan T` is receive-only. Directional channels enforce communication patterns at compile time.

9. **What is `iota` and how does it work?**
   - A predeclared identifier for auto-incrementing constants. Starts at 0, increments per `const` spec. Reset to 0 at each `const` block. Used for enums: `const (A = iota; B; C)`.

10. **What are the predeclared types in Go?**
    - `bool`, `byte`, `complex64/128`, `error`, `float32/64`, `int`/`int8/16/32/64`, `rune`, `string`, `uint`/`uint8/16/32/64`, `uintptr`, `any` (=`interface{}`), `comparable`.
