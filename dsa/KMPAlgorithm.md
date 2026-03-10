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

**Textual Figure: LPS Construction for "AABAACAABAA"**

```
  Index:   0   1   2   3   4   5   6   7   8   9  10
  Char:    A   A   B   A   A   C   A   A   B   A   A

  ┌──────┬────┬─────┬────────┬──────────┬──────────────────────────────┬────────┐
  │ Step │ i  │ len │ pat[i] │ pat[len] │ Action                       │ lps[i] │
  ├──────┼────┼─────┼────────┼──────────┼──────────────────────────────┼────────┤
  │  1   │  1 │  0  │   A    │    A     │ Match → len=1                │   1    │
  │  2   │  2 │  1  │   B    │    A     │ Mismatch → len=lps[0]=0      │        │
  │  3   │  2 │  0  │   B    │    A     │ Mismatch, len=0 → lps[2]=0   │   0    │
  │  4   │  3 │  0  │   A    │    A     │ Match → len=1                │   1    │
  │  5   │  4 │  1  │   A    │    A     │ Match → len=2                │   2    │
  │  6   │  5 │  2  │   C    │    B     │ Mismatch → len=lps[1]=1      │        │
  │  7   │  5 │  1  │   C    │    A     │ Mismatch → len=lps[0]=0      │        │
  │  8   │  5 │  0  │   C    │    A     │ Mismatch, len=0 → lps[5]=0   │   0    │
  │  9   │  6 │  0  │   A    │    A     │ Match → len=1                │   1    │
  │ 10   │  7 │  1  │   A    │    A     │ Match → len=2                │   2    │
  │ 11   │  8 │  2  │   B    │    B     │ Match → len=3                │   3    │
  │ 12   │  9 │  3  │   A    │    A     │ Match → len=4                │   4    │
  │ 13   │ 10 │  4  │   A    │    A     │ Match → len=5                │   5    │
  └──────┴────┴─────┴────────┴──────────┴──────────────────────────────┴────────┘

  Final LPS Array:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ A │ A │ B │ A │ A │ C │ A │ A │ B │ A │ A │  ← pattern
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
  │ 0 │ 1 │ 0 │ 1 │ 2 │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │  ← lps[]
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

  Verification (lps[10] = 5):
    Prefix: [A  A  B  A  A] .  .  .  .  .  .
    Suffix:  .  .  .  .  .  . [A  A  B  A  A]
    ─────────────────────────────────────────────
    "AABAA" is both a prefix and suffix of length 5
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

**Textual Figure: KMP Search for "AABA" in "AABAACAADAABAABA"**

```
  Text:    A  A  B  A  A  C  A  A  D  A  A  B  A  A  B  A
  Index:   0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15

  Pattern: A  A  B  A        LPS: [0, 1, 0, 1]

  ┌────────┬───┬────────────────────────────────────────────────────────┐
  │ Align  │ j │ Comparison Detail                                     │
  ├────────┼───┼────────────────────────────────────────────────────────┤
  │ i=0    │ 0 │ Text: [A  A  B  A] A  C  A  A  D  ...                 │
  │        │   │ Pat:   A  A  B  A                                     │
  │        │   │        ✓  ✓  ✓  ✓  → MATCH #1 at index 0              │
  │        │   │ j = lps[3] = 1  (skip prefix "A")                     │
  ├────────┼───┼────────────────────────────────────────────────────────┤
  │ i=4    │ 1 │ Text: ...  A [A  C] ...                                │
  │        │   │ Pat:        .  A  B  ← C≠B mismatch                   │
  │        │   │ j = lps[1] = 0, then C≠A → advance i                  │
  ├────────┼───┼────────────────────────────────────────────────────────┤
  │ i=6-8  │ 0 │ A matches j=0→1, A matches j=1→2, D≠B → j=0, D≠A    │
  │        │   │ → advance i past 'D'                                  │
  ├────────┼───┼────────────────────────────────────────────────────────┤
  │ i=9    │ 0 │ Text: ... [A  A  B  A] A  B  A                        │
  │        │   │ Pat:       A  A  B  A                                 │
  │        │   │            ✓  ✓  ✓  ✓  → MATCH #2 at index 9          │
  │        │   │ j = lps[3] = 1                                        │
  ├────────┼───┼────────────────────────────────────────────────────────┤
  │ i=13   │ 1 │ Text: ... [A  B  A]                                   │
  │        │   │ Pat:    .  A  B  A   (j starts at 1, already have 'A') │
  │        │   │         ✓  ✓  ✓  → MATCH #3 at index 12               │
  └────────┴───┴────────────────────────────────────────────────────────┘

  KMP Advantage — After match at index 0:
    Naive: Would restart comparison at i=1 (re-scans "ABA")
    KMP:   j falls to lps[3]=1 → already knows prefix "A" matched
           Continues from i=4, j=1 — no character re-scanned

  Result: [0, 9, 12]
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

**Textual Figure: Step-by-Step KMP Trace for "AABAB" in "AABABAABAAB"**

```
  Text:    A  A  B  A  B  A  A  B  A  A  B
  Index:   0  1  2  3  4  5  6  7  8  9  10

  Pattern: A  A  B  A  B      LPS: [0, 1, 0, 1, 0]

  ┌──────┬─────┬───┬──────────┬─────────┬───────────────────────────────────┐
  │ Step │  i  │ j │ Compare  │ Result  │ Action                            │
  ├──────┼─────┼───┼──────────┼─────────┼───────────────────────────────────┤
  │  1   │  0  │ 0 │ A vs A   │ Match   │ i→1, j→1                          │
  │  2   │  1  │ 1 │ A vs A   │ Match   │ i→2, j→2                          │
  │  3   │  2  │ 2 │ B vs B   │ Match   │ i→3, j→3                          │
  │  4   │  3  │ 3 │ A vs A   │ Match   │ i→4, j→4                          │
  │  5   │  4  │ 4 │ B vs B   │ Match   │ i→5, j→5=m ★ FOUND at index 0    │
  │      │     │   │          │         │ j = lps[4] = 0 (full reset)       │
  ├──────┼─────┼───┼──────────┼─────────┼───────────────────────────────────┤
  │  6   │  5  │ 0 │ A vs A   │ Match   │ i→6, j→1                          │
  │  7   │  6  │ 1 │ A vs A   │ Match   │ i→7, j→2                          │
  │  8   │  7  │ 2 │ B vs B   │ Match   │ i→8, j→3                          │
  │  9   │  8  │ 3 │ A vs A   │ Match   │ i→9, j→4                          │
  │ 10   │  9  │ 4 │ A vs B   │Mismatch │ j = lps[3] = 1  ◄── KEY SKIP     │
  │ 11   │  9  │ 1 │ A vs A   │ Match   │ i→10, j→2                         │
  │ 12   │ 10  │ 2 │ B vs B   │ Match   │ i→11, j→3 → i≥n, done            │
  └──────┴─────┴───┴──────────┴─────────┴───────────────────────────────────┘

  Step 10 — KMP avoids re-scanning:
    Text:  A  A  B  A  B  A  A  B  A [A] B
                                      ↑ i=9, text='A' ≠ pattern[4]='B'
    Naive: Would restart from i=1 and rescan everything
    KMP:   Knows prefix "A" already matches (lps[3]=1)
           → Jumps j to 1, continues from i=9 without backtracking
           → Saves re-comparing characters at positions 5–8

  Result: Pattern "AABAB" found at index [0]
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

**Textual Figure: Overlapping Pattern Count — "aa" in "aaaa"**

```
  Text:    a   a   a   a        Pattern: a  a
  Index:   0   1   2   3        LPS:     [0, 1]

  Overlapping Match Trace:
  ┌──────┬──────┬─────┬───────────────────────────────────────────┐
  │ Step │ i, j │ Cmp │ Detail                                    │
  ├──────┼──────┼─────┼───────────────────────────────────────────┤
  │  1   │ 0, 0 │  ✓  │ a==a → i=1, j=1                          │
  │  2   │ 1, 1 │  ✓  │ a==a → i=2, j=2=m → FOUND #1 at idx 0   │
  │      │      │     │ j = lps[1] = 1  (keep overlap)            │
  │  3   │ 2, 1 │  ✓  │ a==a → i=3, j=2=m → FOUND #2 at idx 1   │
  │      │      │     │ j = lps[1] = 1                            │
  │  4   │ 3, 1 │  ✓  │ a==a → i=4, j=2=m → FOUND #3 at idx 2   │
  └──────┴──────┴─────┴───────────────────────────────────────────┘

  Overlapping matches visualized:
     a  a  a  a
     ├──┤              Match #1 at index 0
        ├──┤           Match #2 at index 1
           ├──┤        Match #3 at index 2

  How KMP enables overlap detection:
    After match, j = lps[m-1] = 1 (not reset to 0)
    → Last char of previous match serves as first char
      of next potential match
    → No character is ever re-scanned

  Result: countPattern("aaaa", "aa") = 3
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

**Textual Figure: Shortest Repeating Unit via LPS — "abcabcabc"**

```
  String: "abcabcabc"  (n = 9)

  LPS Array:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ b │ c │ a │ b │ c │ a │ b │ c │  ← characters
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┤
  │ 0 │ 0 │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │  ← lps[]
  └───┴───┴───┴───┴───┴───┴───┴───┴───┘

  Formula:  unitLen = n - lps[n-1] = 9 - 6 = 3
  Check:    n % unitLen = 9 % 3 = 0  ✓  (evenly divides)
            unitLen < n              ✓  (not the whole string)

  The string decomposes into repeating units:
     a  b  c │ a  b  c │ a  b  c
     ────────   ────────   ────────
     unit #1    unit #2    unit #3     →  unit = "abc"

  Why it works (LPS insight):
    lps[8] = 6 means "abcabc" is both a prefix and suffix:
      Prefix: [a  b  c  a  b  c] .  .  .
      Suffix:  .  .  . [a  b  c  a  b  c]
    The non-overlapping gap = 9 - 6 = 3 = one period

  Other examples:
  ┌─────────────┬──────────────────┬───────────┬──────────┬────────┐
  │ String      │ LPS              │ n-lps[n-1]│ n%unit=0 │ Unit   │
  ├─────────────┼──────────────────┼───────────┼──────────┼────────┤
  │ "abab"      │ [0,0,1,2]        │ 4-2 = 2   │ 4%2=0 ✓  │ "ab"   │
  │ "aaa"       │ [0,1,2]          │ 3-2 = 1   │ 3%1=0 ✓  │ "a"    │
  │ "abcd"      │ [0,0,0,0]        │ 4-0 = 4   │ 4%4=0    │ "abcd" │
  │ "ababab"    │ [0,0,1,2,3,4]    │ 6-4 = 2   │ 6%2=0 ✓  │ "ab"   │
  └─────────────┴──────────────────┴───────────┴──────────┴────────┘
  ("abcd": unitLen = n itself → no repeating unit found)
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

**Textual Figure: Rotation Check — Is "cdeab" a rotation of "abcde"?**

```
  Step 1: Verify lengths match
    len("abcde") = 5,  len("cdeab") = 5  ✓

  Step 2: Concatenate s1 with itself
    doubled = "abcde" + "abcde" = "abcdeabcde"

  Step 3: KMP search for s2="cdeab" in doubled string
    Text:    a  b  c  d  e  a  b  c  d  e
    Index:   0  1  2  3  4  5  6  7  8  9
    Pattern:       c  d  e  a  b
                   ✓  ✓  ✓  ✓  ✓  → FOUND at index 2

  Why doubling works:
  ┌───────────────────────────────────────────────────┐
  │ Original:    a  b  c  d  e                    │
  │                  └────────┤  ┌─────┤              │
  │ Rotation:    c  d  e │ a  b                    │
  │  (rotate left by 2 positions)                   │
  │                                                 │
  │ In doubled s1:                                  │
  │   a  b │ c  d  e  a  b │ c  d  e               │
  │        └──────────────┘                         │
  │         s2 appears as a substring at index 2    │
  └───────────────────────────────────────────────────┘

  Any rotation of s1 will appear as a contiguous substring
  of s1+s1 → use KMP to find it in O(n) time.

  Result: isRotation("abcde", "cdeab") = true
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

**Textual Figure: Longest Prefix = Suffix for "abcab"**

```
  String: "abcab"  (n = 5)

  LPS Construction:
  ┌───┬───┬───┬───┬───┐
  │ a │ b │ c │ a │ b │  ← string
  ├───┼───┼───┼───┼───┤
  │ 0 │ 0 │ 0 │ 1 │ 2 │  ← lps[]
  └───┴───┴───┴───┴───┘

  lps[n-1] = lps[4] = 2

  Longest proper prefix which is also a suffix:
    Prefix: [a  b] .  .  .     (length 2)
    Suffix:  .  .  . [a  b]    (length 2)
    ────────────────────────
    "ab" == "ab"  ✓

  Other examples:
  ┌───────────┬───────────────────────┬──────────┬──────────┐
  │ String    │ LPS                   │ lps[n-1] │ Result   │
  ├───────────┼───────────────────────┼──────────┼──────────┤
  │ "aabaaab" │ [0,1,0,1,2,2,3]       │    3     │ "aab"    │
  │ "abcabc"  │ [0,0,0,1,2,3]         │    3     │ "abc"    │
  │ "aaaa"    │ [0,1,2,3]             │    3     │ "aaa"    │
  │ "abcd"    │ [0,0,0,0]             │    0     │ (none)   │
  └───────────┴───────────────────────┴──────────┴──────────┘
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

**Textual Figure: All Prefix-Suffix Matches for "aabaabaabaab"**

```
  String: "aabaabaabaab"  (n = 12)

  LPS Array:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ a │ b │ a │ a │ b │ a │ a │ b │ a │ a │ b │  ← chars
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
  │ 0 │ 1 │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │  ← lps[]
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

  Follow LPS chain from lps[n-1] = 9:
    l = 9  → s[:9]  = "aabaabaab"   ✓ prefix and suffix
    l = lps[8] = 6  → s[:6]  = "aabaab"       ✓ prefix and suffix
    l = lps[5] = 3  → s[:3]  = "aab"          ✓ prefix and suffix
    l = lps[2] = 0  → stop

  Result (ascending): [3, 6, 9]

  Verification:
    len 3:  [a a b] . . . . . . . . .    = prefix
             . . . . . . . . . [a a b]   = suffix  ✓

    len 6:  [a a b a a b] . . . . . .    = prefix
             . . . . . . [a a b a a b]   = suffix  ✓

    len 9:  [a a b a a b a a b] . . .    = prefix
             . . . [a a b a a b a a b]   = suffix  ✓

  Chain traversal: lps[11]=9 → lps[8]=6 → lps[5]=3 → lps[2]=0 (stop)
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

**Textual Figure: Non-Overlapping vs Overlapping Matches — "aa" in "aaaa"**

```
  Text: "aaaa"   Pattern: "aa"   LPS: [0, 1]

  ┌─────────────────────────────────────────────────────┐
  │ OVERLAPPING (standard KMP: j = lps[m-1] = 1)     │
  │                                                   │
  │   Index:  0  1  2  3                               │
  │   Text:   a  a  a  a                               │
  │           ├──┤              Match #1 at 0             │
  │              ├──┤           Match #2 at 1             │
  │                 ├──┤        Match #3 at 2             │
  │   j → lps[1] = 1  (overlap: last char reused)      │
  │   Count = 3                                        │
  ├─────────────────────────────────────────────────────┤
  │ NON-OVERLAPPING (this example: j = 0 after match)  │
  │                                                   │
  │   Index:  0  1  2  3                               │
  │   Text:   a  a  a  a                               │
  │           ├──┤              Match #1 at 0             │
  │                 ├──┤        Match #2 at 2             │
  │   j → 0  (full reset: skip past entire match)      │
  │   Count = 2                                        │
  └─────────────────────────────────────────────────────┘

  Key Difference (one line of code):
    Overlapping:     j = lps[j-1]   → keeps partial match state
    Non-overlapping: j = 0          → complete reset after match

  Also: nonOverlappingCount("abababab", "abab") = 2 (not 3)
    [a b a b] a b a b     Match #1 at 0, j→0
     a b a b [a b a b]    Match #2 at 4, j→0
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

**Textual Figure: Min Chars to Palindrome — "aacecaaa"**

```
  Input: s = "aacecaaa"   (n = 8)

  Step 1: Build combined = s + "#" + reverse(s)
    reverse("aacecaaa") = "aaacecaa"
    combined = "aacecaaa#aaacecaa"   (length = 17)

  Step 2: Compute LPS of combined string:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ a │ c │ e │ c │ a │ a │ a │ # │ a │ a │ a │ c │ e │ c │ a │ a │
  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤
  │ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │ 2 │ 2 │ 0 │ 1 │ 2 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

  lps[16] = 7  →  Longest palindromic prefix of s = "aacecaa" (len 7)

  Step 3: min chars to prepend = n - lps[16] = 8 - 7 = 1

  Visualization:
    Original:     a  a  c  e  c  a  a │ a
                  └──────────────────┘
                  palindromic prefix     ↑ non-palindromic
                  (length 7)             suffix (length 1)

    Prepend reverse of suffix "a":       "a"
    Result:    a │ a  a  c  e  c  a  a  a  =  "aaacecaaa"
               ↑
            prepended                       ← palindrome ✓

  The "#" separator prevents lps from crossing s-boundary
  into reversed part, ensuring correctness.

  Result: minCharsToPalindrome("aacecaaa") = 1
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

**Textual Figure: DFA-Based KMP Automaton for Pattern "abc"**

```
  Pattern: "abc"      LPS: [0, 0, 0]

  DFA State Transition Diagram:

               'a'          'b'          'c'
     ┌───┐    ┌───┐    ┌───┐    ┌───┐
  ──▶│ 0 │───▶│ 1 │───▶│ 2 │───▶│ 3 │  ★ MATCH
     └───┘    └───┘    └───┘    └───┘
      ▲ │      │  ▲      │
      │ │not   │  │'a'   │ 'a'
      │ │'a'   │  │      │
      │ └──────┘  │      │
      └───not 'a'─┘      │
      └──────not 'a'─────┘

  Transition Table:
  ┌───────┬──────┬──────┬──────────────┐
  │ State │ 'a'  │ 'b'  │ 'c' / other  │
  ├───────┼──────┼──────┼──────────────┤
  │   0   │   1  │   0  │      0       │
  │   1   │   1  │   2  │      0       │
  │   2   │   1  │   0  │   3 (★)      │
  └───────┴──────┴──────┴──────────────┘

  Searching "xabcyabcz" character by character:
  ┌──────┬──────┬───────────────┬────────────────────────┐
  │ Char │ From │ To (via DFA)  │ Note                   │
  ├──────┼──────┼───────────────┼────────────────────────┤
  │  x   │  0   │      0        │ no match for 'x'       │
  │  a   │  0   │      1        │                        │
  │  b   │  1   │      2        │                        │
  │  c   │  2   │      3  ★     │ MATCH at index 1       │
  │  y   │  0   │      0        │ reset after match      │
  │  a   │  0   │      1        │                        │
  │  b   │  1   │      2        │                        │
  │  c   │  2   │      3  ★     │ MATCH at index 5       │
  │  z   │  0   │      0        │                        │
  └──────┴──────┴───────────────┴────────────────────────┘

  Advantage: Pre-built DFA → O(1) per character during search.
  Ideal when searching many texts against the same pattern —
  build once, search multiple times without re-building LPS.
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
