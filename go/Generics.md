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
