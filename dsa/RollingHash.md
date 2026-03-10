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

**Textual Figure — Basic Rolling Hash (k=3 on "abcdefgh"):**

```
  Input: s = "abcdefgh"    base = 31    k = 3

  Step 0 — Precompute:
    highPower = base^(k-1) = 31^2 = 961

  Step 1 — Initial window hash (i=0..2):
    ┌───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │ f │ g │ h │
    └───┴───┴───┴───┴───┴───┴───┴───┘
      ▲───────▲
      └─ window 0: "abc"

    hash = 0
    hash = (0  × 31 + 'a') mod M  = 97
    hash = (97 × 31 + 'b') mod M  = 3105
    hash = (3105×31 + 'c') mod M  = 96354

  Step 2 — Slide to window 1 (remove 'a', add 'd'):
    ┌───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │ f │ g │ h │
    └───┴───┴───┴───┴───┴───┴───┴───┘
          ▲───────▲
          └─ window 1: "bcd"

    Formula: new = (old - s[0]×highPower) × base + s[3]
    hash = (96354 - 97×961) mod M  = 3237
    hash = (3237 × 31 + 100) mod M = 100447
             remove 'a'  ──┘  add 'd' ──┘

  Step 3 — Slide to window 2 (remove 'b', add 'e'):
    ┌───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │ f │ g │ h │
    └───┴───┴───┴───┴───┴───┴───┴───┘
              ▲───────▲
              └─ window 2: "cde"

    hash = (100447 - 98×961) × 31 + 101  (mod M)
             remove 'b' ──┘    add 'e' ──┘

  ... continues sliding one position at a time ...

  Result:
    ┌─────────┬────────────────┐
    │ Window  │ Hash Value     │
    ├─────────┼────────────────┤
    │ "abc"   │ hash(abc)      │
    │ "bcd"   │ hash(bcd)      │
    │ "cde"   │ hash(cde)      │
    │ "def"   │ hash(def)      │
    │ "efg"   │ hash(efg)      │
    │ "fgh"   │ hash(fgh)      │
    └─────────┴────────────────┘
    Total windows = n - k + 1 = 8 - 3 + 1 = 6
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

**Textual Figure — Pattern Search (Rabin-Karp) for "AABA" in "AABAACAADAABAABA":**

```
  Input:  text = "AABAACAADAABAABA"   pattern = "AABA"   base=256  mod=101
          Index: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14

  Step 0 — Precompute:
    k = 4,  highPower = 256^3 mod 101 = 56

  Step 1 — Hash the pattern:
    patHash = ((('A'×256 + 'A')×256 + 'B')×256 + 'A') mod 101 = 30

  Step 2 — Hash first window of text [0..3]:
    txtHash = ((('A'×256 + 'A')×256 + 'B')×256 + 'A') mod 101 = 30

  Step 3 — Slide and compare:
   i=0: ┌─────────────────────────────────────────┐
        │ A  A  B  A  A  C  A  A  D  A  A  B  A  A  B  A │
        └─────────────────────────────────────────┘
          [========]                                  txtHash=30 = patHash ✓
          Verify: "AABA" == "AABA" → MATCH at 0

   i=1:   [========]         txtHash = (30 - 'A'×56)×256 + 'A' mod 101
          "ABAA"             Hash ≠ 30, skip

   i=2:      [========]     "BAAC"  Hash ≠ 30, skip
   i=3:         [========]  "AACA"  Hash ≠ 30, skip
   ...           (slides through positions 4..8, no match)

   i=9:                              [========]
        │ A  A  B  A  A  C  A  A  D  A  A  B  A  A  B  A │
                                      "AABA"  txtHash=30 ✓
          Verify → MATCH at 9

   i=12:                                       [========]
                                                "AABA"  txtHash=30 ✓
          Verify → MATCH at 12

  Formula at each slide:
    ┌──────────────────────────────────────────────────────┐
    │ txtHash = (txtHash - text[i]×highPower)×base         │
    │                    + text[i+k]                (mod m)│
    │                                                      │
    │            remove old ──┘          add new ──┘       │
    └──────────────────────────────────────────────────────┘

  Result: Pattern found at positions [0, 9, 12]
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

**Textual Figure — Multi-Pattern Search for ["she","sea","the"] in text:**

```
  Input:  text = "she sells sea shells by the sea shore"
          patterns = ["she", "sea", "the"]   (all length 3)
          base = 31, mod = 1_000_000_007

  Step 1 — Group patterns by length:
    ┌──────────┬──────────────────┐
    │ Length   │ Patterns         │
    ├──────────┼──────────────────┤
    │ k = 3    │ she, sea, the    │
    └──────────┴──────────────────┘

  Step 2 — Hash each pattern (k=3):
    hash("she") = (115×31² + 104×31 + 101) mod M = H_she
    hash("sea") = (115×31² + 101×31 + 97)  mod M = H_sea
    hash("the") = (116×31² + 104×31 + 101) mod M = H_the

    patHashes map:
    ┌────────────┬────────────────┐
    │ Hash       │ Patterns       │
    ├────────────┼────────────────┤
    │ H_she      │ ["she"]        │
    │ H_sea      │ ["sea"]        │
    │ H_the      │ ["the"]        │
    └────────────┴────────────────┘

  Step 3 — Rolling hash over text with k=3:
    highPow = 31^2 = 961

    s  h  e     s  e  l  l  s     s  e  a     ...
    0  1  2  3  4  5  6  7  8  9  10 11 12
    [─────]  window 0: "she" → hash matches H_she
              Verify: "she"=="she" ✓ → found at 0
       [─────] window 1: "he " → no match
          [─────] window 2: "e s" → no match
    ...
              s  e  a           window: "sea"
              [─────]  hash matches H_sea
              Verify ✓ → found at 10
    ...
                    t  h  e     window: "the"
                    [─────]  hash matches H_the
                    Verify ✓ → found at 24
    ...
                       s  e  a  window: "sea" again
                       [─────]  → found at 28

    Hash update formula:
    ┌────────────────────────────────────────────────────┐
    │ hash = (hash×31 + text[i]) mod M                   │
    │ if i >= k:                                         │
    │   hash = (hash - text[i-k]×highPow×31) mod M       │
    │              remove old char ──┘                    │
    └────────────────────────────────────────────────────┘

  Result:
    ┌──────────┬────────────────┐
    │ Pattern  │ Positions      │
    ├──────────┼────────────────┤
    │ "she"    │ [0, 14]        │
    │ "sea"    │ [10, 28]       │
    │ "the"    │ [24]           │
    └──────────┴────────────────┘
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

**Textual Figure — Cyclic Rotation Check: is "cdeab" a rotation of "abcde"?**

```
  Input:  s1 = "abcde"   s2 = "cdeab"   base=31  mod=10^9+7

  Step 1 — Concatenate s1 + s1:
    doubled = "abcde" + "abcde" = "abcdeabcde"
                                   0123456789

    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │ a │ b │ c │ d │ e │
    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
      0   1   2   3   4   5   6   7   8   9

  Step 2 — Hash pattern s2 = "cdeab" (k=5):
    patHash = (c×31⁴ + d×31³ + e×31² + a×31 + b) mod M

    highPow = 31^4 = 923521

  Step 3 — Rolling hash over doubled string:
    Window 0: [a b c d e]  hash ≠ patHash
               ▼
    Window 1:   [b c d e a]  hash ≠ patHash
                 ▼
    Window 2:     [c d e a b]  hash = patHash ✓
    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │ a │ b │ c │ d │ e │
    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
              ▲───────────────▲
              └── "cdeab" ───┘   position 2

    Verify: doubled[2:7] == "cdeab" ✓ → return true!

    Hash update at each slide:
    ┌──────────────────────────────────────────────────────┐
    │ new_hash = (old_hash - old_char×base^(k-1))×base     │
    │                      + new_char             (mod M)  │
    │                                                      │
    │ e.g. Window 0→1:                                     │
    │   remove 'a' (×31⁴), multiply by 31, add 'a'        │
    └──────────────────────────────────────────────────────┘

  Key Insight: If s2 is a rotation of s1, it MUST appear
  as a substring of s1+s1.

  Results:
    isRotation("abcde", "cdeab") → true   (found at pos 2)
    isRotation("abcde", "abced") → false  (not found)
    isRotation("aab",   "aba")  → true   (found in "aabaab")
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

**Textual Figure — Longest Repeated Substring via Binary Search + Rolling Hash on "banana":**

```
  Input: s = "banana"   n = 6    base=31  mod=10^9+7

  Binary search on length L ∈ [0, 5]:
  ┌────────────────────────────────────────────────────────┐
  │ lo=0  hi=5                                             │
  │                                                        │
  │ Iter 1: mid=2 → findRepeated(s, 2)                     │
  │   Rolling hash with k=2, highPow=31                    │
  │   ┌───┬───┬───┬───┬───┬───┐                            │
  │   │ b │ a │ n │ a │ n │ a │                            │
  │   └───┴───┴───┴───┴───┴───┘                            │
  │    [───]  "ba" hash→seen                               │
  │      [───] "an" hash→seen                              │
  │        [───] "na" hash→seen                            │
  │          [───] "an" hash = seen["an"] ✓ FOUND!         │
  │   Result: "an"   → lo = 3                              │
  │                                                        │
  │ Iter 2: mid=4 → findRepeated(s, 4)                     │
  │   Rolling hash with k=4                                │
  │   ┌───┬───┬───┬───┬───┬───┐                            │
  │   │ b │ a │ n │ a │ n │ a │                            │
  │   └───┴───┴───┴───┴───┴───┘                            │
  │    [───────────]  "bana" hash→seen                     │
  │      [───────────] "anan" hash→seen                    │
  │        [───────────] "nana" hash→seen                  │
  │   No duplicate found → hi = 3                          │
  │                                                        │
  │ Iter 3: mid=3 → findRepeated(s, 3)                     │
  │   Rolling hash with k=3                                │
  │   ┌───┬───┬───┬───┬───┬───┐                            │
  │   │ b │ a │ n │ a │ n │ a │                            │
  │   └───┴───┴───┴───┴───┴───┘                            │
  │    [───────]  "ban" hash→seen                          │
  │      [───────] "ana" hash→seen                         │
  │        [───────] "nan" hash→seen                       │
  │          [───────] "ana" hash = seen["ana"] ✓ FOUND!   │
  │   Result: "ana"  → lo = 4                              │
  │                                                        │
  │ Iter 4: lo=4 > hi=3 → STOP                             │
  └────────────────────────────────────────────────────────┘

  Hash update formula (each slide):
    new_hash = (old_hash - s[i-k]×highPow×base) × ... + s[i]  (mod M)
               remove leftmost ──┘           add rightmost ──┘

  Final Result: "ana"  (length 3, appears at indices 1 and 3)
    ┌───┬───┬───┬───┬───┬───┐
    │ b │ a │ n │ a │ n │ a │
    └───┴───┴───┴───┴───┴───┘
          ▲───────▲              "ana" at index 1
                  ▲───────▲      "ana" at index 3
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

**Textual Figure — Window Uniqueness: find windows of size 3 with all unique chars in "abcabcbb":**

```
  Input: s = "abcabcbb"   k = 3

  Sliding window with frequency map:

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ b │ c │ a │ b │ c │ b │ b │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7

  ┌───────┬──────────┬──────────────────────┬─────────┬─────────┐
  │ i     │ Window   │ freq map             │ unique  │ Result? │
  ├───────┼──────────┼──────────────────────┼─────────┼─────────┤
  │ 0–2   │ "abc"    │ {a:1, b:1, c:1}      │ 3 == k  │ ✓ "abc" │
  │       │          │                      │         │         │
  │ 1–3   │ "bca"    │ +a, -a→{b:1,c:1,a:1}│ 3 == k  │ ✓ "bca" │
  │       │          │                      │         │         │
  │ 2–4   │ "cab"    │ +b, -b→{c:1,a:1,b:1}│ 3 == k  │ ✓ "cab" │
  │       │          │                      │         │         │
  │ 3–5   │ "abc"    │ +c, -c→{a:1,b:1,c:1}│ 3 == k  │ ✓ "abc" │
  │       │          │                      │         │         │
  │ 4–6   │ "bcb"    │ +b, -a→{b:2, c:1}   │ 2 < k   │ ✗       │
  │       │          │                      │         │         │
  │ 5–7   │ "cbb"    │ +b, -b→{c:1, b:2}   │ 2 < k   │ ✗       │
  └───────┴──────────┴──────────────────────┴─────────┴─────────┘

  Window operation per slide:
    ┌───────────────────────────────────────────┐
    │ Add s[i]:    freq[s[i]]++              │
    │   if freq[s[i]] == 1 → unique++        │
    │ Remove s[i-k]: freq[s[i-k]]--          │
    │   if freq[s[i-k]] == 0 → unique--      │
    │ Check: unique == k  → all unique!       │
    └───────────────────────────────────────────┘

  Result: ["abc", "bca", "cab", "abc"]
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

**Textual Figure — 2D Rolling Hash: search for pattern in matrix:**

```
  Input Matrix (3×4):                Pattern (2×2):
  ┌───┬───┬───┬───┐                  ┌───┬───┐
  │ a │ b │ c │ d │  row 0          │ f │ g │
  ├───┼───┼───┼───┤                  ├───┼───┤
  │ e │ f │ g │ h │  row 1          │ j │ k │
  ├───┼───┼───┼───┤                  └───┴───┘
  │ i │ j │ k │ l │  row 2
  └───┴───┴───┴───┘

  bases: b1=31 (row), b2=37 (col)   mod=10^9+7

  Step 1 — Hash the pattern (2×2):
    patHash = f×(b1^0)×(b2^0) + g×(b1^0)×(b2^1)
            + j×(b1^1)×(b2^0) + k×(b1^1)×(b2^1)  (mod M)

    patHash = f×1×1 + g×1×37 + j×31×1 + k×31×37 (mod M)

  Step 2 — Try each (r,c) top-left corner:
    (0,0): [a,b]   hash ≠ patHash
           [e,f]

    (0,1): [b,c]   hash ≠ patHash
           [f,g]

    (0,2): [c,d]   hash ≠ patHash
           [g,h]

    (1,0): [e,f]   hash ≠ patHash
           [i,j]

    (1,1): [f,g]   hash = patHash  ✓
           [j,k]
    ┌───┬───┬───┬───┐
    │ a │ b │ c │ d │
    ├───┼═══┼═══┼───┤
    │ e ║ f ║ g ║ h │   ← pattern found here!
    ├───┼═══┼═══┼───┤       at row=1, col=1
    │ i ║ j ║ k ║ l │
    └───┴───┴───┴───┘
           Verify: matrix[1..2][1..2] == pattern ✓

  2D hash formula per submatrix:
    ┌───────────────────────────────────────────────┐
    │ h = Σ  matrix[r+dr][c+dc] × b1^dr × b2^dc  │
    │     dr,dc                       (mod M)  │
    └───────────────────────────────────────────────┘

  Result: Found: true at (1, 1)
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

**Textual Figure — Rolling Hash with Modular Inverse (Append/PopFront on "abcde"):**

```
  Input: s = "abcde"   base=31   mod=10^9+7

  Modular Inverse: invBase = modPow(31, mod-2, mod)
  (Fermat’s little theorem: 31^(mod-2) ≡ 31⁻¹ mod M)

  Step 1 — Build hash for "abc" via Append:
    ┌───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │
    └───┴───┴───┴───┴───┘
    [─────────]

    Append('a'): hash = 0 + a×1       = a        highPow = 31
    Append('b'): hash = a + b×31      = a+31b    highPow = 31²
    Append('c'): hash = a+31b + c×31² = a+31b+961c  highPow = 31³

    Hash scheme: h = c₀×31⁰ + c₁×31¹ + c₂×31²  (positional)
                     a          b          c

  Step 2 — PopFront('a'), Append('d') → "bcd":
    ┌───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │
    └───┴───┴───┴───┴───┘
        [─────────]

    PopFront('a'):
    ┌──────────────────────────────────────────────────┐
    │ hash = (hash - a) × invBase   mod M              │
    │        remove 'a' from position 0                │
    │        divide by base to shift all positions down │
    │ highPow = highPow × invBase  (31³ → 31²)         │
    └──────────────────────────────────────────────────┘

    Now: hash = b×31⁰ + c×31¹    ("bc" at positions 0,1)

    Append('d'):
    ┌──────────────────────────────────────────────────┐
    │ hash = hash + d × highPow                          │
    │ highPow = highPow × base   (31² → 31³)            │
    └──────────────────────────────────────────────────┘
    Now: hash = b×31⁰ + c×31¹ + d×31²  = hash("bcd")

  Step 3 — PopFront('b'), Append('e') → "cde":
    ┌───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ e │
    └───┴───┴───┴───┴───┘
            [─────────]

    PopFront('b'): hash = (hash - b) × invBase  mod M
    Append('e'):   hash = hash + e × highPow

    Final: hash = c×31⁰ + d×31¹ + e×31² = hash("cde")

  Key advantage: modular inverse allows O(1) PopFront
  without storing base^(k-1) separately.
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

**Textual Figure — O(1) Substring Comparison via Prefix Hash on "abcabcabc":**

```
  Input: s = "abcabcabc"   base=31   mod=10^9+7

  Step 1 — Build prefix hash array h[] and power array p[]:
    p[i] = 31^i mod M

    Index:  0   1   2   3   4   5   6   7   8
    Char:   a   b   c   a   b   c   a   b   c

    h[0] = 0
    h[1] = h[0] + a×p[0] = a×1       = 97
    h[2] = h[1] + b×p[1] = 97 + 98×31
    h[3] = h[2] + c×p[2] = ... + 99×961
    ...and so on for all 9 characters

    ┌───┬────┬─────────┬──────────────────┐
    │ i │ p[i]│ h[i]    │ Formula          │
    ├───┼────┼─────────┼──────────────────┤
    │ 0 │  1  │ 0       │ base case        │
    │ 1 │ 31  │ 97      │ h[0]+a*p[0]      │
    │ 2 │ 961 │ 97+3038 │ h[1]+b*p[1]      │
    │...│ ... │ ...     │ h[i-1]+s[i]*p[i] │
    └───┴────┴─────────┴──────────────────┘

  Step 2 — O(1) substring comparison:
    RawHash(l, r) = (h[r+1] - h[l] + M) mod M

    Compare s[0..2] vs s[3..5]:   ("abc" vs "abc")
    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ a │ b │ c │ a │ b │ c │
    └───┴───┴───┴───┴───┴───┴───┴───┴───┘
    [─────────]              l1=0, r1=2
              [─────────]   l2=3, r2=5

    h1 = RawHash(0,2)   h2 = RawHash(3,5)
    Normalize: l1 < l2, so h1 = h1 × p[3]  (shift to align)
    h1 == h2  →  true ✓

    Compare s[0..2] vs s[1..3]:   ("abc" vs "bca")
    [─────────]              l1=0, r1=2
       [─────────]           l2=1, r2=3
    h1 × p[1] ≠ h2  →  false ✗

    Compare s[1..4] vs s[4..7]:   ("bcab" vs "bcab")
       [─────────────]        l1=1, r1=4
                 [─────────────] l2=4, r2=7
    h1 × p[3] == h2  →  true ✓

  Key: No recomputation needed — just array lookups + multiply!
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

**Textual Figure — Repeated String Match: min repetitions of "abcd" to contain "cdabcdab":**

```
  Input: a = "abcd"   b = "cdabcdab"   base=31  mod=10^9+7

  Step 1 — Build repeated string until len >= len(b):
    len(b) = 8,  len(a) = 4

    Repeat 1: "abcd"          len=4  < 8
    Repeat 2: "abcdabcd"      len=8  >= 8 ✓  count=2
    + one more: "abcdabcdabcd" len=12       maxCount=3

    repeated = "abcdabcdabcd"
               0 1 2 3 4 5 6 7 8 9 10 11

  Step 2 — Hash pattern b = "cdabcdab" (k=8):
    patHash = (c×31⁷+d×31⁶+a×31⁵+b×31⁴+c×31³+d×31²+a×31+b) mod M
    highPow = 31^7

  Step 3 — Rolling hash over repeated string:
    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ a │ b │ c │ d │ a │ b │ c │ d │
    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
      0   1   2   3   4   5   6   7   8   9  10  11

    Window [0..7]: "abcdabcd"  hash ≠ patHash

    Slide → remove 'a', add 'a':
    new_hash = (old_hash - 'a'×highPow)×31 + 'a'  mod M

    Window [1..8]: "bcdabcda"  hash ≠ patHash

    Slide → remove 'b', add 'b':
    Window [2..9]: "cdabcdab"  hash = patHash  ✓
    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │ b │ c │ d │ a │ b │ c │ d │ a │ b │ c │ d │
    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
          ▲───────────────────────▲
          └── "cdabcdab" ───────┘   pos 2..9

    Verify: repeated[2:10] == "cdabcdab" ✓

  Step 4 — How many repetitions needed?
    Match at start=2, end=9
    needed = (9) / len(a) + 1 = 9/4 + 1 = 2+1 = 3

  Hash update formula:
    ┌──────────────────────────────────────────────────────┐
    │ new_hash = (old_hash - old_char×base^(k-1))×base     │
    │                      + new_char             (mod M)  │
    └──────────────────────────────────────────────────────┘

  Results:
    ┌──────────────┬────────────────┬────────┐
    │ a            │ b              │ Answer │
    ├──────────────┼────────────────┼────────┤
    │ "abcd"       │ "cdabcdab"     │   3    │
    │ "a"          │ "aa"           │   2    │
    │ "abc"        │ "cab"          │   2    │
    │ "abc"        │ "xyz"          │  -1    │
    └──────────────┴────────────────┴────────┘
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
