# Phase 3: Strings — Palindrome Detection

## What is Palindrome Detection?

A **palindrome** reads the same forwards and backwards. Palindrome detection covers:
- Checking if a string is a palindrome
- Finding palindromic substrings
- **Expand around center** — O(n²) approach
- **Manacher's algorithm** — finds all palindromic substrings in O(n)

---

## Example 1: Basic Palindrome Check

```go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func isPalindrome(s string) bool {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}

// Case-insensitive, alphanumeric only
func isPalindromeClean(s string) bool {
    s = strings.ToLower(s)
    runes := []rune{}
    for _, r := range s {
        if unicode.IsLetter(r) || unicode.IsDigit(r) {
            runes = append(runes, r)
        }
    }
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        if runes[i] != runes[j] {
            return false
        }
    }
    return true
}

func main() {
    fmt.Println(isPalindrome("racecar"))   // true
    fmt.Println(isPalindrome("hello"))     // false
    fmt.Println(isPalindrome("abba"))      // true

    fmt.Println(isPalindromeClean("A man, a plan, a canal: Panama")) // true
    fmt.Println(isPalindromeClean("race a car"))                      // false
}
```

---

## Example 2: Expand Around Center

```go
package main

import "fmt"

func longestPalindrome(s string) string {
    if len(s) < 2 {
        return s
    }

    start, maxLen := 0, 1

    expand := func(left, right int) {
        for left >= 0 && right < len(s) && s[left] == s[right] {
            if right-left+1 > maxLen {
                start = left
                maxLen = right - left + 1
            }
            left--
            right++
        }
    }

    for i := 0; i < len(s); i++ {
        expand(i, i)   // odd length palindromes
        expand(i, i+1) // even length palindromes
    }

    return s[start : start+maxLen]
}

func main() {
    tests := []string{"babad", "cbbd", "racecar", "a", "ac", "aacabdkacaa"}
    for _, s := range tests {
        fmt.Printf("%q → %q\n", s, longestPalindrome(s))
    }
    // "babad"       → "bab" or "aba"
    // "cbbd"        → "bb"
    // "racecar"     → "racecar"
    // "aacabdkacaa" → "aca"
}
```

---

## Example 3: Count All Palindromic Substrings

```go
package main

import "fmt"

func countPalindromicSubstrings(s string) int {
    count := 0

    expand := func(left, right int) {
        for left >= 0 && right < len(s) && s[left] == s[right] {
            count++
            left--
            right++
        }
    }

    for i := 0; i < len(s); i++ {
        expand(i, i)   // odd
        expand(i, i+1) // even
    }

    return count
}

func main() {
    fmt.Println(countPalindromicSubstrings("abc"))    // 3 (a, b, c)
    fmt.Println(countPalindromicSubstrings("aaa"))    // 6 (a,a,a,aa,aa,aaa)
    fmt.Println(countPalindromicSubstrings("abba"))   // 6 (a,b,b,a,bb,abba)
    fmt.Println(countPalindromicSubstrings("racecar")) // 10
}
```

---

## Example 4: Manacher's Algorithm

```go
package main

import "fmt"

// Returns array p where p[i] = radius of palindrome centered at i
// in the transformed string (with separators)
func manacher(s string) (string, []int) {
    // Transform: "abc" → "^#a#b#c#$"
    t := "^#"
    for _, c := range s {
        t += string(c) + "#"
    }
    t += "$"

    n := len(t)
    p := make([]int, n)
    center, right := 0, 0

    for i := 1; i < n-1; i++ {
        mirror := 2*center - i

        if i < right {
            p[i] = min(right-i, p[mirror])
        }

        // Expand
        for t[i+p[i]+1] == t[i-p[i]-1] {
            p[i]++
        }

        // Update center
        if i+p[i] > right {
            center = i
            right = i + p[i]
        }
    }

    return t, p
}

func longestPalindromeManacher(s string) string {
    t, p := manacher(s)

    maxLen := 0
    centerIdx := 0
    for i := 1; i < len(t)-1; i++ {
        if p[i] > maxLen {
            maxLen = p[i]
            centerIdx = i
        }
    }

    // Convert back to original string indices
    start := (centerIdx - maxLen) / 2
    return s[start : start+maxLen]
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    tests := []string{"babad", "cbbd", "racecar", "aacabdkacaa", "abacaba"}
    for _, s := range tests {
        fmt.Printf("%q → %q\n", s, longestPalindromeManacher(s))
    }
}
```

---

## Example 5: Manacher — All Palindromes at Each Center

```go
package main

import "fmt"

func allPalindromes(s string) [][]string {
    t := "^#"
    for _, c := range s {
        t += string(c) + "#"
    }
    t += "$"

    n := len(t)
    p := make([]int, n)
    center, right := 0, 0

    for i := 1; i < n-1; i++ {
        if i < right {
            p[i] = min(right-i, p[2*center-i])
        }
        for t[i+p[i]+1] == t[i-p[i]-1] {
            p[i]++
        }
        if i+p[i] > right {
            center, right = i, i+p[i]
        }
    }

    result := make([][]string, len(s))
    for i := 0; i < len(s); i++ {
        // Odd-length palindromes centered at i
        ti := 2*i + 2 // position in transformed string
        radius := p[ti]
        for r := 1; r <= radius; r++ {
            if (ti-r)%2 == 0 { // only pick actual characters
                continue
            }
            start := (ti - r) / 2
            end := (ti + r) / 2
            result[i] = append(result[i], s[start:end])
        }
    }
    return result
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    s := "abacaba"
    pals := allPalindromes(s)
    for i, list := range pals {
        if len(list) > 0 {
            fmt.Printf("Center %d ('%c'): %v\n", i, s[i], list)
        }
    }
}
```

---

## Example 6: Palindrome Partitioning

```go
package main

import "fmt"

func partition(s string) [][]string {
    var result [][]string
    var current []string

    // Precompute palindrome table
    n := len(s)
    isPalin := make([][]bool, n)
    for i := range isPalin {
        isPalin[i] = make([]bool, n)
        isPalin[i][i] = true
    }
    for l := 2; l <= n; l++ {
        for i := 0; i <= n-l; i++ {
            j := i + l - 1
            if s[i] == s[j] {
                if l == 2 || isPalin[i+1][j-1] {
                    isPalin[i][j] = true
                }
            }
        }
    }

    var backtrack func(start int)
    backtrack = func(start int) {
        if start == n {
            temp := make([]string, len(current))
            copy(temp, current)
            result = append(result, temp)
            return
        }
        for end := start; end < n; end++ {
            if isPalin[start][end] {
                current = append(current, s[start:end+1])
                backtrack(end + 1)
                current = current[:len(current)-1]
            }
        }
    }

    backtrack(0)
    return result
}

func main() {
    s := "aab"
    partitions := partition(s)
    fmt.Printf("Palindrome partitions of %q:\n", s)
    for _, p := range partitions {
        fmt.Println(p)
    }
    // [a a b]
    // [aa b]

    fmt.Println()
    for _, p := range partition("abba") {
        fmt.Println(p)
    }
    // [a b b a], [a bb a], [abba]
}
```

---

## Example 7: Min Cuts for Palindrome Partitioning

```go
package main

import "fmt"

func minCut(s string) int {
    n := len(s)

    // isPalin[i][j] = true if s[i..j] is a palindrome
    isPalin := make([][]bool, n)
    for i := range isPalin {
        isPalin[i] = make([]bool, n)
        isPalin[i][i] = true
    }
    for l := 2; l <= n; l++ {
        for i := 0; i <= n-l; i++ {
            j := i + l - 1
            if s[i] == s[j] && (l == 2 || isPalin[i+1][j-1]) {
                isPalin[i][j] = true
            }
        }
    }

    // dp[i] = min cuts for s[0..i]
    dp := make([]int, n)
    for i := 0; i < n; i++ {
        if isPalin[0][i] {
            dp[i] = 0
            continue
        }
        dp[i] = i // worst case: cut every character
        for j := 1; j <= i; j++ {
            if isPalin[j][i] && dp[j-1]+1 < dp[i] {
                dp[i] = dp[j-1] + 1
            }
        }
    }

    return dp[n-1]
}

func main() {
    fmt.Println(minCut("aab"))     // 1 (aa|b)
    fmt.Println(minCut("a"))       // 0
    fmt.Println(minCut("ab"))      // 1
    fmt.Println(minCut("aaabba"))  // 1 (aaa|bba → wait: aaa|bb|a = 2, a|aabba → not palindrome... let's check)
    fmt.Println(minCut("abcba"))   // 0 (already palindrome)
}
```

---

## Example 8: Valid Palindrome II (Remove at Most One Character)

```go
package main

import "fmt"

func validPalindrome(s string) bool {
    isPalin := func(s string, l, r int) bool {
        for l < r {
            if s[l] != s[r] {
                return false
            }
            l++
            r--
        }
        return true
    }

    l, r := 0, len(s)-1
    for l < r {
        if s[l] != s[r] {
            // Try removing either l or r
            return isPalin(s, l+1, r) || isPalin(s, l, r-1)
        }
        l++
        r--
    }
    return true
}

func main() {
    fmt.Println(validPalindrome("aba"))     // true
    fmt.Println(validPalindrome("abca"))    // true (remove 'c' → "aba")
    fmt.Println(validPalindrome("abc"))     // false
    fmt.Println(validPalindrome("deeee"))   // true (remove 'd')
    fmt.Println(validPalindrome("racecar")) // true
}
```

---

## Example 9: Longest Palindromic Subsequence (DP)

```go
package main

import "fmt"

func longestPalinSubseq(s string) int {
    n := len(s)
    // dp[i][j] = length of longest palindromic subsequence in s[i..j]
    dp := make([][]int, n)
    for i := range dp {
        dp[i] = make([]int, n)
        dp[i][i] = 1
    }

    for length := 2; length <= n; length++ {
        for i := 0; i <= n-length; i++ {
            j := i + length - 1
            if s[i] == s[j] {
                dp[i][j] = dp[i+1][j-1] + 2
            } else {
                dp[i][j] = max(dp[i+1][j], dp[i][j-1])
            }
        }
    }
    return dp[0][n-1]
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func main() {
    fmt.Println(longestPalinSubseq("bbbab"))  // 4 ("bbbb")
    fmt.Println(longestPalinSubseq("cbbd"))   // 2 ("bb")
    fmt.Println(longestPalinSubseq("agbdba")) // 5 ("abdba")
    fmt.Println(longestPalinSubseq("a"))      // 1
}
```

---

## Example 10: Palindrome Pairs

```go
package main

import "fmt"

func palindromePairs(words []string) [][]int {
    wordMap := make(map[string]int)
    for i, w := range words {
        wordMap[w] = i
    }

    var result [][]int

    for i, word := range words {
        for j := 0; j <= len(word); j++ {
            prefix := word[:j]
            suffix := word[j:]

            // If suffix is palindrome, check if reverse(prefix) exists
            if isPalin(suffix) {
                rev := reverse(prefix)
                if idx, ok := wordMap[rev]; ok && idx != i {
                    result = append(result, []int{i, idx})
                }
            }

            // If prefix is palindrome, check if reverse(suffix) exists
            if j > 0 && isPalin(prefix) {
                rev := reverse(suffix)
                if idx, ok := wordMap[rev]; ok && idx != i {
                    result = append(result, []int{idx, i})
                }
            }
        }
    }
    return result
}

func isPalin(s string) bool {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}

func reverse(s string) string {
    b := []byte(s)
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        b[i], b[j] = b[j], b[i]
    }
    return string(b)
}

func main() {
    words := []string{"abcd", "dcba", "lls", "s", "sssll"}
    pairs := palindromePairs(words)
    fmt.Println("Palindrome pairs:")
    for _, p := range pairs {
        i, j := p[0], p[1]
        fmt.Printf("  [%d,%d] → %q + %q = %q\n",
            i, j, words[i], words[j], words[i]+words[j])
    }
}
```

---

## Example 11: Manacher O(n) — Full Implementation with Unique Palindromes

```go
package main

import "fmt"

func uniquePalindromes(s string) []string {
    t := "^#"
    for _, c := range s {
        t += string(c) + "#"
    }
    t += "$"

    n := len(t)
    p := make([]int, n)
    center, right := 0, 0

    for i := 1; i < n-1; i++ {
        if i < right {
            p[i] = min(right-i, p[2*center-i])
        }
        for t[i+p[i]+1] == t[i-p[i]-1] {
            p[i]++
        }
        if i+p[i] > right {
            center, right = i, i+p[i]
        }
    }

    // Collect unique palindromes
    seen := make(map[string]bool)
    for i := 1; i < n-1; i++ {
        if p[i] > 0 {
            // Extract palindrome from original string
            start := (i - p[i]) / 2
            length := p[i]
            if length > 0 {
                sub := s[start : start+length]
                if len(sub) > 1 {
                    seen[sub] = true
                }
            }
        }
    }

    var result []string
    for pal := range seen {
        result = append(result, pal)
    }
    return result
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    s := "abaaba"
    pals := uniquePalindromes(s)
    fmt.Printf("Unique palindromes in %q:\n", s)
    for _, p := range pals {
        fmt.Printf("  %q\n", p)
    }
}
```

---

## Palindrome Detection Complexity

| Algorithm              | Time   | Space  |
|-----------------------|--------|--------|
| Basic check           | O(n)   | O(1)   |
| Expand around center  | O(n²)  | O(1)   |
| Manacher's algorithm  | O(n)   | O(n)   |
| Palindromic subsequence (DP) | O(n²) | O(n²) |
| Palindrome partitioning | O(2^n) | O(n)  |

## Key Takeaways

1. **Two-pointer** check: O(n) for single palindrome verification
2. **Expand around center**: O(n²) — simple, handles both odd/even lengths
3. **Manacher's**: O(n) — precomputes palindrome radii for all centers
4. **Transformation trick**: insert `#` between chars to unify odd/even cases
5. **LPS + KMP**: minimum insertions to make palindrome
6. **Palindrome pairs**: hash map + prefix/suffix decomposition

> **Phase 3 Complete! Next up:** Phase 4 — Hash Tables →
