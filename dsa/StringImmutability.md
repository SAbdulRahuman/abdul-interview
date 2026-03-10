# Phase 3: Strings — String Immutability

## What is String Immutability?

In Go, **strings are immutable** — once created, their bytes cannot be changed. Any "modification" creates a new string. This is important for performance and correctness.

```go
s := "hello"
s[0] = 'H' // COMPILE ERROR: cannot assign to s[0]
```

---

## Example 1: Strings vs Byte Slices

```go
package main

import "fmt"

func main() {
    s := "hello"
    // s[0] = 'H' // ← This won't compile!

    // To modify, convert to []byte
    b := []byte(s)
    b[0] = 'H'
    modified := string(b)

    fmt.Println(s)        // hello — original unchanged
    fmt.Println(modified) // Hello — new string
}
```

**Textual Figure — String vs Byte Slice:**

```
  String (immutable):
  ┌─────────────────────────────┐
  │ s := "hello"                │
  │  ptr ──→ [h][e][l][l][o]   │   Read-only bytes
  │  len = 5                    │   Cannot modify s[0]
  └─────────────────────────────┘

  Convert to []byte (mutable copy):
  ┌─────────────────────────────┐
  │ b := []byte(s)              │   COPIES the bytes
  │  ptr ──→ [h][e][l][l][o]   │   New mutable memory
  │  len=5, cap=5               │
  └─────────────────────────────┘

  Modify and convert back:
  b[0] = 'H'
  ┌─────────────────────────────┐
  │ b ──→ [H][e][l][l][o]      │   Modified in place
  │ modified = string(b)        │   Another COPY back
  └─────────────────────────────┘

  Original s is untouched:  "hello"
  New modified string:      "Hello"

  Total: 2 copies!  string → []byte → string
```

---

## Example 2: String Concatenation Creates New Strings

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    s1 := "hello"
    s2 := s1 + " world" // new allocation

    fmt.Printf("s1: %q at %p\n", s1, unsafe.StringData(s1))
    fmt.Printf("s2: %q at %p\n", s2, unsafe.StringData(s2))
    // Different addresses — s2 is a new string
}
```

**Textual Figure — Concatenation Creates New String:**

```
  s1 := "hello"
  ┌───────┐
  │ ptr ──┼──→ [h][e][l][l][o]     addr: 0xA000
  │ len=5 │
  └───────┘

  s2 := s1 + " world"
  ┌───────┐
  │ ptr ──┼──→ [h][e][l][l][o][ ][w][o][r][l][d]   addr: 0xB000
  │ len=11│
  └───────┘

  s1 and s2 point to DIFFERENT memory!

  0xA000: [h e l l o]           ← s1 (still alive)
  0xB000: [h e l l o   w o r l d]  ← s2 (new allocation)

  Every + creates a new string and copies ALL bytes.
```

---

## Example 3: Inefficient String Concatenation in a Loop

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

func concatNaive(n int) string {
    s := ""
    for i := 0; i < n; i++ {
        s += "a" // O(n) each time → O(n²) total!
    }
    return s
}

func concatBuilder(n int) string {
    var sb strings.Builder
    for i := 0; i < n; i++ {
        sb.WriteByte('a') // O(1) amortized
    }
    return sb.String() // O(n) total
}

func main() {
    n := 100_000

    start := time.Now()
    concatNaive(n)
    fmt.Printf("Naive:   %v\n", time.Since(start))

    start = time.Now()
    concatBuilder(n)
    fmt.Printf("Builder: %v\n", time.Since(start))

    // Builder is ~100x faster for large strings
}
```

**Textual Figure — Naive vs Builder Concatenation:**

```
  Naive: s += "a" in a loop (n = 5)

  Iter 1: s = ""  + "a"     → copy 1 byte  → "a"
  Iter 2: s = "a" + "a"     → copy 2 bytes → "aa"
  Iter 3: s = "aa" + "a"    → copy 3 bytes → "aaa"
  Iter 4: s = "aaa" + "a"   → copy 4 bytes → "aaaa"
  Iter 5: s = "aaaa" + "a"  → copy 5 bytes → "aaaaa"
                  Total copies: 1+2+3+4+5 = 15 = O(n²)

  strings.Builder (amortized O(1) append):
  ┌────────────────────────────────────────┐
  │ internal buffer (grows like []byte):   │
  │  cap=0 → cap=8 → cap=16 → cap=32...   │
  │                                        │
  │  Write("a"): just append, no copy      │
  │  Write("a"): just append, no copy      │
  │  ...                                   │
  │  String(): one final conversion        │
  └────────────────────────────────────────┘
  Total copies: O(n)  ← ~100x faster!

  Performance (n=100,000):
  Naive:   ~500ms
  Builder: ~0.5ms
```

---

## Example 4: strings.Builder — The Right Way

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    var sb strings.Builder

    words := []string{"Go", "strings", "are", "immutable"}
    for i, w := range words {
        if i > 0 {
            sb.WriteByte(' ')
        }
        sb.WriteString(w)
    }

    result := sb.String()
    fmt.Println(result)          // Go strings are immutable
    fmt.Println("Len:", sb.Len()) // 26

    // Builder can also pre-allocate
    var sb2 strings.Builder
    sb2.Grow(100) // pre-allocate 100 bytes
    sb2.WriteString("pre-allocated")
    fmt.Println(sb2.String())
}
```

**Textual Figure — strings.Builder Internals:**

```
  var sb strings.Builder

  sb.WriteString("Go")         sb.WriteByte(' ')
  ┌─────────────────┐          ┌─────────────────┐
  │ buf: [G][o]     │          │ buf: [G][o][ ]  │
  │ len=2, cap=8    │          │ len=3, cap=8    │
  └─────────────────┘          └─────────────────┘

  sb.WriteString("strings")    sb.WriteByte(' ')
  ┌─────────────────────────┐  ┌──────────────────────────┐
  │ buf: [G o   s t r i n]  │  │ buf: [G o   s t r i n g] │
  │      [g s]               │  │      [s  ]               │
  │ len=10, cap=16           │  │ len=11, cap=16           │
  └─────────────────────────┘  └──────────────────────────┘

  ...continues appending "are" and "immutable"...

  sb.String() → "Go strings are immutable"
                  (single conversion, no extra copies)

  sb.Grow(100):  pre-allocate 100 bytes capacity
  ┌──────────────────────────────────────┐
  │ buf: [...........................]    │
  │ len=0, cap=100                       │
  │ No reallocation needed until cap hit │
  └──────────────────────────────────────┘
```

---

## Example 5: String Slicing Shares Memory (Like Slices)

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    s := "hello world"
    sub := s[6:] // "world"

    fmt.Println(sub) // world

    // Both point to same underlying bytes
    fmt.Printf("s starts at:   %p\n", unsafe.StringData(s))
    fmt.Printf("sub starts at: %p\n", unsafe.StringData(sub))
    // sub's pointer = s's pointer + 6

    // This means substrings are O(1) — no copy!
    // But the original string can't be GC'd while sub exists
}
```

**Textual Figure — String Slicing Shares Memory:**

```
  s := "hello world"

  Memory layout:
  addr:  0x1000                           0x100A
         ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  bytes: │ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
         └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
         ↑                       ↑
         s.ptr (len=11)          sub.ptr (len=5)

  sub := s[6:]  →  "world"

  s:   { ptr: 0x1000, len: 11 }    ← string header
  sub: { ptr: 0x1006, len: 5  }    ← points INTO same bytes!

  No copy! O(1) substring creation.

  ⚠ Memory leak risk:
  ┌────────────────────────────────────────────┐
  │ If s = readHugeFile() (1GB)                │
  │ sub = s[:10]                               │
  │ s = ""  // try to free...                  │
  │ But 1GB can't be GC'd! sub still refs it.  │
  │ Fix: sub = string([]byte(s[:10]))  // copy │
  └────────────────────────────────────────────┘
```

---

## Example 6: []rune for Unicode Modification

```go
package main

import "fmt"

func main() {
    // Strings are byte sequences, not character sequences
    s := "café"
    fmt.Println("len(bytes):", len(s))         // 5 (é = 2 bytes)
    fmt.Println("len(runes):", len([]rune(s))) // 4

    // Safe character-level modification
    runes := []rune(s)
    runes[3] = 'E' // replace é with E
    fmt.Println(string(runes)) // cafE

    // Inserting into middle of string
    s2 := "hello world"
    r2 := []rune(s2)
    // Insert "beautiful " at position 6
    insert := []rune("beautiful ")
    result := append(r2[:6], append(insert, r2[6:]...)...)
    fmt.Println(string(result)) // hello beautiful world
}
```

**Textual Figure — []rune for Unicode Modification:**

```
  s := "café"

  Bytes vs Runes:
  Bytes: [c] [a] [f] [0xc3] [0xa9]    len(s) = 5 bytes
          │   │   │   └─────┘
          c   a   f      é            é = 2 bytes in UTF-8

  Runes: [c] [a] [f] [é]              len([]rune(s)) = 4 runes
          0   1   2   3

  Modify rune at index 3:
  runes[3] = 'E'
  [c] [a] [f] [E]  →  "cafE"  ✓

  Insert into middle:
  s2 = "hello world"   r2 = []rune(s2)
  insert = []rune("beautiful ")

  r2[:6]  = [h][e][l][l][o][ ]
  insert  = [b][e][a][u][t][i][f][u][l][ ]
  r2[6:]  = [w][o][r][l][d]

  Result: "hello beautiful world"

  Rule: Use []byte for ASCII, []rune for Unicode!
```

---

## Example 7: String Comparison and Equality

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    s1 := "Hello"
    s2 := "hello"

    // == is exact byte comparison
    fmt.Println(s1 == s2)                          // false
    fmt.Println(strings.EqualFold(s1, s2))         // true (case-insensitive)
    fmt.Println(strings.ToLower(s1) == strings.ToLower(s2)) // true

    // String comparison is lexicographic
    fmt.Println("apple" < "banana")  // true
    fmt.Println("abc" < "abd")       // true
    fmt.Println("abc" < "abcd")      // true

    // strings.Compare returns -1, 0, or 1
    fmt.Println(strings.Compare("a", "b"))  // -1
    fmt.Println(strings.Compare("b", "a"))  // 1
    fmt.Println(strings.Compare("a", "a"))  // 0
}
```

**Textual Figure — String Comparison:**

```
  Byte-by-byte lexicographic comparison:

  "apple" vs "banana":
  a(97) < b(98) → "apple" < "banana"  (stop at first diff)

  "abc" vs "abd":
  a==a, b==b, c(99) < d(100) → "abc" < "abd"

  "abc" vs "abcd":
  a==a, b==b, c==c, then "abc" shorter → "abc" < "abcd"

  Case sensitivity:
  ┌─────────────┬────────┬──────────────────────────┐
  │ Method        │Result │ Notes                      │
  ├─────────────┼────────┼──────────────────────────┤
  │ "Hello"=="hello"  false   │ 'H'(72) ≠ 'h'(104)        │
  │ EqualFold      │ true   │ Case-insensitive           │
  │ ToLower==      │ true   │ Normalize then compare     │
  └─────────────┴────────┴──────────────────────────┘
```

---

## Example 8: String Interning and Memory

```go
package main

import "fmt"

func main() {
    // Go doesn't have explicit string interning, but:
    // 1. String literals are stored in read-only memory
    // 2. Identical literals are deduplicated by the compiler

    s1 := "hello"
    s2 := "hello"
    // These likely share the same backing array (compiler optimization)

    // But runtime-created strings are independent
    b := []byte{'h', 'e', 'l', 'l', 'o'}
    s3 := string(b) // new allocation
    fmt.Println(s1 == s3) // true (value equality, not reference)

    // Manual interning with a map
    intern := make(map[string]string)
    internStr := func(s string) string {
        if existing, ok := intern[s]; ok {
            return existing
        }
        intern[s] = s
        return s
    }
    a := internStr("important")
    c := internStr("important") // reuses memory
    fmt.Println(a == c) // true
    _ = a
    _ = c
}
```

**Textual Figure — String Interning:**

```
  Compile-time literals:
  s1 := "hello"   s2 := "hello"

  Compiler may deduplicate:
  ┌─────┐     ┌──────────────────┐
  │  s1 │───→│ [h][e][l][l][o]  │  read-only segment
  └─────┘  ┌→│                  │
  ┌─────┐  │ └──────────────────┘
  │  s2 │──┘  Same memory! (compiler optimized)
  └─────┘

  Runtime-created strings:
  b := []byte{'h','e','l','l','o'}
  s3 := string(b)
  ┌─────┐     ┌──────────────────┐
  │  s3 │───→│ [h][e][l][l][o]  │  NEW allocation
  └─────┘     └──────────────────┘

  s1 == s3 → true  (value equality, compares bytes)

  Manual interning with map:
  intern["important"] = "important"  (store once)
  Next time: reuse same pointer → saves memory
```

---

## Example 9: Reverse a String (Handling Immutability)

```go
package main

import "fmt"

// Reverse ASCII string
func reverseASCII(s string) string {
    b := []byte(s)
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        b[i], b[j] = b[j], b[i]
    }
    return string(b)
}

// Reverse Unicode string
func reverseUnicode(s string) string {
    r := []rune(s)
    for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
        r[i], r[j] = r[j], r[i]
    }
    return string(r)
}

func main() {
    fmt.Println(reverseASCII("hello"))     // olleh
    fmt.Println(reverseUnicode("hello"))   // olleh
    fmt.Println(reverseUnicode("café"))    // éfac  (correct!)
    fmt.Println(reverseASCII("café"))      // ??fac (WRONG for multi-byte!)
    fmt.Println(reverseUnicode("日本語"))   // 語本日
}
```

**Textual Figure — Reverse String (ASCII vs Unicode):**

```
  reverseASCII("hello"):
  Step 1: b = []byte{'h','e','l','l','o'}
  Two-pointer swap:
    i=0,j=4: swap h↔o  [o,e,l,l,h]
    i=1,j=3: swap e↔l  [o,l,l,e,h]
    i=2,j=2: done!      "olleh"  ✓

  reverseASCII("café"):  ✘ WRONG for multi-byte!
  Bytes: [63 61 66 c3 a9]   (é = 2 bytes)
  Reversed bytes: [a9 c3 66 61 63]  → garbled!

  reverseUnicode("café"):  ✓ Correct
  Step 1: r = []rune{'c','a','f','é'}  (4 runes)
  Two-pointer swap:
    i=0,j=3: swap c↔é  [é,a,f,c]
    i=1,j=2: swap a↔f  [é,f,a,c]
  Result: "éfac"  ✓

  Rule of thumb:
  ┌─────────────┬────────────────────────┐
  │ ASCII only  │ []byte is fine           │
  │ Unicode     │ Must use []rune          │
  └─────────────┴────────────────────────┘
```

---

## Example 10: Performance Comparison — String ops

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

func main() {
    n := 50000

    // Method 1: += concatenation — O(n²)
    start := time.Now()
    s := ""
    for i := 0; i < n; i++ {
        s += "x"
    }
    fmt.Printf("+=:        %v (len=%d)\n", time.Since(start), len(s))

    // Method 2: strings.Builder — O(n)
    start = time.Now()
    var sb strings.Builder
    for i := 0; i < n; i++ {
        sb.WriteByte('x')
    }
    _ = sb.String()
    fmt.Printf("Builder:   %v\n", time.Since(start))

    // Method 3: []byte + string() — O(n)
    start = time.Now()
    buf := make([]byte, 0, n)
    for i := 0; i < n; i++ {
        buf = append(buf, 'x')
    }
    _ = string(buf)
    fmt.Printf("[]byte:    %v\n", time.Since(start))

    // Method 4: strings.Repeat — O(n)
    start = time.Now()
    _ = strings.Repeat("x", n)
    fmt.Printf("Repeat:    %v\n", time.Since(start))

    // Method 5: strings.Join — O(n)
    start = time.Now()
    parts := make([]string, n)
    for i := range parts {
        parts[i] = "x"
    }
    _ = strings.Join(parts, "")
    fmt.Printf("Join:      %v\n", time.Since(start))
}
```

**Textual Figure — Performance Comparison of String Methods:**

```
  Building a string of n=50,000 'x' characters:

  ┌─────────────────┬────────┬─────────┬────────────────────────┐
  │ Method            │ Time    │ Space   │ How it works             │
  ├─────────────────┼────────┼─────────┼────────────────────────┤
  │ s += "x"          │ O(n²)   │ O(n²)   │ Copy entire string each  │
  │ strings.Builder   │ O(n)    │ O(n)    │ Amortized append         │
  │ []byte + string   │ O(n)    │ O(n)    │ Grow slice, convert once │
  │ strings.Repeat    │ O(n)    │ O(n)    │ Single allocation        │
  │ strings.Join      │ O(n)    │ O(n)    │ Pre-calculate total len  │
  └─────────────────┴────────┴─────────┴────────────────────────┘

  Why += is O(n²):
  "" + "x"       → copy 1     total: 1
  "x" + "x"      → copy 2     total: 3
  "xx" + "x"     → copy 3     total: 6
  ...            ...          ...
  n-1 chars + x  → copy n     total: n(n+1)/2 = O(n²)

  Recommendation: Always use strings.Builder for loops!
```

---

## Key Takeaways

1. **Go strings are immutable** — any modification creates a new string
2. **Use `strings.Builder`** for efficient string building: O(n) vs O(n²)
3. **`[]byte` for byte-level**, `[]rune` for character-level modifications
4. **Substrings share memory** — O(1) slicing but potential memory leaks
5. **`len(s)` returns bytes**, not characters — use `len([]rune(s))` for Unicode
6. **String == compares content**, not pointer (value semantics)

> **Next up:** Character Encoding →
