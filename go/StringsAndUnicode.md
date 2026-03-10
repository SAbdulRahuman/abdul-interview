# Chapter 6 — Strings & Unicode

## String Internals

Strings in Go are **immutable byte slices**. Internally, a string is a small **descriptor** (16 bytes on 64-bit: pointer + length) pointing to a read-only byte array — there is no null terminator.

```
┌──────────────────────────────────────────────────────────┐
│                String Internal Structure                 │
│                                                          │
│  s := "Hello, 世界"                                      │
│                                                          │
│  String header:              Byte array (read-only):     │
│  ┌──────────────┐            ┌─────────────────────────┐ │
│  │ ptr ─────────────────────►│ H  e  l  l  o  ,  ░    │ │
│  │ len: 13      │            │ E4 B8 96  E7 95 8C      │ │
│  └──────────────┘            └─────────────────────────┘ │
│                               ◄── ASCII ──►◄─ UTF-8 ──► │
│                               1 byte each   3 bytes each │
│                                                          │
│  len("Hello, 世界") = 13  ← byte count                  │
│  utf8.RuneCountInString() = 9  ← character count        │
│                                                          │
│  Strings are IMMUTABLE: s[0] = 'h' → compile error      │
│  To mutate: convert to []byte, modify, convert back      │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

func main() {
    s := "Hello, 世界"

    // len() returns BYTE count, not character count
    fmt.Println(len(s)) // 13 (5 ASCII bytes + 2 bytes for ", " + 6 bytes for 2 CJK chars)

    // Access individual bytes
    fmt.Println(s[0]) // 72 (byte value of 'H')

    // Strings are IMMUTABLE — cannot modify in place
    // s[0] = 'h' // COMPILE ERROR

    // Convert to byte slice for mutation
    b := []byte(s)
    b[0] = 'h'
    s2 := string(b)
    fmt.Println(s2) // hello, 世界

    // String is just bytes
    fmt.Printf("% x\n", s) // 48 65 6c 6c 6f 2c 20 e4 b8 96 e7 95 8c
}
```

---

## rune vs byte

A `byte` represents a single raw byte (alias for `uint8`). A `rune` represents a Unicode code point (alias for `int32`). When iterating strings, `for i := 0` iterates **bytes** while `for i, r := range` iterates **runes**.

```
┌──────────────────────────────────────────────────────────┐
│           byte vs rune Iteration                        │
│                                                          │
│  s := "Go世界!"                                          │
│                                                          │
│  Byte view:     [G] [o] [E4][B8][96] [E7][95][8C] [!]  │
│  Index:          0   1   2   3   4    5   6   7    8    │
│  len(s) = 9 bytes                                       │
│                                                          │
│  Rune view:     [G]  [o]  [世]       [界]       [!]     │
│  Byte index:     0    1    2          5          8       │
│  utf8.RuneCount = 5 runes                               │
│                                                          │
│  ⚠ len(s) counts bytes, NOT characters                  │
│  ⚠ s[2] gives raw byte 0xE4, NOT the character '世'    │
│  ⚠ range iterates runes (skips multi-byte boundaries)   │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    s := "Go世界!"

    // byte iteration — iterates raw bytes
    fmt.Println("Byte iteration:")
    for i := 0; i < len(s); i++ {
        fmt.Printf("  byte[%d] = %02x (%c)\n", i, s[i], s[i])
    }
    // Multi-byte characters show as individual bytes (broken)

    // rune iteration — iterates Unicode code points
    fmt.Println("\nRune iteration:")
    for i, r := range s {
        fmt.Printf("  byte_index=%d rune=%c (U+%04X)\n", i, r, r)
    }
    // byte_index=0 rune=G (U+0047)
    // byte_index=1 rune=o (U+006F)
    // byte_index=2 rune=世 (U+4E16)
    // byte_index=5 rune=界 (U+754C)
    // byte_index=8 rune=! (U+0021)

    fmt.Println("Byte length:", len(s))                        // 9
    fmt.Println("Rune count:", utf8.RuneCountInString(s))      // 5
}
```

---

## strings Package

The `strings` package provides a rich set of functions for searching, transforming, splitting, joining, trimming, and building strings. Since Go strings are immutable, every operation returns a **new** string — the original is never modified. For efficient concatenation in a loop, use `strings.Builder` which accumulates bytes in an internal buffer and produces the final string in one allocation.

```
┌──────────────────────────────────────────────────────────────┐
│               strings Package — Key Operations              │
│                                                              │
│  Search        Transform       Split/Join       Trim        │
│  ──────        ─────────       ──────────       ────        │
│  Contains()    ToUpper()       Split()          TrimSpace() │
│  HasPrefix()   ToLower()       Join()           Trim()      │
│  HasSuffix()   Title()         Fields()         TrimLeft()  │
│  Index()       Map()           SplitAfter()     TrimRight() │
│  Count()       Replace()                        TrimPrefix()│
│                ReplaceAll()                     TrimSuffix()│
│                                                              │
│  Builder (efficient concat):                                 │
│  ┌─────────────────────────────────────────────┐             │
│  │ builder.WriteString("Go")  ► internal buf   │             │
│  │ builder.WriteString(" is") ► grows in place │             │
│  │ builder.String()           ► final string   │             │
│  └─────────────────────────────────────────────┘             │
└──────────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    s := "Hello, World! Hello, Go!"

    // Search
    fmt.Println(strings.Contains(s, "Go"))       // true
    fmt.Println(strings.HasPrefix(s, "Hello"))    // true
    fmt.Println(strings.HasSuffix(s, "Go!"))      // true
    fmt.Println(strings.Index(s, "World"))         // 7
    fmt.Println(strings.Count(s, "Hello"))         // 2

    // Transform
    fmt.Println(strings.ToUpper("hello"))          // HELLO
    fmt.Println(strings.ToLower("HELLO"))          // hello
    fmt.Println(strings.Title("hello world"))      // Hello World (deprecated, use cases.Title)

    // Split and Join
    parts := strings.Split("a,b,c,d", ",")
    fmt.Println(parts) // [a b c d]

    joined := strings.Join([]string{"Go", "is", "great"}, " ")
    fmt.Println(joined) // Go is great

    // Replace
    fmt.Println(strings.Replace(s, "Hello", "Hi", 1))  // Hi, World! Hello, Go!
    fmt.Println(strings.ReplaceAll(s, "Hello", "Hi"))   // Hi, World! Hi, Go!

    // Trim
    fmt.Println(strings.TrimSpace("  hello  "))          // "hello"
    fmt.Println(strings.Trim("***hello***", "*"))        // "hello"
    fmt.Println(strings.TrimLeft("***hello***", "*"))    // "hello***"
    fmt.Println(strings.TrimRight("***hello***", "*"))   // "***hello"
    fmt.Println(strings.TrimPrefix("hello-world", "hello-")) // "world"
    fmt.Println(strings.TrimSuffix("hello.go", ".go"))       // "hello"

    // Map — apply function to each rune
    result := strings.Map(func(r rune) rune {
        if r == 'o' {
            return '0'
        }
        return r
    }, "Hello World")
    fmt.Println(result) // Hell0 W0rld

    // strings.Builder — efficient string concatenation
    var builder strings.Builder
    for i := 0; i < 5; i++ {
        fmt.Fprintf(&builder, "item%d ", i)
    }
    fmt.Println(builder.String()) // item0 item1 item2 item3 item4

    // NewReader — creates an io.Reader from a string
    reader := strings.NewReader("Hello, Reader!")
    buf := make([]byte, 5)
    n, _ := reader.Read(buf)
    fmt.Println(string(buf[:n])) // Hello
}
```

---

## strconv Package

**Tutorial: Converting Between Strings and Numeric Types**

The `strconv` package handles all conversions between strings and basic Go types (int, float, bool). The most common pair is `Atoi` (ASCII-to-integer) and `Itoa` (integer-to-ASCII). For more control — like parsing hex or binary — use `ParseInt` with a base and bit-size argument. Always check the returned `error` from parse functions, as invalid input will not panic but will return a descriptive error.

```
┌────────────────────────────────────────────────────────────┐
│              strconv — Conversion Flow                     │
│                                                            │
│  String ──────────────────────────────► Numeric            │
│    "42"   ── Atoi(s) ──────────────►  42  (int)           │
│    "FF"   ── ParseInt(s, 16, 64) ──►  255 (int64)         │
│    "1010" ── ParseInt(s, 2, 64) ───►  10  (int64)         │
│    "3.14" ── ParseFloat(s, 64) ────►  3.14 (float64)      │
│    "true" ── ParseBool(s) ─────────►  true (bool)         │
│                                                            │
│  Numeric ──────────────────────────────► String            │
│    42     ── Itoa(n) ──────────────►  "42"                │
│    255    ── FormatInt(n, 16) ─────►  "ff"                │
│    false  ── FormatBool(b) ────────►  "false"             │
│                                                            │
│  ⚠ Always check error: _, err := strconv.Atoi("bad")     │
│    err → strconv.Atoi: parsing "bad": invalid syntax      │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Atoi — string to int
    num, err := strconv.Atoi("42")
    fmt.Println(num, err) // 42 <nil>

    _, err = strconv.Atoi("not_a_number")
    fmt.Println(err) // strconv.Atoi: parsing "not_a_number": invalid syntax

    // Itoa — int to string
    s := strconv.Itoa(42)
    fmt.Println(s) // "42"

    // ParseInt — more control (string, base, bit size)
    n, _ := strconv.ParseInt("FF", 16, 64)    // hex
    fmt.Println(n) // 255

    n2, _ := strconv.ParseInt("1010", 2, 64)  // binary
    fmt.Println(n2) // 10

    // ParseFloat
    f, _ := strconv.ParseFloat("3.14159", 64)
    fmt.Println(f) // 3.14159

    // FormatInt — int to string with base
    fmt.Println(strconv.FormatInt(255, 16)) // ff
    fmt.Println(strconv.FormatInt(10, 2))   // 1010

    // ParseBool
    b, _ := strconv.ParseBool("true")
    fmt.Println(b) // true

    // FormatBool
    fmt.Println(strconv.FormatBool(false)) // "false"
}
```

---

## fmt Package Verbs

**Tutorial: Formatting Output with Printf Verbs**

The `fmt` package uses **verbs** (like `%s`, `%d`, `%v`) as placeholders inside format strings. The three struct verbs — `%v`, `%+v`, `%#v` — give increasing levels of detail: default values, field names, and full Go syntax. Numeric verbs let you print in decimal (`%d`), binary (`%b`), octal (`%o`), or hex (`%x`). The `%w` verb in `fmt.Errorf` wraps errors for use with `errors.Is` and `errors.As`.

```
┌──────────────────────────────────────────────────────────┐
│              fmt Verbs — Quick Reference                 │
│                                                          │
│  ┌─────── General ───────┐  ┌─────── Numeric ──────────┐ │
│  │ %v   default value    │  │ %d  decimal (42)         │ │
│  │ %+v  with field names │  │ %b  binary  (101010)     │ │
│  │ %#v  Go syntax repr   │  │ %o  octal   (52)         │ │
│  │ %T   type name        │  │ %x  hex     (2a)         │ │
│  └───────────────────────┘  └──────────────────────────┘ │
│                                                          │
│  ┌─────── String ────────┐  ┌─────── Other ────────────┐ │
│  │ %s   plain string     │  │ %f  float (3.140000)     │ │
│  │ %q   quoted string    │  │ %e  scientific notation  │ │
│  └───────────────────────┘  │ %p  pointer address      │ │
│                              │ %w  error wrapping       │ │
│                              └──────────────────────────┘ │
│                                                          │
│  User{"Alice",30}                                        │
│    %v  ► {Alice 30}                                      │
│    %+v ► {Name:Alice Age:30}                             │
│    %#v ► main.User{Name:"Alice", Age:30}                │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import "fmt"

type User struct {
    Name string
    Age  int
}

func main() {
    u := User{"Alice", 30}

    fmt.Printf("%v\n", u)   // {Alice 30}           — default format
    fmt.Printf("%+v\n", u)  // {Name:Alice Age:30}  — with field names
    fmt.Printf("%#v\n", u)  // main.User{Name:"Alice", Age:30} — Go syntax
    fmt.Printf("%T\n", u)   // main.User             — type name

    // String
    fmt.Printf("%s\n", "hello")  // hello
    fmt.Printf("%q\n", "hello")  // "hello" (quoted)

    // Integer
    fmt.Printf("%d\n", 42)      // 42 (decimal)
    fmt.Printf("%b\n", 42)      // 101010 (binary)
    fmt.Printf("%o\n", 42)      // 52 (octal)
    fmt.Printf("%x\n", 42)      // 2a (hex lowercase)
    fmt.Printf("%X\n", 42)      // 2A (hex uppercase)

    // Float
    fmt.Printf("%f\n", 3.14)    // 3.140000
    fmt.Printf("%.2f\n", 3.14)  // 3.14
    fmt.Printf("%e\n", 3.14)    // 3.140000e+00 (scientific)

    // Pointer
    x := 42
    fmt.Printf("%p\n", &x)      // 0xc0000b4008 (pointer address)

    // Error wrapping (used with fmt.Errorf)
    // %w wraps an error for errors.Is/errors.As
    err := fmt.Errorf("connection failed: %w", fmt.Errorf("timeout"))
    fmt.Println(err) // connection failed: timeout
}
```

---

## unicode/utf8 Package

**Tutorial: Low-Level UTF-8 Decoding and Validation**

The `unicode/utf8` package provides functions to inspect, validate, and decode UTF-8 encoded strings at the byte level. Use `RuneCountInString` to count characters (not bytes), `DecodeRuneInString` to extract the first rune and its byte width, and `ValidString` to verify a byte sequence is well-formed UTF-8. This package is essential when you need precise control over multi-byte character boundaries.

```
┌──────────────────────────────────────────────────────────────┐
│         unicode/utf8 — Decoding Walk-Through                 │
│                                                              │
│  s := "Hello, 世界!"                                         │
│                                                              │
│  Bytes: [48][65][6C][6C][6F][2C][20][E4 B8 96][E7 95 8C][21]│
│          H   e   l   l   o   ,  ░   世(3B)    界(3B)    !   │
│                                                              │
│  DecodeRuneInString(s):                                      │
│    ┌──────────┐                                              │
│    │ ptr ──►H │  rune='H'  size=1                            │
│    └──────────┘                                              │
│                                                              │
│  DecodeRuneInString("世界"):                                  │
│    ┌──────────────┐                                          │
│    │ ptr ──►E4 B8 96 │  rune='世'  size=3                    │
│    └──────────────┘                                          │
│                                                              │
│  RuneLen('A') = 1    RuneLen('世') = 3                       │
│  ValidString(s) = true                                       │
│  ValidString("\xff\xfe") = false                             │
└──────────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    s := "Hello, 世界!"

    // Count runes (characters), not bytes
    fmt.Println("Byte length:", len(s))                         // 14
    fmt.Println("Rune count:", utf8.RuneCountInString(s))       // 10

    // Check if valid UTF-8
    fmt.Println("Valid UTF-8:", utf8.ValidString(s))             // true
    fmt.Println("Valid UTF-8:", utf8.ValidString("\xff\xfe"))    // false

    // Decode first rune
    r, size := utf8.DecodeRuneInString(s)
    fmt.Printf("First rune: %c, size: %d bytes\n", r, size)
    // First rune: H, size: 1 bytes

    // Decode from "世界"
    r2, size2 := utf8.DecodeRuneInString("世界")
    fmt.Printf("First rune: %c, size: %d bytes\n", r2, size2)
    // First rune: 世, size: 3 bytes

    // Check rune byte length
    fmt.Println(utf8.RuneLen('A'))  // 1
    fmt.Println(utf8.RuneLen('世')) // 3
}
```

---

## String Concatenation Performance

**Tutorial: Why `+=` Is Slow and How Builder Fixes It**

Because strings are immutable in Go, every `+=` in a loop allocates a brand-new string and copies all previous content — resulting in O(n²) time. `strings.Builder` solves this by writing into a growable internal `[]byte` buffer, producing the final string in one call. `bytes.Buffer` works similarly but is heavier (it implements more interfaces). For a pre-existing slice of strings, `strings.Join` is the cleanest and most efficient option.

```
┌──────────────────────────────────────────────────────────────┐
│       String Concatenation — Performance Comparison          │
│                                                              │
│  += operator (O(n²) in loop):                                │
│  ┌─────┐  ┌──────────┐  ┌────────────────┐  ← new alloc     │
│  │"a"  │  │"a" + "b" │  │"ab" + "c"      │     each time    │
│  └─────┘  └──────────┘  └────────────────┘                   │
│    1 byte   2+1 copy      3+1 copy         total: O(n²)     │
│                                                              │
│  strings.Builder (O(n)):                                     │
│  ┌──────────────────────────────────────┐                    │
│  │ internal buffer (grows in place)     │                    │
│  │ [a][b][c][d][e]...                   │ ← appends only    │
│  └──────────────────────────────────────┘                    │
│    .String() ► final string (one alloc)                      │
│                                                              │
│  strings.Join (best for slices):                             │
│  ["Go", "is", "great"] ──► "Go is great"                    │
│    pre-calculates total length, one allocation               │
└──────────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "bytes"
    "fmt"
    "strings"
)

func main() {
    // Method 1: + operator — O(n²) in loops, creates new string each time
    s := ""
    for i := 0; i < 5; i++ {
        s += fmt.Sprintf("item%d ", i) // BAD in loops — copies entire string each time
    }
    fmt.Println(s)

    // Method 2: strings.Builder — O(n), most efficient
    var builder strings.Builder
    for i := 0; i < 5; i++ {
        fmt.Fprintf(&builder, "item%d ", i) // Appends to internal buffer
    }
    fmt.Println(builder.String())

    // Method 3: bytes.Buffer — similar to Builder, also implements io.Writer
    var buf bytes.Buffer
    for i := 0; i < 5; i++ {
        fmt.Fprintf(&buf, "item%d ", i)
    }
    fmt.Println(buf.String())

    // Method 4: strings.Join — best when you have a slice of strings
    parts := []string{"Go", "is", "awesome"}
    fmt.Println(strings.Join(parts, " ")) // Go is awesome

    // Rule of thumb:
    // - Few strings: + is fine
    // - Loop concatenation: strings.Builder
    // - Slice of strings: strings.Join
}
```

---

## Interview Questions

1. **How are strings represented internally in Go?**
   - Strings are immutable byte slices with a header containing a pointer and a length. They store UTF-8 encoded bytes. `len(s)` returns byte count, not character count.

2. **What is the difference between `byte` and `rune` in Go?**
   - `byte` is `uint8` — one byte. `rune` is `int32` — one Unicode code point. A single character (like `'世'`) may be 3 bytes but is always 1 rune.

3. **How do you count the number of characters in a UTF-8 string?**
   - Use `utf8.RuneCountInString(s)` or `len([]rune(s))`. Do NOT use `len(s)` — that gives byte count.

4. **Why is string concatenation with `+=` in a loop inefficient?**
   - Strings are immutable, so each `+=` allocates a new string and copies all previous content — O(n²). Use `strings.Builder` for O(n) concatenation.

5. **How do you iterate over characters (runes) in a Go string?**
   - `for i, r := range s` iterates by rune, giving the byte index and the rune value. `for i := 0; i < len(s); i++` iterates by byte.

6. **What is `strings.Builder` and why is it preferred?**
   - `strings.Builder` efficiently builds strings by accumulating bytes in an internal buffer, minimizing allocations. Call `WriteString()` to append, then `String()` to get the result.

7. **How do you convert between strings and numbers in Go?**
   - `strconv.Atoi("42")` → int. `strconv.Itoa(42)` → string. `strconv.ParseFloat("3.14", 64)` → float64. `strconv.FormatFloat(3.14, 'f', 2, 64)` → string.

8. **Can you modify a character in a Go string?**
   - No. Strings are immutable. Convert to `[]byte` or `[]rune` first, modify, then convert back to `string`.

9. **What is the difference between `%v`, `%+v`, and `%#v` in `fmt`?**
   - `%v`: default format. `%+v`: adds field names for structs. `%#v`: Go-syntax representation (e.g., `main.Person{Name:"Alice", Age:30}`).

10. **What does `strings.Map` do?**
    - It applies a function to every rune in a string and returns a new string of the results. If the function returns -1 for a rune, that rune is dropped.
