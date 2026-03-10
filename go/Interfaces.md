# Chapter 9 — Interfaces

Interfaces in Go are **implicit** — a type satisfies an interface simply by implementing all its methods. There is no `implements` keyword. This enables powerful decoupling: the implementing type doesn't even need to know the interface exists.

```
┌──────────────────────────────────────────────────────────┐
│              Interface Satisfaction                      │
│                                                          │
│  type Shape interface {                                  │
│      Area() float64                                      │
│      Perimeter() float64                                 │
│  }                                                       │
│                                                          │
│  type Rectangle struct { W, H float64 }                  │
│  func (r Rectangle) Area() float64 { ... }       ✓      │
│  func (r Rectangle) Perimeter() float64 { ... }  ✓      │
│  → Rectangle satisfies Shape (all methods matched)       │
│                                                          │
│  type Circle struct { R float64 }                        │
│  func (c Circle) Area() float64 { ... }          ✓      │
│  → Circle does NOT satisfy Shape (missing Perimeter)     │
│                                                          │
│  No "implements" keyword needed!                         │
│  Checked at compile time when used as interface value.   │
└──────────────────────────────────────────────────────────┘
```

## Implicit Interface Satisfaction

**Tutorial: Defining and Satisfying Interfaces Implicitly**

This example defines a `Shape` interface with two methods and two concrete types — `Rectangle` and `Circle` — that satisfy it by implementing both. There is no `implements` keyword; the compiler checks satisfaction when a value is assigned to an interface variable or passed to a function accepting the interface. The `printShape` function accepts any `Shape`, demonstrating polymorphism.

```
┌──────────────────────────────────────────────────────────┐
│         Implicit Satisfaction Check                      │
│                                                          │
│  Shape interface           Rectangle struct              │
│  ┌──────────────────┐      ┌──────────────────┐          │
│  │ Area() float64   │◄─────│ Area() ✓         │          │
│  │ Perimeter() f64  │◄─────│ Perimeter() ✓    │          │
│  └──────────────────┘      └──────────────────┘          │
│          ▲                                               │
│          │                 Circle struct                  │
│          │                 ┌──────────────────┐           │
│          └─────────────────│ Area() ✓         │           │
│                    ◄───────│ Perimeter() ✓    │           │
│                            └──────────────────┘           │
│                                                           │
│  printShape(s Shape) ──► dispatches to concrete methods   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct {
    Width, Height float64
}

// Rectangle implicitly satisfies Shape — no declaration needed!
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * 3.14159 * c.Radius
}

// Works with ANY type that satisfies Shape
func printShape(s Shape) {
    fmt.Printf("Area: %.2f, Perimeter: %.2f\n", s.Area(), s.Perimeter())
}

func main() {
    printShape(Rectangle{Width: 5, Height: 3})  // Area: 15.00, Perimeter: 16.00
    printShape(Circle{Radius: 4})                // Area: 50.27, Perimeter: 25.13
}
```

---

## Empty Interface: interface{} / any

**Tutorial: The Universal Container — any (interface{})**

The empty interface `any` (alias for `interface{}` since Go 1.18) has zero methods, so every type satisfies it automatically. This example shows passing various types to a single function, building heterogeneous slices, and creating maps with mixed value types. Use `any` sparingly — you lose compile-time type safety and must use type assertions to recover the concrete value.

```
┌──────────────────────────────────────────────────────────┐
│         any — Zero-Method Interface                      │
│                                                          │
│  any (interface{})                                       │
│  ┌──────────────┐     int ─────────► satisfies ✓        │
│  │ (no methods) │     string ──────► satisfies ✓        │
│  └──────────────┘     bool ────────► satisfies ✓        │
│                       []int ───────► satisfies ✓        │
│                                                          │
│  []any{ 1, "two", 3.0, true }                           │
│  ┌─────┬───────┬───────┬──────┐                          │
│  │ int │string │float64│ bool │  heterogeneous slice     │
│  └─────┴───────┴───────┴──────┘                          │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// any is an alias for interface{} (Go 1.18+)
// It can hold a value of ANY type

func printAnything(val any) {
    fmt.Printf("Value: %v, Type: %T\n", val, val)
}

func main() {
    printAnything(42)          // Value: 42, Type: int
    printAnything("hello")     // Value: hello, Type: string
    printAnything(true)        // Value: true, Type: bool
    printAnything([]int{1, 2}) // Value: [1 2], Type: []int

    // Slice of any type
    items := []any{1, "two", 3.0, true}
    for _, item := range items {
        fmt.Println(item)
    }

    // Map with any values
    config := map[string]any{
        "port":  8080,
        "debug": true,
        "name":  "myapp",
    }
    fmt.Println(config)
}
```

---

## Interface Composition

**Tutorial: Building Larger Interfaces from Small Ones**

Go encourages composing larger interfaces by embedding smaller ones. Here, `ReadWriter` embeds `Reader` and `Writer`, while `ReadWriteCloser` embeds all three. The `File` struct satisfies `ReadWriteCloser` by implementing `Read`, `Write`, and `Close` — and automatically satisfies every smaller composed interface too. This mirrors how the standard library builds `io.ReadWriter` from `io.Reader` and `io.Writer`.

```
┌──────────────────────────────────────────────────────────┐
│        Interface Composition Hierarchy                   │
│                                                          │
│  Reader        Writer        Closer                      │
│  ┌──────┐     ┌──────┐     ┌───────┐                     │
│  │Read()│     │Write()│    │Close()│                     │
│  └──┬───┘     └──┬───┘     └──┬────┘                     │
│     │            │            │                          │
│     ▼            ▼            │                          │
│  ReadWriter                   │                          │
│  ┌──────────────┐             │                          │
│  │ Reader       │             │                          │
│  │ Writer       │             │                          │
│  └──────┬───────┘             │                          │
│         │                     │                          │
│         ▼                     ▼                          │
│  ReadWriteCloser                                         │
│  ┌──────────────────────────────┐                        │
│  │ Reader + Writer + Closer     │                        │
│  └──────────────────────────────┘                        │
│         ▲                                                │
│         │  File satisfies ALL of them                    │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Small, focused interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Composed interface — embedding interfaces inside interfaces
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Concrete implementation
type File struct {
    name string
}

func (f *File) Read(p []byte) (int, error) {
    fmt.Println("Reading from", f.name)
    return len(p), nil
}

func (f *File) Write(p []byte) (int, error) {
    fmt.Println("Writing to", f.name)
    return len(p), nil
}

func (f *File) Close() error {
    fmt.Println("Closing", f.name)
    return nil
}

func processReadWriter(rw ReadWriter) {
    rw.Read(nil)
    rw.Write(nil)
}

func main() {
    f := &File{name: "data.txt"}

    // File satisfies all three composed interfaces
    var rwc ReadWriteCloser = f
    rwc.Read(nil)
    rwc.Write(nil)
    rwc.Close()

    // File also satisfies the smaller interface
    processReadWriter(f)
}
```

---

## Type Assertion

**Tutorial: Extracting Concrete Types from Interface Values**

A type assertion `i.(ConcreteType)` extracts the underlying concrete value from an interface. The comma-ok form `val, ok := i.(Dog)` is safe — it returns `false` if the type doesn't match. Without comma-ok, a failed assertion panics at runtime. This example shows both safe and unsafe patterns, plus asserting from `any`.

```
┌──────────────────────────────────────────────────────────┐
│          Type Assertion Flow                             │
│                                                          │
│  var a Animal = Dog{Name: "Rex"}                         │
│                                                          │
│  a.(Dog)  ──► dog = Dog{Name:"Rex"}, ok = true          │
│  a.(Cat)  ──► cat = Cat{},           ok = false          │
│                                                          │
│  Interface value:                                        │
│  ┌────────────────┐                                      │
│  │ type: Dog      │──► matches Dog? ✓ → extract value   │
│  │ value: {Rex}   │──► matches Cat? ✗ → ok = false      │
│  └────────────────┘                                      │
│                                                          │
│  ⚠ Without comma-ok: a.(Cat) → PANIC!                  │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Animal interface {
    Sound() string
}

type Dog struct{ Name string }
type Cat struct{ Name string }

func (d Dog) Sound() string { return "Woof!" }
func (c Cat) Sound() string { return "Meow!" }

func main() {
    var a Animal = Dog{Name: "Rex"}

    // Type assertion — extract concrete type from interface
    dog, ok := a.(Dog)
    if ok {
        fmt.Println("It's a dog:", dog.Name) // It's a dog: Rex
    }

    // Failed assertion with comma-ok (safe)
    cat, ok := a.(Cat)
    if !ok {
        fmt.Println("Not a cat!") // Not a cat!
        _ = cat
    }

    // Type assertion WITHOUT comma-ok — PANICS on failure!
    // cat = a.(Cat)  // panic: interface conversion: main.Animal is main.Dog, not main.Cat

    // Asserting to interface type
    var val any = "hello"
    if s, ok := val.(string); ok {
        fmt.Println("String value:", s) // String value: hello
    }
}
```

---

## Type Switch

**Tutorial: Branching on Dynamic Type with a Type Switch**

A type switch `switch v := val.(type)` lets you branch on the concrete type stored in an interface. Each `case` binds `v` to the correctly-typed value, giving you full type-safe access without manual assertions. This is the idiomatic way to handle `any` values when multiple types are expected, and `nil` can be matched as its own case.

```
┌──────────────────────────────────────────────────────────┐
│         Type Switch Decision Tree                        │
│                                                          │
│  val.(type)                                              │
│      │                                                   │
│      ├── int ────────► "int: 42"                         │
│      ├── float64 ────► "float64: 3.14"                   │
│      ├── string ─────► "string: \"hello\" (len=5)"       │
│      ├── bool ───────► "bool: true"                      │
│      ├── []int ──────► "[]int with 2 elements"           │
│      ├── nil ────────► "nil value"                       │
│      └── default ────► "unknown type: ..."               │
│                                                          │
│  Each case binds v to the concrete typed value           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func classifyValue(val any) string {
    switch v := val.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case float64:
        return fmt.Sprintf("float64: %.2f", v)
    case string:
        return fmt.Sprintf("string: %q (len=%d)", v, len(v))
    case bool:
        return fmt.Sprintf("bool: %t", v)
    case []int:
        return fmt.Sprintf("[]int with %d elements", len(v))
    case nil:
        return "nil value"
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

func main() {
    fmt.Println(classifyValue(42))          // int: 42
    fmt.Println(classifyValue(3.14))        // float64: 3.14
    fmt.Println(classifyValue("hello"))     // string: "hello" (len=5)
    fmt.Println(classifyValue(true))        // bool: true
    fmt.Println(classifyValue([]int{1, 2})) // []int with 2 elements
    fmt.Println(classifyValue(nil))         // nil value
}
```

---

## Interface Internals — (type, value) Pair

An interface value is internally stored as **two pointers**: a type descriptor (or itab) and a data pointer. This is why a nil pointer stored in an interface is NOT a nil interface — the type field is populated.

```
┌──────────────────────────────────────────────────────────┐
│           Interface Internal Representation             │
│                                                          │
│  var s Stringer = Person{Name: "Alice"}                  │
│                                                          │
│  Interface value (iface):                                │
│  ┌──────────────────┐                                    │
│  │ type: *itab      │──► Person type descriptor          │
│  │                  │    (method table, type info)       │
│  │ value: *data     │──► Person{Name: "Alice"}          │
│  └──────────────────┘    (actual data, heap-allocated)  │
│                                                          │
│  Nil interface:           Interface holding nil:         │
│  ┌──────────────────┐     ┌──────────────────┐           │
│  │ type: nil        │     │ type: *MyError   │← NOT nil! │
│  │ value: nil       │     │ value: nil       │           │
│  └──────────────────┘     └──────────────────┘           │
│  s == nil → true          s == nil → false ← GOTCHA!    │
│                                                          │
│  ⚠ An interface is nil ONLY when BOTH type AND value   │
│    are nil. This is the #1 Go interface gotcha.         │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Stringer interface {
    String() string
}

type Person struct {
    Name string
}

func (p Person) String() string {
    return "Person: " + p.Name
}

func main() {
    var s Stringer // nil interface: (type=nil, value=nil)
    fmt.Println(s == nil) // true

    s = Person{Name: "Alice"} // (type=Person, value=Person{Name:"Alice"})
    fmt.Println(s == nil)     // false

    // Interface stores the concrete type and value
    fmt.Printf("Type: %T, Value: %v\n", s, s)
    // Type: main.Person, Value: Person: Alice
}
```

---

## Nil Interface vs Interface Holding Nil Pointer (Critical Gotcha!)

**Tutorial: The Nil Interface Trap — Type vs Value**

This is Go's most common interface pitfall. An interface is nil only when **both** its type and value fields are nil. If you assign a typed nil pointer to an interface, the interface stores the type information and becomes non-nil — even though the underlying value is nil. The `riskyFunction` demonstrates the bug; `safeFunction` shows the fix: return `nil` directly rather than a typed nil pointer.

```
┌──────────────────────────────────────────────────────────┐
│         The Nil Interface Trap                           │
│                                                          │
│  riskyFunction(false):                                   │
│  var err *MyError = nil                                  │
│  return err  ──►  error interface:                       │
│                   ┌─────────────────┐                    │
│                   │ type: *MyError  │ ← NOT nil!         │
│                   │ value: nil      │                    │
│                   └─────────────────┘                    │
│                   err == nil → false  ✗ UNEXPECTED       │
│                                                          │
│  safeFunction(false):                                    │
│  return nil  ──►  error interface:                       │
│                   ┌─────────────────┐                    │
│                   │ type: nil       │                    │
│                   │ value: nil      │                    │
│                   └─────────────────┘                    │
│                   err == nil → true   ✓ EXPECTED         │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type MyError struct {
    Code int
}

func (e *MyError) Error() string {
    return fmt.Sprintf("error code: %d", e.Code)
}

// BAD: returns a non-nil interface even when the error is nil!
func riskyFunction(fail bool) error {
    var err *MyError // nil pointer
    if fail {
        err = &MyError{Code: 404}
    }
    return err // DANGER: returns interface{type=*MyError, value=nil}
               // This is NOT a nil interface!
}

// GOOD: return nil explicitly
func safeFunction(fail bool) error {
    if fail {
        return &MyError{Code: 404}
    }
    return nil // nil interface: {type=nil, value=nil}
}

func main() {
    err1 := riskyFunction(false)
    fmt.Println(err1 == nil) // false!! Even though MyError pointer is nil,
                              // the interface value is NOT nil (it has a type)

    err2 := safeFunction(false)
    fmt.Println(err2 == nil) // true — correctly nil

    // This is one of Go's biggest gotchas!
    // Rule: NEVER assign a typed nil pointer to an interface
    // Always return the interface's nil value directly
}
```

---

## Common Standard Library Interfaces

**Tutorial: Implementing Key Standard Library Interfaces**

Go's standard library defines many small interfaces that types can implement for seamless integration. This example shows `fmt.Stringer` (controls how `fmt.Println` displays your type), `error` (custom error messages), `io.Reader`/`io.Writer` (stream-based I/O), and `sort.Interface` (custom sort ordering). Implementing these interfaces unlocks automatic behavior across the entire standard library.

```
┌──────────────────────────────────────────────────────────┐
│       Common Standard Library Interfaces                 │
│                                                          │
│  fmt.Stringer       error            io.Reader           │
│  ┌───────────┐     ┌───────────┐    ┌──────────────┐    │
│  │ String()  │     │ Error()   │    │ Read([]byte) │    │
│  │  string   │     │  string   │    │ (int, error) │    │
│  └─────┬─────┘     └─────┬─────┘    └──────┬───────┘    │
│        │                 │                 │            │
│  fmt.Println()     if err != nil    io.Copy, bufio,     │
│  auto-calls        checks this      all accept Reader   │
│  String()                                                │
│                                                          │
│  sort.Interface                                          │
│  ┌────────────────────────────┐                           │
│  │ Len() int                 │                           │
│  │ Less(i, j int) bool       │                           │
│  │ Swap(i, j int)            │                           │
│  └────────────────────────────┘                           │
│  sort.Sort() works with any implementation               │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "sort"
    "strings"
)

// fmt.Stringer — custom string representation
type User struct {
    Name string
    Age  int
}

func (u User) String() string {
    return fmt.Sprintf("%s (age %d)", u.Name, u.Age)
}

// error — custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// sort.Interface — custom sorting
type ByAge []User

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
    // fmt.Stringer
    u := User{Name: "Alice", Age: 30}
    fmt.Println(u) // Alice (age 30) — calls String() automatically

    // error interface
    err := ValidationError{Field: "email", Message: "invalid format"}
    fmt.Println(err) // validation failed on email: invalid format

    // io.Reader / io.Writer
    reader := strings.NewReader("Hello, io.Reader!")
    buf := make([]byte, 5)
    n, _ := reader.Read(buf)
    fmt.Println(string(buf[:n])) // Hello

    // io.Copy — copies from Reader to Writer
    r := strings.NewReader("Copy this text")
    var w strings.Builder
    io.Copy(&w, r)
    fmt.Println(w.String()) // Copy this text

    // sort.Interface
    users := []User{
        {Name: "Charlie", Age: 25},
        {Name: "Alice", Age: 30},
        {Name: "Bob", Age: 20},
    }
    sort.Sort(ByAge(users))
    for _, u := range users {
        fmt.Println(u)
    }
    // Bob (age 20)
    // Charlie (age 25)
    // Alice (age 30)

    // json.Marshaler — custom JSON output
    data, _ := json.Marshal(u)
    fmt.Println(string(data))
}
```

---

## Best Practices

**Tutorial: Interface Design Rules — Accept Interfaces, Return Structs**

This example illustrates two key Go idioms. First, "accept interfaces, return concrete types" — functions take interface parameters for flexibility and testability but return concrete types so callers get full access without type assertions. Second, keep interfaces small (1-2 methods) following Go convention; large interfaces are hard to implement, hard to mock, and create tight coupling.

```
┌──────────────────────────────────────────────────────────┐
│        Interface Design Rules                            │
│                                                          │
│  ✅ Accept interface        ✅ Return concrete           │
│  ┌────────────────────┐     ┌────────────────────┐       │
│  │ func Process(       │     │ func New()          │       │
│  │   l Logger,         │     │   ConsoleLogger     │       │
│  │   id string,        │     │   { return ... }    │       │
│  │ )                   │     └────────────────────┘       │
│  └────────────────────┘                                  │
│  Any Logger works ►         Caller gets full type ►      │
│  testable, flexible         no type assertion needed     │
│                                                          │
│  ✅ Small interfaces        ❌ Large interfaces          │
│  ┌──────────────┐           ┌──────────────────────┐     │
│  │ 1-2 methods  │           │ 10+ methods          │     │
│  │ easy to      │           │ hard to implement    │     │
│  │ implement    │           │ hard to mock/test    │     │
│  └──────────────┘           └──────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// ✅ Accept interfaces, return concrete types
type Logger interface {
    Log(msg string)
}

type ConsoleLogger struct{}

func (l ConsoleLogger) Log(msg string) {
    fmt.Println("[LOG]", msg)
}

// Accept interface — flexible, testable
func ProcessOrder(logger Logger, orderID string) {
    logger.Log("Processing order: " + orderID)
}

// Return concrete type — caller knows exactly what they get
func NewConsoleLogger() ConsoleLogger {
    return ConsoleLogger{}
}

// ✅ Small interfaces — Go idiom (1-2 methods)
type Doer interface {
    Do() error
}

// ❌ Avoid large interfaces — hard to implement and test
// type EverythingDoer interface {
//     Read() error
//     Write() error
//     Delete() error
//     Update() error
//     ... 20 more methods
// }

func main() {
    logger := NewConsoleLogger()
    ProcessOrder(logger, "ORD-123")
    // [LOG] Processing order: ORD-123
}
```

---

## Interview Questions

1. **How does Go implement interfaces?**
   - Go uses implicit interface satisfaction — a type satisfies an interface by implementing all its methods, without any explicit declaration. Internally, interfaces are stored as `(type, value)` pairs with an `itab` for method dispatch.

2. **What is the empty interface `interface{}` / `any`?**
   - It has zero methods, so every type satisfies it. It's used for generic containers and functions that accept any value (e.g., `fmt.Println`). `any` is an alias for `interface{}` since Go 1.18.

3. **What is the nil interface gotcha?**
   - An interface holding a nil pointer is NOT a nil interface. `var err error = (*MyError)(nil)` — `err != nil` is `true` because the interface has type info `*MyError` even though the value is nil. Always return `nil` explicitly.

4. **What is a type assertion in Go?**
   - `val, ok := i.(ConcreteType)` extracts the concrete value from an interface. If the interface holds that type, `ok` is true. Without the comma-ok, a failed assertion panics.

5. **What is interface composition?**
   - Embedding interfaces inside other interfaces: `type ReadWriter interface { Reader; Writer }`. This creates larger interfaces from smaller ones — a core Go design principle.

6. **What is the Go idiom "Accept interfaces, return concrete types"?**
   - Functions should accept interface parameters (for flexibility/testability) but return concrete types (to give callers full access to the implementation).

7. **Name some common standard library interfaces.**
   - `fmt.Stringer` (String()), `error` (Error()), `io.Reader` (Read()), `io.Writer` (Write()), `sort.Interface` (Len/Less/Swap), `http.Handler` (ServeHTTP()), `context.Context`.

8. **Why does Go prefer small interfaces (1-2 methods)?**
   - Small interfaces are easier to implement and compose. They follow the Interface Segregation Principle. Types naturally satisfy many small interfaces without coupling.

9. **How do you check if a value implements an interface at compile time?**
   - Use a compile-time assertion: `var _ MyInterface = (*MyType)(nil)`. If `MyType` doesn't implement `MyInterface`, the code won't compile.

10. **What is the difference between a type assertion and a type switch?**
    - Type assertion checks for ONE specific type: `v, ok := i.(int)`. Type switch checks multiple types: `switch v := i.(type) { case int: ... case string: ... }`. Type switch is preferred when handling multiple types.
