# Phase 3: Strings — Rolling Hash

## What is a Rolling Hash?

A **rolling hash** updates the hash value incrementally as a fixed-size window slides over the string. Instead of recomputing the hash from scratch (O(k) per position), rolling hash updates in **O(1)** by removing the outgoing character and adding the incoming one.

**Formula:**
```
hash_new = (hash_old - s[i] × base^(k-1)) × base + s[i+k]   (mod m)
```

---

## Example 1: Basic Rolling Hash

```go
package main

import "fmt"

const (
    base = 31
    mod  = 1_000_000_007
)

func rollingHashes(s string, k int) []int64 {
    if k > len(s) {
        return nil
    }
    n := len(s)

    // Compute base^(k-1) mod m
    highPower := int64(1)
    for i := 0; i < k-1; i++ {
        highPower = (highPower * base) % mod
    }

    // Initial hash for first window
    hash := int64(0)
    for i := 0; i < k; i++ {
        hash = (hash*base + int64(s[i])) % mod
    }

    result := []int64{hash}

    // Slide the window
    for i := k; i < n; i++ {
        // Remove leftmost char, add new char
        hash = (hash - int64(s[i-k])*highPower%mod + mod) % mod
        hash = (hash*base + int64(s[i])) % mod
        result = append(result, hash)
    }
    return result
}

func main() {
    s := "abcdefgh"
    hashes := rollingHashes(s, 3)
    for i, h := range hashes {
        fmt.Printf("hash(%q) = %d\n", s[i:i+3], h)
    }
}
```

---

## Example 2: Pattern Search Using Rolling Hash

```go
package main

import "fmt"

func searchPattern(text, pattern string) []int {
    const (
        b = 256
        m = 101 // small prime for demo
    )
    n, k := len(text), len(pattern)
    if k > n {
        return nil
    }

    // base^(k-1)
    highPower := 1
    for i := 0; i < k-1; i++ {
        highPower = (highPower * b) % m
    }

    // Hash pattern and first window
    patHash, txtHash := 0, 0
    for i := 0; i < k; i++ {
        patHash = (patHash*b + int(pattern[i])) % m
        txtHash = (txtHash*b + int(text[i])) % m
    }

    var result []int
    for i := 0; i <= n-k; i++ {
        if patHash == txtHash {
            // Verify
            if text[i:i+k] == pattern {
                result = append(result, i)
            }
        }
        if i < n-k {
            txtHash = ((txtHash-int(text[i])*highPower)*b + int(text[i+k])) % m
            if txtHash < 0 {
                txtHash += m
            }
        }
    }
    return result
}

func main() {
    text := "AABAACAADAABAABA"
    pattern := "AABA"
    fmt.Println("Pattern found at:", searchPattern(text, pattern))
    // [0, 9, 12]
}
```

---

## Example 3: Multi-Pattern Search with Rolling Hash

```go
package main

import "fmt"

func multiPatternSearch(text string, patterns []string) map[string][]int {
    const (
        b = 31
        m = 1_000_000_007
    )

    result := make(map[string][]int)

    // Group patterns by length
    byLength := make(map[int][]string)
    for _, p := range patterns {
        byLength[len(p)] = append(byLength[len(p)], p)
    }

    for k, pats := range byLength {
        if k > len(text) {
            continue
        }

        // Hash all patterns of this length
        patHashes := make(map[int64][]string)
        for _, p := range pats {
            h := int64(0)
            for _, c := range []byte(p) {
                h = (h*b + int64(c)) % m
            }
            patHashes[h] = append(patHashes[h], p)
        }

        // Rolling hash over text
        highPow := int64(1)
        for i := 0; i < k-1; i++ {
            highPow = (highPow * b) % m
        }

        hash := int64(0)
        for i := 0; i < len(text); i++ {
            hash = (hash*b + int64(text[i])) % m
            if i >= k {
                hash = (hash - int64(text[i-k])*highPow%m*b%m + m*2) % m
            }
            if i >= k-1 {
                if candidates, ok := patHashes[hash]; ok {
                    sub := text[i-k+1 : i+1]
                    for _, p := range candidates {
                        if sub == p {
                            result[p] = append(result[p], i-k+1)
                        }
                    }
                }
            }
        }
    }
    return result
}

func main() {
    text := "she sells sea shells by the sea shore"
    patterns := []string{"she", "sea", "the"}
    res := multiPatternSearch(text, patterns)
    for p, positions := range res {
        fmt.Printf("%q found at: %v\n", p, positions)
    }
}
```

---

## Example 4: Cyclic Rotation Check

```go
package main

import "fmt"

func isRotation(s1, s2 string) bool {
    if len(s1) != len(s2) {
        return false
    }
    // Concatenate s1+s1 and search for s2
    doubled := s1 + s1

    const (
        b = 31
        m = 1_000_000_007
    )
    k := len(s2)

    highPow := int64(1)
    for i := 0; i < k-1; i++ {
        highPow = (highPow * b) % m
    }

    // Hash pattern
    patHash := int64(0)
    for i := 0; i < k; i++ {
        patHash = (patHash*b + int64(s2[i])) % m
    }

    // Rolling hash on doubled
    hash := int64(0)
    for i := 0; i < len(doubled); i++ {
        hash = (hash*b + int64(doubled[i])) % m
        if i >= k {
            hash = (hash - int64(doubled[i-k])*highPow%m*b%m + m*2) % m
        }
        if i >= k-1 && hash == patHash {
            if doubled[i-k+1:i+1] == s2 {
                return true
            }
        }
    }
    return false
}

func main() {
    fmt.Println(isRotation("abcde", "cdeab")) // true
    fmt.Println(isRotation("abcde", "abced")) // false
    fmt.Println(isRotation("aab", "aba"))      // true
}
```

---

## Example 5: Longest Repeated Substring (Binary Search + Rolling Hash)

```go
package main

import "fmt"

func longestRepeatedSubstring(s string) string {
    n := len(s)
    lo, hi := 0, n-1
    result := ""

    for lo <= hi {
        mid := (lo + hi) / 2
        if sub := findRepeated(s, mid); sub != "" {
            result = sub
            lo = mid + 1
        } else {
            hi = mid - 1
        }
    }
    return result
}

func findRepeated(s string, length int) string {
    if length == 0 {
        return ""
    }
    const (
        b = 31
        m = 1_000_000_007
    )

    highPow := int64(1)
    for i := 0; i < length-1; i++ {
        highPow = (highPow * b) % m
    }

    seen := make(map[int64]int) // hash → starting index
    hash := int64(0)

    for i := 0; i < len(s); i++ {
        hash = (hash*b + int64(s[i])) % m
        if i >= length {
            hash = (hash - int64(s[i-length])*highPow%m*b%m + m*2) % m
        }
        if i >= length-1 {
            start := i - length + 1
            if prevStart, ok := seen[hash]; ok {
                // Verify
                if s[prevStart:prevStart+length] == s[start:start+length] {
                    return s[start : start+length]
                }
            }
            seen[hash] = start
        }
    }
    return ""
}

func main() {
    fmt.Println(longestRepeatedSubstring("banana"))       // "ana"
    fmt.Println(longestRepeatedSubstring("abcabcabc"))    // "abcabc"
    fmt.Println(longestRepeatedSubstring("aaaa"))          // "aaa"
    fmt.Println(longestRepeatedSubstring("abcd"))          // ""
}
```

---

## Example 6: Rolling Hash for Window Uniqueness

```go
package main

import "fmt"

// Find all windows of size k with all unique characters
func uniqueWindows(s string, k int) []string {
    if k > len(s) {
        return nil
    }

    var result []string
    freq := make(map[byte]int)
    unique := 0

    for i := 0; i < len(s); i++ {
        // Add new character
        freq[s[i]]++
        if freq[s[i]] == 1 {
            unique++
        }

        // Remove old character
        if i >= k {
            freq[s[i-k]]--
            if freq[s[i-k]] == 0 {
                unique--
                delete(freq, s[i-k])
            }
        }

        if i >= k-1 && unique == k {
            result = append(result, s[i-k+1:i+1])
        }
    }
    return result
}

func main() {
    s := "abcabcbb"
    fmt.Println("Windows with 3 unique chars:", uniqueWindows(s, 3))
    // [abc, bca, cab, abc]
}
```

---

## Example 7: 2D Rolling Hash

```go
package main

import "fmt"

func search2D(matrix [][]byte, pattern [][]byte) (int, int, bool) {
    const (
        b1 = 31
        b2 = 37
        m  = 1_000_000_007
    )

    rows, cols := len(matrix), len(matrix[0])
    pr, pc := len(pattern), len(pattern[0])
    if pr > rows || pc > cols {
        return 0, 0, false
    }

    // Hash of pattern
    patHash := int64(0)
    rowPow := int64(1)
    for r := 0; r < pr; r++ {
        colPow := int64(1)
        for c := 0; c < pc; c++ {
            patHash = (patHash + int64(pattern[r][c])*rowPow%m*colPow%m) % m
            colPow = (colPow * int64(b2)) % m
        }
        rowPow = (rowPow * int64(b1)) % m
    }

    // Try each possible top-left corner
    for r := 0; r <= rows-pr; r++ {
        for c := 0; c <= cols-pc; c++ {
            h := int64(0)
            rp := int64(1)
            for dr := 0; dr < pr; dr++ {
                cp := int64(1)
                for dc := 0; dc < pc; dc++ {
                    h = (h + int64(matrix[r+dr][c+dc])*rp%m*cp%m) % m
                    cp = (cp * int64(b2)) % m
                }
                rp = (rp * int64(b1)) % m
            }
            if h == patHash {
                // Verify
                match := true
                for dr := 0; dr < pr && match; dr++ {
                    for dc := 0; dc < pc && match; dc++ {
                        if matrix[r+dr][c+dc] != pattern[dr][dc] {
                            match = false
                        }
                    }
                }
                if match {
                    return r, c, true
                }
            }
        }
    }
    return 0, 0, false
}

func main() {
    matrix := [][]byte{
        {'a', 'b', 'c', 'd'},
        {'e', 'f', 'g', 'h'},
        {'i', 'j', 'k', 'l'},
    }
    pattern := [][]byte{
        {'f', 'g'},
        {'j', 'k'},
    }
    r, c, found := search2D(matrix, pattern)
    fmt.Printf("Found: %v at (%d, %d)\n", found, r, c) // Found: true at (1, 1)
}
```

---

## Example 8: Rolling Hash with Modular Inverse

```go
package main

import "fmt"

const (
    base = 31
    mod  = 1_000_000_007
)

func modPow(b, e, m int64) int64 {
    result := int64(1)
    b %= m
    for e > 0 {
        if e%2 == 1 {
            result = result * b % m
        }
        e /= 2
        b = b * b % m
    }
    return result
}

// Modular inverse using Fermat's little theorem (mod is prime)
func modInverse(a, m int64) int64 {
    return modPow(a, m-2, m)
}

type RollingHash struct {
    hash    int64
    size    int
    highPow int64
    invBase int64
}

func NewRollingHash() *RollingHash {
    return &RollingHash{
        hash:    0,
        size:    0,
        highPow: 1,
        invBase: modInverse(base, mod),
    }
}

func (rh *RollingHash) Append(c byte) {
    rh.hash = (rh.hash + int64(c)*rh.highPow) % mod
    rh.highPow = (rh.highPow * base) % mod
    rh.size++
}

func (rh *RollingHash) PopFront(c byte) {
    rh.hash = (rh.hash - int64(c) + mod) % mod
    rh.hash = rh.hash * rh.invBase % mod
    rh.highPow = rh.highPow * rh.invBase % mod
    rh.size--
}

func main() {
    rh := NewRollingHash()
    s := "abcde"

    // Build hash for "abc"
    for i := 0; i < 3; i++ {
        rh.Append(s[i])
    }
    fmt.Printf("hash(abc) = %d\n", rh.hash)

    // Slide: remove 'a', add 'd' → "bcd"
    rh.PopFront(s[0])
    rh.Append(s[3])
    fmt.Printf("hash(bcd) = %d\n", rh.hash)

    // Slide: remove 'b', add 'e' → "cde"
    rh.PopFront(s[1])
    rh.Append(s[4])
    fmt.Printf("hash(cde) = %d\n", rh.hash)
}
```

---

## Example 9: Check If Substrings Match at Two Positions (O(1))

```go
package main

import "fmt"

const (
    b = 31
    m = 1_000_000_007
)

type PrefixHash struct {
    h []int64
    p []int64
}

func BuildPrefixHash(s string) *PrefixHash {
    n := len(s)
    ph := &PrefixHash{
        h: make([]int64, n+1),
        p: make([]int64, n+1),
    }
    ph.p[0] = 1
    for i := 0; i < n; i++ {
        ph.h[i+1] = (ph.h[i] + int64(s[i])*ph.p[i]) % m
        ph.p[i+1] = (ph.p[i] * b) % m
    }
    return ph
}

// RawHash returns unnormalized hash of s[l..r]
func (ph *PrefixHash) RawHash(l, r int) int64 {
    return (ph.h[r+1] - ph.h[l] + m) % m
}

// Compare substrings s[l1..r1] and s[l2..r2] in O(1)
func (ph *PrefixHash) Equal(l1, r1, l2, r2 int) bool {
    if r1-l1 != r2-l2 {
        return false
    }
    h1 := ph.RawHash(l1, r1)
    h2 := ph.RawHash(l2, r2)
    // Normalize: multiply by power to align positions
    if l1 < l2 {
        h1 = h1 * ph.p[l2-l1] % m
    } else {
        h2 = h2 * ph.p[l1-l2] % m
    }
    return h1 == h2
}

func main() {
    s := "abcabcabc"
    ph := BuildPrefixHash(s)

    fmt.Println(ph.Equal(0, 2, 3, 5)) // "abc" == "abc" → true
    fmt.Println(ph.Equal(0, 2, 6, 8)) // "abc" == "abc" → true
    fmt.Println(ph.Equal(0, 2, 1, 3)) // "abc" == "bca" → false
    fmt.Println(ph.Equal(1, 4, 4, 7)) // "bcab" == "bcab" → true
}
```

---

## Example 10: Repeated String Match

```go
package main

import "fmt"

// Given strings a and b, find minimum number of times a must be repeated
// so that b is a substring of the repeated string.
func repeatedStringMatch(a, b string) int {
    const (
        base = 31
        mod  = 1_000_000_007
    )

    // Build repeated string just long enough
    repeated := a
    count := 1
    for len(repeated) < len(b) {
        repeated += a
        count++
    }
    // Also try one more repetition
    repeated += a
    maxCount := count + 1

    k := len(b)
    // Hash of b
    patHash := int64(0)
    for i := 0; i < k; i++ {
        patHash = (patHash*base + int64(b[i])) % mod
    }

    highPow := int64(1)
    for i := 0; i < k-1; i++ {
        highPow = (highPow * base) % mod
    }

    hash := int64(0)
    for i := 0; i < len(repeated); i++ {
        hash = (hash*base + int64(repeated[i])) % mod
        if i >= k {
            hash = (hash - int64(repeated[i-k])*highPow%mod*base%mod + mod*2) % mod
        }
        if i >= k-1 && hash == patHash {
            start := i - k + 1
            if repeated[start:start+k] == b {
                // How many repetitions needed to cover position start..start+k-1
                needed := (start+k-1)/len(a) + 1
                if needed <= maxCount {
                    return needed
                }
            }
        }
    }
    return -1
}

func main() {
    fmt.Println(repeatedStringMatch("abcd", "cdabcdab")) // 3
    fmt.Println(repeatedStringMatch("a", "aa"))           // 2
    fmt.Println(repeatedStringMatch("abc", "cab"))        // 2
    fmt.Println(repeatedStringMatch("abc", "xyz"))        // -1
}
```

---

## Rolling Hash Complexity

| Operation         | Time  | Space |
|-------------------|-------|-------|
| Build initial hash | O(k)  | O(1)  |
| Slide window      | O(1)  | O(1)  |
| Full scan of text | O(n)  | O(1)  |
| Binary search + hash | O(n log n) | O(n) |

## Key Takeaways

1. **O(1) update** — remove leftmost, add rightmost character
2. **Always verify** — hash collisions are possible
3. **Double hashing** — use two different (base, mod) pairs for safety
4. **Modular inverse** enables cleaner pop-front operations
5. **Prefix hashing** + power array → O(1) substring comparison
6. **Binary search + rolling hash** → powerful for longest substring problems

> **Next up:** Rabin-Karp Algorithm →
