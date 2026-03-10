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

**Tutorial: Loop and Branch Control Keywords**

Go provides `break`, `continue`, `goto`, `return`, and `fallthrough` to control execution flow. `break` exits a loop or switch, `continue` skips to the next iteration, and `goto` jumps to a labeled statement (rarely used). Notice that `fallthrough` in a `switch` is unconditional — it always enters the next case body, which differs from C where fall-through is the default.

```
┌─────────────────────────────────────────────┐
│        Flow Control Keywords                │
│                                             │
│  for i := 0; i < 10; i++                   │
│    ├── if i==5 ──► break    ──► exit loop   │
│    └── if i%2==0 ► continue ──► next iter   │
│                                             │
│  switch val {                               │
│    case 1: ──► fallthrough ──┐              │
│    case 2: ◄─────────────────┘ (runs too)   │
│    case 3: (not reached)                    │
│  }                                          │
│                                             │
│  goto label ──► jumps to label:             │
│  return val ──► exits function              │
└─────────────────────────────────────────────┘
```

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

**Tutorial: Branching with if/else and switch**

Go's `if/else` chain works like other languages but requires braces (no single-line bodies). The `switch` statement is more powerful than C's: cases don't fall through by default, each case can list multiple values, and you can switch on expressions or even use a tagless switch for boolean conditions. The `default` case handles unmatched values.

```
┌────────────────────────────────────────┐
│            if / else chain             │
│                                        │
│  x = 42                                │
│    │                                   │
│    ▼                                   │
│  x > 50? ──yes──► "big"                │
│    │no                                 │
│    ▼                                   │
│  x > 20? ──yes──► "medium" ◄── chosen  │
│    │no                                 │
│    ▼                                   │
│  else ──────────► "small"              │
│                                        │
│         switch x {                     │
│  ┌──────┬──────┬──────────┐            │
│  │1,2,3 │  42  │ default  │            │
│  │tiny  │answer│something │            │
│  └──────┴──┬───┴──────────┘            │
│            ▼                           │
│         matched                        │
└────────────────────────────────────────┘
```

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

**Tutorial: Go's Single Loop Keyword — for**

Go has only one loop keyword: `for`. It covers all loop patterns — classic three-clause loops, while-style loops (`for condition {}`), and infinite loops (`for {}`). The `range` keyword iterates over slices (index + value), maps (key + value), strings (index + rune), channels (values), and integers (Go 1.22+). Note that map iteration order is intentionally randomized.

```
┌──────────────────────────────────────────────┐
│  for — Go's only loop construct              │
│                                              │
│  Classic:   for i:=0; i<5; i++ { ... }       │
│  While:     for condition { ... }            │
│  Infinite:  for { ... }                      │
│                                              │
│  range over slice:                           │
│  ┌───┬───┬───┐                               │
│  │ a │ b │ c │  ──► i=0,v="a"  i=1,v="b" ... │
│  └───┴───┴───┘                               │
│                                              │
│  range over map:                             │
│  { "a":1, "b":2 } ──► k="a",v=1  k="b",v=2  │
│                        (random order!)       │
│                                              │
│  range over int (Go 1.22+):                  │
│  range 5 ──► 0, 1, 2, 3, 4                  │
└──────────────────────────────────────────────┘
```

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

**Tutorial: Function Declaration and Multiple Returns**

The `func` keyword declares functions. Go functions can return multiple values, which is the idiomatic way to handle errors — the last return value is typically an `error`. Parameters of the same type can share a type annotation (`a, b float64`). Functions are first-class values and can be assigned to variables, passed as arguments, or returned from other functions.

```
┌───────────────────────────────────────────────┐
│  func declaration                             │
│                                               │
│  func greet(name string) string               │
│       │        │              │                │
│       │        └─ parameter   └─ return type   │
│       └─ function name                        │
│                                               │
│  Multiple returns (idiomatic error handling):  │
│  func divide(a, b float64) (float64, error)   │
│       │                       │        │      │
│       │                       │        └─ err │
│       │                       └─ result       │
│       └─ shared param type                    │
│                                               │
│  Caller:                                      │
│  result, err := divide(10, 3)                 │
│  if err != nil { handle... }                  │
└───────────────────────────────────────────────┘
```

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

**Tutorial: Variables, Constants, and Custom Types**

Go uses `var` for variable declarations (with optional type inference), `:=` for short declarations inside functions, and `const` for compile-time constants. The `type` keyword defines new named types, enabling method sets — here `Celsius` is a distinct type from `float64` with its own `ToF()` method. This provides type safety: you can't accidentally mix Celsius and Fahrenheit values.

```
┌──────────────────────────────────────────────────┐
│  Declaration Keywords                            │
│                                                  │
│  var globalVar = "I'm global"   ◄── package-level│
│  const Pi = 3.14159             ◄── compile-time │
│                                                  │
│  Inside func:                                    │
│    var x int = 10               ◄── explicit type│
│    y := 20                      ◄── short decl   │
│                                                  │
│  type Celsius float64           ◄── new type     │
│    │                                             │
│    └─► func (c Celsius) ToF() Fahrenheit         │
│              │                                   │
│              └─ method on Celsius                │
│                                                  │
│  Celsius(100).ToF() ──► 212.0°F                  │
└──────────────────────────────────────────────────┘
```

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

**Tutorial: Package and Import System**

Every Go file begins with a `package` declaration, which groups related files. The `import` keyword brings in external packages — grouped imports use parentheses. Package `main` with `func main()` defines an executable entry point. All other packages are libraries. Unused imports cause compilation errors, enforcing clean dependency trees.

```
┌──────────────────────────────────────────────┐
│  Go File Structure                           │
│                                              │
│  ┌────────────────────────────┐              │
│  │ package main               │ ◄── 1st line│
│  │                            │              │
│  │ import (                   │ ◄── deps    │
│  │     "fmt"                  │              │
│  │     "strings"              │              │
│  │ )                          │              │
│  │                            │              │
│  │ func main() { ... }        │ ◄── entry   │
│  └────────────────────────────┘              │
│                                              │
│  package main ──► executable                 │
│  package foo  ──► library (importable)       │
└──────────────────────────────────────────────┘
```

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

**Tutorial: Structs and Interfaces**

The `struct` keyword defines composite types with named fields — Go's primary way to model data. The `interface` keyword defines behavior contracts as sets of method signatures. Interfaces are satisfied implicitly: any type that implements all the methods automatically satisfies the interface, with no `implements` keyword needed. This enables loose coupling and powerful polymorphism.

```
┌───────────────────────────────────────────────┐
│  struct + interface (implicit satisfaction)    │
│                                               │
│  ┌──────────────┐   ┌───────────────────────┐ │
│  │ struct Person │   │ interface Greeter     │ │
│  │  Name string  │   │  Greet() string       │ │
│  │  Age  int     │   └───────────┬───────────┘ │
│  └──────┬───────┘               │             │
│         │                       │ satisfies   │
│         └─► func (p Person)     │ implicitly  │
│               Greet() string ───┘             │
│                                               │
│  var g Greeter = Person{...}  ◄── works!      │
│  g.Greet() ──► "Hi, I'm Alice!"              │
└───────────────────────────────────────────────┘
```

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

**Tutorial: Maps — Built-in Hash Tables**

The `map` keyword creates hash tables with typed keys and values. Maps must be initialized with `make` or a literal before use (a nil map panics on write). Keys must be comparable types (no slices or maps as keys). Map lookups return a zero value for missing keys, so use the comma-ok idiom (`val, ok := m[key]`) to distinguish missing entries from zero values.

```
┌─────────────────────────────────────────────┐
│  map[string]int — hash table                │
│                                             │
│  scores := map[string]int{                  │
│    "Alice":   95,                           │
│    "Bob":     87,                           │
│  }                                          │
│                                             │
│  Bucket layout (conceptual):                │
│  ┌────────┬───────┐                         │
│  │  Key   │ Value │                         │
│  ├────────┼───────┤                         │
│  │"Alice" │  95   │                         │
│  │"Bob"   │  87   │                         │
│  │"Charlie│  92   │ ◄── added later         │
│  └────────┴───────┘                         │
│                                             │
│  scores["Charlie"] = 92  ──► insert/update  │
│  delete(scores, "Bob")   ──► remove entry   │
└─────────────────────────────────────────────┘
```

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

**Tutorial: Goroutines, Channels, and Select**

The `go` keyword launches a goroutine — a lightweight concurrent function (initial stack ~2KB). The `chan` keyword creates typed channels for safe communication between goroutines. The `select` keyword multiplexes on multiple channel operations, blocking until one is ready. Here a goroutine sends a message through a channel, and `select` either receives it or times out after 1 second.

```
┌──────────────────────────────────────────────────┐
│  Goroutine + Channel + Select                    │
│                                                  │
│  main goroutine              spawned goroutine   │
│  ┌──────────────┐            ┌────────────────┐  │
│  │ ch := make() │            │                │  │
│  │ go func()    │──launch──►│ ch <- "hello"   │  │
│  │              │            └───────┬────────┘  │
│  │ select {     │                    │           │
│  │   case <-ch ◄├────────────────────┘           │
│  │   case <-timeout                              │
│  │ }            │                                │
│  └──────────────┘                                │
│                                                  │
│  select picks whichever case is ready first      │
│  (random if multiple ready simultaneously)       │
└──────────────────────────────────────────────────┘
```

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

**Tutorial: defer — LIFO Cleanup Scheduling**

The `defer` keyword schedules a function call to execute when the enclosing function returns, regardless of how it returns (normal return, panic, etc.). Multiple defers stack in LIFO (Last In, First Out) order. Arguments to deferred calls are evaluated immediately at the `defer` statement, not when the deferred function runs. `defer` is the standard pattern for cleanup: closing files, unlocking mutexes, releasing resources.

```
┌──────────────────────────────────────────┐
│  defer — LIFO execution order            │
│                                          │
│  func main() {                           │
│    defer Println("third")  ──► push ─┐   │
│    defer Println("second") ──► push ─┤   │
│    defer Println("first")  ──► push ─┤   │
│    Println("start")                  │   │
│  } ◄── function returns              │   │
│                                      │   │
│  Defer Stack (LIFO):      Output:    │   │
│  ┌──────────┐             start      │   │
│  │ "first"  │ ◄── pop 1   first    ◄─┤   │
│  │ "second" │ ◄── pop 2   second   ◄─┤   │
│  │ "third"  │ ◄── pop 3   third    ◄─┘   │
│  └──────────┘                            │
└──────────────────────────────────────────┘
```

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

**Tutorial: Predeclared Built-in Functions**

Go's built-in functions are predeclared identifiers, not keywords — they can be shadowed (though you shouldn't). `make` creates slices/maps/channels with initial state, while `new` allocates zeroed memory and returns a pointer. `append` grows slices, `copy` transfers elements, `delete` removes map entries, and `close` shuts a channel. `panic`/`recover` provide emergency error handling. Go 1.21+ added `clear`, `min`, and `max`.

```
┌──────────────────────────────────────────────────┐
│  Built-in Functions — Data Flow                  │
│                                                  │
│  Creation:                                       │
│  make([]int, 5, 10) ──► [0 0 0 0 0] len=5 cap=10│
│  make(map[K]V)      ──► {} empty map             │
│  make(chan int, 3)   ──► buffered channel cap=3   │
│  new(int)            ──► *int → 0                │
│                                                  │
│  Mutation:                                       │
│  append(s, 1,2) ──► s grows                      │
│  copy(dst, src) ──► elements copied              │
│  delete(m, key) ──► entry removed                │
│  close(ch)      ──► channel closed               │
│                                                  │
│  Inspection:                                     │
│  len(x) ──► current length                       │
│  cap(x) ──► current capacity                     │
│                                                  │
│  Error:                                          │
│  panic(v)   ──► unwind stack ──► recover()       │
└──────────────────────────────────────────────────┘
```

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

**Tutorial: Go's Type System and Memory Sizes**

Go has a fixed set of predeclared types: `bool`, signed/unsigned integers of various sizes, `float32/64`, `complex64/128`, `string`, `byte` (alias for `uint8`), `rune` (alias for `int32` for Unicode code points), `any` (alias for `interface{}`), and `error`. The `int` and `uint` types are platform-dependent (32 or 64 bits). Use `unsafe.Sizeof` to inspect actual memory sizes at runtime.

```
┌─────────────────────────────────────────────────┐
│  Predeclared Types — Memory Layout              │
│                                                 │
│  bool     ──► 1 byte                            │
│  int8     ──► 1 byte   [-128, 127]              │
│  int16    ──► 2 bytes  [-32768, 32767]          │
│  int32    ──► 4 bytes  (alias: rune)            │
│  int64    ──► 8 bytes                           │
│  int      ──► 4 or 8 bytes (platform)           │
│                                                 │
│  uint8    ──► 1 byte   [0, 255] (alias: byte)   │
│  uint16   ──► 2 bytes                           │
│  uint32   ──► 4 bytes                           │
│  uint64   ──► 8 bytes                           │
│                                                 │
│  float32  ──► 4 bytes                           │
│  float64  ──► 8 bytes                           │
│                                                 │
│  string   ──► 16 bytes header (ptr + len)       │
│  any      ──► interface{} (Go 1.18+)            │
│  error    ──► interface { Error() string }      │
└─────────────────────────────────────────────────┘
```

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
