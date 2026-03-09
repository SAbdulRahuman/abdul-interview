# Chapter 7 — Pointers

Go has pointers but **no pointer arithmetic** — this prevents entire classes of memory bugs (buffer overflows, out-of-bounds access). Pointers let you share data between functions without copying.

```
┌──────────────────────────────────────────────────────────┐
│                 Pointer Concept                          │
│                                                          │
│  x := 42                                                 │
│  p := &x     ← & takes address                          │
│  *p          ← * dereferences (reads value at address)   │
│                                                          │
│  Stack:                Memory:                           │
│  ┌────────────┐        ┌────────────────┐                │
│  │ x: 42      │◄───────│ p: 0xc000014088│                │
│  │ addr: ...88│        │ (holds x's addr)│               │
│  └────────────┘        └────────────────┘                │
│                                                          │
│  *p = 100  → changes x to 100 (same memory location)    │
│                                                          │
│  Go Pointer Rules:                                       │
│  ✓ &variable  — get address of variable                  │
│  ✓ *pointer   — dereference (read/write value)           │
│  ✗ p++        — NO pointer arithmetic                    │
│  ✗ p + 4      — NO offset calculations                   │
│  ✓ p == nil   — can compare to nil                       │
│  ✓ p == q     — can compare two pointers                 │
└──────────────────────────────────────────────────────────┘
```

## Pointer Basics

```go
package main

import "fmt"

func main() {
    x := 42

    // &x gives the memory address of x
    p := &x
    fmt.Println("Value of x:", x)     // 42
    fmt.Println("Address of x:", p)   // 0xc0000b4008 (some address)

    // *p dereferences the pointer — reads the value at the address
    fmt.Println("Value at p:", *p)    // 42

    // Modify value through pointer
    *p = 100
    fmt.Println("x after *p = 100:", x) // 100

    // Pointer type
    var intPtr *int
    fmt.Println("Nil pointer:", intPtr) // <nil>

    intPtr = &x
    fmt.Println("Now points to x:", *intPtr) // 100
}
```

---

## No Pointer Arithmetic

```go
package main

// Go does NOT allow pointer arithmetic for safety
// This prevents buffer overflows and memory corruption

// In C: ptr++ moves to next memory address — Go forbids this
// p := &x
// p++ // COMPILE ERROR in Go

// If you truly need it, use the unsafe package (avoid if possible)
```

---

## new(T) — Allocate Zeroed Memory

```go
package main

import "fmt"

func main() {
    // new(T) allocates zeroed memory for type T, returns *T
    p := new(int)       // allocates an int, returns *int
    fmt.Println(*p)     // 0 (zero value)

    *p = 42
    fmt.Println(*p)     // 42

    // new for struct
    type Point struct {
        X, Y int
    }

    pt := new(Point)
    fmt.Println(*pt)    // {0 0}
    pt.X = 10           // Go allows pt.X instead of (*pt).X
    pt.Y = 20
    fmt.Println(*pt)    // {10 20}

    // Equivalent to:
    pt2 := &Point{}      // more common idiom
    fmt.Println(*pt2)    // {0 0}
}
```

---

## make(T, ...) — Create Slices, Maps, Channels

```go
package main

import "fmt"

func main() {
    // make creates and initializes slices, maps, and channels
    // Returns T, NOT *T (unlike new)

    // Slice
    s := make([]int, 3, 10)     // len=3, cap=10
    fmt.Println(s, len(s), cap(s)) // [0 0 0] 3 10

    // Map
    m := make(map[string]int)   // initialized, ready to use
    m["key"] = 42
    fmt.Println(m) // map[key:42]

    // Channel
    ch := make(chan int, 5)     // buffered channel with capacity 5
    ch <- 1
    fmt.Println(<-ch) // 1
}
```

---

## new vs make

These are the two built-in allocation functions in Go. They serve different purposes:

```
┌──────────────────────────────────────────────────────────┐
│                  new(T) vs make(T)                       │
│                                                          │
│  new(T)                      make(T, args...)            │
│  ───────────────────         ───────────────────         │
│  Works on ANY type           ONLY slice, map, channel    │
│  Returns *T (pointer)        Returns T (not a pointer)   │
│  Memory is zeroed            Memory is initialized       │
│  Value is NOT ready to use   Value IS ready to use       │
│  (nil slice, nil map)        (usable slice/map/chan)     │
│                                                          │
│  p := new(int)               s := make([]int, 3, 10)    │
│  *p → 0                      s → [0 0 0], len=3, cap=10 │
│                                                          │
│  p := new([]int)             m := make(map[string]int)   │
│  *p → nil (can't append!)   m["k"]=1 → works!           │
│                                                          │
│  In practice, prefer:                                    │
│  &MyStruct{}  over  new(MyStruct)                        │
│  make([]int,0) over new([]int)                           │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // new(T):
    // - Allocates zeroed memory for ANY type
    // - Returns *T (pointer to T)
    // - The value is zero-valued
    p := new([]int)
    fmt.Println(*p)          // [] (nil slice)
    fmt.Println(*p == nil)   // true — not usable with append without initializing

    // make(T):
    // - ONLY for slice, map, channel
    // - Returns T (NOT a pointer)
    // - The value is initialized and ready to use
    s := make([]int, 0, 5)
    fmt.Println(s)           // []
    fmt.Println(s == nil)    // false — initialized, ready to use

    // Summary:
    // Use new() when you need a pointer to a zero-valued type
    // Use make() when creating slices, maps, or channels

    // In practice, struct literals with & are more common than new():
    type Config struct{ Port int }
    cfg := &Config{Port: 8080}  // preferred over new(Config)
    fmt.Println(cfg)            // &{8080}
}
```

---

## Nil Pointers

```go
package main

import "fmt"

func main() {
    var p *int // nil pointer

    fmt.Println(p)       // <nil>
    fmt.Println(p == nil) // true

    // Dereferencing a nil pointer causes PANIC
    // fmt.Println(*p) // panic: runtime error: invalid memory address or nil pointer dereference

    // Always check for nil before dereferencing
    if p != nil {
        fmt.Println(*p)
    } else {
        fmt.Println("Pointer is nil, cannot dereference")
    }
}
```

---

## Passing Pointers to Functions

```go
package main

import "fmt"

type User struct {
    Name  string
    Score int
}

// Pass by value — gets a COPY, original unchanged
func incrementByValue(score int) {
    score++
    fmt.Println("Inside function:", score) // 101
}

// Pass by pointer — modifies the ORIGINAL
func incrementByPointer(score *int) {
    *score++
    fmt.Println("Inside function:", *score) // 101
}

// Large struct — pass pointer for efficiency (avoids copy)
func updateName(u *User, name string) {
    u.Name = name // Go allows u.Name instead of (*u).Name
}

func main() {
    score := 100

    incrementByValue(score)
    fmt.Println("After byValue:", score) // 100 (unchanged!)

    incrementByPointer(&score)
    fmt.Println("After byPointer:", score) // 101 (modified!)

    user := User{Name: "Alice", Score: 90}
    updateName(&user, "Bob")
    fmt.Println(user.Name) // Bob
}
```

---

## Pointer vs Value Semantics

```go
package main

import "fmt"

// Value semantics — data is copied, each variable has its own copy
// Use for: small types, immutable data, types that should not be shared

// Pointer semantics — data is shared via pointer
// Use for: large structs, mutable data, when you need to modify the original

type SmallConfig struct {
    Debug bool
}

type LargeData struct {
    Records [1000]int
    Name    string
}

// Value semantics — fine for small structs
func processConfig(cfg SmallConfig) SmallConfig {
    cfg.Debug = true
    return cfg
}

// Pointer semantics — efficient for large structs
func processData(data *LargeData) {
    data.Name = "processed"
}

func main() {
    cfg := SmallConfig{Debug: false}
    newCfg := processConfig(cfg)
    fmt.Println(cfg.Debug)    // false (original unchanged)
    fmt.Println(newCfg.Debug) // true

    data := &LargeData{Name: "raw"}
    processData(data)
    fmt.Println(data.Name) // processed (modified in place)
}
```

---

## Pointers to Interfaces

```go
package main

import "fmt"

type Speaker interface {
    Speak() string
}

type Dog struct {
    Name string
}

func (d Dog) Speak() string {
    return d.Name + " says: Woof!"
}

func main() {
    dog := Dog{Name: "Rex"}

    // Interface already holds a pointer internally
    // An interface value is a (type, value) pair
    var s Speaker = dog
    fmt.Println(s.Speak()) // Rex says: Woof!

    // Pointer to interface — almost NEVER needed
    // var sp *Speaker = &s  // This is rarely useful and often a bug
    // Use Speaker, not *Speaker, as parameter types

    // The correct pattern:
    var s2 Speaker = &dog // pointer to DOG, stored in interface
    fmt.Println(s2.Speak())

    // WRONG (rarely needed):
    // func doSomething(s *Speaker) {} // avoid this
    // RIGHT:
    // func doSomething(s Speaker) {}  // interface already handles it
}
```

---

## Stack vs Heap — Escape Analysis

```go
package main

import "fmt"

// The Go compiler decides whether to allocate on stack or heap
// using "escape analysis"

// Stack: fast allocation/deallocation, function-scoped
// Heap: slower, garbage collected, survives function scope

// Does NOT escape — allocated on stack
func addLocal(a, b int) int {
    result := a + b // stays on stack
    return result
}

// ESCAPES to heap — pointer returned, must outlive function
func createUser(name string) *string {
    s := name     // s escapes to heap because we return a pointer to it
    return &s
}

func main() {
    sum := addLocal(3, 5)
    fmt.Println(sum) // 8

    name := createUser("Alice")
    fmt.Println(*name) // Alice — still valid because it's on the heap

    // To see escape analysis:
    // go build -gcflags="-m" main.go
    // Output shows which variables "escape to heap"
}
```

---

## Interview Questions

1. **Does Go support pointer arithmetic?**
   - No. Go deliberately disallows pointer arithmetic for safety. You can only use `unsafe.Pointer` for low-level pointer manipulation.

2. **What is the difference between `new` and `make` in Go?**
   - `new(T)` allocates zeroed memory and returns `*T` (works for any type). `make(T, ...)` initializes slices, maps, and channels and returns `T` (not a pointer). `make` sets up internal data structures.

3. **What is a nil pointer dereference?**
   - Dereferencing a pointer whose value is `nil` causes a runtime panic. Always check for nil before dereferencing: `if p != nil { *p }`.

4. **When should you pass a pointer vs a value to a function?**
   - Pass a pointer when: you need to mutate the original, the struct is large (avoid copy cost), or for consistency with other methods. Pass a value when: the data is small, you don't need mutation, or you want immutability guarantees.

5. **What is escape analysis in Go?**
   - The compiler analyzes whether variables escape their function scope. If a variable's address is returned or captured, it "escapes" to the heap. Otherwise it stays on the stack. Use `go build -gcflags="-m"` to inspect.

6. **Should you use pointers to interfaces?**
   - Almost never. An interface value already holds a pointer internally (to the concrete value). `*io.Reader` is redundant and confusing.

7. **What is the zero value of a pointer?**
   - `nil`. All uninitialized pointers are `nil`.

8. **Does Go have references like C++?**
   - No. Go has pointers (`*T`, `&x`) but no references. Slices, maps, and channels have reference-like behavior because they contain internal pointers, but they are still values.

9. **Can a function return a pointer to a local variable?**
   - Yes. Go's escape analysis detects this and allocates the variable on the heap instead of the stack. The pointer remains valid after the function returns.

10. **What is the difference between stack and heap allocation in Go?**
    - Stack: fast, automatic cleanup, goroutine-local. Heap: GC-managed, slower, survives function scope. The compiler decides via escape analysis — the programmer doesn't explicitly choose.
