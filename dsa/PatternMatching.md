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

---

## Key Takeaways

1. **Naive pattern matching**: O(n×m) — check every position
2. **Advanced algorithms** (KMP, Rabin-Karp): O(n+m) — covered in separate files
3. **`strings.Index`** uses optimized algorithms internally
4. **Wildcard matching**: DP approach for `?` and `*`
5. **Subsequence check**: two-pointer O(n)
6. **Overlap between strings**: suffix-prefix matching for merging

> **Next up:** Sliding Window on Strings →
