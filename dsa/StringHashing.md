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

**Textual Figure — Polynomial Hash Computation for "hello":**

```
  Input strings: "hello", "world", "hello", "olleh", "abc"
  Formula: hash(s) = Σ s[i] × base^i  (mod 10^9+7)
  Character values: a=1, b=2, ... h=8, e=5, l=12, o=15

  Trace for "hello":
  ┌───────┬──────┬──────┬────────────┬───────────────────────────────┐
  │   i   │ s[i] │ val  │ power(31^i)│ hash (cumulative mod 10^9+7)  │
  ├───────┼──────┼──────┼────────────┼───────────────────────────────┤
  │   0   │ 'h'  │   8  │          1 │ 0 + 8×1         =          8 │
  │   1   │ 'e'  │   5  │         31 │ 8 + 5×31        =        163 │
  │   2   │ 'l'  │  12  │        961 │ 163 + 12×961    =     11,695 │
  │   3   │ 'l'  │  12  │     29,791 │ 11695 + 12×29791=    369,187 │
  │   4   │ 'o'  │  15  │    923,521 │ 369187+15×923521= 14,222,002 │
  └───────┴──────┴──────┴────────────┴───────────────────────────────┘
                                          hash("hello") = 14,222,002

  Results:
    "hello" ──hash──→ 14,222,002  ◄── same
    "world" ──hash──→ (different value)
    "hello" ──hash──→ 14,222,002  ◄── same (duplicate detected!)
    "olleh" ──hash──→ (different — character order matters)
    "abc"   ──hash──→ (different value)
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

**Textual Figure — Double Hash Collision Reduction:**

```
  Input: "abc", "bca", "abc", "xyz"
  Hash 1: base=31, mod=10^9+7    Hash 2: base=37, mod=10^9+9

  Trace for "abc" (a=1, b=2, c=3):
  ┌──────────────────────────────────┬──────────────────────────────────┐
  │         Hash 1 (base=31)        │         Hash 2 (base=37)        │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ i=0: 0 + 1×1     =    1        │ i=0: 0 + 1×1     =    1        │
  │ i=1: 1 + 2×31    =   63        │ i=1: 1 + 2×37    =   75        │
  │ i=2: 63 + 3×961  = 2946        │ i=2: 75 + 3×1369 = 4182        │
  └──────────────────────────────────┴──────────────────────────────────┘

  Results (h1, h2):
  ┌────────┬──────────────────┬──────────────────────────────┐
  │ String │ (h1, h2)         │ Notes                        │
  ├────────┼──────────────────┼──────────────────────────────┤
  │ "abc"  │ (2946, 4182)     │ Stored                       │
  │ "bca"  │ (1056, 1482)     │ Different (not an anagram    │
  │        │                  │  match in positional hash)    │
  │ "abc"  │ (2946, 4182)     │ Match with first "abc" ✓     │
  │ "xyz"  │ (different pair) │ No match                     │
  └────────┴──────────────────┴──────────────────────────────┘

  Collision probability comparison:
  ┌──────────────────────────────────────────────────┐
  │ Single hash:  P(collision) ≈ 1 / 10^9           │
  │ Double hash:  P(collision) ≈ 1 / 10^18          │
  │                              ↑                   │
  │               Drastically reduced!               │
  └──────────────────────────────────────────────────┘
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

**Textual Figure — Prefix Hash Array & O(1) Substring Hashing:**

```
  Input: s = "abcabcabc"   (a=1, b=2, c=3)   base=31, mod=10^9+7

  Step 1 — Build power[] array:
    power[0]=1  power[1]=31  power[2]=961  power[3]=29791 ...

  Step 2 — Build prefixHash[] array:
    prefixHash[i+1] = prefixHash[i] + val(s[i]) × power[i]

    Index:  0    1    2    3    4    5    6    7    8
    Char:   a    b    c    a    b    c    a    b    c
    Val:    1    2    3    1    2    3    1    2    3

    ┌───┬──────────────────────────────────────────────────┐
    │ i │ prefixHash[i]                                    │
    ├───┼──────────────────────────────────────────────────┤
    │ 0 │ 0                                                │
    │ 1 │ 0 + 1×1           =                    1        │
    │ 2 │ 1 + 2×31          =                   63        │
    │ 3 │ 63 + 3×961        =                2,946        │
    │ 4 │ 2946 + 1×29,791   =               32,737        │
    │ 5 │ 32737 + 2×923,521 =            1,879,779        │
    │ 6 │ 1879779 + 3×28,629,151 =      87,767,232        │
    │...│ (continues)                                      │
    └───┴──────────────────────────────────────────────────┘

  Step 3 — O(1) SubHash(l, r) = prefixHash[r+1] − prefixHash[l]:
    ┌──────────────────┬───────────────────────────────────────┐
    │ Substring        │ Computation                           │
    ├──────────────────┼───────────────────────────────────────┤
    │ s[0..2] = "abc"  │ prefixHash[3] − prefixHash[0] = 2946 │
    │ s[3..5] = "abc"  │ prefixHash[6] − prefixHash[3]        │
    │ s[6..8] = "abc"  │ prefixHash[9] − prefixHash[6]        │
    └──────────────────┴───────────────────────────────────────┘

    Note: Raw SubHash values differ across positions due to
    power offset. Normalize by multiplying by modular inverse
    of base^l for direct cross-position comparison.

    ┌────────────────────────────────────────────────────┐
    │  Without normalization:  O(1) per SubHash call     │
    │  With normalization:     O(log mod) via modPow     │
    │  Equality check:  compare hash × power[shift]      │
    └────────────────────────────────────────────────────┘
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

**Textual Figure — Rolling Hash Duplicate Detection for "banana":**

```
  Input: s = "banana", length = 3
  Rolling hash: base=31, mod=10^9+7
  Char values: b=2, a=1, n=14

  Sliding window (size 3) across "banana":
  ┌─────┬─────┬─────┬─────┬─────┬─────┐
  │  b  │  a  │  n  │  a  │  n  │  a  │
  │  2  │  1  │ 14  │  1  │ 14  │  1  │
  └─────┴─────┴─────┴─────┴─────┴─────┘
   ├───────────┤                          window 0: "ban"
         ├───────────┤                    window 1: "ana"
               ├───────────┤              window 2: "nan"
                     ├───────────┤        window 3: "ana"

  Rolling hash trace:
  ┌──────┬────────┬────────┬────────────────────────────────────┐
  │ Pos  │ Window │  Hash  │ Action                             │
  ├──────┼────────┼────────┼────────────────────────────────────┤
  │  0   │ "ban"  │  h₁    │ New → seen[h₁] = [0]              │
  │  1   │ "ana"  │  h₂    │ New → seen[h₂] = [1]              │
  │  2   │ "nan"  │  h₃    │ New → seen[h₃] = [2]              │
  │  3   │ "ana"  │  h₂    │ h₂ found! Verify: s[1:4]=="ana"   │
  │      │        │        │   s[3:6]=="ana" → Match ✓          │
  │      │        │        │   duplicates = ["ana"]             │
  └──────┴────────┴────────┴────────────────────────────────────┘

  Hash update (rolling):
    ┌──────────────────────────────────────────────────┐
    │  new_hash = (old_hash × base + new_char) mod m   │
    │          − (removed_char × base^len) mod m       │
    │                                                  │
    │  "ban" → "ana": drop 'b', add 'a'                │
    │  "ana" → "nan": drop 'a', add 'n'                │
    │  "nan" → "ana": drop 'n', add 'a'                │
    └──────────────────────────────────────────────────┘

  Result: ["ana"]
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

**Textual Figure — Anagram Grouping via Character Frequency Keys:**

```
  Input: ["eat", "tea", "tan", "ate", "nat", "bat"]

  Step 1 — Compute [26]int frequency key for each string:
  ┌────────┬──────────────────────────┬──────────────────────┐
  │ String │ Char Frequencies         │ Key (non-zero only)  │
  ├────────┼──────────────────────────┼──────────────────────┤
  │ "eat"  │ a:1, e:1, t:1            │ KEY_A                │
  │ "tea"  │ a:1, e:1, t:1            │ KEY_A  ◄── same      │
  │ "ate"  │ a:1, e:1, t:1            │ KEY_A  ◄── same      │
  │ "tan"  │ a:1, n:1, t:1            │ KEY_B                │
  │ "nat"  │ a:1, n:1, t:1            │ KEY_B  ◄── same      │
  │ "bat"  │ a:1, b:1, t:1            │ KEY_C                │
  └────────┴──────────────────────────┴──────────────────────┘

  Step 2 — Group by frequency key:
  ┌─────────┬──────────────────────────────────┐
  │   Key   │ map[[26]int][]string             │
  ├─────────┼──────────────────────────────────┤
  │  KEY_A  │ ──→ ["eat", "tea", "ate"]        │
  │  KEY_B  │ ──→ ["tan", "nat"]               │
  │  KEY_C  │ ──→ ["bat"]                      │
  └─────────┴──────────────────────────────────┘

  Why [26]int works as map key in Go:
  ┌────────────────────────────────────────────────────┐
  │ Arrays (not slices) are comparable in Go, so      │
  │ [26]int can be used directly as a map key.         │
  │ Anagrams share identical frequency arrays.         │
  └────────────────────────────────────────────────────┘

  Output: [[eat tea ate] [tan nat] [bat]]
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

**Textual Figure — Binary Search + Hashing for Longest Common Substring:**

```
  Input: s1 = "abcdef", s2 = "zbcdf"
  Binary search on substring length: lo=0, hi=min(6,5)=5

  ┌──────┬────┬────┬─────┬───────────────────────────────────────┐
  │ Step │ lo │ hi │ mid │ hasCommonSubstring(s1,s2,mid)?       │
  ├──────┼────┼────┼─────┼───────────────────────────────────────┤
  │   1  │  0 │  5 │  2  │ Yes → found "bc"  → result="bc"    │
  │   2  │  3 │  5 │  4  │ No  → no length-4 common substring  │
  │   3  │  3 │  3 │  3  │ Yes → found "bcd" → result="bcd"   │
  │   4  │  4 │  3 │  —  │ lo > hi → stop                     │
  └──────┴────┴────┴─────┴───────────────────────────────────────┘

  Detail: hasCommonSubstring(s1, s2, 3):

    s1 hash set (len=3):         s2 hash set (len=3):
    ┌────────────────┐           ┌────────────────┐
    │ "abc" → h₁   │           │ "zbc" → h₅   │
    │ "bcd" → h₂   │           │ "bcd" → h₆   │ ◄─ h₂==h₆!
    │ "cde" → h₃   │           │ "cdf" → h₇   │
    │ "def" → h₄   │           └────────────────┘
    └────────────────┘
                   │                    │
                   └───── Match! ─────┘
                   Verify: "bcd" == "bcd" ✓

  Result: "bcd"
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

**Textual Figure — O(1) Hash-Based String Equality Detection:**

```
  Input: ["apple", "banana", "apple", "cherry", "banana"]
  Polynomial hash: base=31, mod=10^9+7

  Processing each string:
  ┌──────────┬───────────────┬─────────────────────────────────┐
  │  String  │  Hash Value   │ Action                          │
  ├──────────┼───────────────┼─────────────────────────────────┤
  │ "apple"  │  h₁           │ Not in map → Store hashes[h₁]  │
  │ "banana" │  h₂           │ Not in map → Store hashes[h₂]  │
  │ "apple"  │  h₁           │ Found! matches "apple"  ✓      │
  │ "cherry" │  h₃           │ Not in map → Store hashes[h₃]  │
  │ "banana" │  h₂           │ Found! matches "banana" ✓      │
  └──────────┴───────────────┴─────────────────────────────────┘

  Flow diagram:
    "apple"  ──O(n)──→ h₁ ──lookup──→ Miss  → Store
    "banana" ──O(n)──→ h₂ ──lookup──→ Miss  → Store
    "apple"  ──O(n)──→ h₁ ──lookup──→ Hit!  → "matches 'apple'"
    "cherry" ──O(n)──→ h₃ ──lookup──→ Miss  → Store
    "banana" ──O(n)──→ h₂ ──lookup──→ Hit!  → "matches 'banana'"
                  │         │
              O(n) hash   O(1) map
              compute     lookup
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

**Textual Figure — Counting Distinct Substrings via Rolling Hash:**

```
  Input: s = "abc"    (a=1, b=2, c=3)

  Enumerate all substrings by length, compute rolling hash:
  ┌────────┬────────────┬───────────┬──────────────────────────┐
  │ Length │ Substring  │ Hash      │ seen key (hash×(len+1)) │
  ├────────┼────────────┼───────────┼──────────────────────────┤
  │   1    │ "a"        │ h₁        │ h₁×2 → new ✓             │
  │   1    │ "b"        │ h₂        │ h₂×2 → new ✓             │
  │   1    │ "c"        │ h₃        │ h₃×2 → new ✓             │
  │   2    │ "ab"       │ h₄        │ h₄×3 → new ✓             │
  │   2    │ "bc"       │ h₅        │ h₅×3 → new ✓             │
  │   3    │ "abc"      │ h₆        │ h₆×4 → new ✓             │
  └────────┴────────────┴───────────┴──────────────────────────┘
  |seen| = 6  →  6 distinct substrings ✓

  Contrast with s = "aaa":
  ┌────────┬────────────┬───────────┬──────────────────────────┐
  │ Length │ Substring  │ Hash      │ Status                   │
  ├────────┼────────────┼───────────┼──────────────────────────┤
  │   1    │ "a" (×3)   │ h₁        │ Same hash → 1 entry      │
  │   2    │ "aa" (×2)  │ h₂        │ Same hash → 1 entry      │
  │   3    │ "aaa" (×1) │ h₃        │ 1 entry                  │
  └────────┴────────────┴───────────┴──────────────────────────┘
  |seen| = 3  →  3 distinct substrings ✓
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

**Textual Figure — Isomorphic String Mapping Trace:**

```
  Input: s = "egg", t = "add"

  Step-by-step character mapping:
  ┌─────┬──────┬──────┬────────────────┬────────────────┬────────┐
  │  i  │ s[i] │ t[i] │ sToT           │ tToS           │ Valid? │
  ├─────┼──────┼──────┼────────────────┼────────────────┼────────┤
  │  0  │ 'e'  │ 'a'  │ {e→a}          │ {a→e}          │   ✓    │
  │  1  │ 'g'  │ 'd'  │ {e→a, g→d}     │ {a→e, d→g}     │   ✓    │
  │  2  │ 'g'  │ 'd'  │ g→d exists,    │ d→g exists,    │   ✓    │
  │     │      │      │ mapped='d' ✓   │ mapped='g' ✓   │        │
  └─────┴──────┴──────┴────────────────┴────────────────┴────────┘
  Result: true — "egg" ≅ "add"

  Bijective mapping:
  ┌───────────────────────────────────────────────┐
  │   e ──→ a        (one-to-one, both ways)  │
  │   g ──→ d        each char maps uniquely   │
  └───────────────────────────────────────────────┘

  Counter-example: s = "foo", t = "bar"
  ┌─────┬──────┬──────┬────────────────┬────────────────┬────────┐
  │  i  │ s[i] │ t[i] │ sToT           │ tToS           │ Valid? │
  ├─────┼──────┼──────┼────────────────┼────────────────┼────────┤
  │  0  │ 'f'  │ 'b'  │ {f→b}          │ {b→f}          │   ✓    │
  │  1  │ 'o'  │ 'a'  │ {f→b, o→a}     │ {b→f, a→o}     │   ✓    │
  │  2  │ 'o'  │ 'r'  │ o→a exists,    │                │   ✗    │
  │     │      │      │ but 'a'≠'r' !  │                │        │
  └─────┴──────┴──────┴────────────────┴────────────────┴────────┘
  Result: false — 'o' can't map to both 'a' and 'r'
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

**Textual Figure — Fingerprint-Based Plagiarism Detection:**

```
  Input:
    t1 = "the quick brown fox jumps over the lazy dog"
    t2 = "the quick brown cat jumps over the lazy dog"
    t3 = "completely different text here nothing alike"
  windowSize = 5

  Step 1 — Compute rolling hash fingerprints (window=5):

  t1:  t h e   q u i c k   b r o w n   f o x ...
       │─────│ │─────│ │─────│
       "the q"  "he qu"  "e qui"  ...  (39 windows total)
         h₁       h₂       h₃            hₙ

  t2:  t h e   q u i c k   b r o w n   c a t ...
       │─────│ │─────│ │─────│
       "the q"  "he qu"  "e qui"  ...  (39 windows total)
         h₁       h₂       h₃            hₙ
         ↑same   ↑same   ↑same    ↑differ where "fox"≠"cat"

  Step 2 — Compare fingerprint sets:
  ┌───────────────┬──────────────┬─────────────────────────────────┐
  │  Comparison   │    Formula     │ Result                          │
  ├───────────────┼──────────────┼─────────────────────────────────┤
  │  t1 vs t2     │ 2×common/total │ ≈35/39 overlap → ~0.90 (high)  │
  │  t1 vs t3     │ 2×common/total │ ≈1/39 overlap  → ~0.03 (low)   │
  └───────────────┴──────────────┴─────────────────────────────────┘

  Visual overlap:
    t1: [===========================|XXX|==============]
    t2: [===========================|YYY|==============]
                                     ↑
                              only "fox" vs "cat" differs
                              (plus neighboring windows)

    t1: [==========================================]
    t3: [XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX]
                    ↑
             almost no windows match
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
