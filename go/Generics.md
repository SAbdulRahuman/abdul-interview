# Chapter 15 — Generics (Go 1.18+)

Generics allow writing functions and types that work with **any type** satisfying a constraint. Before generics, the only way was `interface{}` (losing type safety) or code generation. Generics provide compile-time type safety with zero runtime overhead.

```
┌──────────────────────────────────────────────────────────┐
│               Generics Concepts                         │
│                                                          │
│  func Max[T cmp.Ordered](a, b T) T { ... }              │
│           │  └──────────┘                                │
│           │   constraint (what T must support)           │
│           └── type parameter                             │
│                                                          │
│  Built-in Constraints:                                  │
│  ┌───────────────┬───────────────────────────────┐       │
│  │ any           │ accepts ALL types              │      │
│  │ comparable    │ supports == and !=             │      │
│  │ cmp.Ordered   │ supports < > <= >= == !=       │      │
│  └───────────────┴───────────────────────────────┘       │
│                                                          │
│  Custom Constraints:                                    │
│  type Number interface {                                │
│      ~int | ~int8 | ~float32 | ~float64                 │
│  }                                                       │
│  │                                                       │
│  └─ ~ (tilde) means "any type whose UNDERLYING type is"│
│     This allows type MyInt int to satisfy Number        │
│                                                          │
│  Limitations:                                           │
│  ✗ No generic methods (only functions & types)          │
│  ✗ No specialization (all T treated the same)           │
│  ✗ No operator constraints beyond ~type unions          │
└──────────────────────────────────────────────────────────┘
```

## Type Parameters

**Tutorial: Declaring and Using Type Parameters**

Type parameters are declared in square brackets after the function name: `func Print[T any](val T)`. The `T` is a placeholder that the compiler replaces with a concrete type at the call site. You can specify the type explicitly (`Print[int](42)`) or let the compiler infer it (`Print(42)`). Multiple type parameters are separated by commas. Watch how `Pair[T, U]` accepts two independent type parameters.

```
┌──────────────────────────────────────────────────┐
│      Type Parameter Resolution                   │
│                                                  │
│  func Print[T any](val T)                        │
│                                                  │
│  Call site            │  T resolves to           │
│  ─────────────────────┼──────────────────        │
│  Print[int](42)       │  int (explicit)          │
│  Print("hello")       │  string (inferred)       │
│  Print(3.14)          │  float64 (inferred)      │
│                                                  │
│  func Pair[T, U any](first T, second U)          │
│                                                  │
│  Pair("name", 42)                                │
│    T ──► string                                  │
│    U ──► int                                     │
│    Output: (name, 42)                            │
└──────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Generic function with type parameter T
func Print[T any](val T) {
    fmt.Println(val)
}

// Multiple type parameters
func Pair[T, U any](first T, second U) {
    fmt.Printf("(%v, %v)\n", first, second)
}

func main() {
    Print[int](42)        // 42 — explicit type argument
    Print[string]("hello") // hello

    // Type inference — compiler figures out T
    Print(3.14)            // 3.14 (T inferred as float64)
    Print(true)            // true (T inferred as bool)

    Pair("name", 42)       // (name, 42)
    Pair(1, true)          // (1, true)
}
```

---

## Type Constraints

**Tutorial: Restricting What Types Are Allowed**

Constraints are interfaces that limit which types can be used as type arguments. `any` allows all types but provides no operations. `comparable` allows types that support `==` and `!=`, which is required for things like map keys or equality checks. `cmp.Ordered` allows types that support ordering operators (`<`, `>`, `<=`, `>=`). The constraint you choose determines what operations are available inside the generic function body.

```
┌──────────────────────────────────────────────────┐
│      Constraint Hierarchy                        │
│                                                  │
│  any (all types)                                 │
│   │   Operations: none (only assign/pass)        │
│   │                                              │
│   ├──► comparable                                │
│   │     Operations: == !=                        │
│   │     Use for: map keys, Contains()            │
│   │                                              │
│   └──► cmp.Ordered                               │
│         Operations: == != < > <= >=              │
│         Use for: Max(), Min(), sorting           │
│                                                  │
│  func Identity[T any](v T) T          ✓ compile  │
│  func Contains[T comparable](...) bool ✓ uses == │
│  func Max[T cmp.Ordered](a,b T) T     ✓ uses >  │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "cmp"
    "fmt"
)

// 'any' constraint — accepts any type
func Identity[T any](val T) T {
    return val
}

// 'comparable' constraint — types that support == and !=
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// cmp.Ordered constraint — types that support < > <= >=
func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Identity("hello")) // hello
    fmt.Println(Identity(42))      // 42

    fmt.Println(Contains([]int{1, 2, 3}, 2))          // true
    fmt.Println(Contains([]string{"a", "b"}, "c"))     // false

    fmt.Println(Max(10, 20))       // 20
    fmt.Println(Max("hello", "world")) // world (lexicographic)

    fmt.Println(Min(3.14, 2.71))   // 2.71
}
```

---

## Generic Functions — Practical Examples

**Tutorial: Map, Filter, Reduce, and Sort with Generics**

These are the classic functional programming patterns implemented with Go generics. `Map` transforms each element using a function, `Filter` keeps elements matching a predicate, and `Reduce` accumulates elements into a single value. `SortSlice` demonstrates using the `cmp.Ordered` constraint to write a type-safe sort. Notice how `Map[T, U]` uses two type parameters — the input slice type and the output type can differ.

```
┌──────────────────────────────────────────────────┐
│   Map / Filter / Reduce Data Flow                │
│                                                  │
│   nums = [1, 2, 3, 4, 5]                        │
│                                                  │
│   Map(nums, ×2)                                  │
│   [1, 2, 3, 4, 5] ──► [2, 4, 6, 8, 10]          │
│    each elem ──► fn(elem) ──► new slice          │
│                                                  │
│   Filter(nums, even?)                            │
│   [1, 2, 3, 4, 5] ──► [2, 4]                    │
│    keep if predicate(elem) == true               │
│                                                  │
│   Reduce(nums, 0, +)                             │
│   [1, 2, 3, 4, 5] ──► 15                        │
│    acc=0 ──► +1=1 ──► +2=3 ──► +3=6 ──► ...      │
│                                                  │
│   SortSlice[T cmp.Ordered]                       │
│   [5, 3, 1, 4, 2] ──► [1, 2, 3, 4, 5]           │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "cmp"
    "fmt"
)

// Generic Map function
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Generic Filter function
func Filter[T any](slice []T, predicate func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// Generic Reduce function
func Reduce[T, U any](slice []T, initial U, fn func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = fn(result, v)
    }
    return result
}

// Generic Sort (using cmp.Ordered)
func SortSlice[T cmp.Ordered](s []T) {
    for i := 0; i < len(s); i++ {
        for j := i + 1; j < len(s); j++ {
            if s[j] < s[i] {
                s[i], s[j] = s[j], s[i]
            }
        }
    }
}

func main() {
    // Map: transform each element
    nums := []int{1, 2, 3, 4, 5}
    doubled := Map(nums, func(n int) int { return n * 2 })
    fmt.Println("Doubled:", doubled) // [2 4 6 8 10]

    strings := Map(nums, func(n int) string {
        return fmt.Sprintf("item-%d", n)
    })
    fmt.Println("Strings:", strings) // [item-1 item-2 item-3 item-4 item-5]

    // Filter: keep matching elements
    evens := Filter(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println("Evens:", evens) // [2 4]

    // Reduce: accumulate to single value
    sum := Reduce(nums, 0, func(acc, n int) int { return acc + n })
    fmt.Println("Sum:", sum) // 15

    // Sort
    unsorted := []int{5, 3, 1, 4, 2}
    SortSlice(unsorted)
    fmt.Println("Sorted:", unsorted) // [1 2 3 4 5]
}
```

---

## Generic Types

**Tutorial: Parameterizing Structs with Type Parameters**

Generic types let you define data structures that work with any element type while keeping full type safety. `Stack[T any]` stores elements of any type, while `Set[T comparable]` requires its element type to support equality checking (needed for map keys). Note the method syntax: `func (s *Stack[T]) Push(item T)` — the type parameter declared on the struct is used in method signatures, but you cannot add new type parameters to methods.

```
┌──────────────────────────────────────────────────┐
│    Generic Stack[T] Memory Layout                │
│                                                  │
│  intStack := &Stack[int]{}                       │
│                                                  │
│  Push(1)  Push(2)  Push(3)     Pop()             │
│  ┌───┐   ┌───┬───┐ ┌───┬───┬───┐  ┌───┬───┐     │
│  │ 1 │   │ 1 │ 2 │ │ 1 │ 2 │ 3 │  │ 1 │ 2 │     │
│  └───┘   └───┴───┘ └───┴───┴───┘  └───┴───┘     │
│                          top ▲       returns 3   │
│                                                  │
│  Generic Set[T comparable]                       │
│  ┌──────────────────────┐                        │
│  │ items: map[T]struct{}│                        │
│  │  "go"   → struct{}   │                        │
│  │  "rust" → struct{}   │                        │
│  │  "go"   → (duplicate,│ no effect)             │
│  └──────────────────────┘                        │
└──────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Generic Stack
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Size() int {
    return len(s.items)
}

// Generic Set
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{items: make(map[T]struct{})}
}

func (s *Set[T]) Add(item T) {
    s.items[item] = struct{}{}
}

func (s *Set[T]) Contains(item T) bool {
    _, ok := s.items[item]
    return ok
}

func (s *Set[T]) Size() int {
    return len(s.items)
}

func main() {
    // Integer stack
    intStack := &Stack[int]{}
    intStack.Push(1)
    intStack.Push(2)
    intStack.Push(3)

    val, _ := intStack.Pop()
    fmt.Println("Popped:", val)    // 3
    fmt.Println("Size:", intStack.Size()) // 2

    // String stack
    strStack := &Stack[string]{}
    strStack.Push("hello")
    strStack.Push("world")
    top, _ := strStack.Peek()
    fmt.Println("Top:", top) // world

    // Set
    set := NewSet[string]()
    set.Add("go")
    set.Add("rust")
    set.Add("go") // duplicate, ignored

    fmt.Println("Contains go:", set.Contains("go"))     // true
    fmt.Println("Contains python:", set.Contains("python")) // false
    fmt.Println("Size:", set.Size())                    // 2
}
```

---

## Interface Type Sets and Tilde (~)

**Tutorial: Union Types and the Tilde Approximation Operator**

Go 1.18 interfaces can now define **type sets** using `|` (union). `type Number interface { int | float64 }` means "T must be exactly `int` or `float64`." The tilde `~` is an approximation element: `~int` means "any type whose *underlying type* is `int`," so `type MyInt int` satisfies `~int` but not plain `int`. This distinction is critical for libraries that need to accept user-defined types built on primitives.

```
┌──────────────────────────────────────────────────┐
│    Tilde (~) — Underlying Type Matching           │
│                                                  │
│  type Number interface { int | float64 }         │
│    int      ✓                                    │
│    float64  ✓                                    │
│    MyInt    ✗ (MyInt is not int, just based on)  │
│                                                  │
│  type Number2 interface { ~int | ~float64 }      │
│    int      ✓                                    │
│    float64  ✓                                    │
│    MyInt    ✓ (underlying type IS int)            │
│                                                  │
│  type Stringish interface { ~string }            │
│                                                  │
│  type MyString string                            │
│    string    ──► underlying: string ──► ✓ ~string│
│    MyString  ──► underlying: string ──► ✓ ~string│
└──────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Interface type set — union of types
type Number interface {
    int | int8 | int16 | int32 | int64 | float32 | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// Tilde (~) — underlying type approximation
// ~int matches: int AND any type defined as `type MyInt int`
type Stringish interface {
    ~string
}

type MyString string

func ToUpper[T Stringish](s T) string {
    result := make([]byte, len(s))
    for i := 0; i < len(s); i++ {
        b := s[i]
        if b >= 'a' && b <= 'z' {
            b -= 32
        }
        result[i] = b
    }
    return string(result)
}

func main() {
    ints := []int{1, 2, 3, 4, 5}
    fmt.Println("Sum ints:", Sum(ints))         // 15

    floats := []float64{1.1, 2.2, 3.3}
    fmt.Println("Sum floats:", Sum(floats))     // 6.6

    // Tilde allows custom types based on string
    var ms MyString = "hello"
    fmt.Println(ToUpper(ms))      // HELLO
    fmt.Println(ToUpper("world")) // WORLD
}
```

---

## Limitations of Generics

**Tutorial: What Go Generics Cannot Do**

Go's generics are deliberately limited compared to languages like Rust or C++. Methods on types cannot introduce new type parameters — only the type itself can be generic. You cannot use type assertions on type parameters, and there is no specialization (providing different implementations for different types). The workaround for method-level type parameters is to use standalone generic functions like `Transform[T, U]` shown below.

```
┌──────────────────────────────────────────────────┐
│   Generic Limitations & Workarounds              │
│                                                  │
│  ✗ Generic methods:                              │
│    func (c Container[T]) Transform[U](...) U     │
│    ──► COMPILE ERROR                             │
│                                                  │
│  ✓ Workaround — standalone function:             │
│    func Transform[T, U any](val T, fn func(T)U)U│
│                                                  │
│  ✗ Type assertions on type params:               │
│    func foo[T any](v T) { s := v.(string) }     │
│    ──► ERROR                                     │
│                                                  │
│  ✗ Specialization:                               │
│    Can't have Max[int] use one body              │
│    and Max[string] use another                   │
└──────────────────────────────────────────────────┘
```

```go
package main

// 1. No generic METHODS — only generic functions and types
// type Container[T any] struct { item T }
// func (c Container[T]) Transform[U any](fn func(T) U) U { ... }
// ^^^ COMPILE ERROR: methods cannot have type parameters

// 2. No specialization — can't provide different implementations for different types

// 3. No operator methods — can't define + - * / for custom types

// 4. No type assertions on type parameters
// func foo[T any](val T) {
//     s := val.(string) // ERROR: cannot use type assertion on type parameter
// }

// Workaround for method type parameters:
// Use a standalone generic function instead

func Transform[T, U any](val T, fn func(T) U) U {
    return fn(val)
}
```

---

## cmp Package (Go 1.21+)

**Tutorial: Comparison Utilities in the Standard Library**

`cmp.Compare` is Go's three-way comparison function — it returns -1 if `a < b`, 0 if equal, and +1 if `a > b`, working with any `cmp.Ordered` type. `cmp.Or` returns the first non-zero value from its arguments, similar to SQL's `COALESCE`. These utilities eliminate boilerplate when writing sorting functions, default-value logic, or custom comparators.

```
┌──────────────────────────────────────────────────┐
│   cmp.Compare and cmp.Or                         │
│                                                  │
│  cmp.Compare(a, b) ──► result                    │
│    a < b  ──► -1                                 │
│    a == b ──►  0                                 │
│    a > b  ──► +1                                 │
│                                                  │
│  cmp.Or(v1, v2, v3, ...) ──► first non-zero      │
│    Or(0, 0, 42, 100) ──► 42                      │
│    Or("", "", "default") ──► "default"            │
│                                                  │
│  Like SQL COALESCE:                              │
│    ┌───┐  ┌───┐  ┌────┐  ┌─────┐                │
│    │ 0 │─►│ 0 │─►│ 42 │─►│ 100 │                │
│    └───┘  └───┘  └──┬─┘  └─────┘                │
│     skip   skip   ◄─┘ first non-zero → return    │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "cmp"
    "fmt"
)

func main() {
    // cmp.Compare — returns -1, 0, or +1
    fmt.Println(cmp.Compare(1, 2))       // -1
    fmt.Println(cmp.Compare(2, 2))       //  0
    fmt.Println(cmp.Compare(3, 2))       //  1
    fmt.Println(cmp.Compare("a", "b"))   // -1

    // cmp.Or — returns the first non-zero value (like COALESCE)
    result := cmp.Or(0, 0, 42, 100)
    fmt.Println("First non-zero:", result) // 42

    name := cmp.Or("", "", "default")
    fmt.Println("Name:", name) // default
}
```

---

## When NOT to Use Generics

**Tutorial: Generics Add Complexity — Use Them Only When Justified**

Go's philosophy is simplicity. Generics should be used when they eliminate significant duplication or provide meaningful type safety. Overusing them leads to hard-to-read, over-abstracted code. Here are clear guidelines.

```
┌──────────────────────────────────────────────────────────┐
│  When to Use vs NOT Use Generics                        │
│                                                          │
│  ✅ USE generics when:                                    │
│  ├── Same algorithm for multiple types (Min, Max, Sort)  │
│  ├── Generic data structures (Stack, Queue, Set, Tree)   │
│  ├── Reducing code duplication (3+ copies of same code)  │
│  ├── Type-safe containers (Result[T], Optional[T])       │
│  └── Standard library utilities (slices.Map, maps.Keys)  │
│                                                          │
│  ❌ DON'T use generics when:                              │
│  ├── Only one or two concrete types needed               │
│  │   → Just write concrete functions                     │
│  ├── An interface already works                          │
│  │   → io.Reader, sort.Interface etc.                    │
│  ├── Behavior differs by type                            │
│  │   → Use interface + concrete implementations          │
│  ├── It makes the API harder to understand               │
│  │   → Readability > cleverness                          │
│  └── You're designing for hypothetical future types      │
│      → YAGNI (You Aren't Gonna Need It)                  │
│                                                          │
│  Rule of thumb:                                          │
│  "If adding a type parameter doesn't save significant    │
│   duplication, don't add it."                            │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// ❌ BAD — over-generic, no real benefit
// Only ever used with string, adds readability cost
func Wrap[T any](val T) []T {
	return []T{val}
}

// ✅ BETTER — just use a concrete function
func WrapString(val string) []string {
	return []string{val}
}

// ❌ BAD — behavior differs per type, use interfaces instead
// Generic function that switches on type defeats the purpose
func Process[T any](val T) string {
	// You can't switch on T directly — this is a code smell
	// If logic differs per type, generics are wrong
	return fmt.Sprintf("%v", val)
}

// ✅ GOOD — interface when behavior varies
type Processable interface {
	Process() string
}

type Order struct{ ID int }
func (o Order) Process() string { return fmt.Sprintf("order-%d", o.ID) }

type Payment struct{ Amount float64 }
func (p Payment) Process() string { return fmt.Sprintf("$%.2f", p.Amount) }

// ✅ GOOD — generics when algorithm is identical across types
func Contains[T comparable](slice []T, target T) bool {
	for _, v := range slice {
		if v == target {
			return true
		}
	}
	return false
}

func main() {
	fmt.Println(Contains([]int{1, 2, 3}, 2))          // true
	fmt.Println(Contains([]string{"a", "b"}, "c"))     // false

	fmt.Println(Order{42}.Process())   // order-42
	fmt.Println(Payment{9.99}.Process()) // $9.99
}
```

---

## Generic Methods Limitation — Deep Dive

**Tutorial: Why Methods Can't Have Type Parameters and How to Work Around It**

Go deliberately prohibits type parameters on methods. A method like `func (c Container[T]) Map[U any](fn func(T) U) U` is **illegal**. This is because Go's interface dispatch needs to know the complete method set at compile time, and method-level type parameters would make that impossible. There are two workarounds: (1) use standalone generic functions, or (2) make both types parameters of the struct.

```
┌──────────────────────────────────────────────────────────┐
│  Why No Generic Methods?                                │
│                                                          │
│  Go interfaces work via method sets.                    │
│  Method sets must be fixed at compile time.             │
│                                                          │
│  If methods had type params:                            │
│    type Stringer interface { String[T]() T }            │
│      → vtable would need infinite entries               │
│      → impossible for interface dispatch                │
│                                                          │
│  ❌ ILLEGAL:                                             │
│  func (c Container[T]) Map[U any](fn func(T) U) []U    │
│                             ^^^                          │
│                 methods cannot have type parameters      │
│                                                          │
│  ✅ WORKAROUND 1: Standalone generic function            │
│  func Map[T, U any](c Container[T], fn func(T) U) []U  │
│                                                          │
│  ✅ WORKAROUND 2: Both type params on the struct         │
│  type Converter[T, U any] struct { ... }                │
│  func (c Converter[T, U]) Convert(v T) U               │
│                                                          │
│  ✅ WORKAROUND 3: Accept interface in method             │
│  func (c Container[T]) Apply(fn func(T) any) []any     │
│  (loses type safety — less ideal)                       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type Container[T any] struct {
	items []T
}

func NewContainer[T any](items ...T) Container[T] {
	return Container[T]{items: items}
}

// ❌ COMPILE ERROR — methods cannot have type parameters
// func (c Container[T]) Map[U any](fn func(T) U) []U { ... }

// ✅ Workaround 1: Standalone generic function
func Map[T, U any](c Container[T], fn func(T) U) []U {
	result := make([]U, len(c.items))
	for i, v := range c.items {
		result[i] = fn(v)
	}
	return result
}

// ✅ Workaround 2: Both types on the struct
type Mapper[T, U any] struct {
	fn func(T) U
}

func (m Mapper[T, U]) MapSlice(items []T) []U {
	result := make([]U, len(items))
	for i, v := range items {
		result[i] = m.fn(v)
	}
	return result
}

func main() {
	c := NewContainer(1, 2, 3, 4, 5)

	// Workaround 1 — standalone function
	strs := Map(c, func(n int) string {
		return fmt.Sprintf("#%d", n)
	})
	fmt.Println(strs) // [#1 #2 #3 #4 #5]

	doubles := Map(c, func(n int) int { return n * 2 })
	fmt.Println(doubles) // [2 4 6 8 10]

	// Workaround 2 — mapper struct
	m := Mapper[int, string]{fn: func(n int) string {
		return fmt.Sprintf("item-%d", n)
	}}
	fmt.Println(m.MapSlice([]int{10, 20})) // [item-10 item-20]
}
```

---

## Performance Considerations

**Tutorial: How Go Compiles Generics and What It Means for Performance**

Go uses a hybrid approach called **GC shape stenciling with dictionaries**. Types that share the same GC shape (e.g., all pointer types, all 64-bit integers) share one compiled function, with a dictionary passed to provide type-specific information. This is a middle ground between full monomorphization (C++/Rust — fastest but binary bloat) and full boxing (Java — slower).

```
┌──────────────────────────────────────────────────────────┐
│  How Go Compiles Generics: GC Shape Stenciling          │
│                                                          │
│  func Max[T cmp.Ordered](a, b T) T                      │
│                                                          │
│  Concrete calls:  Max[int](1, 2)                         │
│                   Max[int32](1, 2)                        │
│                   Max[string]("a", "b")                   │
│                   Max[*MyType](p1, p2)                    │
│                                                          │
│  Compiled as:                                            │
│  ┌─────────────────────────────────────────────────┐     │
│  │ Max_int     ← one function for int-shaped       │     │
│  │ Max_int32   ← one function for int32-shaped     │     │
│  │ Max_string  ← one function for string-shaped    │     │
│  │ Max_pointer ← ONE function for ALL pointer types│     │
│  │               (uses dictionary for type info)   │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
│  Comparison with other languages:                        │
│  ┌────────────────┬───────────────┬───────────────┐      │
│  │ C++ / Rust     │ Full Mono     │ Fastest, big  │      │
│  │                │ (one per type)│ binaries      │      │
│  ├────────────────┼───────────────┼───────────────┤      │
│  │ Go             │ GC Shape +   │ Balanced, some│      │
│  │                │ Dictionaries  │ overhead      │      │
│  ├────────────────┼───────────────┼───────────────┤      │
│  │ Java           │ Type erasure  │ Slower (boxing│      │
│  │                │ + boxing      │ heap allocs)  │      │
│  └────────────────┴───────────────┴───────────────┘      │
│                                                          │
│  Performance implications:                               │
│  • Generics with value types (int, float64): ~same speed│
│    as hand-written concrete functions                    │
│  • Generics with pointer types: small overhead from     │
│    dictionary lookup (typically negligible)              │
│  • No heap allocations for value types in generics      │
│  • Interface{}-based code IS slower (boxing + assertion) │
│  • Generic code may not inline as aggressively          │
│                                                          │
│  When performance matters:                               │
│  • Benchmark generic vs concrete in hot paths           │
│  • For ultra-hot loops, concrete functions may inline    │
│    better than generic ones                              │
│  • The difference is usually <5-10%, rarely matters     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"time"
)

// Generic sum
func SumGeneric[T int | int64 | float64](nums []T) T {
	var total T
	for _, n := range nums {
		total += n
	}
	return total
}

// Concrete sum (hand-written for int)
func SumConcrete(nums []int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

// interface{}-based sum (pre-generics approach)
func SumInterface(nums []any) int {
	total := 0
	for _, n := range nums {
		total += n.(int) // type assertion at runtime
	}
	return total
}

func main() {
	// Build test data
	const N = 10_000_000
	ints := make([]int, N)
	anys := make([]any, N)
	for i := range ints {
		ints[i] = i
		anys[i] = i
	}

	// Benchmark comparison (approximate)
	start := time.Now()
	SumConcrete(ints)
	concreteDur := time.Since(start)

	start = time.Now()
	SumGeneric(ints)
	genericDur := time.Since(start)

	start = time.Now()
	SumInterface(anys)
	interfaceDur := time.Since(start)

	fmt.Printf("Concrete:  %v\n", concreteDur)
	fmt.Printf("Generic:   %v  (~same as concrete)\n", genericDur)
	fmt.Printf("Interface: %v  (slower — boxing + type assertion)\n", interfaceDur)

	// Typical results:
	// Concrete:  ~8ms
	// Generic:   ~8ms  (negligible difference)
	// Interface: ~25ms (3x slower from boxing)
}
```

---

## Interview Questions

1. **What are generics in Go and when were they introduced?**
   - Generics (type parameters) were introduced in Go 1.18. They allow writing functions and types that work with multiple types while maintaining type safety: `func Max[T cmp.Ordered](a, b T) T`.

2. **What is a type constraint?**
   - An interface that restricts what types can be used as type arguments. `any` allows all types. `comparable` allows types that support `==`. `cmp.Ordered` allows ordered types (integers, floats, strings).

3. **What is the tilde `~` operator in type constraints?**
   - `~int` matches `int` and any type whose underlying type is `int` (e.g., `type MyInt int`). Without `~`, only the exact type matches.

4. **Can you have generic methods in Go?**
   - No. Go does not support type parameters on methods. Only functions and types can be generic. To work around this, use generic functions or make the entire type generic.

5. **What is type inference in Go generics?**
   - The compiler can often infer type arguments from function arguments: `Max(3, 5)` instead of `Max[int](3, 5)`. This makes generic code cleaner.

6. **What are interface type sets?**
   - In Go 1.18+, interfaces can specify union type sets: `type Number interface { int | float64 }`. A type satisfies the interface if it is one of the listed types.

7. **What is `comparable` used for?**
   - It's a built-in constraint that allows types supporting `==` and `!=`. Required for using type parameters as map keys or comparing values.

8. **What are the limitations of Go generics?**
   - No generic methods (only functions and types). No specialization. No operator overloading. No covariance/contravariance. No type parameters on interfaces used as constraints simultaneously.

9. **How do generics compare to `interface{}` / `any`?**
   - Generics provide compile-time type safety and avoid runtime type assertions. `interface{}` loses type information and requires runtime casting. Generics are preferred when type safety matters.

10. **What is `cmp.Compare` and `cmp.Or`?**
    - `cmp.Compare(a, b)` returns -1, 0, or 1 (like Java's compareTo). `cmp.Or(vals...)` returns the first non-zero value. Both were added in Go 1.21 for cleaner comparison logic.

11. **When should you NOT use generics?**
    - When only one or two concrete types are needed, when behavior differs by type (use interfaces), when an existing interface like `io.Reader` already works, or when it makes the API harder to read. Generics should eliminate real duplication, not add premature abstraction.

12. **Why can't Go methods have type parameters?**
    - Go interfaces dispatch via method sets which must be fixed at compile time. Method-level type parameters would require infinite vtable entries. Workarounds: standalone generic functions, or make both types parameters of the struct itself.

13. **How does Go compile generics? Is there a performance cost?**
    - Go uses **GC shape stenciling with dictionaries**. Types sharing the same GC shape (e.g., all pointers) share one compiled function with a dictionary for type info. For value types, performance is nearly identical to hand-written concrete code. Pointer types have minor dictionary overhead. Generic code may not inline as aggressively as concrete code, but the difference is typically <5-10%.

14. **Generics vs `interface{}` — what's the performance difference?**
    - Generics avoid boxing/unboxing overhead. `interface{}` requires heap allocation for value types and runtime type assertions. In benchmarks, generics can be 2-3x faster than `interface{}`-based code for tight loops with value types.
