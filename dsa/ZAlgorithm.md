# Phase 3: Strings — Z Algorithm

## What is the Z Algorithm?

The **Z algorithm** constructs a **Z-array** where `Z[i]` is the length of the longest substring starting at position `i` that matches a prefix of the string. It runs in **O(n)** time.

**Pattern matching**: Concatenate `pattern + "$" + text`, then every position where `Z[i] == len(pattern)` is a match.

---

## Example 1: Building the Z-Array

```go
package main

import "fmt"

func buildZArray(s string) []int {
    n := len(s)
    z := make([]int, n)
    // z[0] is undefined (or set to n by convention)

    l, r := 0, 0 // [l, r) is the Z-box
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    strings := []string{"aabxaab", "aaaaaa", "abcabcabc", "aabaaab"}
    for _, s := range strings {
        fmt.Printf("String: %q\n", s)
        fmt.Printf("Z:      %v\n\n", buildZArray(s))
    }
    // "aabxaab" → [0 1 0 0 3 1 0]
    // "aaaaaa"  → [0 5 4 3 2 1]
    // "abcabcabc" → [0 0 0 6 0 0 3 0 0]
}
```

**Textual Figure: Z-Array Construction for "aabxaab"**

```
┌─────────────────────────────────────────────────────────────────┐
│  Input String                                                   │
│  Index:   0     1     2     3     4     5     6                 │
│  Char:  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐             │
│         │ a │ │ a │ │ b │ │ x │ │ a │ │ a │ │ b │             │
│         └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘             │
├─────────────────────────────────────────────────────────────────┤
│  Step-by-Step Z-box [L, R) Tracking                             │
│                                                                 │
│  i=1 │ i≥R → compare from scratch                              │
│      │ s[0]='a' == s[1]='a' ✓  extend                          │
│      │ s[1]='a' != s[2]='b' ✗  stop                            │
│      │ z[1]=1     Z-box → [L=1, R=2)                           │
│      │   a [a] b   x   a   a   b                               │
│      │       ↑ matched 1 char                                   │
│                                                                 │
│  i=2 │ i≥R → s[0]='a' != s[2]='b' ✗                            │
│      │ z[2]=0     Z-box unchanged                               │
│                                                                 │
│  i=3 │ i≥R → s[0]='a' != s[3]='x' ✗                            │
│      │ z[3]=0     Z-box unchanged                               │
│                                                                 │
│  i=4 │ i≥R → compare from scratch                              │
│      │ s[0]='a' == s[4]='a' ✓                                  │
│      │ s[1]='a' == s[5]='a' ✓                                  │
│      │ s[2]='b' == s[6]='b' ✓                                  │
│      │ i+z[i]=7 reached end → stop                              │
│      │ z[4]=3     Z-box → [L=4, R=7)                           │
│      │   a   a   b   x  [a   a   b]  ← Z-box window            │
│      │                    └───┬───┘                              │
│      │                  matches prefix "aab"                     │
│                                                                 │
│  i=5 │ i<R=7 → inside Z-box                                    │
│      │ z[5] = min(R−i, z[i−L]) = min(2, z[1]) = min(2,1) = 1  │
│      │ Try extend: s[1]='a' != s[6]='b' ✗                      │
│      │ z[5]=1     Z-box unchanged                               │
│                                                                 │
│  i=6 │ i<R=7 → inside Z-box                                    │
│      │ z[6] = min(R−i, z[i−L]) = min(1, z[2]) = min(1,0) = 0  │
│      │ Try extend: s[0]='a' != s[6]='b' ✗                      │
│      │ z[6]=0     Z-box unchanged                               │
├─────────────────────────────────────────────────────────────────┤
│  Final Z-Array                                                  │
│  ┌───────┬───┬───┬───┬───┬───┬───┬───┐                         │
│  │ Index │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │                         │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                         │
│  │ Char  │ a │ a │ b │ x │ a │ a │ b │                         │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                         │
│  │ Z[i]  │ — │ 1 │ 0 │ 0 │ 3 │ 1 │ 0 │                         │
│  └───────┴───┴───┴───┴───┴───┴───┴───┘                         │
│  Z[4]=3 → s[4..6]="aab" matches prefix s[0..2]="aab"           │
│  Z[1]=1, Z[5]=1 → single char 'a' matches prefix "a"           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Example 2: Z-Algorithm Pattern Matching

```go
package main

import "fmt"

func zSearch(text, pattern string) []int {
    // Concatenate pattern + "$" + text
    combined := pattern + "$" + text
    z := buildZArray(combined)

    m := len(pattern)
    var result []int
    for i := m + 1; i < len(combined); i++ {
        if z[i] == m {
            result = append(result, i-m-1) // position in original text
        }
    }
    return result
}

func buildZArray(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    text := "AABAACAADAABAABA"
    pattern := "AABA"
    fmt.Printf("Pattern %q found at: %v\n", pattern, zSearch(text, pattern))
    // [0 9 12]
}
```

**Textual Figure: Z-Algorithm Pattern Matching — "AABA" in "AABAACAADAABAABA"**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Step 1: Concatenate  pattern + "$" + text                          │
│                                                                      │
│  Index: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19  │
│  Char:  A  A  B  A  $  A  A  B  A  A  C  A  A  D  A  A  B  A  A  B  │
│         ├─ pattern ─┤     ├──────────── text ─────────────────────┤   │
│                     ↑                                                │
│                 separator                                            │
│                                                                      │
│  (Full combined string is 21 chars: index 20 = 'A')                  │
├──────────────────────────────────────────────────────────────────────┤
│  Step 2: Build Z-array on combined string                            │
│                                                                      │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐  │
│  │ i │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │13 │14 │15 │16 │17 │18 │19 │  │
│  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤  │
│  │chr│ A │ A │ B │ A │ A │ C │ A │ A │ D │ A │ A │ B │ A │ A │ B │  │
│  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤  │
│  │Z[i]│ 4│ 1│ 0│ 1│ 2│ 0│ 1│ 1│ 0│ 4│ 1│ 0│ 4│ 1│ 0│              │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘  │
│        ★                                 ★           ★              │
│     Z[5]=4                           Z[14]=4      Z[17]=4            │
├──────────────────────────────────────────────────────────────────────┤
│  Step 3: Where Z[i] == m == 4  →  match!                            │
│                                                                      │
│   i= 5: Z[ 5]=4 → text pos = 5−4−1 =  0   ✓                       │
│   i=14: Z[14]=4 → text pos = 14−4−1 = 9   ✓                       │
│   i=17: Z[17]=4 → text pos = 17−4−1 = 12  ✓                       │
├──────────────────────────────────────────────────────────────────────┤
│  Matches highlighted in text:                                        │
│                                                                      │
│  Position: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15           │
│  Text:     A  A  B  A  A  C  A  A  D  A  A  B  A  A  B  A           │
│            ╚══╤══╤══╗                                                │
│            └─ AABA ─┘  pos 0                                        │
│                                       ╚══╤══╤══╗                    │
│                                       └─ AABA ─┘  pos 9            │
│                                                ╚══╤══╤══╗           │
│                                                └─ AABA ─┘ pos 12   │
├──────────────────────────────────────────────────────────────────────┤
│  Result: Pattern "AABA" found at positions [0, 9, 12]                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Example 3: Step-by-Step Z-Array Construction

```go
package main

import "fmt"

func buildZArrayVerbose(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0

    for i := 1; i < n; i++ {
        fmt.Printf("\ni=%d, char='%c', Z-box=[%d,%d)\n", i, s[i], l, r)

        if i < r {
            z[i] = min(r-i, z[i-l])
            fmt.Printf("  Inside Z-box: z[%d] initialized to min(%d, %d) = %d\n",
                i, r-i, z[i-l], z[i])
        }

        // Extend
        extensions := 0
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
            extensions++
        }
        if extensions > 0 {
            fmt.Printf("  Extended %d times, z[%d] = %d\n", extensions, i, z[i])
        }

        if i+z[i] > r {
            fmt.Printf("  Updated Z-box: [%d, %d) → [%d, %d)\n", l, r, i, i+z[i])
            l, r = i, i+z[i]
        }
    }

    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    s := "aabxaab"
    z := buildZArrayVerbose(s)
    fmt.Printf("\nFinal Z-array for %q: %v\n", s, z)
}
```

**Textual Figure: Verbose Z-Array Trace for "aabxaab"**

```
┌──────────────────────────────────────────────────────────────────┐
│  Input: s = "aabxaab"   (n = 7)                                 │
│  Index:  0   1   2   3   4   5   6                               │
│  Char:   a   a   b   x   a   a   b                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─ i=1, char='a', Z-box=[0,0) ──────────────────────────────┐  │
│  │  Not inside Z-box (i ≥ R)                                  │  │
│  │  Compare s[0]='a' vs s[1]='a' → ✓ extend (1)              │  │
│  │  Compare s[1]='a' vs s[2]='b' → ✗ stop                    │  │
│  │  z[1] = 1                                                  │  │
│  │  Z-box updated: [0,0) ──→ [1,2)                           │  │
│  └────────────────────────────────────────────────────────────┘  │
│       a  [a]  b   x   a   a   b       Z-box: [1,2)              │
│                                                                  │
│  ┌─ i=2, char='b', Z-box=[1,2) ──────────────────────────────┐  │
│  │  Not inside Z-box (i ≥ R)                                  │  │
│  │  Compare s[0]='a' vs s[2]='b' → ✗ stop                    │  │
│  │  z[2] = 0   (no extensions)                                │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ i=3, char='x', Z-box=[1,2) ──────────────────────────────┐  │
│  │  Not inside Z-box (i ≥ R)                                  │  │
│  │  Compare s[0]='a' vs s[3]='x' → ✗ stop                    │  │
│  │  z[3] = 0   (no extensions)                                │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ i=4, char='a', Z-box=[1,2) ──────────────────────────────┐  │
│  │  Not inside Z-box (i ≥ R)                                  │  │
│  │  Compare s[0]='a' vs s[4]='a' → ✓ extend (1)              │  │
│  │  Compare s[1]='a' vs s[5]='a' → ✓ extend (2)              │  │
│  │  Compare s[2]='b' vs s[6]='b' → ✓ extend (3)              │  │
│  │  i+z[i] = 7 = n → stop (end of string)                    │  │
│  │  z[4] = 3                                                  │  │
│  │  Z-box updated: [1,2) ──→ [4,7)                           │  │
│  └────────────────────────────────────────────────────────────┘  │
│       a   a   b   x  [a   a   b]      Z-box: [4,7)              │
│                        ═══════════ matches prefix "aab"          │
│                                                                  │
│  ┌─ i=5, char='a', Z-box=[4,7) ──────────────────────────────┐  │
│  │  Inside Z-box! z[5] = min(R−i, z[i−L])                    │  │
│  │                     = min(7−5, z[5−4])                     │  │
│  │                     = min(2, z[1]) = min(2, 1) = 1         │  │
│  │  Try extend: s[1]='a' vs s[6]='b' → ✗ stop                │  │
│  │  z[5] = 1   (Z-box not updated, i+z[i]=6 ≤ R=7)          │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ i=6, char='b', Z-box=[4,7) ──────────────────────────────┐  │
│  │  Inside Z-box! z[6] = min(R−i, z[i−L])                    │  │
│  │                     = min(7−6, z[6−4])                     │  │
│  │                     = min(1, z[2]) = min(1, 0) = 0         │  │
│  │  Try extend: s[0]='a' vs s[6]='b' → ✗ stop                │  │
│  │  z[6] = 0   (Z-box not updated)                           │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│  Final Z-array for "aabxaab":                                    │
│  ┌───────┬───┬───┬───┬───┬───┬───┬───┐                          │
│  │ Index │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │                          │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                          │
│  │ Char  │ a │ a │ b │ x │ a │ a │ b │                          │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                          │
│  │ Z[i]  │ — │ 1 │ 0 │ 0 │ 3 │ 1 │ 0 │                          │
│  └───────┴───┴───┴───┴───┴───┴───┴───┘                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example 4: Count Occurrences Using Z

```go
package main

import "fmt"

func countOccurrences(text, pattern string) int {
    combined := pattern + "$" + text
    z := buildZ(combined)

    count := 0
    m := len(pattern)
    for i := m + 1; i < len(combined); i++ {
        if z[i] == m {
            count++
        }
    }
    return count
}

func buildZ(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(countOccurrences("aaaa", "aa"))       // 3
    fmt.Println(countOccurrences("abcabcabc", "abc")) // 3
    fmt.Println(countOccurrences("hello", "xyz"))     // 0
}
```

**Textual Figure: Count Occurrences — pattern "aa" in text "aaaa"**

```
┌──────────────────────────────────────────────────────────────┐
│  Step 1: Concatenate  pattern + "$" + text                   │
│                                                              │
│  combined = "aa$aaaa"                                        │
│  Index:  0   1   2   3   4   5   6                           │
│  Char:   a   a   $   a   a   a   a                           │
│          ├─pat─┤     ├── text ──┤                            │
├──────────────────────────────────────────────────────────────┤
│  Step 2: Build Z-array                                       │
│                                                              │
│  i=1: s[0]='a'==s[1]='a' ✓, s[1]='a'!=s[2]='$' ✗           │
│       z[1]=1, Z-box=[1,2)                                    │
│  i=2: s[0]='a'!=s[2]='$' ✗ → z[2]=0                         │
│  i=3: s[0]='a'==s[3]='a' ✓, s[1]='a'==s[4]='a' ✓           │
│       s[2]='$'!=s[5]='a' ✗ → z[3]=2, Z-box=[3,5)            │
│  i=4: inside Z-box, z[4]=min(1,z[1])=1                       │
│       extend: s[1]='a'==s[5]='a' ✓ → z[4]=2, Z-box=[4,6)    │
│  i=5: inside Z-box, z[5]=min(1,z[1])=1                       │
│       extend: s[1]='a'==s[6]='a' ✓ → z[5]=2, Z-box=[5,7)    │
│  i=6: inside Z-box, z[6]=min(1,z[1])=1                       │
│       no further extension → z[6]=1                          │
│                                                              │
│  ┌───────┬───┬───┬───┬───┬───┬───┬───┐                      │
│  │ Index │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │                      │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                      │
│  │ Char  │ a │ a │ $ │ a │ a │ a │ a │                      │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                      │
│  │ Z[i]  │ — │ 1 │ 0 │ 2 │ 2 │ 2 │ 1 │                      │
│  └───────┴───┴───┴───┴───┴───┴───┴───┘                      │
├──────────────────────────────────────────────────────────────┤
│  Step 3: Count positions where Z[i] == m == 2                │
│                                                              │
│   i=3: Z[3]=2 == 2 ✓  → text pos 3−2−1 = 0                 │
│   i=4: Z[4]=2 == 2 ✓  → text pos 4−2−1 = 1                 │
│   i=5: Z[5]=2 == 2 ✓  → text pos 5−2−1 = 2                 │
│   i=6: Z[6]=1 != 2 ✗                                        │
│                                                              │
│  Text "aaaa" with overlapping matches:                       │
│   [a a] a  a    pos 0                                        │
│    a[a  a] a    pos 1                                        │
│    a  a[a  a]   pos 2                                        │
├──────────────────────────────────────────────────────────────┤
│  Result: count = 3                                           │
└──────────────────────────────────────────────────────────────┘
```

---

## Example 5: Shortest Period of a String

```go
package main

import "fmt"

func shortestPeriod(s string) int {
    n := len(s)
    z := buildZ(s)

    for p := 1; p < n; p++ {
        // Check if period p works
        // s has period p iff every z[i] >= n-i for positions that are multiples of p
        valid := true
        for i := p; i < n; i += p {
            needed := n - i
            if needed > p {
                needed = p
            }
            if z[i] < needed {
                valid = false
                break
            }
        }
        if valid {
            return p
        }
    }
    return n // string itself
}

func buildZ(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    tests := []string{"abcabcabc", "abab", "aaaa", "abcde"}
    for _, s := range tests {
        p := shortestPeriod(s)
        fmt.Printf("%q → period %d (%q)\n", s, p, s[:p])
    }
    // "abcabcabc" → 3 ("abc")
    // "abab"      → 2 ("ab")
    // "aaaa"      → 1 ("a")
    // "abcde"     → 5 ("abcde")
}
```

**Textual Figure: Shortest Period Detection for "abcabcabc"**

```
┌──────────────────────────────────────────────────────────────────┐
│  Input: s = "abcabcabc"   (n = 9)                                │
│                                                                  │
│  Index:  0   1   2   3   4   5   6   7   8                      │
│  Char:   a   b   c   a   b   c   a   b   c                      │
├──────────────────────────────────────────────────────────────────┤
│  Step 1: Build Z-array                                           │
│                                                                  │
│  ┌───────┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                  │
│  │ Index │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │                  │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┼───┼───┤                  │
│  │ Char  │ a │ b │ c │ a │ b │ c │ a │ b │ c │                  │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┼───┼───┤                  │
│  │ Z[i]  │ — │ 0 │ 0 │ 6 │ 0 │ 0 │ 3 │ 0 │ 0 │                  │
│  └───────┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                  │
├──────────────────────────────────────────────────────────────────┤
│  Step 2: Try candidate periods p = 1, 2, 3, ...                  │
│                                                                  │
│  p=1: i=1, needed=min(n−i, p)=min(8,1)=1                        │
│       z[1]=0 < 1 → ✗ invalid                                    │
│                                                                  │
│  p=2: i=2, needed=min(7,2)=2                                    │
│       z[2]=0 < 2 → ✗ invalid                                    │
│                                                                  │
│  p=3: Check all multiples of 3:                                  │
│       i=3: needed=min(6,3)=3, z[3]=6 ≥ 3 → ✓                   │
│       i=6: needed=min(3,3)=3, z[6]=3 ≥ 3 → ✓                   │
│       All passed → ✓ VALID PERIOD                                │
├──────────────────────────────────────────────────────────────────┤
│  Visualization of period p=3:                                    │
│                                                                  │
│  ┌───────────┬───────────┬───────────┐                           │
│  │  a  b  c  │  a  b  c  │  a  b  c  │                           │
│  └───────────┴───────────┴───────────┘                           │
│   repeat #1    repeat #2    repeat #3                            │
│   └─── period p=3 ───┘                                           │
│                                                                  │
│  "abcabcabc" = "abc" × 3                                        │
├──────────────────────────────────────────────────────────────────┤
│  Result: shortest period = 3, unit = "abc"                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example 6: Longest Prefix That Appears at Position i

```go
package main

import "fmt"

func prefixAppearances(s string) {
    z := buildZ(s)
    n := len(s)

    fmt.Printf("String: %q\n", s)
    fmt.Printf("Z-array: %v\n\n", z)

    for i := 1; i < n; i++ {
        if z[i] > 0 {
            fmt.Printf("Position %d: prefix %q (length %d) matches\n",
                i, s[:z[i]], z[i])
        }
    }
}

func buildZ(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    prefixAppearances("aabaaab")
    // Position 1: "a" matches
    // Position 3: "aa" matches
    // Position 4: "aab" matches (wait, need to verify)
    fmt.Println()
    prefixAppearances("abcabcabc")
}
```

**Textual Figure: Prefix Appearances in "aabaaab"**

```
┌──────────────────────────────────────────────────────────────────┐
│  Input: s = "aabaaab"   (n = 7)                                  │
│                                                                  │
│  Index:  0   1   2   3   4   5   6                               │
│  Char:   a   a   b   a   a   a   b                               │
├──────────────────────────────────────────────────────────────────┤
│  Z-Array Construction                                           │
│                                                                  │
│  ┌───────┬───┬───┬───┬───┬───┬───┬───┐                          │
│  │ Index │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │                          │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                          │
│  │ Char  │ a │ a │ b │ a │ a │ a │ b │                          │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┤                          │
│  │ Z[i]  │ — │ 1 │ 0 │ 2 │ 3 │ 1 │ 0 │                          │
│  └───────┴───┴───┴───┴───┴───┴───┴───┘                          │
├──────────────────────────────────────────────────────────────────┤
│  Prefix Matches (Z[i] > 0):                                      │
│                                                                  │
│  Pos 1 │ Z[1]=1 │ s[1..1]  = "a"   matches prefix "a"          │
│       │        │  a [a] b  a  a  a  b                            │
│       │        │    ↑                                              │
│  Pos 3 │ Z[3]=2 │ s[3..4]  = "aa"  matches prefix "aa"          │
│       │        │  a  a  b [a  a] a  b                            │
│       │        │          └──┘                                     │
│  Pos 4 │ Z[4]=3 │ s[4..6]  = "aab" matches prefix "aab"         │
│       │        │  a  a  b  a [a  a  b]                           │
│       │        │             └─────┘                              │
│  Pos 5 │ Z[5]=1 │ s[5..5]  = "a"   matches prefix "a"          │
│       │        │  a  a  b  a  a [a] b                            │
│       │        │                ↑                                  │
├──────────────────────────────────────────────────────────────────┤
│  Result: Prefix reappears at positions 1, 3, 4, 5               │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example 7: Z-Function for Matching with Multiple Patterns

```go
package main

import "fmt"

func multiPatternZ(text string, patterns []string) map[string][]int {
    result := make(map[string][]int)
    for _, pattern := range patterns {
        combined := pattern + "$" + text
        z := buildZ(combined)
        m := len(pattern)
        for i := m + 1; i < len(combined); i++ {
            if z[i] >= m {
                result[pattern] = append(result[pattern], i-m-1)
            }
        }
    }
    return result
}

func buildZ(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    text := "the cat sat on the mat"
    patterns := []string{"the", "at", "cat"}
    res := multiPatternZ(text, patterns)
    for p, positions := range res {
        fmt.Printf("%q → %v\n", p, positions)
    }
}
```

**Textual Figure: Multi-Pattern Z Matching in "the cat sat on the mat"**

```
┌──────────────────────────────────────────────────────────────────┐
│  Input text: "the cat sat on the mat"                            │
│  Patterns:   ["the", "at", "cat"]                                │
├──────────────────────────────────────────────────────────────────┤
│  Text layout:                                                    │
│  Index: 0  1  2  3  4  5  6  7  8  9 10 11 12 ...15 ...19 20 21 │
│  Char:  t  h  e     c  a  t     s  a  t     o     t     m  a  t  │
├──────────────────────────────────────────────────────────────────┤
│  Pattern-by-pattern Z search:                                    │
│                                                                  │
│  ┌─ Pattern "the" (m=3) ────────────────────────────────────┐  │
│  │  combined = "the$the cat sat on the mat"                    │  │
│  │  Z[i]==3 at text positions: 0, 15                           │  │
│  │  "the" found → [0, 15]                                     │  │
│  │    [t h e] c a t   s a t   o n   [t h e]  m a t             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ Pattern "at" (m=2) ─────────────────────────────────────┐  │
│  │  combined = "at$the cat sat on the mat"                     │  │
│  │  Z[i]==2 at text positions: 5, 9, 20                        │  │
│  │  "at" found → [5, 9, 20]                                   │  │
│  │    t h e   c[a t]  s[a t]  o n   t h e   m[a t]             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ Pattern "cat" (m=3) ────────────────────────────────────┐  │
│  │  combined = "cat$the cat sat on the mat"                    │  │
│  │  Z[i]==3 at text position: 4                                │  │
│  │  "cat" found → [4]                                         │  │
│  │    t h e  [c a t]  s a t   o n   t h e   m a t              │  │
│  └────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────┤
│  Result:                                                         │
│    "the" → [0, 15]                                              │
│    "at"  → [5, 9, 20]                                           │
│    "cat" → [4]                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example 8: Longest Common Prefix Between Suffix and Full String

```go
package main

import "fmt"

// For each suffix s[i..], find length of longest common prefix with s
func longestCommonPrefixZ(s string) {
    z := buildZ(s)
    n := len(s)

    fmt.Printf("String: %q (length %d)\n\n", s, n)
    for i := 1; i < n; i++ {
        fmt.Printf("Suffix s[%d..] = %q  LCP with s = %d",
            i, s[i:], z[i])
        if z[i] > 0 {
            fmt.Printf("  (%q)", s[:z[i]])
        }
        fmt.Println()
    }
}

func buildZ(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    longestCommonPrefixZ("aabxaab")
}
```

**Textual Figure: Longest Common Prefix (Suffix vs Full String) for "aabxaab"**

```
┌──────────────────────────────────────────────────────────────────┐
│  Input: s = "aabxaab"   (n = 7)                                  │
│                                                                  │
│  Z-Array: [—, 1, 0, 0, 3, 1, 0]                                 │
├──────────────────────────────────────────────────────────────────┤
│  Suffix-by-suffix LCP with full string:                          │
│                                                                  │
│  Full string:    a  a  b  x  a  a  b                             │
│                                                                  │
│  ┌────┬───────────────┬─────┬─────────────────────────┐     │
│  │ i  │  Suffix s[i..] │ Z[i] │    LCP Match               │     │
│  ├────┼───────────────┼─────┼─────────────────────────┤     │
│  │  1 │  "abxaab"      │  1   │    "a" matches prefix     │     │
│  ├────┼───────────────┼─────┼─────────────────────────┤     │
│  │  2 │  "bxaab"       │  0   │    (no common prefix)     │     │
│  ├────┼───────────────┼─────┼─────────────────────────┤     │
│  │  3 │  "xaab"        │  0   │    (no common prefix)     │     │
│  ├────┼───────────────┼─────┼─────────────────────────┤     │
│  │  4 │  "aab"         │  3   │    "aab" matches prefix   │     │
│  ├────┼───────────────┼─────┼─────────────────────────┤     │
│  │  5 │  "ab"          │  1   │    "a" matches prefix     │     │
│  ├────┼───────────────┼─────┼─────────────────────────┤     │
│  │  6 │  "b"           │  0   │    (no common prefix)     │     │
│  └────┴───────────────┴─────┴─────────────────────────┘     │
│                                                                  │
│  Visual alignment for Z[4]=3:                                    │
│    Full string:  a  a  b  x  a  a  b                             │
│    Suffix [4..]:             a  a  b                             │
│                              ════════  LCP = 3 ("aab")           │
├──────────────────────────────────────────────────────────────────┤
│  Result: Z[i] gives LCP of each suffix with the full string     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example 9: Check If String is a Concatenation of a Pattern

```go
package main

import "fmt"

// Check if text is formed by concatenating pattern one or more times
func isConcatenation(text, pattern string) bool {
    n, m := len(text), len(pattern)
    if n == 0 || m == 0 || n%m != 0 {
        return false
    }

    combined := pattern + "$" + text
    z := buildZ(combined)

    for i := m + 1; i < len(combined); i += m {
        pos := i - m - 1
        if z[i] < m {
            return false
        }
        _ = pos
    }
    return true
}

func buildZ(s string) []int {
    n := len(s)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && s[z[i]] == s[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }
    return z
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(isConcatenation("abcabcabc", "abc"))  // true
    fmt.Println(isConcatenation("abababab", "ab"))     // true
    fmt.Println(isConcatenation("abcabd", "abc"))      // false
    fmt.Println(isConcatenation("aaa", "a"))           // true
    fmt.Println(isConcatenation("abcab", "abc"))       // false (5 % 3 != 0)
}
```

**Textual Figure: Concatenation Check — "abcabcabc" vs pattern "abc"**

```
┌──────────────────────────────────────────────────────────────────┐
│  Input: text = "abcabcabc",  pattern = "abc"                      │
│  len(text)=9, len(pattern)=3, 9 % 3 == 0  ✓  (divisible)        │
├──────────────────────────────────────────────────────────────────┤
│  Step 1: Concatenate  pattern + "$" + text                       │
│                                                                  │
│  combined = "abc$abcabcabc"                                      │
│  Index:  0  1  2  3  4  5  6  7  8  9 10 11 12                   │
│  Char:   a  b  c  $  a  b  c  a  b  c  a  b  c                  │
│          ├─pat─┤     ├──────── text ────────┤                  │
├──────────────────────────────────────────────────────────────────┤
│  Step 2: Build Z-array on combined string                        │
│                                                                  │
│  ┌───────┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐  │
│  │ Index │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12│  │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤  │
│  │ Char  │ a │ b │ c │ $ │ a │ b │ c │ a │ b │ c │ a │ b │ c │  │
│  ├───────┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤  │
│  │ Z[i]  │ — │ 0 │ 0 │ 0 │ 3 │ 0 │ 0 │ 3 │ 0 │ 0 │ 3 │ 0 │ 0 │  │
│  └───────┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘  │
├──────────────────────────────────────────────────────────────────┤
│  Step 3: Check every m-th position (stride by m=3)               │
│                                                                  │
│    i = m+1 = 4:  Z[4]  = 3 ≥ m=3  ✓  (text[0..2] = "abc")      │
│    i = 4+3  = 7:  Z[7]  = 3 ≥ m=3  ✓  (text[3..5] = "abc")      │
│    i = 7+3  = 10: Z[10] = 3 ≥ m=3  ✓  (text[6..8] = "abc")      │
│                                                                  │
│    All positions passed → true                                  │
│                                                                  │
│  Visualization:                                                  │
│    ┌─────┬─────┬─────┐                                             │
│    │ abc │ abc │ abc │   =  "abc" × 3                             │
│    └─────┴─────┴─────┘                                             │
├──────────────────────────────────────────────────────────────────┤
│  Result: isConcatenation("abcabcabc", "abc") = true              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example 10: Z vs KMP Comparison

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

func zSearch(text, pattern string) int {
    combined := pattern + "$" + text
    n := len(combined)
    z := make([]int, n)
    l, r := 0, 0
    for i := 1; i < n; i++ {
        if i < r {
            z[i] = min(r-i, z[i-l])
        }
        for i+z[i] < n && combined[z[i]] == combined[i+z[i]] {
            z[i]++
        }
        if i+z[i] > r {
            l, r = i, i+z[i]
        }
    }

    count := 0
    m := len(pattern)
    for i := m + 1; i < n; i++ {
        if z[i] == m {
            count++
        }
    }
    return count
}

func kmpSearch(text, pattern string) int {
    m := len(pattern)
    lps := make([]int, m)
    length := 0
    for i := 1; i < m; {
        if pattern[i] == pattern[length] {
            length++
            lps[i] = length
            i++
        } else if length != 0 {
            length = lps[length-1]
        } else {
            i++
        }
    }

    count := 0
    j := 0
    for i := 0; i < len(text); i++ {
        for j > 0 && text[i] != pattern[j] {
            j = lps[j-1]
        }
        if text[i] == pattern[j] {
            j++
        }
        if j == m {
            count++
            j = lps[j-1]
        }
    }
    return count
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    text := strings.Repeat("abcabc", 100000) // 600K chars
    pattern := "abcabc"

    start := time.Now()
    c1 := zSearch(text, pattern)
    zTime := time.Since(start)

    start = time.Now()
    c2 := kmpSearch(text, pattern)
    kmpTime := time.Since(start)

    fmt.Printf("Z-algorithm: count=%d time=%v\n", c1, zTime)
    fmt.Printf("KMP:         count=%d time=%v\n", c2, kmpTime)
    // Both are O(n+m) — performance is comparable
}
```

**Textual Figure: Z-Algorithm vs KMP — Conceptual Comparison**

```
┌──────────────────────────────────────────────────────────────────┐
│  Text:    "abcabc" × 100000  (600K chars)                         │
│  Pattern: "abcabc"          (m = 6)                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│    Z-Algorithm Approach             KMP Approach                 │
│   ┌──────────────────────┐     ┌───────────────────────┐    │
│   │  1. Concat:           │     │  1. Build LPS array: │    │
│   │     pat + "$" + text  │     │     O(m) on pattern   │    │
│   │  2. Build Z-array:    │     │  2. Scan text with   │    │
│   │     O(n+m) combined   │     │     failure function   │    │
│   │  3. Scan Z for        │     │     O(n) scan          │    │
│   │     Z[i] == m          │     │  3. j falls back on  │    │
│   │                       │     │     mismatch via LPS   │    │
│   └──────────────────────┘     └───────────────────────┘    │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│  Example trace for Z on combined = "abcabc$abcabc..."            │
│                                                                  │
│  Index:  0  1  2  3  4  5  6  7  8  9  10 11 12 ...             │
│  Char:   a  b  c  a  b  c  $  a  b  c   a  b  c  ...            │
│  Z[i]:   —  0  0  3  0  0  0  6  0  0   3  0  0  ...            │
│                               ★                                  │
│                            Z[7]=6==m  →  match at text pos 0    │
│                                                                  │
│  Z-box window walks through the combined string,                 │
│  reusing previously computed prefix matches to stay O(n+m).      │
├──────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┬───────────────────────┬─────────────┐    │
│  │     Feature     │     Z Algorithm      │     KMP     │    │
│  ├─────────────────┼───────────────────────┼─────────────┤    │
│  │ Time           │ O(n+m)                │ O(n+m)      │    │
│  ├─────────────────┼───────────────────────┼─────────────┤    │
│  │ Space          │ O(n+m) combined str   │ O(m) LPS    │    │
│  ├─────────────────┼───────────────────────┼─────────────┤    │
│  │ Preprocessing  │ Z-array on combined   │ LPS on pat  │    │
│  ├─────────────────┼───────────────────────┼─────────────┤    │
│  │ Simplicity     │ Slightly simpler      │ More common │    │
│  └─────────────────┴───────────────────────┴─────────────┘    │
├──────────────────────────────────────────────────────────────────┤
│  Both algorithms yield same count with O(n+m) time.              │
│  KMP uses less space; Z is conceptually simpler.                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## Z-Algorithm Complexity

| Operation       | Time  | Space  |
|----------------|-------|--------|
| Build Z-array  | O(n)  | O(n)   |
| Pattern search | O(n+m)| O(n+m) |

## Z vs KMP Comparison

| Feature          | Z Algorithm          | KMP                    |
|-----------------|---------------------|------------------------|
| Preprocessing   | Z-array O(n)        | LPS array O(m)         |
| Extra space     | O(n + m) combined   | O(m) pattern only      |
| Concept         | Prefix matching at i | Failure function       |
| Implementation  | Slightly simpler    | Slightly more complex  |
| Both            | O(n + m) guaranteed | O(n + m) guaranteed    |

## Key Takeaways

1. **Z[i]** = length of longest substring starting at i matching a prefix
2. **Z-box optimization** makes construction O(n) not O(n²)
3. **Pattern matching**: concat `pattern + "$" + text`, check `Z[i] == m`
4. **Period detection**: smallest p where string can be built from repeating s[0:p]
5. **Simpler to understand** than KMP for many developers
6. **Same O(n+m) complexity** as KMP — choose based on preference

> **Next up:** Palindrome Detection →
