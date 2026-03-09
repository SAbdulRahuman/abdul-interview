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

---

## Key Takeaways

1. **Go strings are immutable** — any modification creates a new string
2. **Use `strings.Builder`** for efficient string building: O(n) vs O(n²)
3. **`[]byte` for byte-level**, `[]rune` for character-level modifications
4. **Substrings share memory** — O(1) slicing but potential memory leaks
5. **`len(s)` returns bytes**, not characters — use `len([]rune(s))` for Unicode
6. **String == compares content**, not pointer (value semantics)

> **Next up:** Character Encoding →
