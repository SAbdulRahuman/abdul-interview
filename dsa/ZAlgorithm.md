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
