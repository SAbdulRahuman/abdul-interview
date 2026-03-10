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

**Tutorial: Integer Types Demonstration**

This example declares every integer type Go offers. Each variable gets the zero value `0` because no explicit value is assigned. The key insight is that `int` and `uint` are platform-dependent (64-bit on modern systems) — use them for general purposes. Use sized types (`int32`, `int64`) when you need exact control over memory layout (e.g., binary protocols, FFI, or struct layout for cache alignment).

```
┌──────────────────────────────────────────────────────────┐
│        Memory Layout — Each Variable in RAM              │
│                                                          │
│  var a int     →  [8 bytes on 64-bit]  = 0000...0000     │
│  var b int8    →  [1 byte ]  = 00000000                  │
│  var c int16   →  [2 bytes]  = 00000000 00000000         │
│  var d int32   →  [4 bytes]  = 00000000 ... (×4)         │
│  var e int64   →  [8 bytes]  = 00000000 ... (×8)         │
│                                                          │
│  var g uint8   →  [1 byte ]  = 00000000 (0-255)          │
│  var k uintptr →  [8 bytes]  = stores raw pointer value  │
│                                                          │
│  All zero-initialized — Go guarantees no garbage values! │
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

**Tutorial: Floating-Point Precision**

Go offers two floating-point sizes: `float32` (7 digits precision) and `float64` (15 digits precision). Always prefer `float64` — it's the default inferred type for float literals and provides adequate precision for most tasks. This example demonstrates the classic floating-point imprecision issue: `0.1 + 0.2 ≠ 0.3` because these decimal fractions can't be represented exactly in IEEE 754 binary. Use `fmt.Printf("%.2f")` when displaying to avoid showing imprecision.

```
┌──────────────────────────────────────────────────────────┐
│         IEEE 754 — Why 0.1 + 0.2 ≠ 0.3                  │
│                                                          │
│  Decimal 0.1 in binary:                                  │
│  0.0001100110011001100110011... (repeating forever)      │
│                                                          │
│  float64 stores 52 bits of mantissa → truncated!        │
│  0.1 ≈ 0.1000000000000000055511151231257827021181583...  │
│  0.2 ≈ 0.2000000000000000111022302462515654042363166...  │
│  sum  = 0.30000000000000004  (not exactly 0.3)           │
│                                                          │
│  Fix: use fmt.Printf("%.2f", result) → "0.30"           │
│  For exact decimals (money): use integer cents or        │
│  a decimal library like shopspring/decimal               │
└──────────────────────────────────────────────────────────┘
```

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

**Tutorial: Complex Number Arithmetic**

Go has built-in complex number support — unusual for a systems language. `complex64` uses two `float32` values (real + imaginary), while `complex128` uses two `float64` values. The `complex()` built-in constructs a complex number, `real()` extracts the real part, and `imag()` extracts the imaginary part. Complex numbers are used in signal processing, physics simulations, and fractal generation.

```
┌──────────────────────────────────────────────────────────┐
│       Complex Number Memory Layout                       │
│                                                          │
│  complex128 = 3 + 4i                                     │
│  ┌──────────────────┬──────────────────┐                 │
│  │ real: float64    │ imag: float64    │                 │
│  │ 3.0 (8 bytes)    │ 4.0 (8 bytes)    │                 │
│  └──────────────────┴──────────────────┘                 │
│  Total: 16 bytes                                         │
│                                                          │
│  Arithmetic: (3+4i) + (1+1i) = (4+5i)                   │
│  real(3+4i) = 3.0     imag(3+4i) = 4.0                  │
└──────────────────────────────────────────────────────────┘
```

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

**Tutorial: byte vs rune vs string**

This is one of the most important concepts in Go. A `byte` is an alias for `uint8` — used for raw data and ASCII. A `rune` is an alias for `int32` — used for Unicode code points. A `string` is an immutable slice of bytes encoded in UTF-8. The critical insight: `len(s)` returns the **byte count**, not the character count! The character '世' takes 3 bytes in UTF-8, so `"Hello, 世界"` is 13 bytes but only 9 characters.

```go
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

**Tutorial: Zero Values — No Uninitialized Variables**

This example shows that every type in Go has a predictable zero value. Unlike C/C++ where uninitialized variables contain garbage, Go guarantees deterministic defaults. This eliminates an entire class of bugs. Note that `nil` types (pointers, slices, maps, channels, functions, interfaces) are safe to read but may panic when written to or methods are called on them — always check for `nil` before use.

```go
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

**Tutorial: 7 Ways to Declare Variables**

Go offers multiple declaration styles for different contexts. The `var` keyword with explicit type is the most verbose but clearest. Short declaration `:=` is the most common inside functions — it infers the type automatically. Block declarations group related variables. The swap idiom `x, y = y, x` is unique to Go — it swaps values without a temp variable because the right side is fully evaluated before assignment.

```
┌──────────────────────────────────────────────────────────┐
│       Variable Declaration Decision Guide                │
│                                                          │
│  Where?          Style              Example              │
│  ──────          ─────              ───────              │
│  Package level   var name Type      var count int        │
│  Inside func     := (preferred)     count := 0           │
│  Multiple vars   var block          var ( a=1; b=2 )     │
│  Explicit type   var name Type = v  var s string = "hi"  │
│  Type inference  var name = v       var s = "hi"         │
│                                                          │
│  ⚠ := only works inside functions (not package level)   │
│  ⚠ := requires at least one NEW variable on the left    │
└──────────────────────────────────────────────────────────┘
```

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

**Tutorial: Constants as Compile-Time Values & iota Magic**

Constants are evaluated at compile time — they never appear in runtime memory as mutable data. The `iota` keyword is a compile-time counter that auto-increments within each `const` block. It resets to 0 in every new `const` block. This example shows three patterns: simple constants, day-of-week enum via raw `iota`, byte-size units via `iota` with bit shifting, and Unix file permission flags via `iota` with `1 << iota`. The bit flag pattern is especially powerful: combine permissions with `|` (OR) and check them with `&` (AND).

```
┌──────────────────────────────────────────────────────────┐
│       Bit Flags — How Permissions Work                   │
│                                                          │
│  const (                                                 │
│    Read  = 1 << 0  = 001 = 1                             │
│    Write = 1 << 1  = 010 = 2                             │
│    Exec  = 1 << 2  = 100 = 4                             │
│  )                                                       │
│                                                          │
│  Combine: Read | Write = 001 | 010 = 011 = 3            │
│  Check:   perms & Read != 0 → true  (has read)          │
│  Check:   perms & Exec != 0 → false (no exec)           │
│                                                          │
│  Bit visualization:                                      │
│  perms = 011                                             │
│          │││                                             │
│          ││└─ Read  (bit 0) = 1 ✓                        │
│          │└── Write (bit 1) = 1 ✓                        │
│          └─── Exec  (bit 2) = 0 ✗                        │
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

**Tutorial: Untyped Constants — Go's Hidden Superpower**

Untyped constants have no fixed type until they're used in an expression. This makes them far more flexible than typed constants. The constant `42` can become an `int`, `float64`, `float32`, or any numeric type depending on the context it's used in. Untyped constants also have much higher precision — you can declare `const huge = 1e1000` which no runtime type can hold, as long as the final result fits in a real type.

```
┌──────────────────────────────────────────────────────────┐
│       Untyped vs Typed Constants                         │
│                                                          │
│  const x = 42           (untyped integer)                │
│  ┌──────────────────────────────────────┐                │
│  │ var i int     = x  → x becomes int     ✓             │
│  │ var f float64 = x  → x becomes float64 ✓             │
│  │ var b byte    = x  → x becomes byte    ✓             │
│  └──────────────────────────────────────┘                │
│  x ADAPTS to whatever type context it appears in         │
│                                                          │
│  const typedX int = 42  (typed integer)                  │
│  ┌──────────────────────────────────────┐                │
│  │ var i int     = typedX  → ✓ (same type)              │
│  │ var f float64 = typedX  → ✗ COMPILE ERROR            │
│  └──────────────────────────────────────┘                │
│  typedX is LOCKED to int — no adaptation                 │
│                                                          │
│  High precision:                                         │
│  const huge = 1e1000     ← OK as untyped (compiler math)│
│  var h float64 = huge    ← ERROR: overflow at assignment │
│  const small = huge/1e999 ← OK: result is 10            │
│  var s float64 = small   ← OK: 10 fits in float64       │
└──────────────────────────────────────────────────────────┘
```

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

**Tutorial: Explicit Conversions — No Surprises**

Go refuses to do implicit type conversions. This is intentional — in C, `int x = 3.99` silently truncates to `3`, and `char c = 256` silently overflows. Go forces you to write `int(3.99)` so the truncation is visible in the code. The most common gotcha: `string(65)` does NOT produce `"65"` — it produces `"A"` (Unicode code point 65). For number-to-string conversion, always use `strconv.Itoa()`.

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

**Tutorial: Two Distinct Mechanisms**

These are two mechanisms that look similar but serve very different purposes. A type alias (`type MyInt = int`) creates another name for the exact same type — `MyInt` and `int` are interchangeable with no conversion needed. A type definition (`type Celsius float64`) creates a brand-new type — `Celsius` and `float64` are incompatible, requiring explicit conversion. You can only add methods to type definitions (new types), not aliases (same type).

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
