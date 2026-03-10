# Phase 3: Strings — Pattern Matching

## What is Pattern Matching?

Pattern matching searches for a **pattern** string within a **text** string. The naive approach is O(n×m), but advanced algorithms achieve O(n+m).

---

## Example 1: Naive/Brute Force — O(n×m)

```go
package main

import "fmt"

func naiveSearch(text, pattern string) []int {
    var matches []int
    n, m := len(text), len(pattern)

    for i := 0; i <= n-m; i++ {
        match := true
        for j := 0; j < m; j++ {
            if text[i+j] != pattern[j] {
                match = false
                break
            }
        }
        if match {
            matches = append(matches, i)
        }
    }
    return matches
}

func main() {
    fmt.Println(naiveSearch("AABAACAADAABAABA", "AABA")) // [0 9 12]
    fmt.Println(naiveSearch("abcabcabc", "abc"))          // [0 3 6]
    fmt.Println(naiveSearch("hello", "xyz"))               // []
}
```

**Textual Figure — Naive Pattern Matching:**

```
  text = "AABAACAADAABAABA",  pattern = "AABA"

  Slide pattern across text, compare char by char:

  i=0: A A B A A C A A D A A B A A B A
       A A B A               ✓ match at 0!

  i=1: A A B A A C A A D A A B A A B A
         A A B A             ✘ (B≠A at j=2)

  i=2: A A B A A C A A D A A B A A B A
           A A B A           ✘ (A≠A? no, A≠B at j=1)
  ...skip...

  i=9: A A B A A C A A D A A B A A B A
                             A A B A   ✓ match at 9!

  i=12: A A B A A C A A D A A B A A B A
                                A A B A ✓ match at 12!

  Matches: [0, 9, 12]

  Complexity: O((n-m+1) × m) = O(n×m) worst case
  Each position: up to m comparisons
```

---

## Example 2: strings.Index and strings.Contains

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    text := "the quick brown fox jumps over the lazy dog"

    // Find first occurrence
    fmt.Println(strings.Index(text, "fox"))      // 16
    fmt.Println(strings.Index(text, "cat"))      // -1

    // Find last occurrence
    fmt.Println(strings.LastIndex(text, "the"))  // 31

    // Contains
    fmt.Println(strings.Contains(text, "quick")) // true

    // Count occurrences
    fmt.Println(strings.Count("abababab", "ab")) // 4

    // Find all occurrences
    findAll := func(text, pattern string) []int {
        var indices []int
        start := 0
        for {
            idx := strings.Index(text[start:], pattern)
            if idx == -1 {
                break
            }
            indices = append(indices, start+idx)
            start += idx + 1
        }
        return indices
    }
    fmt.Println(findAll("abababab", "aba")) // [0 2 4]
}
```

**Textual Figure — strings.Index Find All:**

```
  text = "abababab",  pattern = "aba"

  Finding ALL overlapping occurrences:

  start=0: strings.Index("abababab", "aba") = 0
           a b a b a b a b
           a b a           ✓ found at 0
           start = 0 + 1 = 1

  start=1: strings.Index("bababab", "aba") = 1 → actual pos = 2
           a b a b a b a b
               a b a       ✓ found at 2
           start = 2 + 1 = 3

  start=3: strings.Index("babab", "aba") = 1 → actual pos = 4
           a b a b a b a b
                   a b a   ✓ found at 4
           start = 4 + 1 = 5

  start=5: strings.Index("bab", "aba") = -1 → done

  Result: [0, 2, 4]  (overlapping matches!)
```

---

## Example 3: Wildcard Matching

```go
package main

import "fmt"

// ? matches any single character, * matches any sequence
func isMatch(s, p string) bool {
    m, n := len(s), len(p)
    dp := make([][]bool, m+1)
    for i := range dp {
        dp[i] = make([]bool, n+1)
    }
    dp[0][0] = true

    // Handle leading *s
    for j := 1; j <= n; j++ {
        if p[j-1] == '*' {
            dp[0][j] = dp[0][j-1]
        }
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if p[j-1] == '*' {
                dp[i][j] = dp[i-1][j] || dp[i][j-1]
            } else if p[j-1] == '?' || s[i-1] == p[j-1] {
                dp[i][j] = dp[i-1][j-1]
            }
        }
    }
    return dp[m][n]
}

func main() {
    fmt.Println(isMatch("adceb", "*a*b"))       // true
    fmt.Println(isMatch("acdcb", "a*c?b"))      // false
    fmt.Println(isMatch("hello", "h?ll*"))       // true
    fmt.Println(isMatch("abc", "a*"))            // true
    fmt.Println(isMatch("abc", "a?c"))           // true
}
```

**Textual Figure — Wildcard Matching DP:**

```
  s = "adceb",  p = "*a*b"
  ? = any single char,  * = any sequence (including empty)

  DP table: dp[i][j] = s[:i] matches p[:j]

        ""   *    a    *    b
    j:  0    1    2    3    4
  ┌───┬───┬───┬───┬───┬───┐
  │   │ "" │ * │ a │ * │ b │
  ├───┼───┼───┼───┼───┼───┤
  │"" │ T │ T │ F │ F │ F │  i=0
  │ a │ F │ T │ T │ T │ F │  i=1
  │ d │ F │ T │ F │ T │ F │  i=2
  │ c │ F │ T │ F │ T │ F │  i=3
  │ e │ F │ T │ F │ T │ F │  i=4
  │ b │ F │ T │ F │ T │ T │  i=5  ← answer!
  └───┴───┴───┴───┴───┴───┘

  dp[5][4] = T → "adceb" matches "*a*b"  ✓

  Rules:
  '*': dp[i][j] = dp[i-1][j] || dp[i][j-1]  (use or skip)
  '?': dp[i][j] = dp[i-1][j-1]              (must match 1)
```

---

## Example 4: Regular Expression Matching (. and *)

```go
package main

import "fmt"

func isMatchRegex(s, p string) bool {
    m, n := len(s), len(p)
    dp := make([][]bool, m+1)
    for i := range dp {
        dp[i] = make([]bool, n+1)
    }
    dp[0][0] = true

    // Handle patterns like a*, a*b*, a*b*c*
    for j := 2; j <= n; j++ {
        if p[j-1] == '*' {
            dp[0][j] = dp[0][j-2]
        }
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if p[j-1] == '*' {
                // * = zero occurrences: skip pattern pair
                dp[i][j] = dp[i][j-2]
                // * = one or more: if pattern[j-2] matches s[i-1]
                if p[j-2] == '.' || p[j-2] == s[i-1] {
                    dp[i][j] = dp[i][j] || dp[i-1][j]
                }
            } else if p[j-1] == '.' || p[j-1] == s[i-1] {
                dp[i][j] = dp[i-1][j-1]
            }
        }
    }
    return dp[m][n]
}

func main() {
    fmt.Println(isMatchRegex("aa", "a"))     // false
    fmt.Println(isMatchRegex("aa", "a*"))    // true
    fmt.Println(isMatchRegex("ab", ".*"))    // true
    fmt.Println(isMatchRegex("aab", "c*a*b")) // true (c*=empty, a*=aa)
    fmt.Println(isMatchRegex("mississippi", "mis*is*ip*.")) // true
}
```

**Textual Figure — Regex Matching with . and *:**

```
  s = "aab",  p = "c*a*b"
  . = any single char,  x* = zero or more of x

  DP table:
         ""   c    *    a    *    b
    j:   0    1    2    3    4    5
  ┌───┬───┬───┬───┬───┬───┬───┐
  │   │ "" │ c │ * │ a │ * │ b │
  ├───┼───┼───┼───┼───┼───┼───┤
  │"" │ T │ F │ T │ F │ T │ F │  c*=ε, a*=ε
  │ a │ F │ F │ F │ T │ T │ F │
  │ a │ F │ F │ F │ F │ T │ F │  a* matches "aa"
  │ b │ F │ F │ F │ F │ F │ T │  ← answer!
  └───┴───┴───┴───┴───┴───┴───┘

  Key: c* = zero c's, a* = two a's, b = b
  So "c*a*b" matches "aab"  ✓

  x* rules:
  • Zero occurrences: dp[i][j] = dp[i][j-2]  (skip x*)
  • One+ occurrences: dp[i][j] = dp[i-1][j] if x matches s[i-1]
```

---

## Example 5: Implement strStr (LeetCode 28)

```go
package main

import "fmt"

func strStr(haystack, needle string) int {
    if len(needle) == 0 {
        return 0
    }
    n, m := len(haystack), len(needle)
    for i := 0; i <= n-m; i++ {
        if haystack[i:i+m] == needle {
            return i
        }
    }
    return -1
}

func main() {
    fmt.Println(strStr("hello", "ll"))     // 2
    fmt.Println(strStr("aaaaa", "bba"))    // -1
    fmt.Println(strStr("sadbutsad", "sad")) // 0
    fmt.Println(strStr("leetcode", "leeto")) // -1
}
```

**Textual Figure — strStr (Find Substring):**

```
  haystack = "hello",  needle = "ll"

  i=0: haystack[0:2] = "he" ≠ "ll"
  i=1: haystack[1:3] = "el" ≠ "ll"
  i=2: haystack[2:4] = "ll" == "ll" → return 2  ✓

  Visualization:
  h e l l o
  └─┘         "he" ✘
    └─┘       "el" ✘
      └─┘     "ll" ✓  → return 2!

  Key optimization: haystack[i:i+m] uses Go's slice
  comparison which is efficient for short patterns.
  For long patterns, use KMP or Rabin-Karp instead.
```

---

## Example 6: Two-Pointer Pattern Match

```go
package main

import "fmt"

// Check if pattern is a subsequence of text
func isSubsequence(sub, text string) bool {
    i, j := 0, 0
    for i < len(sub) && j < len(text) {
        if sub[i] == text[j] {
            i++
        }
        j++
    }
    return i == len(sub)
}

func main() {
    fmt.Println(isSubsequence("ace", "abcde"))  // true
    fmt.Println(isSubsequence("aec", "abcde"))  // false
    fmt.Println(isSubsequence("", "hello"))      // true
    fmt.Println(isSubsequence("abc", "ahbgdc"))  // true
    fmt.Println(isSubsequence("axc", "ahbgdc"))  // false
}
```

**Textual Figure — Subsequence Check (Two Pointers):**

```
  sub = "ace",  text = "abcde"
  i = pointer into sub,  j = pointer into text

  j=0: text[0]='a' == sub[0]='a' → i=1, j=1
       a b c d e
       ↑               match!
       a c e
       ↑

  j=1: text[1]='b' ≠ sub[1]='c' → j=2
  j=2: text[2]='c' == sub[1]='c' → i=2, j=3
       a b c d e
           ↑             match!
       a c e
         ↑

  j=3: text[3]='d' ≠ sub[2]='e' → j=4
  j=4: text[4]='e' == sub[2]='e' → i=3, j=5

  i == len(sub) = 3 → true!  (all chars found in order)

  "axc" in "ahbgdc":
  a✓ x? (never found) → false

  O(n) time, O(1) space
```

---

## Example 7: Repeated Substring Pattern

```go
package main

import (
    "fmt"
    "strings"
)

// Check if s can be formed by repeating a substring
func repeatedSubstringPattern(s string) bool {
    // Trick: if s is a repeated pattern, then (s+s)[1:-1] contains s
    doubled := s + s
    return strings.Contains(doubled[1:len(doubled)-1], s)
}

// Alternative: check all divisor lengths
func repeatedSubstringDivisor(s string) bool {
    n := len(s)
    for l := 1; l <= n/2; l++ {
        if n%l == 0 {
            pattern := s[:l]
            match := true
            for i := l; i < n; i += l {
                if s[i:i+l] != pattern {
                    match = false
                    break
                }
            }
            if match {
                return true
            }
        }
    }
    return false
}

func main() {
    fmt.Println(repeatedSubstringPattern("abab"))     // true (ab)
    fmt.Println(repeatedSubstringPattern("aba"))      // false
    fmt.Println(repeatedSubstringPattern("abcabcabc")) // true (abc)
    fmt.Println(repeatedSubstringDivisor("aaa"))       // true (a)
}
```

**Textual Figure — Repeated Substring Pattern:**

```
  s = "abab"  → Is it made of a repeating pattern?

  Method 1: Double-string trick
  doubled = s + s = "abababab"
  Remove first and last char: "bababab"
  Does "bababab" contain "abab"?  YES! → true

  Why this works:
  "abab" + "abab" = "abab|abab"
  Remove edges:     "_bab ab ab_"
                      still contains "abab"!

  If NOT repeating: "abc" + "abc" = "abcabc"
  Remove edges: "_bcab_"  does NOT contain "abc"

  Method 2: Check all divisor lengths
  n = 4,  check l = 1, 2 (divisors of 4, ≤ n/2)

  l=1: pattern="a", check "a"|"b"|"a"|"b" → "b"≠"a" ✘
  l=2: pattern="ab", check "ab"|"ab" → all match ✓

  Result: true ("ab" repeats twice)
```

---

## Example 8: Shortest Superstring Containing All Words

```go
package main

import "fmt"

// Find overlap between suffix of a and prefix of b
func overlap(a, b string) int {
    maxOverlap := len(a)
    if len(b) < maxOverlap {
        maxOverlap = len(b)
    }
    for i := maxOverlap; i > 0; i-- {
        if a[len(a)-i:] == b[:i] {
            return i
        }
    }
    return 0
}

func main() {
    a, b := "abcde", "cdefg"
    ov := overlap(a, b)
    merged := a + b[ov:]
    fmt.Printf("'%s' + '%s' overlap=%d → '%s'\n", a, b, ov, merged)
    // 'abcde' + 'cdefg' overlap=3 → 'abcdefg'

    // More examples
    pairs := [][2]string{
        {"GCTA", "CTAG"},
        {"hello", "world"},
        {"abcabc", "abcabc"},
    }
    for _, p := range pairs {
        ov := overlap(p[0], p[1])
        fmt.Printf("'%s' + '%s' overlap=%d\n", p[0], p[1], ov)
    }
}
```

**Textual Figure — String Overlap Merging:**

```
  a = "abcde",  b = "cdefg"

  Check suffix of a vs prefix of b:
  len=5: "abcde" == "cdefg"[:5]? No (different lengths ok, but no match)
  len=4: "bcde"  == "cdef"? No
  len=3: "cde"   == "cde"?  YES!  overlap = 3

  Merge:   a      +    b[overlap:]
         "abcde" +    "fg"
       = "abcdefg"

  Visual:
  a: [a][b][c][d][e]
  b:         [c][d][e][f][g]
             └─overlap─┘

  Merged: [a][b][c][d][e][f][g]  (no duplication)

  No overlap example: "hello" + "world" → overlap=0
  Merge: "helloworld"
```

---

## Example 9: Count Pattern Occurrences (Non-Overlapping)

```go
package main

import (
    "fmt"
    "strings"
)

func countNonOverlapping(text, pattern string) int {
    count := 0
    start := 0
    for {
        idx := strings.Index(text[start:], pattern)
        if idx == -1 {
            break
        }
        count++
        start += idx + len(pattern) // skip past the match
    }
    return count
}

func countOverlapping(text, pattern string) int {
    count := 0
    start := 0
    for {
        idx := strings.Index(text[start:], pattern)
        if idx == -1 {
            break
        }
        count++
        start += idx + 1 // advance by 1 for overlapping
    }
    return count
}

func main() {
    text := "ababababab"
    fmt.Println("Non-overlapping 'aba':", countNonOverlapping(text, "aba")) // 2
    fmt.Println("Overlapping 'aba':", countOverlapping(text, "aba"))       // 4
    fmt.Println("Non-overlapping 'ab':", countNonOverlapping(text, "ab"))   // 5
}
```

**Textual Figure — Overlapping vs Non-Overlapping Counts:**

```
  text = "ababababab",  pattern = "aba"

  Non-overlapping (skip past match):
  a b a b a b a b a b
  a b a                 ✓ found at 0, next start = 0+3 = 3
        a b a           ✘ "bab" (start=3, found at idx 1 → pos=4)
          a b a         ✓ found at 4, next start = 4+3 = 7
                a b a   ✘ not found
  Count: 2

  Overlapping (advance by 1):
  a b a b a b a b a b
  a b a                 ✓ found at 0, next start = 1
      a b a             ✓ found at 2, next start = 3
          a b a         ✓ found at 4, next start = 5
              a b a     ✓ found at 6, next start = 7
  Count: 4

  Key difference:
  Non-overlapping: start += len(pattern)  (skip match)
  Overlapping:     start += 1             (advance 1)
```

---

## Example 10: Pattern Matching with Backtracking

```go
package main

import "fmt"

// Match pattern with letters mapping to words
// pattern = "abba", s = "dog cat cat dog" → true
func wordPattern(pattern, s string) bool {
    words := splitWords(s)
    if len(pattern) != len(words) {
        return false
    }

    charToWord := make(map[byte]string)
    wordToChar := make(map[string]byte)

    for i := 0; i < len(pattern); i++ {
        c := pattern[i]
        w := words[i]

        if mapped, ok := charToWord[c]; ok {
            if mapped != w {
                return false
            }
        } else {
            charToWord[c] = w
        }

        if mapped, ok := wordToChar[w]; ok {
            if mapped != c {
                return false
            }
        } else {
            wordToChar[w] = c
        }
    }
    return true
}

func splitWords(s string) []string {
    var words []string
    start := 0
    for i := 0; i <= len(s); i++ {
        if i == len(s) || s[i] == ' ' {
            if i > start {
                words = append(words, s[start:i])
            }
            start = i + 1
        }
    }
    return words
}

func main() {
    fmt.Println(wordPattern("abba", "dog cat cat dog"))   // true
    fmt.Println(wordPattern("abba", "dog cat cat fish"))  // false
    fmt.Println(wordPattern("aaaa", "dog cat cat dog"))   // false
    fmt.Println(wordPattern("abba", "dog dog dog dog"))   // false
}
```

**Textual Figure — Word Pattern Matching:**

```
  pattern = "abba",  s = "dog cat cat dog"
  words = ["dog", "cat", "cat", "dog"]

  Build bidirectional mapping:

  i=0: pattern='a', word="dog"
       charToWord: {a:"dog"}    wordToChar: {"dog":a}

  i=1: pattern='b', word="cat"
       charToWord: {a:"dog", b:"cat"}
       wordToChar: {"dog":a, "cat":b}

  i=2: pattern='b', word="cat"
       charToWord[b]="cat" == "cat" ✓
       wordToChar["cat"]=b == b ✓

  i=3: pattern='a', word="dog"
       charToWord[a]="dog" == "dog" ✓
       wordToChar["dog"]=a == a ✓

  All match → true!

  Counterexample: "abba" vs "dog dog dog dog"
  i=0: a → "dog"
  i=1: b → "dog"  but wordToChar["dog"]=a ≠ b → false!

  Both maps needed to prevent many-to-one mappings.
```

---

## Key Takeaways

1. **Naive pattern matching**: O(n×m) — check every position
2. **Advanced algorithms** (KMP, Rabin-Karp): O(n+m) — covered in separate files
3. **`strings.Index`** uses optimized algorithms internally
4. **Wildcard matching**: DP approach for `?` and `*`
5. **Subsequence check**: two-pointer O(n)
6. **Overlap between strings**: suffix-prefix matching for merging

> **Next up:** Sliding Window on Strings →
