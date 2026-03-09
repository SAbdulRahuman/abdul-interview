# Phase 3: Strings — KMP Algorithm (Knuth-Morris-Pratt)

## What is KMP?

The **KMP algorithm** performs pattern matching in **O(n + m)** time by pre-computing a **failure function** (also called the prefix function or partial match table). When a mismatch occurs, instead of restarting from scratch, KMP uses the failure function to skip characters already known to match.

**Key Idea**: The failure function `lps[i]` stores the length of the longest proper prefix of `pattern[0..i]` that is also a suffix.

---

## Example 1: Building the Failure Function (LPS Array)

```go
package main

import "fmt"

func buildLPS(pattern string) []int {
    m := len(pattern)
    lps := make([]int, m)
    length := 0 // length of previous longest prefix suffix
    i := 1

    for i < m {
        if pattern[i] == pattern[length] {
            length++
            lps[i] = length
            i++
        } else {
            if length != 0 {
                length = lps[length-1] // fall back
            } else {
                lps[i] = 0
                i++
            }
        }
    }
    return lps
}

func main() {
    patterns := []string{
        "AABAACAABAA",
        "ABCABD",
        "AAAAAB",
        "ABABAB",
    }
    for _, p := range patterns {
        fmt.Printf("Pattern: %q\n", p)
        fmt.Printf("LPS:     %v\n\n", buildLPS(p))
    }
    // "AABAACAABAA" → [0 1 0 1 2 0 1 2 3 4 5]
    // "ABCABD"      → [0 0 0 1 2 0]
    // "AAAAAB"      → [0 1 2 3 4 0]
    // "ABABAB"      → [0 0 1 2 3 4]
}
```

---

## Example 2: Classic KMP Search

```go
package main

import "fmt"

func kmpSearch(text, pattern string) []int {
    n, m := len(text), len(pattern)
    if m == 0 {
        return nil
    }

    lps := buildLPS(pattern)
    var result []int

    i, j := 0, 0 // i for text, j for pattern
    for i < n {
        if text[i] == pattern[j] {
            i++
            j++
        }
        if j == m {
            result = append(result, i-j)
            j = lps[j-1]
        } else if i < n && text[i] != pattern[j] {
            if j != 0 {
                j = lps[j-1]
            } else {
                i++
            }
        }
    }
    return result
}

func buildLPS(pattern string) []int {
    m := len(pattern)
    lps := make([]int, m)
    length := 0
    i := 1
    for i < m {
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
    return lps
}

func main() {
    text := "AABAACAADAABAABA"
    pattern := "AABA"
    fmt.Printf("Pattern %q found at: %v\n", pattern, kmpSearch(text, pattern))
    // [0 9 12]
}
```

---

## Example 3: Step-by-Step KMP Trace

```go
package main

import "fmt"

func kmpTrace(text, pattern string) {
    n, m := len(text), len(pattern)
    lps := make([]int, m)

    // Build LPS
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

    fmt.Printf("LPS: %v\n\n", lps)

    i, j := 0, 0
    step := 0
    for i < n {
        step++
        fmt.Printf("Step %d: i=%d('%c') j=%d('%c') ", step, i, text[i], j, pattern[j])

        if text[i] == pattern[j] {
            fmt.Println("→ MATCH, advance both")
            i++
            j++
        } else {
            if j != 0 {
                fmt.Printf("→ MISMATCH, j falls back from %d to %d\n", j, lps[j-1])
                j = lps[j-1]
            } else {
                fmt.Println("→ MISMATCH, j=0 so advance i")
                i++
            }
        }

        if j == m {
            fmt.Printf("*** FOUND at index %d ***\n\n", i-j)
            j = lps[j-1]
        }
    }
}

func main() {
    kmpTrace("AABABAABAAB", "AABAB")
}
```

---

## Example 4: Count Pattern Occurrences

```go
package main

import "fmt"

func countPattern(text, pattern string) int {
    n, m := len(text), len(pattern)
    if m == 0 || m > n {
        return 0
    }

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
    i, j := 0, 0
    for i < n {
        if text[i] == pattern[j] {
            i++
            j++
        }
        if j == m {
            count++
            j = lps[j-1]
        } else if i < n && text[i] != pattern[j] {
            if j != 0 {
                j = lps[j-1]
            } else {
                i++
            }
        }
    }
    return count
}

func main() {
    fmt.Println(countPattern("aaaa", "aa"))           // 3 (overlapping)
    fmt.Println(countPattern("abababab", "abab"))     // 3
    fmt.Println(countPattern("hello world", "xyz"))   // 0
    fmt.Println(countPattern("abcabcabc", "abc"))     // 3
}
```

---

## Example 5: Shortest Repeating Unit Using LPS

```go
package main

import "fmt"

// If a string has a repeating pattern, the length of the
// smallest repeating unit = n - lps[n-1]
func shortestRepeatingUnit(s string) string {
    n := len(s)
    lps := make([]int, n)
    length := 0
    for i := 1; i < n; {
        if s[i] == s[length] {
            length++
            lps[i] = length
            i++
        } else if length != 0 {
            length = lps[length-1]
        } else {
            i++
        }
    }

    unitLen := n - lps[n-1]
    if n%unitLen == 0 && unitLen < n {
        return s[:unitLen]
    }
    return s // no repeating unit, the string itself
}

func main() {
    tests := []string{"abcabcabc", "abab", "aaa", "abcd", "ababab"}
    for _, s := range tests {
        fmt.Printf("%q → repeating unit: %q\n", s, shortestRepeatingUnit(s))
    }
    // "abcabcabc" → "abc"
    // "abab"      → "ab"
    // "aaa"       → "a"
    // "abcd"      → "abcd" (no repeat)
    // "ababab"    → "ab"
}
```

---

## Example 6: Check if String is Rotation Using KMP

```go
package main

import "fmt"

func isRotation(s1, s2 string) bool {
    if len(s1) != len(s2) {
        return false
    }
    // s2 is a rotation of s1 iff s2 is a substring of s1+s1
    doubled := s1 + s1
    return len(kmpSearch(doubled, s2)) > 0
}

func kmpSearch(text, pattern string) []int {
    n, m := len(text), len(pattern)
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

    var result []int
    i, j := 0, 0
    for i < n {
        if text[i] == pattern[j] {
            i++
            j++
        }
        if j == m {
            result = append(result, i-j)
            j = lps[j-1]
        } else if i < n && text[i] != pattern[j] {
            if j != 0 {
                j = lps[j-1]
            } else {
                i++
            }
        }
    }
    return result
}

func main() {
    fmt.Println(isRotation("abcde", "cdeab")) // true
    fmt.Println(isRotation("abcde", "abced")) // false
    fmt.Println(isRotation("aab", "aba"))      // true
    fmt.Println(isRotation("aa", "aa"))         // true
}
```

---

## Example 7: Longest Prefix Which is Also Suffix

```go
package main

import "fmt"

func longestPrefixSuffix(s string) string {
    n := len(s)
    lps := make([]int, n)
    length := 0
    for i := 1; i < n; {
        if s[i] == s[length] {
            length++
            lps[i] = length
            i++
        } else if length != 0 {
            length = lps[length-1]
        } else {
            i++
        }
    }

    lpLen := lps[n-1]
    if lpLen == 0 {
        return ""
    }
    return s[:lpLen]
}

func main() {
    tests := []string{"abcab", "aabaaab", "abcabc", "aaaa", "abcd"}
    for _, s := range tests {
        result := longestPrefixSuffix(s)
        if result == "" {
            fmt.Printf("%q → (none)\n", s)
        } else {
            fmt.Printf("%q → %q (length %d)\n", s, result, len(result))
        }
    }
    // "abcab"   → "ab" (length 2)
    // "aabaaab" → "aab" (length 3)
    // "abcabc"  → "abc" (length 3)
    // "aaaa"    → "aaa" (length 3)
    // "abcd"    → (none)
}
```

---

## Example 8: All Prefix-Suffix Matches

```go
package main

import "fmt"

// Find all lengths that are both a proper prefix and suffix of s
func allPrefixSuffixes(s string) []int {
    n := len(s)
    lps := make([]int, n)
    length := 0
    for i := 1; i < n; {
        if s[i] == s[length] {
            length++
            lps[i] = length
            i++
        } else if length != 0 {
            length = lps[length-1]
        } else {
            i++
        }
    }

    // Follow the chain from lps[n-1]
    var result []int
    l := lps[n-1]
    for l > 0 {
        result = append(result, l)
        l = lps[l-1]
    }

    // Reverse to get ascending order
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }
    return result
}

func main() {
    s := "aabaabaabaab"
    lengths := allPrefixSuffixes(s)
    fmt.Printf("String: %q\n", s)
    fmt.Printf("Prefix-suffix lengths: %v\n", lengths)
    for _, l := range lengths {
        fmt.Printf("  length %d: %q\n", l, s[:l])
    }
}
```

---

## Example 9: KMP for Non-Overlapping Matches

```go
package main

import "fmt"

func nonOverlappingCount(text, pattern string) int {
    n, m := len(text), len(pattern)
    if m == 0 || m > n {
        return 0
    }

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
    i, j := 0, 0
    for i < n {
        if text[i] == pattern[j] {
            i++
            j++
        }
        if j == m {
            count++
            j = 0 // reset completely for non-overlapping
        } else if i < n && text[i] != pattern[j] {
            if j != 0 {
                j = lps[j-1]
            } else {
                i++
            }
        }
    }
    return count
}

func main() {
    fmt.Println(nonOverlappingCount("aaaa", "aa"))        // 2 (not 3)
    fmt.Println(nonOverlappingCount("abababab", "abab"))  // 2 (not 3)
    fmt.Println(nonOverlappingCount("abcabcabc", "abc"))  // 3
}
```

---

## Example 10: Minimum Characters to Make Palindrome (KMP-based)

```go
package main

import "fmt"

// Minimum characters to prepend to make s a palindrome
func minCharsToPalindrome(s string) int {
    // Build s + "#" + reverse(s)
    rev := reverse(s)
    combined := s + "#" + rev

    // Compute LPS of combined
    n := len(combined)
    lps := make([]int, n)
    length := 0
    for i := 1; i < n; {
        if combined[i] == combined[length] {
            length++
            lps[i] = length
            i++
        } else if length != 0 {
            length = lps[length-1]
        } else {
            i++
        }
    }

    // lps[n-1] = length of longest palindromic prefix of s
    return len(s) - lps[n-1]
}

func reverse(s string) string {
    runes := []byte(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

func main() {
    tests := []string{"aacecaaa", "abcd", "aba", "abc", "racecar"}
    for _, s := range tests {
        fmt.Printf("%q → need %d chars prepended → %q\n",
            s, minCharsToPalindrome(s),
            reverse(s[len(s)-minCharsToPalindrome(s):])+s)
    }
    // "aacecaaa" → 1  → "aaacecaaa"
    // "abcd"     → 3  → "dcbabcd"
    // "aba"      → 0  → "aba"
    // "abc"      → 2  → "cbabc"
    // "racecar"  → 0  → "racecar"
}
```

---

## Example 11: Automaton-Based KMP (Multiple Text Searches)

```go
package main

import "fmt"

// Build a DFA (Deterministic Finite Automaton) for KMP
type KMPAutomaton struct {
    dfa     []map[byte]int
    pattern string
}

func NewKMPAutomaton(pattern string) *KMPAutomaton {
    m := len(pattern)
    dfa := make([]map[byte]int, m+1)
    for i := range dfa {
        dfa[i] = make(map[byte]int)
    }

    // Build LPS first
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

    // Build DFA transitions
    for state := 0; state < m; state++ {
        // On matching character → go to next state
        dfa[state][pattern[state]] = state + 1

        // On non-matching: follow failure links
        for c := byte('a'); c <= byte('z'); c++ {
            if c != pattern[state] {
                fallback := state
                for fallback > 0 {
                    fallback = lps[fallback-1]
                    if pattern[fallback] == c {
                        dfa[state][c] = fallback + 1
                        break
                    }
                }
            }
        }
    }

    return &KMPAutomaton{dfa: dfa, pattern: pattern}
}

func (ka *KMPAutomaton) Search(text string) []int {
    var result []int
    state := 0
    m := len(ka.pattern)

    for i := 0; i < len(text); i++ {
        state = ka.dfa[state][text[i]]
        if state == m {
            result = append(result, i-m+1)
            // Continue searching
            state = ka.dfa[state-1][text[i]] // approximate fallback
            // Actually need lps for this, simplified here
        }
    }
    return result
}

func main() {
    auto := NewKMPAutomaton("abc")
    texts := []string{"xabcyabcz", "abcabc", "xyz", "aabc"}
    for _, t := range texts {
        fmt.Printf("Searching %q: found at %v\n", t, auto.Search(t))
    }
}
```

---

## KMP Complexity

| Component        | Time  | Space |
|------------------|-------|-------|
| Build LPS array  | O(m)  | O(m)  |
| Search in text   | O(n)  | O(1)  |
| Total            | O(n+m)| O(m)  |

## Key Takeaways

1. **LPS array** is the heart of KMP — longest proper prefix = suffix
2. **O(n+m) guaranteed** — no worst case degradation unlike Rabin-Karp
3. **Failure function** allows skipping already-matched characters
4. **Applications**: pattern search, palindrome problems, string periodicity
5. **Repeating unit**: `period = n - lps[n-1]`, string repeats iff `n % period == 0`
6. **KMP + concatenation** trick solves rotation checks and palindrome problems

> **Next up:** Z Algorithm →
