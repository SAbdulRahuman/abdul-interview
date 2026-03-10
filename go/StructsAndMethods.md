# Chapter 8 — Structs & Methods

Methods in Go are functions with a **receiver** argument. The receiver binds a function to a type. There are two kinds: **value receivers** (get a copy) and **pointer receivers** (get the address). The choice between them affects whether the method can modify the receiver and which interfaces the type satisfies.

```
┌──────────────────────────────────────────────────────────┐
│         Value Receiver vs Pointer Receiver               │
│                                                          │
│  func (c Circle) Area() float64    ← VALUE receiver     │
│  func (c *Counter) Increment()     ← POINTER receiver   │
│                                                          │
│  Value Receiver:          Pointer Receiver:              │
│  ┌────────────────┐       ┌────────────────┐             │
│  │ Gets a COPY    │       │ Gets ADDRESS   │             │
│  │ Cannot modify  │       │ CAN modify     │             │
│  │ Safe to call   │       │ Can mutate     │             │
│  │ concurrently   │       │ the original   │             │
│  └────────────────┘       └────────────────┘             │
│                                                          │
│  Method Set Rules (for interface satisfaction):          │
│  ┌──────────┬──────────────────────────────────┐         │
│  │ Type     │ Method set includes              │         │
│  ├──────────┼──────────────────────────────────┤         │
│  │ T        │ value-receiver methods only      │         │
│  │ *T       │ BOTH value + pointer receiver    │         │
│  └──────────┴──────────────────────────────────┘         │
│                                                          │
│  ⚠ If ANY method uses pointer receiver,                │
│    use pointer receiver for ALL methods (consistency)    │
└──────────────────────────────────────────────────────────┘
```

## Defining Methods

**Tutorial: Attaching Methods to a Struct**

Go has no classes — instead you attach methods to types by declaring a **receiver** between `func` and the method name. This example defines two value-receiver methods on a `Circle` struct. Notice the receiver `(c Circle)` gives the method a copy of the struct, so reading fields is fine but mutations would not affect the caller.

```
┌──────────────────────────────────────────────────────────┐
│          Method Declaration Anatomy                      │
│                                                          │
│  func (c Circle) Area() float64 { ... }                  │
│        ▲           ▲        ▲                            │
│        │           │        └─ return type                │
│        │           └─ method name                        │
│        └─ receiver (copy of Circle)                      │
│                                                          │
│  Calling:                                                │
│  c := Circle{Radius: 5}                                  │
│  c.Area()   ─►  (c Circle).Area()                        │
│                  copy of c passed in                      │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Circle struct {
    Radius float64
}

// Method with value receiver
func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * 3.14159 * c.Radius
}

func main() {
    c := Circle{Radius: 5}
    fmt.Printf("Area: %.2f\n", c.Area())           // Area: 78.54
    fmt.Printf("Perimeter: %.2f\n", c.Perimeter())  // Perimeter: 31.42
}
```

---

## Value Receivers vs Pointer Receivers

**Tutorial: Copy vs Address — Choosing the Right Receiver**

A value receiver `(c Counter)` works on a copy — mutations inside the method vanish when it returns. A pointer receiver `(c *Counter)` receives the memory address, so `c.Count++` modifies the original struct. Go automatically takes the address or dereferences as needed at the call site, so `c.Increment()` works whether `c` is a value or a pointer.

```
┌──────────────────────────────────────────────────────────┐
│       Value Receiver              Pointer Receiver       │
│                                                          │
│  c := Counter{0}                c := Counter{0}          │
│       ┌──────┐                       ┌──────┐            │
│  c    │  0   │                  c    │  0   │            │
│       └──┬───┘                       └──┬───┘            │
│          │ copy                         │ &c (address)   │
│          ▼                              ▼                │
│  ┌────────────┐                 ┌────────────┐           │
│  │ copy: 0    │ ← method sees   │ *ptr → c   │ ← same   │
│  │ copy++ = 1 │   its own copy  │ c.Count++  │   memory  │
│  └────────────┘                 └────────────┘           │
│  c is still 0 ✗                 c is now 1 ✓            │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Counter struct {
    Count int
}

// Value receiver — gets a COPY, cannot modify original
func (c Counter) DisplayValue() {
    fmt.Println("Count (value):", c.Count)
}

// Pointer receiver — gets the ADDRESS, CAN modify original
func (c *Counter) Increment() {
    c.Count++
}

// Pointer receiver — mutates the receiver
func (c *Counter) Reset() {
    c.Count = 0
}

func main() {
    c := Counter{Count: 0}

    c.Increment()   // Go auto-takes address: (&c).Increment()
    c.Increment()
    c.DisplayValue() // Count (value): 2

    c.Reset()
    c.DisplayValue() // Count (value): 0

    // With pointer variable
    p := &Counter{Count: 10}
    p.Increment()
    p.DisplayValue() // Count (value): 11
}
```

---

## When to Use Pointer Receivers

**Tutorial: Pointer Receiver Guidelines — Mutate, Large, Consistent**

Use a pointer receiver when: (1) the method mutates the receiver's state, (2) the struct is large and copying is expensive, or (3) for consistency if any method already uses a pointer receiver. This example models a `Database` with mutable state — every method uses a pointer receiver, even `Get` which only reads, to keep the method set uniform.

```
┌──────────────────────────────────────────────────────────┐
│         Decision: Value or Pointer Receiver?             │
│                                                          │
│  ┌─ Does method MUTATE state? ──► YES ──► *T (pointer)   │
│  │                                                       │
│  ├─ Is struct LARGE (many fields)? ──► YES ──► *T        │
│  │                                                       │
│  ├─ Do OTHER methods use *T? ──► YES ──► *T (consistent) │
│  │                                                       │
│  └─ None of the above? ──► T (value) is fine             │
│                                                          │
│  Database example:                                       │
│  Connect()  → mutates connections  → *Database           │
│  Set(k,v)   → mutates data map    → *Database            │
│  Get(k)     → reads only BUT      → *Database            │
│               consistency wins                           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Use POINTER receiver when:
// 1. Method needs to MODIFY the receiver
// 2. Struct is LARGE (avoid copying)
// 3. CONSISTENCY — if one method uses pointer, all should

type Database struct {
    connections int
    data        map[string]string // large internal state
}

func NewDatabase() *Database {
    return &Database{
        data: make(map[string]string),
    }
}

// Pointer receiver — modifies state
func (db *Database) Connect() {
    db.connections++
    fmt.Println("Connected. Total:", db.connections)
}

// Pointer receiver — modifies state
func (db *Database) Set(key, value string) {
    db.data[key] = value
}

// Pointer receiver for consistency (even though it doesn't modify)
func (db *Database) Get(key string) (string, bool) {
    v, ok := db.data[key]
    return v, ok
}

func main() {
    db := NewDatabase()
    db.Connect()
    db.Set("name", "Alice")

    val, ok := db.Get("name")
    fmt.Println(val, ok) // Alice true
}
```

---

## Method Sets

A type's **method set** determines which interfaces it satisfies:
- **Value type (T)** — only value-receiver methods
- **Pointer type (*T)** — both value-receiver AND pointer-receiver methods

**Tutorial: Method Sets and Interface Satisfaction**

The method set rules matter most when passing values to interface parameters. A value `buf` of type `Buffer` can satisfy `Sizer` (value-receiver methods only), but only `&buf` satisfies `Resizer` (which requires a pointer-receiver method). This is the most common source of "X does not implement Y" compile errors.

```
┌──────────────────────────────────────────────────────────┐
│         Method Set Rules                                 │
│                                                          │
│  type Buffer struct { data []byte }                      │
│  func (b Buffer)  Size() int    ← value receiver        │
│  func (b *Buffer) Resize(n int) ← pointer receiver      │
│                                                          │
│  Interface satisfaction:                                 │
│                                                          │
│  buf := Buffer{}                                         │
│  ┌──────────────────────────────────────────────┐        │
│  │ Type    │ Method Set          │ Satisfies     │        │
│  ├─────────┼─────────────────────┼───────────────┤        │
│  │ Buffer  │ { Size }            │ Sizer ✓       │        │
│  │         │                     │ Resizer ✗     │        │
│  ├─────────┼─────────────────────┼───────────────┤        │
│  │ *Buffer │ { Size, Resize }    │ Sizer ✓       │        │
│  │         │                     │ Resizer ✓     │        │
│  └─────────┴─────────────────────┴───────────────┘        │
│                                                          │
│  var s Sizer = buf    ✓  (value-receiver in T's set)    │
│  var r Resizer = buf  ✗  COMPILE ERROR                  │
│  var r Resizer = &buf ✓  (*T has both method kinds)     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Sizer interface {
    Size() int
}

type Resizer interface {
    Resize(n int)
}

type Buffer struct {
    data []byte
}

// Value receiver — in method set of BOTH Buffer and *Buffer
func (b Buffer) Size() int {
    return len(b.data)
}

// Pointer receiver — ONLY in method set of *Buffer
func (b *Buffer) Resize(n int) {
    b.data = make([]byte, n)
}

func main() {
    buf := Buffer{data: make([]byte, 10)}

    // Value type satisfies Sizer (has value-receiver method)
    var s Sizer = buf
    fmt.Println(s.Size()) // 10

    // Value type does NOT satisfy Resizer (missing pointer-receiver method)
    // var r Resizer = buf  // COMPILE ERROR: Buffer does not implement Resizer

    // Pointer type satisfies BOTH
    var s2 Sizer = &buf
    var r2 Resizer = &buf
    fmt.Println(s2.Size()) // 10
    r2.Resize(20)
    fmt.Println(buf.Size()) // 20
}
```

---

## Exported vs Unexported

**Tutorial: Visibility by Naming Convention — Uppercase vs Lowercase**

Go controls visibility with a single rule: identifiers starting with an **uppercase** letter are exported (visible outside the package), and **lowercase** are unexported (package-private). This applies to struct fields, methods, types, and functions alike. The code below shows `Name` and `Email` as exported fields while `age` is hidden from external packages.

```
┌──────────────────────────────────────────────────────────┐
│       Exported vs Unexported Visibility                  │
│                                                          │
│  type User struct {                                      │
│      Name  string   ← Uppercase = EXPORTED  (public)    │
│      Email string   ← Uppercase = EXPORTED  (public)    │
│      age   int      ← lowercase = UNEXPORTED (private)  │
│  }                                                       │
│                                                          │
│  From another package:                                   │
│  ┌─────────────────────────────────┐                     │
│  │  u.Name  ✓  accessible         │                     │
│  │  u.Email ✓  accessible         │                     │
│  │  u.age   ✗  compile error      │                     │
│  └─────────────────────────────────┘                     │
│                                                          │
│  Same package: all fields accessible ✓                   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type User struct {
    Name  string // Exported — accessible from other packages
    Email string // Exported
    age   int    // Unexported — only accessible within this package
}

// Exported method
func (u User) String() string {
    return fmt.Sprintf("%s (%s)", u.Name, u.Email)
}

// Unexported method — only callable within this package
func (u *User) setAge(age int) {
    u.age = age
}

func main() {
    u := User{Name: "Alice", Email: "alice@test.com", age: 30}
    fmt.Println(u.String()) // Alice (alice@test.com)
    u.setAge(31)            // works within same package
    fmt.Println(u.age)      // 31
}
```

---

## Struct Embedding — Composition Over Inheritance

**Tutorial: Embedding Structs for Method Promotion**

Go replaces inheritance with **embedding**: placing a type inside a struct without a field name promotes all its fields and methods to the outer type. `Dog` embeds `Animal`, so `dog.Name` and `dog.Move()` work directly. If `Dog` defines its own `Speak()`, it **shadows** the embedded version — but you can still call `dog.Animal.Speak()` explicitly. Multi-level embedding chains promotions further.

```
┌──────────────────────────────────────────────────────────┐
│       Struct Embedding — Promotion & Shadowing           │
│                                                          │
│  ┌────────────────┐                                      │
│  │    Animal       │  Name string                        │
│  │    Speak()      │  Move()                             │
│  └───────┬────────┘                                      │
│          │ embed                                         │
│          ▼                                               │
│  ┌────────────────┐                                      │
│  │    Dog          │  Breed string                       │
│  │    Speak()      │  ◄── shadows Animal.Speak()         │
│  │    Move()       │  ◄── promoted from Animal           │
│  │    Name         │  ◄── promoted from Animal           │
│  └───────┬────────┘                                      │
│          │ embed                                         │
│          ▼                                               │
│  ┌────────────────┐                                      │
│  │  ServiceDog     │  Task string                        │
│  │  Speak(), Move()│  ◄── promoted through Dog           │
│  │  Name, Breed    │  ◄── promoted through Dog           │
│  └────────────────┘                                      │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Base "class" — Go uses composition, not inheritance
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return a.Name + " makes a sound"
}

func (a Animal) Move() string {
    return a.Name + " moves"
}

// Dog "extends" Animal via embedding
type Dog struct {
    Animal          // Embedded — promotes all fields and methods
    Breed  string
}

// Override the Speak method
func (d Dog) Speak() string {
    return d.Name + " barks!" // d.Name is promoted from Animal
}

// ServiceDog adds another layer
type ServiceDog struct {
    Dog
    Task string
}

func main() {
    dog := Dog{
        Animal: Animal{Name: "Rex"},
        Breed:  "German Shepherd",
    }

    // Promoted methods from Animal
    fmt.Println(dog.Move())   // Rex moves (promoted from Animal)
    fmt.Println(dog.Speak())  // Rex barks! (overridden by Dog)
    fmt.Println(dog.Name)     // Rex (promoted field)

    // Access the embedded struct explicitly
    fmt.Println(dog.Animal.Speak()) // Rex makes a sound (Animal's original method)

    // Multi-level embedding
    sd := ServiceDog{
        Dog:  Dog{Animal: Animal{Name: "Buddy"}, Breed: "Labrador"},
        Task: "Guide",
    }
    fmt.Println(sd.Name)    // Buddy (promoted through Dog → Animal)
    fmt.Println(sd.Speak()) // Buddy barks! (from Dog)
    fmt.Println(sd.Task)    // Guide
}
```

---

## Constructor Pattern

**Tutorial: NewXxx Constructor Functions**

Go has no built-in constructors. By convention, you write a `NewXxx` function that validates inputs, sets defaults, and returns a pointer (often with an error). This pattern gives you full control over initialization — the `Server` struct below keeps its fields unexported, and `NewServer` is the only way to create a valid instance with a sensible `maxConn` default.

```
┌──────────────────────────────────────────────────────────┐
│       Constructor Pattern Flow                           │
│                                                          │
│  caller                                                  │
│    │                                                     │
│    │  NewServer("localhost", 8080)                        │
│    ▼                                                     │
│  ┌──────────────────────────┐                             │
│  │  Validate inputs         │                             │
│  │  host == "" ? → error    │                             │
│  │  port < 1 ?   → error    │                             │
│  └──────────┬───────────────┘                             │
│             │ ok                                         │
│             ▼                                            │
│  ┌──────────────────────────┐                             │
│  │  &Server{                │                             │
│  │    host:    "localhost", │                             │
│  │    port:    8080,        │                             │
│  │    maxConn: 100, ← default                            │
│  │  }                       │                             │
│  └──────────┬───────────────┘                             │
│             │                                            │
│             ▼                                            │
│  return (*Server, nil)                                   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "errors"
    "fmt"
)

type Server struct {
    host    string
    port    int
    maxConn int
}

// Constructor pattern: NewXxx returns *Xxx
func NewServer(host string, port int) (*Server, error) {
    if host == "" {
        return nil, errors.New("host cannot be empty")
    }
    if port < 1 || port > 65535 {
        return nil, errors.New("invalid port number")
    }

    return &Server{
        host:    host,
        port:    port,
        maxConn: 100, // default value
    }, nil
}

func (s *Server) Address() string {
    return fmt.Sprintf("%s:%d", s.host, s.port)
}

func main() {
    srv, err := NewServer("localhost", 8080)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Server:", srv.Address()) // Server: localhost:8080
}
```

---

## Methods on Non-Struct Types

**Tutorial: Methods on Named Types — Slices, Floats, and More**

You can attach methods to any **named type**, not just structs. Define `type StringSlice []string` and you can add `Join`, `Contains`, and `Add` methods to it. The same works for numeric types like `Celsius` and `Fahrenheit`. Note that `Add` uses a pointer receiver because `append` may reallocate the underlying array — you must write back through the pointer with `*ss = append(*ss, s)`.

```
┌──────────────────────────────────────────────────────────┐
│       Methods on Named Types                             │
│                                                          │
│  type StringSlice []string                               │
│       ▲                                                  │
│       │ named type wraps built-in []string               │
│       │                                                  │
│  ┌────┴────────────────────────────────────┐              │
│  │  (ss StringSlice) Join(sep) string      │ value recv  │
│  │  (ss StringSlice) Contains(t) bool      │ value recv  │
│  │  (ss *StringSlice) Add(s string)        │ ptr recv    │
│  └─────────────────────────────────────────┘              │
│                                                          │
│  type Celsius float64   ──► ToFahrenheit() Fahrenheit    │
│  type Fahrenheit float64 ──► ToCelsius() Celsius         │
│                                                          │
│  Cannot add methods to unnamed types:                    │
│  func (s []string) Bad() {} ← COMPILE ERROR             │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "strings"
)

// You can define methods on any named type (not just structs)

type StringSlice []string

func (ss StringSlice) Join(sep string) string {
    return strings.Join(ss, sep)
}

func (ss StringSlice) Contains(target string) bool {
    for _, s := range ss {
        if s == target {
            return true
        }
    }
    return false
}

func (ss *StringSlice) Add(s string) {
    *ss = append(*ss, s)
}

type Celsius float64
type Fahrenheit float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

func main() {
    tags := StringSlice{"go", "programming", "tutorial"}
    fmt.Println(tags.Join(", "))        // go, programming, tutorial
    fmt.Println(tags.Contains("go"))    // true

    tags.Add("interview")
    fmt.Println(tags.Join(", "))        // go, programming, tutorial, interview

    temp := Celsius(100)
    fmt.Printf("%.1f°C = %.1f°F\n", temp, temp.ToFahrenheit())
    // 100.0°C = 212.0°F

    f := Fahrenheit(72)
    fmt.Printf("%.1f°F = %.1f°C\n", f, f.ToCelsius())
    // 72.0°F = 22.2°C
}
```

---

## Interview Questions

1. **What is the difference between a value receiver and a pointer receiver?**
   - Value receiver `func (s T) M()` operates on a copy — cannot mutate the original. Pointer receiver `func (s *T) M()` operates on the original — can mutate fields. Use pointer receivers for mutation, large structs, or consistency.

2. **What is a method set in Go?**
   - The method set of a type determines which interfaces it satisfies. A value type `T` has only value-receiver methods. A pointer type `*T` has both value and pointer-receiver methods.

3. **Can you define methods on built-in types in Go?**
   - Not directly. You must create a named type: `type MySlice []int`, then define methods on `MySlice`.

4. **What is struct embedding and how does it differ from inheritance?**
   - Embedding places a type inside a struct without a field name. Its methods are promoted but it's composition, not inheritance — there's no polymorphism, no virtual dispatch, and the outer type doesn't "is-a" the inner type.

5. **What is the constructor pattern in Go?**
   - Since Go has no constructors, the convention is a `NewXxx` function: `func NewServer(port int) *Server { ... }`. It returns an initialized instance.

6. **How are exported and unexported fields different?**
   - Exported fields start with uppercase — visible outside the package. Unexported fields start with lowercase — only accessible within the package. This is Go's encapsulation mechanism.

7. **Can you have multiple methods with the same name on different types?**
   - Yes. Methods are defined per type, so `func (a TypeA) Foo()` and `func (b TypeB) Foo()` are separate methods.

8. **What happens when an embedded struct has a method with the same name as the outer struct?**
   - The outer struct's method takes precedence (shadows the embedded method). The embedded method is still accessible via the embedded type name.

9. **Why should you use consistent receiver types (all value or all pointer)?**
   - Mixing can cause confusion about the method set. If any method needs a pointer receiver, all methods should use pointer receivers for consistency and to ensure the type satisfies interfaces predictably.

10. **Can you create methods on function types in Go?**
    - Yes. Define a function type and attach methods: `type HandlerFunc func(w Writer, r *Request)` then `func (f HandlerFunc) ServeHTTP(w, r)`. This is how `http.HandlerFunc` works.
