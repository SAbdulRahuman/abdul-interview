# Phase 3: Strings — Rabin-Karp Algorithm

## What is Rabin-Karp?

**Rabin-Karp** is a string pattern matching algorithm that uses **rolling hash** to find occurrences of a pattern in a text. It hashes the pattern once, then slides a window across the text, updating the hash in O(1) per step.

- **Average case**: O(n + m) — where n = text length, m = pattern length
- **Worst case**: O(n × m) — when all hashes collide (rare with good hash)
- **Best for**: Multiple pattern search, plagiarism detection

---

## Example 1: Classic Rabin-Karp

```go
package main

import "fmt"

func rabinKarp(text, pattern string) []int {
    const (
        base = 256
        mod  = 101 // prime
    )

    n, m := len(text), len(pattern)
    if m > n {
        return nil
    }

    // Compute base^(m-1) % mod
    highPow := 1
    for i := 0; i < m-1; i++ {
        highPow = (highPow * base) % mod
    }

    // Initial hashes
    patHash, txtHash := 0, 0
    for i := 0; i < m; i++ {
        patHash = (patHash*base + int(pattern[i])) % mod
        txtHash = (txtHash*base + int(text[i])) % mod
    }

    var result []int
    for i := 0; i <= n-m; i++ {
        if patHash == txtHash {
            // Verify character by character
            if text[i:i+m] == pattern {
                result = append(result, i)
            }
        }
        // Roll the hash
        if i < n-m {
            txtHash = ((txtHash-int(text[i])*highPow)*base + int(text[i+m])) % mod
            if txtHash < 0 {
                txtHash += mod
            }
        }
    }
    return result
}

func main() {
    text := "AABAACAADAABAABA"
    pattern := "AABA"
    fmt.Printf("Pattern %q found at indices: %v\n", pattern, rabinKarp(text, pattern))
    // [0 9 12]
}
```

---

## Example 2: Rabin-Karp with Large Prime Mod

```go
package main

import "fmt"

func rabinKarpLargeMod(text, pattern string) []int {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    n, m := len(text), len(pattern)
    if m > n {
        return nil
    }

    highPow := int64(1)
    for i := 0; i < m-1; i++ {
        highPow = (highPow * base) % mod
    }

    patHash := int64(0)
    txtHash := int64(0)
    for i := 0; i < m; i++ {
        patHash = (patHash*base + int64(pattern[i]-'a'+1)) % mod
        txtHash = (txtHash*base + int64(text[i]-'a'+1)) % mod
    }

    var result []int
    for i := 0; i <= n-m; i++ {
        if patHash == txtHash && text[i:i+m] == pattern {
            result = append(result, i)
        }
        if i < n-m {
            txtHash = ((txtHash-int64(text[i]-'a'+1)*highPow%mod+mod)*base + int64(text[i+m]-'a'+1)) % mod
        }
    }
    return result
}

func main() {
    fmt.Println(rabinKarpLargeMod("abcabcabc", "abc"))  // [0 3 6]
    fmt.Println(rabinKarpLargeMod("aaaaa", "aa"))        // [0 1 2 3]
    fmt.Println(rabinKarpLargeMod("hello world", "world")) // [6]
}
```

---

## Example 3: Multi-Pattern Rabin-Karp

```go
package main

import "fmt"

func rabinKarpMulti(text string, patterns []string) map[string][]int {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    result := make(map[string][]int)
    n := len(text)

    // Group patterns by length
    byLength := make(map[int]map[int64][]string)
    for _, p := range patterns {
        m := len(p)
        if _, ok := byLength[m]; !ok {
            byLength[m] = make(map[int64][]string)
        }
        h := int64(0)
        for _, c := range []byte(p) {
            h = (h*base + int64(c-'a'+1)) % mod
        }
        byLength[m][h] = append(byLength[m][h], p)
    }

    for m, patHashes := range byLength {
        if m > n {
            continue
        }

        highPow := int64(1)
        for i := 0; i < m-1; i++ {
            highPow = (highPow * base) % mod
        }

        txtHash := int64(0)
        for i := 0; i < m; i++ {
            txtHash = (txtHash*base + int64(text[i]-'a'+1)) % mod
        }

        for i := 0; i <= n-m; i++ {
            if pats, ok := patHashes[txtHash]; ok {
                sub := text[i : i+m]
                for _, p := range pats {
                    if sub == p {
                        result[p] = append(result[p], i)
                    }
                }
            }
            if i < n-m {
                txtHash = ((txtHash-int64(text[i]-'a'+1)*highPow%mod+mod)*base + int64(text[i+m]-'a'+1)) % mod
            }
        }
    }
    return result
}

func main() {
    text := "the quick brown fox jumps over the lazy dog"
    patterns := []string{"the", "fox", "dog", "lazy"}
    res := rabinKarpMulti(text, patterns)
    for p, indices := range res {
        fmt.Printf("%q → %v\n", p, indices)
    }
}
```

---

## Example 4: Double Hash Rabin-Karp (Collision-Safe)

```go
package main

import "fmt"

func rabinKarpDouble(text, pattern string) []int {
    const (
        b1  = 31
        m1  = 1_000_000_007
        b2  = 37
        m2  = 1_000_000_009
    )

    n, k := len(text), len(pattern)
    if k > n {
        return nil
    }

    hp1, hp2 := int64(1), int64(1)
    for i := 0; i < k-1; i++ {
        hp1 = (hp1 * b1) % m1
        hp2 = (hp2 * b2) % m2
    }

    var ph1, ph2, th1, th2 int64
    for i := 0; i < k; i++ {
        pc := int64(pattern[i])
        tc := int64(text[i])
        ph1 = (ph1*b1 + pc) % m1
        ph2 = (ph2*b2 + pc) % m2
        th1 = (th1*b1 + tc) % m1
        th2 = (th2*b2 + tc) % m2
    }

    var result []int
    for i := 0; i <= n-k; i++ {
        if th1 == ph1 && th2 == ph2 {
            result = append(result, i) // double hash match ≈ guaranteed correct
        }
        if i < n-k {
            tc_out := int64(text[i])
            tc_in := int64(text[i+k])
            th1 = ((th1-tc_out*hp1%m1+m1)*b1 + tc_in) % m1
            th2 = ((th2-tc_out*hp2%m2+m2)*b2 + tc_in) % m2
        }
    }
    return result
}

func main() {
    text := "abracadabra"
    fmt.Println(rabinKarpDouble(text, "abra")) // [0 7]
    fmt.Println(rabinKarpDouble(text, "cad"))  // [4]
}
```

---

## Example 5: Rabin-Karp for Longest Duplicate Substring

```go
package main

import "fmt"

func longestDupSubstring(s string) string {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    check := func(length int) string {
        if length == 0 {
            return ""
        }
        highPow := int64(1)
        for i := 0; i < length-1; i++ {
            highPow = (highPow * base) % mod
        }

        seen := make(map[int64][]int)
        h := int64(0)
        for i := 0; i < len(s); i++ {
            h = (h*base + int64(s[i]-'a'+1)) % mod
            if i >= length {
                h = (h - int64(s[i-length]-'a'+1)*highPow%mod*base%mod + mod*2) % mod
            }
            if i >= length-1 {
                start := i - length + 1
                if starts, ok := seen[h]; ok {
                    sub := s[start : start+length]
                    for _, prev := range starts {
                        if s[prev:prev+length] == sub {
                            return sub
                        }
                    }
                }
                seen[h] = append(seen[h], start)
            }
        }
        return ""
    }

    lo, hi := 0, len(s)-1
    result := ""
    for lo <= hi {
        mid := (lo + hi) / 2
        if sub := check(mid); sub != "" {
            result = sub
            lo = mid + 1
        } else {
            hi = mid - 1
        }
    }
    return result
}

func main() {
    fmt.Println(longestDupSubstring("banana"))    // "ana"
    fmt.Println(longestDupSubstring("abcd"))      // ""
    fmt.Println(longestDupSubstring("aaaaaa"))    // "aaaaa"
}
```

---

## Example 6: Counting Occurrences of a Pattern

```go
package main

import "fmt"

func countOccurrences(text, pattern string) int {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    n, m := len(text), len(pattern)
    if m > n {
        return 0
    }

    highPow := int64(1)
    for i := 0; i < m-1; i++ {
        highPow = (highPow * base) % mod
    }

    patHash := int64(0)
    txtHash := int64(0)
    for i := 0; i < m; i++ {
        patHash = (patHash*base + int64(pattern[i])) % mod
        txtHash = (txtHash*base + int64(text[i])) % mod
    }

    count := 0
    for i := 0; i <= n-m; i++ {
        if patHash == txtHash && text[i:i+m] == pattern {
            count++
        }
        if i < n-m {
            txtHash = ((txtHash-int64(text[i])*highPow%mod+mod)*base + int64(text[i+m])) % mod
        }
    }
    return count
}

func main() {
    fmt.Println(countOccurrences("aaaa", "aa"))           // 3
    fmt.Println(countOccurrences("abababab", "abab"))     // 3
    fmt.Println(countOccurrences("hello world", "xyz"))   // 0
    fmt.Println(countOccurrences("abcabc", "abc"))        // 2
}
```

---

## Example 7: First Unique Substring of Length K

```go
package main

import "fmt"

func firstUniqueSubstring(s string, k int) string {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    n := len(s)
    if k > n {
        return ""
    }

    highPow := int64(1)
    for i := 0; i < k-1; i++ {
        highPow = (highPow * base) % mod
    }

    // First pass: count occurrences
    freq := make(map[int64]int)
    first := make(map[int64]int) // first occurrence index
    h := int64(0)

    for i := 0; i < n; i++ {
        h = (h*base + int64(s[i])) % mod
        if i >= k {
            h = (h - int64(s[i-k])*highPow%mod*base%mod + mod*2) % mod
        }
        if i >= k-1 {
            start := i - k + 1
            freq[h]++
            if _, exists := first[h]; !exists {
                first[h] = start
            }
        }
    }

    // Find first hash with count 1
    bestIdx := n
    for hash, count := range freq {
        if count == 1 && first[hash] < bestIdx {
            bestIdx = first[hash]
        }
    }

    if bestIdx < n {
        return s[bestIdx : bestIdx+k]
    }
    return ""
}

func main() {
    fmt.Println(firstUniqueSubstring("aabaabaab", 3)) // first unique 3-char substring
    fmt.Println(firstUniqueSubstring("abcabc", 3))     // ""  (both "abc" appear twice)
    fmt.Println(firstUniqueSubstring("abcdef", 2))     // "ab" (all are unique, first wins)
}
```

---

## Example 8: Rabin-Karp on Binary Strings

```go
package main

import "fmt"

func searchBinary(text, pattern string) []int {
    const mod = 1_000_000_007

    n, m := len(text), len(pattern)
    if m > n {
        return nil
    }

    // For binary: base = 2
    highPow := int64(1)
    for i := 0; i < m-1; i++ {
        highPow = (highPow * 2) % mod
    }

    patHash := int64(0)
    txtHash := int64(0)
    for i := 0; i < m; i++ {
        patHash = (patHash*2 + int64(pattern[i]-'0')) % mod
        txtHash = (txtHash*2 + int64(text[i]-'0')) % mod
    }

    var result []int
    for i := 0; i <= n-m; i++ {
        if patHash == txtHash && text[i:i+m] == pattern {
            result = append(result, i)
        }
        if i < n-m {
            txtHash = ((txtHash-int64(text[i]-'0')*highPow%mod+mod)*2 + int64(text[i+m]-'0')) % mod
        }
    }
    return result
}

func main() {
    text := "110101101011011010"
    pattern := "1010"
    fmt.Printf("Binary pattern %q found at: %v\n", pattern, searchBinary(text, pattern))
}
```

---

## Example 9: Rabin-Karp with Wildcard Support

```go
package main

import "fmt"

// '*' in pattern matches any single character
func rabinKarpWildcard(text, pattern string) []int {
    n, m := len(text), len(pattern)
    if m > n {
        return nil
    }

    var result []int
    for i := 0; i <= n-m; i++ {
        match := true
        for j := 0; j < m; j++ {
            if pattern[j] != '*' && pattern[j] != text[i+j] {
                match = false
                break
            }
        }
        if match {
            result = append(result, i)
        }
    }
    return result
}

// Optimized: hash only non-wildcard positions
func rabinKarpWildcardHash(text, pattern string) []int {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    n, m := len(text), len(pattern)
    if m > n {
        return nil
    }

    // Compute pattern hash (skip wildcards)
    patHash := int64(0)
    for j := 0; j < m; j++ {
        if pattern[j] != '*' {
            patHash = (patHash + int64(pattern[j])*modPow(base, int64(j), mod)) % mod
        }
    }

    var result []int
    for i := 0; i <= n-m; i++ {
        txtHash := int64(0)
        for j := 0; j < m; j++ {
            if pattern[j] != '*' {
                txtHash = (txtHash + int64(text[i+j])*modPow(base, int64(j), mod)) % mod
            }
        }
        if txtHash == patHash && matchWithWildcard(text[i:i+m], pattern) {
            result = append(result, i)
        }
    }
    return result
}

func matchWithWildcard(s, p string) bool {
    for i := 0; i < len(p); i++ {
        if p[i] != '*' && p[i] != s[i] {
            return false
        }
    }
    return true
}

func modPow(b, e, m int64) int64 {
    r := int64(1)
    b %= m
    for e > 0 {
        if e%2 == 1 {
            r = r * b % m
        }
        e /= 2
        b = b * b % m
    }
    return r
}

func main() {
    text := "abcaecafc"
    pattern := "a*c"
    fmt.Printf("Wildcard %q found at: %v\n", pattern, rabinKarpWildcard(text, pattern))
    // [0 3 6]
}
```

---

## Example 10: Rabin-Karp vs Naive — Performance Comparison

```go
package main

import (
    "fmt"
    "strings"
    "time"
)

func naiveSearch(text, pattern string) int {
    count := 0
    n, m := len(text), len(pattern)
    for i := 0; i <= n-m; i++ {
        if text[i:i+m] == pattern {
            count++
        }
    }
    return count
}

func rabinKarpCount(text, pattern string) int {
    const (
        base = 256
        mod  = 1_000_000_007
    )

    n, m := len(text), len(pattern)
    if m > n {
        return 0
    }

    highPow := int64(1)
    for i := 0; i < m-1; i++ {
        highPow = (highPow * base) % mod
    }

    ph, th := int64(0), int64(0)
    for i := 0; i < m; i++ {
        ph = (ph*base + int64(pattern[i])) % mod
        th = (th*base + int64(text[i])) % mod
    }

    count := 0
    for i := 0; i <= n-m; i++ {
        if ph == th && text[i:i+m] == pattern {
            count++
        }
        if i < n-m {
            th = ((th-int64(text[i])*highPow%mod+mod)*base + int64(text[i+m])) % mod
        }
    }
    return count
}

func main() {
    // Build a large text
    text := strings.Repeat("abcdefghij", 100000) // 1M chars
    pattern := "fghij"

    start := time.Now()
    c1 := naiveSearch(text, pattern)
    naiveTime := time.Since(start)

    start = time.Now()
    c2 := rabinKarpCount(text, pattern)
    rkTime := time.Since(start)

    fmt.Printf("Naive:      count=%d  time=%v\n", c1, naiveTime)
    fmt.Printf("Rabin-Karp: count=%d  time=%v\n", c2, rkTime)
    // Rabin-Karp is typically faster for long patterns or multi-pattern search
}
```

---

## Rabin-Karp Complexity Summary

| Scenario           | Time       | Space |
|-------------------|------------|-------|
| Single pattern    | O(n + m) avg | O(1) |
| Worst case        | O(n × m)    | O(1) |
| Multi-pattern (k) | O(n × k_lengths + Σm) | O(k) |
| With binary search | O(n log n) | O(n) |

## Key Takeaways

1. **Rabin-Karp = rolling hash + verification**
2. **Average O(n+m)** — hash collisions are rare with good mod
3. **Best for multi-pattern** — hash all patterns, check each window
4. **Double hashing** practically eliminates false positives
5. **Always verify** — never trust hash match alone in production
6. **Binary search + Rabin-Karp** → powerful for optimization problems

> **Next up:** KMP Algorithm →
