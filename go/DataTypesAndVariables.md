# Chapter 2 — Data Types & Variables

Go is a statically typed language — every variable has a fixed type determined at compile time. Go's type system is intentionally simple: no implicit conversions, no type hierarchy, and no union types.

```
┌──────────────────────────────────────────────────────────────┐
│                    Go Type Classification                    │
│                                                              │
│  Basic Types              Composite Types     Other Types    │
│  ├── bool                 ├── array [N]T      ├── pointer *T │
│  ├── string               ├── slice []T       ├── function   │
│  ├── int, int8..int64     ├── map[K]V         ├── interface  │
│  ├── uint, uint8..uint64  ├── struct           └── channel    │
│  ├── float32, float64     │                                  │
│  ├── complex64, complex128│    Reference-like:               │
│  ├── byte (= uint8)       │    slice, map, channel, pointer  │
│  └── rune (= int32)       │    (contain internal pointers)   │
│                            │                                  │
│  uintptr                   │    Value types:                  │
│  (holds pointer as int)    │    array, struct, basic types    │
│                            │    (copied on assignment)        │
└──────────────────────────────────────────────────────────────┘
```

## Basic Integer Types

Go has specific integer types with explicit sizes. Using sized types (`int32`, `int64`) gives you control over memory layout, which matters for binary protocols and interop. The default `int` is platform-dependent (64-bit on modern systems).

```
┌──────────────────────────────────────────────────────────┐
│              Integer Types — Memory Sizes                │
│                                                          │
│  Type     Bytes   Bits    Range                          │
│  ───────  ─────   ────    ────────────────────           │
│  int8     1       8       -128 to 127                    │
│  int16    2       16      -32,768 to 32,767              │
│  int32    4       32      -2.1B to 2.1B                  │
│  int64    8       64      -9.2E18 to 9.2E18              │
│                                                          │
│  uint8    1       8       0 to 255                        │
│  uint16   2       16      0 to 65,535                    │
│  uint32   4       32      0 to 4.2B                      │
│  uint64   8       64      0 to 18.4E18                   │
│                                                          │
│  int      4 or 8  32/64   platform-dependent             │
│  uint     4 or 8  32/64   platform-dependent             │
│  uintptr  4 or 8  32/64   big enough to hold a pointer   │
│                                                          │
│  byte  = uint8    (alias for raw data)                   │
│  rune  = int32    (alias for Unicode code points)        │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // Signed integers
    var a int     // Platform-dependent: 32 or 64 bits
    var b int8    // -128 to 127
    var c int16   // -32768 to 32767
    var d int32   // -2 billion to 2 billion
    var e int64   // -9 quintillion to 9 quintillion

    // Unsigned integers
    var f uint    // Platform-dependent: 32 or 64 bits
    var g uint8   // 0 to 255
    var h uint16  // 0 to 65535
    var i uint32  // 0 to ~4 billion
    var j uint64  // 0 to ~18 quintillion

    var k uintptr // Large enough to hold a pointer value

    fmt.Println(a, b, c, d, e, f, g, h, i, j, k)
    // Output: 0 0 0 0 0 0 0 0 0 0 0
}
```

---

## Floating Point Types

```go
package main

import "fmt"

func main() {
    var f32 float32 = 3.14          // 32-bit IEEE 754
    var f64 float64 = 3.141592653589793 // 64-bit IEEE 754 (preferred)

    fmt.Println(f32)  // 3.14
    fmt.Println(f64)  // 3.141592653589793

    // Float arithmetic
    result := 0.1 + 0.2
    fmt.Println(result)             // 0.30000000000000004 (floating point imprecision)
    fmt.Printf("%.2f\n", result)    // 0.30 (formatted output)
}
```

---

## Complex Types

```go
package main

import "fmt"

func main() {
    var c64 complex64 = complex(float32(1), float32(2))   // 1+2i
    var c128 complex128 = complex(3.0, 4.0)               // 3+4i

    fmt.Println(c64)              // (1+2i)
    fmt.Println(c128)             // (3+4i)
    fmt.Println(real(c128))       // 3 — real part
    fmt.Println(imag(c128))       // 4 — imaginary part

    // Complex arithmetic
    sum := c128 + complex(1, 1)
    fmt.Println(sum)              // (4+5i)
}
```

---

## byte, rune, string, bool

Understanding the difference between `byte` and `rune` is critical for working with text in Go. A Go string is a **read-only slice of bytes** (UTF-8 encoded), not a slice of characters.

```
┌──────────────────────────────────────────────────────────┐
│            String "Hello, 世界" in Memory                │
│                                                          │
│  Index:  0  1  2  3  4  5  6  7  8  9  10 11 12         │
│  Byte:   H  e  l  l  o  ,     [世=3bytes] [界=3bytes]   │
│          48 65 6C 6C 6F 2C 20 E4B896     E7958C         │
│                                                          │
│  len("Hello, 世界") = 13   ← counts BYTES, not chars    │
│  []rune("Hello, 世界") = 9 runes  ← actual characters   │
│                                                          │
│  byte = uint8  → 1 byte  → ASCII character               │
│  rune = int32  → 4 bytes → Unicode code point (any char) │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // byte — alias for uint8, represents a single ASCII character
    var b byte = 'A'
    fmt.Println(b)       // 65 (ASCII value)
    fmt.Printf("%c\n", b) // A

    // rune — alias for int32, represents a Unicode code point
    var r rune = '世'
    fmt.Println(r)        // 19990 (Unicode code point)
    fmt.Printf("%c\n", r) // 世

    // string — immutable sequence of bytes (UTF-8 encoded)
    var s string = "Hello, 世界"
    fmt.Println(s)        // Hello, 世界
    fmt.Println(len(s))   // 13 (byte count, NOT character count!)

    // bool — true or false
    var isReady bool = true
    var isDone bool       // zero value is false
    fmt.Println(isReady, isDone) // true false
}
```

---

## Zero Values

Every type in Go has a **zero value** — the default value assigned when you don't explicitly initialize. This is a key design choice: Go guarantees no uninitialized variables.

```
┌──────────────────────────────────────────────────────────┐
│                    Go Zero Values                        │
│                                                          │
│  Type Category     Zero Value      Example               │
│  ──────────────    ──────────      ──────────────────    │
│  bool              false           var b bool    → false │
│  integers          0               var i int     → 0    │
│  floats            0.0             var f float64 → 0    │
│  complex           (0+0i)          var c complex128      │
│  string            "" (empty)      var s string  → ""   │
│  pointer           nil             var p *int    → nil  │
│  slice             nil             var sl []int  → nil  │
│  map               nil             var m map[K]V → nil  │
│  channel           nil             var ch chan T → nil   │
│  interface         nil             var i error   → nil  │
│  function          nil             var fn func() → nil  │
│  struct            all fields zero var s MyStruct        │
│  array             all elems zero  var a [3]int → [0,0,0]│
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    var i int         // 0
    var f float64     // 0
    var b bool        // false
    var s string      // "" (empty string)
    var p *int        // nil
    var sl []int      // nil
    var m map[string]int // nil
    var ch chan int    // nil
    var fn func()     // nil
    var iface error   // nil

    fmt.Println(i)     // 0
    fmt.Println(f)     // 0
    fmt.Println(b)     // false
    fmt.Println(s)     // (empty string)
    fmt.Println(p)     // <nil>
    fmt.Println(sl)    // []
    fmt.Println(m)     // map[]
    fmt.Println(ch)    // <nil>
    fmt.Println(fn)    // <nil>
    fmt.Println(iface) // <nil>
}
```

---

## Variable Declaration

```go
package main

import "fmt"

func main() {
    // 1. var declaration with type
    var name string = "Alice"

    // 2. var with type inference
    var age = 30

    // 3. Short declaration (most common, only inside functions)
    city := "New York"

    // 4. Multiple variable declaration
    var x, y, z int = 1, 2, 3

    // 5. Multiple short declaration
    a, b := "hello", 42

    // 6. Block declaration
    var (
        firstName = "Bob"
        lastName  = "Smith"
        score     = 95
    )

    fmt.Println(name, age, city)
    fmt.Println(x, y, z)
    fmt.Println(a, b)
    fmt.Println(firstName, lastName, score)

    // 7. Multiple assignment (swap values)
    x, y = y, x
    fmt.Println("After swap:", x, y) // After swap: 2 1
}
```

---

## Constants (const, iota)

Constants in Go are computed at **compile time**. They must be assignable from constant expressions (no function calls, except `len` of string/array literals). The `iota` keyword is a compile-time constant generator that auto-increments within a `const` block.

```
┌──────────────────────────────────────────────────────────┐
│                  iota Progression                        │
│                                                          │
│  const (                                                 │
│      A = iota     // iota=0 → A=0                        │
│      B            // iota=1 → B=1  (expression repeats) │
│      C            // iota=2 → C=2                        │
│      _            // iota=3 → skipped                    │
│      E            // iota=4 → E=4                        │
│  )                                                       │
│                                                          │
│  const (          // iota resets to 0 in new block       │
│      X = iota+10  // iota=0 → X=10                       │
│      Y            // iota=1 → Y=11                       │
│      Z            // iota=2 → Z=12                       │
│  )                                                       │
│                                                          │
│  Bit flags with iota:                                    │
│  const (                                                 │
│      Read  = 1 << iota  // 1 << 0 = 0001 = 1            │
│      Write              // 1 << 1 = 0010 = 2            │
│      Exec               // 1 << 2 = 0100 = 4            │
│  )                                                       │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // Simple constants
    const pi = 3.14159
    const greeting = "Hello"

    // Block constants
    const (
        StatusOK    = 200
        StatusNotFound = 404
        StatusError = 500
    )

    // iota — auto-incrementing constant generator
    const (
        Sunday    = iota // 0
        Monday           // 1
        Tuesday          // 2
        Wednesday        // 3
        Thursday         // 4
        Friday           // 5
        Saturday         // 6
    )

    fmt.Println(Sunday, Monday, Saturday) // 0 1 6

    // iota with expressions
    const (
        _  = iota             // 0 (skip with blank identifier)
        KB = 1 << (10 * iota) // 1 << 10 = 1024
        MB                    // 1 << 20 = 1048576
        GB                    // 1 << 30 = 1073741824
        TB                    // 1 << 40
    )

    fmt.Println(KB, MB, GB) // 1024 1048576 1073741824

    // iota for bit flags
    const (
        ReadPermission   = 1 << iota // 1  (001)
        WritePermission              // 2  (010)
        ExecutePermission            // 4  (100)
    )

    // Combine permissions with bitwise OR
    userPerms := ReadPermission | WritePermission
    fmt.Printf("User permissions: %03b\n", userPerms) // 011

    // Check permission with bitwise AND
    canRead := userPerms&ReadPermission != 0
    canExec := userPerms&ExecutePermission != 0
    fmt.Println("Can read:", canRead)   // true
    fmt.Println("Can execute:", canExec) // false
}
```

---

## Untyped Constants

Untyped constants in Go have no fixed type until used. They have much higher precision and adapt to context.

```go
package main

import "fmt"

func main() {
    // Untyped constant — adapts to whatever type context it's used in
    const x = 42       // untyped integer constant
    const y = 3.14     // untyped floating-point constant

    var i int = x      // x adapts to int
    var f float64 = x  // x adapts to float64 (becomes 42.0)
    var f2 float32 = y // y adapts to float32

    fmt.Println(i, f, f2) // 42 42 3.14

    // Typed constant — fixed type, no adaptation
    const typedX int = 42
    // var f3 float64 = typedX  // ERROR: cannot use typedX (int) as float64

    // High precision of untyped constants
    const huge = 1e1000  // This is fine as an untyped constant!
    // var h float64 = huge  // ERROR: overflow — only fails when assigned to a type
    const small = huge / 1e999
    fmt.Println(small) // 10
}
```

---

## Type Conversion

Go has **no implicit conversions**. All conversions must be explicit. This prevents subtle bugs from automatic widening/narrowing that occurs in C/Java.

```
┌──────────────────────────────────────────────────────────┐
│              Type Conversion Rules                       │
│                                                          │
│  int → float64    ✓  float64(myInt)    (safe)           │
│  float64 → int    ✓  int(myFloat)      (truncates!)    │
│  int → string     ⚠  string(65) = "A"  (Unicode, NOT "65")│
│  int → "65"       ✓  strconv.Itoa(65)  (use strconv!)  │
│  string → int     ✓  strconv.Atoi("65")                 │
│  int8 → int64     ✓  int64(myInt8)     (safe, widens)  │
│  int64 → int8     ⚠  int8(bigNum)      (overflows!)    │
│  []byte → string  ✓  string(byteSlice) (copies data)   │
│  string → []byte  ✓  []byte(myString)  (copies data)   │
│                                                          │
│  ⚠ Overflow wraps silently: int8(256) = 0               │
│  ⚠ Float→Int truncates: int(3.99) = 3  (NOT 4)         │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    // Integer conversions
    var i int = 42
    var f float64 = float64(i)    // int to float64
    var u uint = uint(f)          // float64 to uint

    fmt.Println(i, f, u) // 42 42 42

    // Truncation warning
    var bigFloat float64 = 3.99
    var truncated int = int(bigFloat)
    fmt.Println(truncated) // 3 (truncated, NOT rounded!)

    // String conversions
    var num int = 65
    s := string(num)       // Converts to Unicode character, NOT "65"!
    fmt.Println(s)         // A (Unicode code point 65)

    // For number-to-string, use strconv
    // import "strconv"
    // s = strconv.Itoa(num) // "65"

    // Between integer sizes
    var small int8 = 127
    var big int64 = int64(small)
    fmt.Println(big) // 127

    // Overflow wraps silently
    var overflow int8 = int8(256) // Wraps: 256 % 256 = 0
    fmt.Println(overflow)         // 0
}
```

---

## Type Alias vs Type Definition

These are two distinct mechanisms in Go that look similar but behave very differently:

```
┌──────────────────────────────────────────────────────────┐
│          Type Alias vs Type Definition                   │
│                                                          │
│  Type ALIAS:     type MyInt = int                        │
│  ─────────────────────────────────────────               │
│  • MyInt IS int (same type identity)                     │
│  • Fully interchangeable, no conversion needed           │
│  • Cannot add methods (it's the same type)               │
│  • Use case: gradual code refactoring/migration          │
│                                                          │
│  Type DEFINITION: type Celsius float64                   │
│  ─────────────────────────────────────────               │
│  • Celsius is a NEW distinct type                        │
│  • Based on float64 but NOT interchangeable              │
│  • CAN add methods to it                                 │
│  • Requires explicit conversion: float64(c)              │
│  • Use case: domain types, type safety, methods          │
│                                                          │
│  Example:                                                │
│    var a MyInt = 10                                       │
│    var b int = a           ✓ alias — same type           │
│                                                          │
│    var c Celsius = 100.0                                  │
│    var d float64 = c       ✗ definition — different type │
│    var d float64 = float64(c) ✓ explicit conversion      │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Type ALIAS — MyInt IS int (same type, fully interchangeable)
type MyAlias = int

// Type DEFINITION — Celsius is a NEW type based on float64
type Celsius float64

// You can add methods to type definitions
func (c Celsius) ToFahrenheit() float64 {
    return float64(c)*9/5 + 32
}

// You CANNOT add methods to type aliases (they're the same type)
// func (m MyAlias) Double() int { ... } // ERROR if int is from another package

func main() {
    // Type alias — interchangeable with the original type
    var a MyAlias = 10
    var b int = a      // Works! MyAlias IS int
    fmt.Println(a, b)  // 10 10

    // Type definition — NOT interchangeable, needs conversion
    var temp Celsius = 100
    // var f float64 = temp  // ERROR: cannot use Celsius as float64
    var f float64 = float64(temp) // Must convert explicitly
    fmt.Println(f)                // 100

    // Methods work on type definitions
    fmt.Println(temp.ToFahrenheit()) // 212

    // Type alias is mainly used for gradual code migrations
    // Type definition is used to create distinct types with their own methods
}
```

---

## Interview Questions

1. **What are the zero values for different types in Go?**
   - `int`: `0`, `float64`: `0.0`, `bool`: `false`, `string`: `""`, pointers/slices/maps/channels/interfaces/functions: `nil`.

2. **What is the difference between `var x int` and `x := 0`?**
   - `var x int` is a full variable declaration (can be used at package level). `:=` is a short declaration (only inside functions). Both initialize to zero value, but `:=` infers the type.

3. **Does Go support implicit type conversion?**
   - No. Go requires explicit type conversion: `float64(myInt)`. There are no implicit conversions, even between `int` and `int64`.

4. **What is `iota` and how is it used?**
   - `iota` is a compile-time constant generator that auto-increments (starting from 0) within a `const` block. It's commonly used for enums: `const (A = iota; B; C)` gives 0, 1, 2.

5. **What is the difference between a type alias and a type definition?**
   - Type alias (`type MyInt = int`): creates an alternate name, same underlying type, fully interchangeable. Type definition (`type MyInt int`): creates a new distinct type — can have its own methods, requires explicit conversion.

6. **What is the difference between `byte` and `rune`?**
   - `byte` is an alias for `uint8` (represents a single ASCII byte). `rune` is an alias for `int32` (represents a Unicode code point). Use `rune` for character-level operations on UTF-8 strings.

7. **What are untyped constants in Go?**
   - Untyped constants have no fixed type until used in a context that requires one. They have higher precision and can be used flexibly: `const x = 42` can be used as `int`, `float64`, etc.

8. **Can you compare two variables of different integer types in Go?**
   - No. You cannot compare `int32` and `int64` directly. You must explicitly convert one to the other first: `int64(myInt32) == myInt64`.

9. **What happens if you declare a variable but don't use it?**
   - Go treats unused local variables as a compile error. Package-level variables and the blank identifier `_` are exempt.

10. **What is `uintptr` and when should you use it?**
    - `uintptr` is an unsigned integer type large enough to hold a pointer value. It's used in low-level programming with `unsafe.Pointer`. It is not a pointer — the GC does not track it, so values may become invalid after GC moves objects.
