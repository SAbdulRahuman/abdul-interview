# Phase 3: Strings — String Hashing

## What is String Hashing?

String hashing converts a string to a numeric value, enabling **O(1) string comparison** after O(n) preprocessing. Used for pattern matching, deduplication, and hash tables.

Common formula: `hash(s) = Σ s[i] × p^i  (mod m)`

---

## Example 1: Polynomial Hash Function

```go
package main

import "fmt"

const (
    base = 31
    mod  = 1_000_000_007
)

func polyHash(s string) int64 {
    var hash int64
    power := int64(1)
    for i := 0; i < len(s); i++ {
        hash = (hash + int64(s[i]-'a'+1)*power) % mod
        power = (power * base) % mod
    }
    return hash
}

func main() {
    strings := []string{"hello", "world", "hello", "olleh", "abc"}
    for _, s := range strings {
        fmt.Printf("hash(%q) = %d\n", s, polyHash(s))
    }
    // Same strings → same hash
    // "hello" appears twice with same hash
}
```

---

## Example 2: Hash with Collision Detection

```go
package main

import "fmt"

const (
    base1 = 31
    mod1  = 1_000_000_007
    base2 = 37
    mod2  = 1_000_000_009
)

type DoubleHash struct {
    h1, h2 int64
}

func doubleHash(s string) DoubleHash {
    var h1, h2 int64
    p1, p2 := int64(1), int64(1)
    for i := 0; i < len(s); i++ {
        ch := int64(s[i] - 'a' + 1)
        h1 = (h1 + ch*p1) % mod1
        h2 = (h2 + ch*p2) % mod2
        p1 = (p1 * base1) % mod1
        p2 = (p2 * base2) % mod2
    }
    return DoubleHash{h1, h2}
}

func main() {
    strs := []string{"abc", "bca", "abc", "xyz"}
    for _, s := range strs {
        h := doubleHash(s)
        fmt.Printf("hash(%q) = (%d, %d)\n", s, h.h1, h.h2)
    }
    // Double hashing drastically reduces collision probability
}
```

---

## Example 3: Prefix Hash for O(1) Substring Hashing

```go
package main

import "fmt"

const (
    base = 31
    mod  = 1_000_000_007
)

type HashArray struct {
    prefixHash []int64
    power      []int64
}

func NewHashArray(s string) *HashArray {
    n := len(s)
    h := &HashArray{
        prefixHash: make([]int64, n+1),
        power:      make([]int64, n+1),
    }
    h.power[0] = 1
    for i := 1; i <= n; i++ {
        h.power[i] = (h.power[i-1] * base) % mod
    }
    for i := 0; i < n; i++ {
        h.prefixHash[i+1] = (h.prefixHash[i] + int64(s[i]-'a'+1)*h.power[i]) % mod
    }
    return h
}

// Hash of s[l..r] (0-indexed, inclusive)
func (h *HashArray) SubHash(l, r int) int64 {
    raw := (h.prefixHash[r+1] - h.prefixHash[l] + mod) % mod
    // Normalize by dividing out p^l (multiply by modular inverse)
    // Simpler: compare raw * power[shift] for equality checks
    return raw
}

func main() {
    s := "abcabcabc"
    h := NewHashArray(s)

    // Check if s[0:3] == s[3:6] == s[6:9]
    h1 := h.SubHash(0, 2)
    h2 := h.SubHash(3, 5)
    h3 := h.SubHash(6, 8)
    fmt.Printf("hash(abc@0) = %d\n", h1)
    fmt.Printf("hash(abc@3) = %d\n", h2)
    fmt.Printf("hash(abc@6) = %d\n", h3)

    // To compare, normalize: h * power[shift]
    // h(s[l..r]) at position l needs power normalization
    norm1 := h1 // already at position 0
    norm2 := (h2 * modPow(base, mod-1-3, mod)) % mod
    _ = norm1
    _ = norm2
    fmt.Println("Substrings 'abc' all match:", s[0:3] == s[3:6] && s[3:6] == s[6:9])
}

func modPow(base, exp, mod int64) int64 {
    result := int64(1)
    base = base % mod
    for exp > 0 {
        if exp%2 == 1 {
            result = (result * base) % mod
        }
        exp /= 2
        base = (base * base) % mod
    }
    return result
}
```

---

## Example 4: Finding Duplicate Substrings Using Hashing

```go
package main

import "fmt"

func findDuplicateSubstrings(s string, length int) []string {
    if length > len(s) {
        return nil
    }

    const (
        b = 31
        m = 1_000_000_007
    )

    // Compute base^length
    power := int64(1)
    for i := 0; i < length; i++ {
        power = (power * int64(b)) % int64(m)
    }

    seen := make(map[int64][]int)
    var duplicates []string

    hash := int64(0)
    for i := 0; i < len(s); i++ {
        hash = (hash*int64(b) + int64(s[i]-'a'+1)) % int64(m)

        if i >= length {
            hash = (hash - int64(s[i-length]-'a'+1)*power%int64(m) + int64(m)) % int64(m)
        }

        if i >= length-1 {
            if positions, exists := seen[hash]; exists {
                // Verify to handle collisions
                sub := s[i-length+1 : i+1]
                for _, pos := range positions {
                    if s[pos:pos+length] == sub {
                        duplicates = append(duplicates, sub)
                        break
                    }
                }
            }
            seen[hash] = append(seen[hash], i-length+1)
        }
    }
    return duplicates
}

func main() {
    fmt.Println(findDuplicateSubstrings("banana", 3))   // ["ana"]
    fmt.Println(findDuplicateSubstrings("abcabc", 3))   // ["abc"]
    fmt.Println(findDuplicateSubstrings("aaaaaa", 2))   // ["aa" "aa" "aa" "aa"]
}
```

---

## Example 5: Group Strings by Hash

```go
package main

import "fmt"

func groupAnagrams(strs []string) [][]string {
    // Use sorted character frequency as hash key
    groups := make(map[[26]int][]string)

    for _, s := range strs {
        var key [26]int
        for _, c := range s {
            key[c-'a']++
        }
        groups[key] = append(groups[key], s)
    }

    var result [][]string
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}

func main() {
    strs := []string{"eat", "tea", "tan", "ate", "nat", "bat"}
    groups := groupAnagrams(strs)
    for _, g := range groups {
        fmt.Println(g)
    }
    // [eat tea ate]
    // [tan nat]
    // [bat]
}
```

---

## Example 6: Longest Common Substring Using Hashing + Binary Search

```go
package main

import "fmt"

func longestCommonSubstring(s1, s2 string) string {
    // Binary search on the length of common substring
    lo, hi := 0, len(s1)
    if len(s2) < hi {
        hi = len(s2)
    }
    result := ""

    for lo <= hi {
        mid := (lo + hi) / 2
        if sub := hasCommonSubstring(s1, s2, mid); sub != "" {
            result = sub
            lo = mid + 1
        } else {
            hi = mid - 1
        }
    }
    return result
}

func hasCommonSubstring(s1, s2 string, length int) string {
    if length == 0 {
        return ""
    }
    const (
        b = 31
        m = 1_000_000_007
    )

    // Collect all hashes of s1 substrings of given length
    hashes := make(map[int64]string)
    hash := int64(0)
    power := int64(1)
    for i := 0; i < length-1; i++ {
        power = (power * int64(b)) % int64(m)
    }

    for i := 0; i < len(s1); i++ {
        hash = (hash*int64(b) + int64(s1[i])) % int64(m)
        if i >= length {
            hash = (hash - int64(s1[i-length])*power%int64(m)*int64(b)%int64(m) + int64(m)*2) % int64(m)
        }
        if i >= length-1 {
            hashes[hash] = s1[i-length+1 : i+1]
        }
    }

    // Check s2 substrings
    hash = 0
    for i := 0; i < len(s2); i++ {
        hash = (hash*int64(b) + int64(s2[i])) % int64(m)
        if i >= length {
            hash = (hash - int64(s2[i-length])*power%int64(m)*int64(b)%int64(m) + int64(m)*2) % int64(m)
        }
        if i >= length-1 {
            if candidate, ok := hashes[hash]; ok {
                sub := s2[i-length+1 : i+1]
                if sub == candidate { // verify
                    return sub
                }
            }
        }
    }
    return ""
}

func main() {
    fmt.Println(longestCommonSubstring("abcdef", "zbcdf"))  // "bcd"
    fmt.Println(longestCommonSubstring("GeeksforGeeks", "GeeksQuiz")) // "Geeks"
}
```

---

## Example 7: String Equality Check Using Hash

```go
package main

import "fmt"

const (
    hashBase = 31
    hashMod  = 1_000_000_007
)

func computeHash(s string) int64 {
    h := int64(0)
    p := int64(1)
    for i := 0; i < len(s); i++ {
        h = (h + int64(s[i]-'a'+1)*p) % hashMod
        p = (p * hashBase) % hashMod
    }
    return h
}

func main() {
    // O(1) comparison after O(n) hash computation
    strings := []string{"apple", "banana", "apple", "cherry", "banana"}
    hashes := make(map[int64]string)

    for _, s := range strings {
        h := computeHash(s)
        if existing, ok := hashes[h]; ok {
            fmt.Printf("'%s' matches '%s' (hash=%d)\n", s, existing, h)
        } else {
            hashes[h] = s
        }
    }
}
```

---

## Example 8: Count Distinct Substrings

```go
package main

import "fmt"

func countDistinctSubstrings(s string) int {
    n := len(s)
    seen := make(map[int64]bool)

    const (
        b = 31
        m = 1_000_000_007
    )

    for length := 1; length <= n; length++ {
        hash := int64(0)
        power := int64(1)
        for i := 0; i < length-1; i++ {
            power = (power * int64(b)) % int64(m)
        }

        for i := 0; i < n; i++ {
            hash = (hash*int64(b) + int64(s[i]-'a'+1)) % int64(m)
            if i >= length {
                hash = (hash - int64(s[i-length]-'a'+1)*power%int64(m)*int64(b)%int64(m) + int64(m)*2) % int64(m)
            }
            if i >= length-1 {
                seen[hash*int64(length+1)] = true // include length to reduce collisions
            }
        }
    }
    return len(seen)
}

func main() {
    fmt.Println(countDistinctSubstrings("abc"))   // 6: a,b,c,ab,bc,abc
    fmt.Println(countDistinctSubstrings("aaa"))   // 3: a,aa,aaa
    fmt.Println(countDistinctSubstrings("abab"))  // 7: a,b,ab,ba,aba,bab,abab
}
```

---

## Example 9: Isomorphic Strings Using Hash Mapping

```go
package main

import "fmt"

func isIsomorphic(s, t string) bool {
    if len(s) != len(t) {
        return false
    }
    sToT := make(map[byte]byte)
    tToS := make(map[byte]byte)

    for i := 0; i < len(s); i++ {
        sc, tc := s[i], t[i]
        if mapped, ok := sToT[sc]; ok {
            if mapped != tc {
                return false
            }
        } else {
            sToT[sc] = tc
        }
        if mapped, ok := tToS[tc]; ok {
            if mapped != sc {
                return false
            }
        } else {
            tToS[tc] = sc
        }
    }
    return true
}

func main() {
    fmt.Println(isIsomorphic("egg", "add"))     // true
    fmt.Println(isIsomorphic("foo", "bar"))     // false
    fmt.Println(isIsomorphic("paper", "title")) // true
    fmt.Println(isIsomorphic("ab", "aa"))       // false
}
```

---

## Example 10: Hash-Based Plagiarism Detection (Fingerprinting)

```go
package main

import "fmt"

func fingerprint(text string, windowSize int) map[int64]int {
    const (
        b = 31
        m = 1_000_000_007
    )

    hashes := make(map[int64]int)
    power := int64(1)
    for i := 0; i < windowSize-1; i++ {
        power = (power * int64(b)) % int64(m)
    }

    hash := int64(0)
    for i := 0; i < len(text); i++ {
        hash = (hash*int64(b) + int64(text[i])) % int64(m)
        if i >= windowSize {
            hash = (hash - int64(text[i-windowSize])*power%int64(m)*int64(b)%int64(m) + int64(m)*2) % int64(m)
        }
        if i >= windowSize-1 {
            hashes[hash]++
        }
    }
    return hashes
}

func similarity(text1, text2 string, windowSize int) float64 {
    fp1 := fingerprint(text1, windowSize)
    fp2 := fingerprint(text2, windowSize)

    common := 0
    total := 0
    for h, c1 := range fp1 {
        if c2, ok := fp2[h]; ok {
            if c1 < c2 {
                common += c1
            } else {
                common += c2
            }
        }
        total += c1
    }
    for _, c := range fp2 {
        total += c
    }
    if total == 0 {
        return 0
    }
    return 2.0 * float64(common) / float64(total)
}

func main() {
    t1 := "the quick brown fox jumps over the lazy dog"
    t2 := "the quick brown cat jumps over the lazy dog"
    t3 := "completely different text here nothing alike"

    fmt.Printf("t1 vs t2: %.2f\n", similarity(t1, t2, 5)) // high
    fmt.Printf("t1 vs t3: %.2f\n", similarity(t1, t3, 5)) // low
}
```

---

## Key Takeaways

1. **Polynomial hashing**: `hash = Σ s[i] × base^i mod m`
2. **Double hashing** reduces collision probability dramatically
3. **Prefix hashes** enable O(1) substring hash computation
4. **Always verify** hash matches with actual string comparison
5. **Choose coprime base and mod** — common: base=31, mod=10^9+7
6. **Applications**: pattern matching, dedup, anagram grouping, plagiarism detection

> **Next up:** Rolling Hash →
