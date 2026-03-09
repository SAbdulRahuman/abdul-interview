# Chapter 5 — Composite Types

Composite types aggregate other values. Go has four composite types: **arrays**, **slices**, **maps**, and **structs**. Understanding how these are stored in memory — value semantics vs reference semantics — is key to avoiding bugs.

```
┌──────────────────────────────────────────────────────────┐
│           Composite Types — Value vs Reference           │
│                                                          │
│  VALUE types (copied on assignment/pass):                │
│  ┌─────────┐  ┌─────────────────────────────┐            │
│  │  Array  │  │ [3]int{1,2,3}               │            │
│  │  Struct │  │ Person{Name:"Al", Age:30}    │            │
│  └─────────┘  └─────────────────────────────┘            │
│  b := a    → creates a full independent copy             │
│                                                          │
│  REFERENCE-like types (header copied, data shared):      │
│  ┌─────────┐  ┌──────────────────┐  ┌──────────────┐    │
│  │  Slice  │  │ ptr | len | cap  │──►│ backing array│   │
│  │  Map    │  │ ptr to hash table│──►│ hash buckets │   │
│  └─────────┘  └──────────────────┘  └──────────────┘    │
│  b := a    → copies the header, both share same data     │
└──────────────────────────────────────────────────────────┘
```

## Arrays

### Fixed-Size, Value Semantics

Arrays in Go have a **fixed size** that is part of the type. `[3]int` and `[4]int` are completely different types. Arrays are **values** — assigning or passing an array creates a full copy.

```go
package main

import "fmt"

func main() {
    // Declaration
    var a [3]int                    // [0 0 0] — zero valued
    b := [3]int{1, 2, 3}           // explicit values
    c := [...]int{10, 20, 30, 40}  // compiler counts: [4]int

    fmt.Println(a) // [0 0 0]
    fmt.Println(b) // [1 2 3]
    fmt.Println(c) // [10 20 30 40]

    // Value semantics — arrays are COPIED on assignment
    original := [3]string{"a", "b", "c"}
    copied := original
    copied[0] = "Z"

    fmt.Println(original) // [a b c]  — original unchanged!
    fmt.Println(copied)   // [Z b c]  — only copy changed

    // Array comparison with ==
    x := [3]int{1, 2, 3}
    y := [3]int{1, 2, 3}
    z := [3]int{1, 2, 4}

    fmt.Println(x == y) // true
    fmt.Println(x == z) // false
    // Note: arrays of DIFFERENT sizes are different types
    // [3]int and [4]int cannot be compared

    // Iterating
    for i, v := range b {
        fmt.Printf("b[%d] = %d\n", i, v)
    }

    // Length
    fmt.Println("Length of c:", len(c)) // 4
}

// Arrays passed to functions are COPIED
func modifyArray(arr [3]int) {
    arr[0] = 999 // modifies the copy, not the original
}
```

---

## Slices

### Slice Internals — Header: Pointer, Length, Capacity

A slice is a **descriptor** (24 bytes on 64-bit systems) that references a contiguous segment of an underlying array. It consists of three fields:
- **Pointer** — to the first element visible through the slice
- **Length** — number of elements in the slice (`len(s)`)
- **Capacity** — number of elements from the pointer to the end of the underlying array (`cap(s)`)

```
┌──────────────────────────────────────────────────────────┐
│                  Slice Internals                         │
│                                                          │
│  s := []int{10, 20, 30, 40, 50}                         │
│  sub := s[1:3]                                           │
│                                                          │
│  Slice header "s":        Underlying array:              │
│  ┌─────────────┐          ┌────┬────┬────┬────┬────┐    │
│  │ ptr ────────────────►  │ 10 │ 20 │ 30 │ 40 │ 50 │    │
│  │ len: 5      │          └────┴────┴────┴────┴────┘    │
│  │ cap: 5      │            [0]  [1]  [2]  [3]  [4]    │
│  └─────────────┘                                        │
│                                                          │
│  Slice header "sub":                                     │
│  ┌─────────────┐               ▼                        │
│  │ ptr ──────────────────────► 20 │ 30 │ 40 │ 50 │      │
│  │ len: 2      │              ─────────────────────      │
│  │ cap: 4      │              visible   beyond len       │
│  └─────────────┘              ◄─len─►◄────────►          │
│                               ◄──────cap────────►        │
│                                                          │
│  ⚠ sub[0] = 999 also changes s[1] (shared memory!)     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // Slice literal
    s := []int{10, 20, 30, 40, 50}
    fmt.Println("Slice:", s)
    fmt.Println("Length:", len(s))   // 5
    fmt.Println("Capacity:", cap(s)) // 5

    // Slicing an existing slice
    sub := s[1:3] // elements at index 1, 2
    fmt.Println("Sub:", sub)         // [20 30]
    fmt.Println("Sub len:", len(sub)) // 2
    fmt.Println("Sub cap:", cap(sub)) // 4 (from index 1 to end of underlying array)

    // Sub shares the same underlying array!
    sub[0] = 999
    fmt.Println("s after modifying sub:", s) // [10 999 30 40 50]
}
```

### Creating Slices

```go
package main

import "fmt"

func main() {
    // 1. Slice literal
    s1 := []int{1, 2, 3}

    // 2. make([]T, length, capacity)
    s2 := make([]int, 3)      // [0 0 0], len=3, cap=3
    s3 := make([]int, 3, 10)  // [0 0 0], len=3, cap=10

    // 3. Slicing an array
    arr := [5]int{10, 20, 30, 40, 50}
    s4 := arr[1:4] // [20 30 40]

    // 4. Nil slice — has no underlying array
    var s5 []int // nil slice: len=0, cap=0, s5 == nil

    // 5. Empty slice — has an underlying array (of size 0)
    s6 := []int{} // empty slice: len=0, cap=0, s6 != nil

    fmt.Println(s1, s2, s3, s4)
    fmt.Println(s5 == nil) // true
    fmt.Println(s6 == nil) // false
}
```

### append() — Growth Strategy

When `append` exceeds the slice's capacity, Go allocates a new, larger backing array and copies existing elements over. The old and new slices then point to **different** arrays. The growth factor is roughly 2x for small slices, gradually decreasing for larger ones (Go 1.21+).

```
┌──────────────────────────────────────────────────────────┐
│             append() Reallocation                        │
│                                                          │
│  s := make([]int, 3, 3)   →  s = [1, 2, 3]              │
│                                                          │
│  Before append(s, 4):  len=3, cap=3                      │
│  ┌─────────┐     ┌───┬───┬───┐                           │
│  │ ptr ─────────► │ 1 │ 2 │ 3 │  ← array A (full!)     │
│  │ len: 3  │     └───┴───┴───┘                           │
│  │ cap: 3  │                                             │
│  └─────────┘                                             │
│                                                          │
│  After s = append(s, 4):  len=4, cap=6                   │
│  ┌─────────┐     ┌───┬───┬───┬───┬───┬───┐               │
│  │ ptr ─────────► │ 1 │ 2 │ 3 │ 4 │ 0 │ 0 │ ← NEW array B│
│  │ len: 4  │     └───┴───┴───┴───┴───┴───┘               │
│  │ cap: 6  │     cap doubled (3 → 6)                     │
│  └─────────┘                                             │
│                                                          │
│  old slice still points to array A (independent now)     │
│                                                          │
│  ⚠ Always use: s = append(s, ...) — reassign the result!│
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    s := make([]int, 0, 3)
    fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)

    // Append within capacity — no reallocation
    s = append(s, 1, 2, 3)
    fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
    // len=3 cap=3 [1 2 3]

    // Append beyond capacity — new underlying array allocated!
    old := s
    s = append(s, 4)
    fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
    // len=4 cap=6 [1 2 3 4] — capacity doubled

    // old and s now point to DIFFERENT arrays
    old[0] = 999
    fmt.Println("old:", old) // [999 2 3]
    fmt.Println("s:", s)     // [1 2 3 4] — not affected!

    // Append slice to slice with ...
    a := []int{1, 2}
    b := []int{3, 4, 5}
    a = append(a, b...)
    fmt.Println(a) // [1 2 3 4 5]
}
```

### copy()

```go
package main

import "fmt"

func main() {
    src := []int{1, 2, 3, 4, 5}
    dst := make([]int, 3)

    // copy returns min(len(dst), len(src))
    n := copy(dst, src)
    fmt.Println(n, dst) // 3 [1 2 3]

    // dst and src are independent — no shared memory
    dst[0] = 999
    fmt.Println(src) // [1 2 3 4 5] — unchanged
}
```

### Nil Slice vs Empty Slice

These look similar but behave differently in specific contexts (especially JSON marshaling):

```
┌──────────────────────────────────────────────────────────┐
│            Nil Slice vs Empty Slice                      │
│                                                          │
│  var nilSlice []int       │  emptySlice := []int{}       │
│  ┌─────────────┐          │  ┌─────────────┐             │
│  │ ptr: nil    │          │  │ ptr: 0xaddr │→ (empty)    │
│  │ len: 0     │          │  │ len: 0      │             │
│  │ cap: 0     │          │  │ cap: 0      │             │
│  └─────────────┘          │  └─────────────┘             │
│  nilSlice == nil → true   │  emptySlice == nil → false   │
│  json: null               │  json: []                    │
│                           │                              │
│  Both: len=0, cap=0, safe to append, range works         │
│                                                          │
│  ⚠ Use nil slice as default (var s []int)               │
│  ⚠ Use empty slice when you need JSON "[]" not "null"   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "fmt"
)

func main() {
    var nilSlice []int    // nil — no underlying array
    emptySlice := []int{} // empty — has an (empty) underlying array

    fmt.Println(nilSlice == nil)  // true
    fmt.Println(emptySlice == nil) // false

    // Both behave the same with len, cap, range, append
    fmt.Println(len(nilSlice), len(emptySlice))   // 0 0
    fmt.Println(cap(nilSlice), cap(emptySlice))   // 0 0

    nilSlice = append(nilSlice, 1) // works fine!

    // But they differ in JSON marshaling!
    j1, _ := json.Marshal(nilSlice)   // "null"  (after append, it's [1])
    j2, _ := json.Marshal(emptySlice) // "[]"

    // For fresh nil vs empty:
    var fresh []int
    j3, _ := json.Marshal(fresh)
    j4, _ := json.Marshal([]int{})

    fmt.Println(string(j1)) // [1]
    fmt.Println(string(j2)) // []
    fmt.Println(string(j3)) // null
    fmt.Println(string(j4)) // []
}
```

### Slice Tricks

```go
package main

import "fmt"

func main() {
    // Delete element at index i
    s := []int{1, 2, 3, 4, 5}
    i := 2 // delete element at index 2 (value 3)
    s = append(s[:i], s[i+1:]...)
    fmt.Println("After delete:", s) // [1 2 4 5]

    // Insert at index i
    s = []int{1, 2, 4, 5}
    i = 2
    s = append(s[:i], append([]int{3}, s[i:]...)...)
    fmt.Println("After insert:", s) // [1 2 3 4 5]

    // Filter (keep even numbers)
    nums := []int{1, 2, 3, 4, 5, 6}
    evens := nums[:0] // reuse underlying array
    for _, n := range nums {
        if n%2 == 0 {
            evens = append(evens, n)
        }
    }
    fmt.Println("Evens:", evens) // [2 4 6]

    // Reverse
    r := []int{1, 2, 3, 4, 5}
    for left, right := 0, len(r)-1; left < right; left, right = left+1, right-1 {
        r[left], r[right] = r[right], r[left]
    }
    fmt.Println("Reversed:", r) // [5 4 3 2 1]
}
```

### Full Slice Expression

The three-index slice `a[low:high:max]` controls the **capacity** of the resulting slice, preventing accidental overwrites of elements in the original backing array.

```
┌──────────────────────────────────────────────────────────┐
│           Full Slice Expression: a[low:high:max]         │
│                                                          │
│  s := []int{1, 2, 3, 4, 5}                              │
│                                                          │
│  Normal:  sub := s[1:3]      len=2, cap=4                │
│           ┌───┬───┬───┬───┐                              │
│           │ 2 │ 3 │ 4 │ 5 │  ← can append into s[3],s[4]│
│           └───┴───┴───┴───┘                              │
│           ◄─len─►◄──extra──►                             │
│                                                          │
│  Full:    sub := s[1:3:3]    len=2, cap=2                │
│           ┌───┬───┐                                      │
│           │ 2 │ 3 │  ← no room, append creates new array│
│           └───┴───┘                                      │
│           ◄─len─►                                        │
│           cap = max - low = 3 - 1 = 2                    │
│                                                          │
│  ⚠ Use full slice to protect original array from append │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    s := []int{1, 2, 3, 4, 5}

    // Normal slice: sub can see beyond its length (up to original capacity)
    sub1 := s[1:3]
    fmt.Printf("sub1: %v, len=%d, cap=%d\n", sub1, len(sub1), cap(sub1))
    // sub1: [2 3], len=2, cap=4

    // Full slice expression: a[low:high:max] — limits capacity
    sub2 := s[1:3:3] // capacity limited to high-low = 3-1 = 2
    fmt.Printf("sub2: %v, len=%d, cap=%d\n", sub2, len(sub2), cap(sub2))
    // sub2: [2 3], len=2, cap=2

    // append to sub2 will create a NEW array (can't overwrite s[3])
    sub2 = append(sub2, 99)
    fmt.Println("s after append to sub2:", s) // [1 2 3 4 5] — safe!
}
```

---

## Maps

Maps are Go's built-in hash table type. They provide O(1) average-case lookups, insertions, and deletions. Maps are **not safe for concurrent use** — use `sync.Map` or a mutex for concurrent access.

```
┌──────────────────────────────────────────────────────────┐
│                  Map Internal Structure                  │
│                                                          │
│  m := map[string]int{"alice": 90, "bob": 85}            │
│                                                          │
│  map header (hmap)        Buckets (arrays of key-value): │
│  ┌──────────────┐         ┌────────────────────────┐     │
│  │ count: 2     │         │ bucket 0:              │     │
│  │ buckets ─────────────► │   "alice" → 90         │     │
│  │ hash seed    │         │   "bob"   → 85         │     │
│  │ bucket count │         ├────────────────────────┤     │
│  └──────────────┘         │ bucket 1: (empty)      │     │
│                           ├────────────────────────┤     │
│  Assigning map copies     │ bucket 2: (empty)      │     │
│  the header pointer →     └────────────────────────┘     │
│  both variables share                                    │
│  the same hash table.     Each bucket holds ~8 entries.  │
│                           Grows automatically.           │
│                                                          │
│  ⚠ var m map[K]V → nil map (read OK, write PANICS!)    │
│  ⚠ Iteration order is RANDOMIZED by design             │
│  ⚠ NOT safe for concurrent read+write                  │
└──────────────────────────────────────────────────────────┘
```

### Creation and Operations

```go
package main

import "fmt"

func main() {
    // Create with make
    m := make(map[string]int)
    m["alice"] = 90
    m["bob"] = 85

    // Map literal
    scores := map[string]int{
        "alice": 90,
        "bob":   85,
        "carol": 92,
    }

    // Access
    fmt.Println(scores["alice"]) // 90
    fmt.Println(scores["dave"])  // 0 (zero value for missing key)

    // Comma-ok idiom — distinguish missing key from zero value
    val, ok := scores["dave"]
    if ok {
        fmt.Println("Dave's score:", val)
    } else {
        fmt.Println("Dave not found") // Dave not found
    }

    // Delete
    delete(scores, "bob")
    fmt.Println(scores) // map[alice:90 carol:92]

    // Iteration — order is NOT guaranteed (randomized!)
    for name, score := range scores {
        fmt.Printf("%s: %d\n", name, score)
    }

    // Length
    fmt.Println("Size:", len(scores)) // 2
}
```

### Nil Map Behavior

```go
package main

import "fmt"

func main() {
    var m map[string]int // nil map

    // Reading from nil map is safe — returns zero value
    fmt.Println(m["key"]) // 0
    fmt.Println(len(m))   // 0

    // Writing to nil map PANICS!
    // m["key"] = 1 // panic: assignment to entry in nil map

    // Always initialize before writing
    m = make(map[string]int)
    m["key"] = 1 // works now
    fmt.Println(m) // map[key:1]
}
```

### Maps Package (Go 1.21+)

```go
package main

import (
    "fmt"
    "maps"
    "slices"
)

func main() {
    original := map[string]int{
        "a": 1, "b": 2, "c": 3,
    }

    // Clone a map
    cloned := maps.Clone(original)
    cloned["d"] = 4
    fmt.Println("Original:", original) // No "d"
    fmt.Println("Cloned:", cloned)     // Has "d"

    // Collect keys (use slices.Sorted for sorted output)
    keys := slices.Sorted(maps.Keys(original))
    fmt.Println("Keys:", keys) // [a b c]

    // Collect values
    vals := slices.Sorted(maps.Values(original))
    fmt.Println("Values:", vals) // [1 2 3]
}
```

---

## Structs

Structs are Go's way of defining custom data types that group related fields. Structs are **value types** — assigning or passing a struct creates a copy. Go uses struct embedding (composition) instead of inheritance.

```
┌──────────────────────────────────────────────────────────┐
│              Struct Embedding (Composition)              │
│                                                          │
│  type Address struct {                                   │
│      City, Country string                                │
│  }                                                       │
│                                                          │
│  type Employee struct {                                  │
│      Name string                                         │
│      Address         ← embedded (anonymous field)        │
│  }                                                       │
│                                                          │
│  Memory layout:            Field promotion:              │
│  ┌───────────────────┐     emp.City         ← promoted  │
│  │ Name: "Alice"     │     emp.Address.City ← explicit  │
│  ├───────────────────┤     (both access the same field)  │
│  │ Address:          │                                   │
│  │   City: "SF"      │     If Employee also had a City  │
│  │   Country: "USA"  │     field, promotion is shadowed: │
│  └───────────────────┘     emp.City → Employee.City      │
│                            emp.Address.City → Address.City│
└──────────────────────────────────────────────────────────┘
```

### Struct Definition and Initialization

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
    Email string
}

func main() {
    // Named field initialization (preferred)
    p1 := Person{
        Name:  "Alice",
        Age:   30,
        Email: "alice@example.com",
    }

    // Positional initialization (fragile, avoid)
    p2 := Person{"Bob", 25, "bob@example.com"}

    // Partial initialization — unset fields get zero values
    p3 := Person{Name: "Carol"}

    // Zero-valued struct
    var p4 Person

    fmt.Println(p1)  // {Alice 30 alice@example.com}
    fmt.Println(p2)  // {Bob 25 bob@example.com}
    fmt.Println(p3)  // {Carol 0 }
    fmt.Println(p4)  // { 0 }

    // Access and modify fields
    p1.Age = 31
    fmt.Println(p1.Name, "is", p1.Age) // Alice is 31
}
```

### Anonymous / Embedded Fields (Struct Embedding)

```go
package main

import "fmt"

type Address struct {
    City    string
    Country string
}

type Employee struct {
    Name    string
    Address // Embedded (anonymous) field — gets promoted fields
}

func main() {
    emp := Employee{
        Name: "Alice",
        Address: Address{
            City:    "San Francisco",
            Country: "USA",
        },
    }

    // Access promoted fields directly
    fmt.Println(emp.City)    // San Francisco (promoted from Address)
    fmt.Println(emp.Country) // USA

    // Or access via the embedded field explicitly
    fmt.Println(emp.Address.City) // San Francisco

    // Embedding is Go's way of composition over inheritance
}
```

### Struct Tags

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID        int    `json:"id" db:"user_id"`
    FirstName string `json:"first_name" db:"first_name"`
    LastName  string `json:"last_name" db:"last_name"`
    Password  string `json:"-"` // "-" means omit from JSON
    Email     string `json:"email,omitempty"` // omit if empty
}

func main() {
    user := User{
        ID:        1,
        FirstName: "Alice",
        LastName:  "Smith",
        Password:  "secret123",
    }

    data, _ := json.Marshal(user)
    fmt.Println(string(data))
    // {"id":1,"first_name":"Alice","last_name":"Smith"}
    // Note: Password is omitted (-), Email is omitted (empty + omitempty)

    // Unmarshal from JSON
    jsonStr := `{"id":2,"first_name":"Bob","last_name":"Jones","email":"bob@test.com"}`
    var user2 User
    json.Unmarshal([]byte(jsonStr), &user2)
    fmt.Printf("%+v\n", user2)
    // {ID:2 FirstName:Bob LastName:Jones Password: Email:bob@test.com}
}
```

### Struct Comparison

```go
package main

import "fmt"

type Point struct {
    X, Y int
}

type Shape struct {
    Name  string
    Sides int
}

type Container struct {
    Items []int // slices are NOT comparable
}

func main() {
    // Structs are comparable if ALL fields are comparable
    p1 := Point{1, 2}
    p2 := Point{1, 2}
    p3 := Point{3, 4}

    fmt.Println(p1 == p2) // true
    fmt.Println(p1 == p3) // false

    s1 := Shape{"Triangle", 3}
    s2 := Shape{"Triangle", 3}
    fmt.Println(s1 == s2) // true

    // Structs with non-comparable fields (slice, map, func) CANNOT use ==
    // c1 := Container{[]int{1, 2}}
    // c2 := Container{[]int{1, 2}}
    // fmt.Println(c1 == c2) // COMPILE ERROR: invalid operation
}
```

---

## Interview Questions

1. **What is the difference between an array and a slice in Go?**
   - Arrays have a fixed size that's part of the type (`[3]int` ≠ `[4]int`). They are values (copied on assignment). Slices are dynamic, reference a backing array via a header (pointer, length, capacity), and are the preferred type.

2. **What happens when you `append` to a slice that exceeds its capacity?**
   - Go allocates a new, larger underlying array, copies existing elements, appends the new elements, and returns a new slice header pointing to the new array. The old array (if not referenced) is garbage collected.

3. **What is the difference between a nil slice and an empty slice?**
   - Nil slice (`var s []int`): pointer is nil, len=0, cap=0. Empty slice (`s := []int{}`): pointer is non-nil (points to zero-length array), len=0, cap=0. Both have `len(s) == 0`. `json.Marshal` encodes nil as `null`, empty as `[]`.

4. **What is the full slice expression `a[low:high:max]`?**
   - The third index `max` sets the capacity of the resulting slice to `max - low`. This prevents `append` from overwriting elements in the original slice's backing array.

5. **What happens when you read a key that doesn't exist in a map?**
   - It returns the zero value for the value type. Use the comma-ok idiom to distinguish: `val, ok := m[key]`.

6. **What happens when you write to a nil map?**
   - It panics at runtime. You must initialize maps with `make(map[K]V)` or a map literal before writing.

7. **Is map iteration order guaranteed in Go?**
   - No. Map iteration order is intentionally randomized. Never rely on a specific order.

8. **Can you compare structs in Go?**
   - Only if all fields are comparable types. Structs with slice, map, or function fields cannot use `==`. Use `reflect.DeepEqual` for deep comparison.

9. **What is struct embedding in Go?**
   - Embedding places one struct inside another without a field name. The embedded struct's fields and methods are "promoted" and accessible directly. It's Go's composition mechanism (no inheritance).

10. **What are struct tags used for?**
    - Struct tags are metadata attached to fields: `json:"name,omitempty"`. They are read at runtime via reflection. Common uses: JSON serialization, database column mapping, validation.
