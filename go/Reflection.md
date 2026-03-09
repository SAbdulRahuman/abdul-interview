# Chapter 16 — Reflection

Reflection lets you inspect and manipulate types and values at **runtime**. It's how `encoding/json`, `fmt.Printf`, and ORMs work under the hood. The `reflect` package provides `TypeOf` (what type?) and `ValueOf` (what value?).

```
┌──────────────────────────────────────────────────────────┐
│              Reflection Overview                        │
│                                                          │
│  interface{} value                                       │
│       │                                                  │
│       ▼                                                  │
│  reflect.TypeOf(v)  ──► reflect.Type                    │
│  │                      ├── Name()    → "User"          │
│  │                      ├── Kind()    → reflect.Struct  │
│  │                      ├── NumField() → 3              │
│  │                      └── Field(i)  → StructField     │
│  │                                                       │
│  reflect.ValueOf(v) ──► reflect.Value                   │
│                         ├── Int()     → 42              │
│                         ├── String()  → "hello"         │
│                         ├── CanSet()  → true/false      │
│                         └── Set(v)    → modifies value  │
│                                                          │
│  Kind vs Type:                                          │
│  type UserID int                                        │
│  • Type = "main.UserID" (specific Go type)              │
│  • Kind = reflect.Int   (underlying category)           │
│                                                          │
│  ⚠ Reflection is slow — avoid in hot paths             │
│  ⚠ To modify via reflect: pass pointer + call Elem()   │
│  ⚠ Only exported fields are settable via reflection     │
└──────────────────────────────────────────────────────────┘
```

## reflect.TypeOf and reflect.ValueOf

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x int = 42
    var s string = "hello"
    var f float64 = 3.14

    // reflect.TypeOf — returns the type
    fmt.Println(reflect.TypeOf(x)) // int
    fmt.Println(reflect.TypeOf(s)) // string
    fmt.Println(reflect.TypeOf(f)) // float64

    // reflect.ValueOf — returns the value wrapped in reflect.Value
    fmt.Println(reflect.ValueOf(x)) // 42
    fmt.Println(reflect.ValueOf(s)) // hello
    fmt.Println(reflect.ValueOf(f)) // 3.14

    // Get the underlying value back
    val := reflect.ValueOf(x)
    fmt.Println(val.Int())        // 42
    fmt.Println(val.Interface())  // 42 (as interface{})
}
```

---

## Kind vs Type

```go
package main

import (
    "fmt"
    "reflect"
)

type UserID int
type User struct {
    Name string
    Age  int
}

func main() {
    var id UserID = 42
    u := User{Name: "Alice", Age: 30}

    // Type is the specific Go type
    fmt.Println("Type of id:", reflect.TypeOf(id))   // main.UserID
    fmt.Println("Type of u:", reflect.TypeOf(u))     // main.User

    // Kind is the underlying category
    fmt.Println("Kind of id:", reflect.TypeOf(id).Kind())  // int
    fmt.Println("Kind of u:", reflect.TypeOf(u).Kind())    // struct

    // Common kinds:
    // reflect.Int, reflect.String, reflect.Bool, reflect.Float64
    // reflect.Slice, reflect.Map, reflect.Struct, reflect.Ptr
    // reflect.Chan, reflect.Func, reflect.Interface

    vals := []any{42, "hello", true, []int{1, 2}, map[string]int{"a": 1}}
    for _, v := range vals {
        t := reflect.TypeOf(v)
        fmt.Printf("Value: %-12v Type: %-15v Kind: %v\n", v, t, t.Kind())
    }
}
```

---

## Settability

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    x := 42

    // reflect.ValueOf(x) — NOT settable (it's a copy)
    v := reflect.ValueOf(x)
    fmt.Println("CanSet:", v.CanSet()) // false

    // Must pass POINTER to make it settable
    v = reflect.ValueOf(&x).Elem() // Elem() dereferences the pointer
    fmt.Println("CanSet:", v.CanSet()) // true

    // Now we can modify the original variable
    v.SetInt(100)
    fmt.Println("x after SetInt:", x) // 100

    // Works with strings too
    s := "hello"
    sv := reflect.ValueOf(&s).Elem()
    sv.SetString("world")
    fmt.Println("s:", s) // world
}
```

---

## Iterating Struct Fields and Reading Tags

```go
package main

import (
    "fmt"
    "reflect"
)

type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"email"`
    Age   int    `json:"age" validate:"min=0,max=150"`
}

func inspectStruct(v any) {
    t := reflect.TypeOf(v)
    val := reflect.ValueOf(v)

    fmt.Printf("Struct: %s (Kind: %s)\n", t.Name(), t.Kind())
    fmt.Printf("Fields: %d\n\n", t.NumField())

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        value := val.Field(i)

        fmt.Printf("  Field: %s\n", field.Name)
        fmt.Printf("    Type:     %s\n", field.Type)
        fmt.Printf("    Value:    %v\n", value)
        fmt.Printf("    JSON tag: %s\n", field.Tag.Get("json"))
        fmt.Printf("    Validate: %s\n", field.Tag.Get("validate"))
        fmt.Println()
    }
}

func main() {
    u := User{Name: "Alice", Email: "alice@test.com", Age: 30}
    inspectStruct(u)

    // Access field by name
    val := reflect.ValueOf(u)
    nameField := val.FieldByName("Name")
    fmt.Println("Name field:", nameField.String()) // Alice

    // Check if field exists
    missing := val.FieldByName("Phone")
    fmt.Println("Phone valid:", missing.IsValid()) // false
}
```

---

## Creating Values with Reflection

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    // Create new value with reflect.New
    t := reflect.TypeOf(0) // int type
    v := reflect.New(t)    // *int
    v.Elem().SetInt(42)
    fmt.Println("Created int:", v.Elem().Int()) // 42

    // Create a slice
    sliceType := reflect.SliceOf(reflect.TypeOf(0))
    slice := reflect.MakeSlice(sliceType, 0, 5)
    slice = reflect.Append(slice, reflect.ValueOf(1))
    slice = reflect.Append(slice, reflect.ValueOf(2))
    slice = reflect.Append(slice, reflect.ValueOf(3))
    fmt.Println("Created slice:", slice.Interface()) // [1 2 3]

    // Create a map
    mapType := reflect.MapOf(reflect.TypeOf(""), reflect.TypeOf(0))
    m := reflect.MakeMap(mapType)
    m.SetMapIndex(reflect.ValueOf("age"), reflect.ValueOf(30))
    m.SetMapIndex(reflect.ValueOf("score"), reflect.ValueOf(95))
    fmt.Println("Created map:", m.Interface()) // map[age:30 score:95]
}
```

---

## Calling Functions with Reflection

```go
package main

import (
    "fmt"
    "reflect"
)

func add(a, b int) int {
    return a + b
}

func greet(name string) string {
    return "Hello, " + name + "!"
}

func main() {
    // Call function via reflection
    addFn := reflect.ValueOf(add)
    args := []reflect.Value{
        reflect.ValueOf(3),
        reflect.ValueOf(5),
    }
    result := addFn.Call(args)
    fmt.Println("add(3, 5) =", result[0].Int()) // 8

    // Another function
    greetFn := reflect.ValueOf(greet)
    result2 := greetFn.Call([]reflect.Value{reflect.ValueOf("Go")})
    fmt.Println(result2[0].String()) // Hello, Go!
}
```

---

## Laws of Reflection and reflect.DeepEqual

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    // Law 1: Reflection goes from interface value to reflection object
    var x float64 = 3.14
    v := reflect.ValueOf(x) // interface{} → reflect.Value
    t := reflect.TypeOf(x)  // interface{} → reflect.Type
    fmt.Println(v, t)       // 3.14 float64

    // Law 2: Reflection goes from reflection object to interface value
    y := v.Interface().(float64) // reflect.Value → interface{} → float64
    fmt.Println(y)               // 3.14

    // Law 3: To modify, value must be settable (pass pointer)
    z := 10
    vz := reflect.ValueOf(&z).Elem()
    vz.SetInt(20)
    fmt.Println(z) // 20

    // reflect.DeepEqual — recursive comparison (useful in tests)
    a := []int{1, 2, 3}
    b := []int{1, 2, 3}
    c := []int{1, 2, 4}

    fmt.Println(reflect.DeepEqual(a, b)) // true
    fmt.Println(reflect.DeepEqual(a, c)) // false

    // Works with maps, structs, nested types
    m1 := map[string][]int{"a": {1, 2}, "b": {3, 4}}
    m2 := map[string][]int{"a": {1, 2}, "b": {3, 4}}
    m3 := map[string][]int{"a": {1, 2}, "b": {3, 5}}

    fmt.Println(reflect.DeepEqual(m1, m2)) // true
    fmt.Println(reflect.DeepEqual(m1, m3)) // false
}
```

---

## Performance and Use Cases

```go
package main

import "fmt"

func main() {
    // Reflection is SLOW — avoid in hot paths (tight loops, request handlers)
    // Benchmark: reflection calls are 10-100x slower than direct calls

    // Use cases (where the flexibility is worth the cost):
    fmt.Println("Reflection use cases:")
    fmt.Println("  - JSON/XML serialization (encoding/json)")
    fmt.Println("  - ORM frameworks (GORM, sqlx)")
    fmt.Println("  - Dependency injection (wire, fx)")
    fmt.Println("  - Test assertions (reflect.DeepEqual)")
    fmt.Println("  - Configuration loading (viper)")
    fmt.Println("  - Validation frameworks")

    // Prefer generics (Go 1.18+) over reflection when possible
    // Generics are checked at compile time, no runtime overhead
}
```

---

## Interview Questions

1. **What is reflection in Go?**
   - Reflection allows inspecting and manipulating types and values at runtime via the `reflect` package. It enables examining struct fields, calling functions dynamically, and reading struct tags.

2. **What are the three laws of reflection?**
   - (1) Reflection goes from interface value to reflection object (`reflect.ValueOf`). (2) Reflection goes back from reflection object to interface value (`.Interface()`). (3) To modify a reflection object, the value must be settable (pass pointer).

3. **What is the difference between `reflect.Type` and `reflect.Value`?**
   - `Type` represents the Go type (name, kind, methods, fields). `Value` represents the actual data and allows reading/modifying values. Get them via `reflect.TypeOf(x)` and `reflect.ValueOf(x)`.

4. **What is `Kind` vs `Type` in reflection?**
   - `Type` is the specific type name (e.g., `main.Person`). `Kind` is the underlying category (e.g., `reflect.Struct`, `reflect.Ptr`, `reflect.Slice`). A custom `type MyInt int` has Kind `Int` but Type `MyInt`.

5. **How do you make a reflected value settable?**
   - Pass a pointer: `v := reflect.ValueOf(&x).Elem()`. Now `v.CanSet()` returns true. Without the pointer, the reflection operates on a copy and is not settable.

6. **How do you read struct tags via reflection?**
   - `field.Tag.Get("json")` returns the tag value. Get the field via `t.Field(i)` or `t.FieldByName("Name")` where `t` is a `reflect.Type` of a struct.

7. **What is `reflect.DeepEqual` used for?**
   - It performs recursive deep comparison of two values, including slices, maps, and nested structs. Commonly used in tests to compare complex data structures.

8. **Why is reflection slow?**
   - Reflection bypasses compile-time optimizations, involves interface boxing/unboxing, runtime type checks, and cannot be inlined. It can be 10-100x slower than direct operations.

9. **When should you use reflection?**
   - Serialization/deserialization (JSON, ORM), dependency injection, testing utilities, code generation tools. Avoid in hot paths. Prefer generics (Go 1.18+) when possible.

10. **Can you call a function dynamically using reflection?**
    - Yes. `reflect.Value.Call(args)` where args is `[]reflect.Value`. The function must be wrapped in a `reflect.Value` first via `reflect.ValueOf(fn)`.
