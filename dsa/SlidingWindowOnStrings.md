# Phase 3: Strings — Sliding Window on Strings

## Sliding Window on Strings

Applies the same sliding window technique from arrays, but operates on **characters** in strings. Uses frequency arrays or hash maps to track window state.

---

## Example 1: Longest Substring Without Repeating Characters

```go
package main

import "fmt"

func lengthOfLongestSubstring(s string) int {
    lastSeen := make(map[byte]int)
    maxLen := 0
    left := 0

    for right := 0; right < len(s); right++ {
        if idx, found := lastSeen[s[right]]; found && idx >= left {
            left = idx + 1
        }
        lastSeen[s[right]] = right
        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(lengthOfLongestSubstring("abcabcbb"))  // 3 → "abc"
    fmt.Println(lengthOfLongestSubstring("bbbbb"))     // 1
    fmt.Println(lengthOfLongestSubstring("pwwkew"))    // 3 → "wke"
    fmt.Println(lengthOfLongestSubstring(""))          // 0
    fmt.Println(lengthOfLongestSubstring("dvdf"))      // 3 → "vdf"
}
```

**Textual Figure — Longest Substring Without Repeating Characters:**

```
  Input: s = "abcabcbb"

  Step    left  right  char   lastSeen update          Window       maxLen
  ─────── ───── ────── ────── ──────────────────────── ──────────── ──────
   1       0     0      'a'   a→0                      [a]           1
   2       0     1      'b'   b→1                      [ab]          2
   3       0     2      'c'   c→2                      [abc]         3
   4       3     3      'a'   a seen@0 → left=1→3      [a]           3
                                a→3
   5       3     4      'b'   b seen@1 → left stays 3  [ab]          3
                                b→4
   6       3     5      'c'   c seen@2 → left stays 3  [abc]         3
                                c→5
   7       5     6      'b'   b seen@4 → left=5        [cb]→[b]      3
                                b→6
   8       7     7      'b'   b seen@6 → left=7        [b]           3
                                b→7

  Pointer movement on string:

  Index:   0   1   2   3   4   5   6   7
  Char:    a   b   c   a   b   c   b   b
           ├───────────┤                     ← window "abc" (maxLen=3)
                       ├───────────┤         ← window "abc" (maxLen=3)
                               ├───┤         ← window "cb"  (shrinks)

  Result: 3 ("abc")
```

---

## Example 2: Minimum Window Substring

```go
package main

import (
    "fmt"
    "math"
)

func minWindow(s string, t string) string {
    need := make(map[byte]int)
    for i := 0; i < len(t); i++ {
        need[t[i]]++
    }

    have := make(map[byte]int)
    formed, required := 0, len(need)
    minLen := math.MaxInt64
    start := 0
    left := 0

    for right := 0; right < len(s); right++ {
        c := s[right]
        have[c]++
        if need[c] > 0 && have[c] == need[c] {
            formed++
        }

        for formed == required {
            if right-left+1 < minLen {
                minLen = right - left + 1
                start = left
            }
            lc := s[left]
            have[lc]--
            if need[lc] > 0 && have[lc] < need[lc] {
                formed--
            }
            left++
        }
    }
    if minLen == math.MaxInt64 {
        return ""
    }
    return s[start : start+minLen]
}

func main() {
    fmt.Println(minWindow("ADOBECODEBANC", "ABC")) // "BANC"
    fmt.Println(minWindow("a", "a"))                 // "a"
    fmt.Println(minWindow("a", "aa"))                // ""
}
```

**Textual Figure — Minimum Window Substring:**

```
  Input: s = "ADOBECODEBANC", t = "ABC"
  need: {A:1, B:1, C:1}   required = 3

  Index:  0  1  2  3  4  5  6  7  8  9 10 11 12
  Char:   A  D  O  B  E  C  O  D  E  B  A  N  C

  Step   left right  formed   Window              Action
  ────── ──── ─────  ──────── ─────────────────── ─────────────────────
   1      0    0      1       [A]                  have A=1 ✓
   2      0    1      1       [AD]                 D not needed
   3      0    2      1       [ADO]                O not needed
   4      0    3      2       [ADOB]               have B=1 ✓
   5      0    4      2       [ADOBE]              E not needed
   6      0    5      3       [ADOBEC]             have C=1 ✓ → formed!
          ┌─ shrink: save "ADOBEC" len=6, start=0
          │  left=1: remove A → formed=2, stop
   7      1    6      2       [DOBECO]             O not needed
   8      1    7      2       [DOBECOD]            D not needed
   9      1    8      2       [DOBECODЕ]           E not needed
  10      1    9      3       [DOBECODEB]          have B→ formed!
          ┌─ shrink: save len=9 (worse), skip
          │  left=2,3,4,5: shrink until formed<3
  11      5   10      3       [CODEBA]             … → formed!
          ┌─ shrink → save "CODEBA" len=6
  12      6   10      2       [ODEBA]              …
  ...     ..  ..      ..      ...                  ...
  final   9   12      3       [BANC]               len=4 ← minimum!
          ┌─ shrink: left=10 → formed drops

  Windows found (formed == required):
  ┌──────────────────────────────────────────────┐
  │  "ADOBEC"  (len 6)   start=0                │
  │  "CODEBA"  (len 6)   start=5                │
  │  "BANC"    (len 4)   start=9   ← BEST       │
  └──────────────────────────────────────────────┘

  Result: "BANC"
```

---

## Example 3: Find All Anagrams in a String

```go
package main

import "fmt"

func findAnagrams(s string, p string) []int {
    if len(s) < len(p) {
        return nil
    }
    var pCount, wCount [26]int
    for _, c := range p {
        pCount[c-'a']++
    }

    var result []int
    for i := 0; i < len(s); i++ {
        wCount[s[i]-'a']++
        if i >= len(p) {
            wCount[s[i-len(p)]-'a']--
        }
        if wCount == pCount {
            result = append(result, i-len(p)+1)
        }
    }
    return result
}

func main() {
    fmt.Println(findAnagrams("cbaebabacd", "abc")) // [0 6]
    fmt.Println(findAnagrams("abab", "ab"))         // [0 1 2]
    fmt.Println(findAnagrams("aaaa", "aa"))         // [0 1 2]
}
```

**Textual Figure — Find All Anagrams in a String:**

```
  Input: s = "cbaebabacd", p = "abc"
  pCount: a=1  b=1  c=1       window size = 3

  Index:  0   1   2   3   4   5   6   7   8   9
  Char:   c   b   a   e   b   a   b   a   c   d

  i   Window    wCount (a,b,c)   Match?   Result
  ─── ───────── ──────────────── ──────── ──────────
  0   [c]       a=0 b=0 c=1      (size<3)
  1   [cb]      a=0 b=1 c=1      (size<3)
  2   [cba]     a=1 b=1 c=1      ✓ YES    [0]
  3   [bae]     a=1 b=1 c=0,e=1  ✗
  4   [aeb]     a=1 b=1 e=1      ✗
  5   [eba]     a=1 b=1 e=1      ✗
  6   [bab]     a=1 b=2          ✗
  7   [aba]     a=2 b=1          ✗         (wait...)
       ─ remove s[4]='b' ─
  6   [bab]     a=1 b=2 c=0      ✗
  7   [aba]     a=2 b=1 c=0      ✗
  8   [bac]     a=1 b=1 c=1      ✓ YES    [0, 6]
  9   [acd]     a=1 c=1 d=1      ✗

  Sliding the fixed-size window:

    c  b  a  e  b  a  b  a  c  d
   ├──────┤                          i=2: "cba" ✓ → anagram at 0
      ├──────┤                       i=3: "bae" ✗
         ├──────┤                    i=4: "aeb" ✗
            ├──────┤                 i=5: "eba" ✗
               ├──────┤              i=6: "bab" ✗
                  ├──────┤           i=7: "aba" ✗
                     ├──────┤        i=8: "bac" ✓ → anagram at 6
                        ├──────┤     i=9: "acd" ✗

  Result: [0, 6]
```

---

## Example 4: Longest Repeating Character Replacement

```go
package main

import "fmt"

func characterReplacement(s string, k int) int {
    var count [26]int
    maxFreq := 0
    left := 0
    maxLen := 0

    for right := 0; right < len(s); right++ {
        count[s[right]-'A']++
        if count[s[right]-'A'] > maxFreq {
            maxFreq = count[s[right]-'A']
        }

        // Window is invalid if replacements needed > k
        for (right - left + 1) - maxFreq > k {
            count[s[left]-'A']--
            left++
        }

        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(characterReplacement("ABAB", 2))    // 4
    fmt.Println(characterReplacement("AABABBA", 1)) // 4
    fmt.Println(characterReplacement("AAAA", 0))    // 4
}
```

**Textual Figure — Longest Repeating Character Replacement:**

```
  Input: s = "AABABBA", k = 1
  Rule: windowLen - maxFreqInWindow <= k  (replacements needed ≤ k)

  Index:   0   1   2   3   4   5   6
  Char:    A   A   B   A   B   B   A

  Step  left right  Window   Counts       maxFreq  winLen  repl  Valid?
  ───── ──── ─────  ──────── ──────────── ──────── ─────── ───── ──────
   1     0    0     [A]      A=1          1        1       0     ✓
   2     0    1     [AA]     A=2          2        2       0     ✓
   3     0    2     [AAB]    A=2,B=1      2        3       1     ✓ (1≤1)
   4     0    3     [AABA]   A=3,B=1      3        4       1     ✓ maxLen=4
   5     0    4     [AABAB]  A=3,B=2      3        5       2     ✗ (2>1)
         ┌─ shrink: left=0→1, A=2        
         └─ now [ABAB]  A=2,B=2  maxF=2  len=4  repl=2   ✗ (2>1)
         ┌─ shrink: left=1→2, A=1
         └─ now [BAB]   A=1,B=2  maxF=2  len=3  repl=1   ✓
   6     2    5     [BABB]   A=1,B=3      3        4       1     ✓ maxLen=4
   7     2    6     [BABBA]  A=2,B=3      3        5       2     ✗ (2>1)
         ┌─ shrink: left=2→3, B=2
         └─ now [ABBA] A=2,B=2  maxF=2   len=4  repl=2   ✗
         ┌─ shrink: left=3→4, A=1
         └─ now [BBA]  A=1,B=2  maxF=2   len=3  repl=1   ✓

  Best valid windows (length = 4):
    ┌─────────────┐
    │  [A A B A]  │  replace B→A → "AAAA" (len=4)
    │  [B A B B]  │  replace A→B → "BBBB" (len=4)
    └─────────────┘

  Result: 4
```

---

## Example 5: Permutation in String

```go
package main

import "fmt"

func checkInclusion(s1, s2 string) bool {
    if len(s1) > len(s2) {
        return false
    }
    var s1Count, winCount [26]int
    for _, c := range s1 {
        s1Count[c-'a']++
    }

    for i := 0; i < len(s2); i++ {
        winCount[s2[i]-'a']++
        if i >= len(s1) {
            winCount[s2[i-len(s1)]-'a']--
        }
        if winCount == s1Count {
            return true
        }
    }
    return false
}

func main() {
    fmt.Println(checkInclusion("ab", "eidbaooo"))  // true
    fmt.Println(checkInclusion("ab", "eidboaoo"))  // false
    fmt.Println(checkInclusion("adc", "dcda"))     // true
}
```

**Textual Figure — Permutation in String:**

```
  Input: s1 = "ab", s2 = "eidbaooo"
  s1Count: a=1  b=1        window size = len(s1) = 2

  Index:  0   1   2   3   4   5   6   7
  Char:   e   i   d   b   a   o   o   o

  i   Window    winCount (a,b)    Match s1Count?   
  ─── ───────── ────────────────  ─────────────
  0   [e]       e=1               (size < 2)
  1   [ei]      e=1,i=1           ✗
  2   [id]      i=1,d=1           ✗  (remove e)
  3   [db]      d=1,b=1           ✗  (remove i)
  4   [ba]      b=1,a=1           ✓  MATCH!
                                  └─ return true

  Sliding fixed-size window (size 2):

    e   i   d   b   a   o   o   o
   ├───┤                              "ei" ✗
       ├───┤                          "id" ✗
           ├───┤                      "db" ✗
               ├───┤                  "ba" ✓ ← permutation of "ab"!

  ┌────────────────────────────────────────────────┐
  │  "ba" has counts a=1,b=1 == s1Count a=1,b=1   │
  │  → It is a permutation of "ab"                │
  └────────────────────────────────────────────────┘

  Result: true
```

---

## Example 6: Longest Substring with At Most K Distinct Characters

```go
package main

import "fmt"

func lengthOfLongestSubstringKDistinct(s string, k int) int {
    freq := make(map[byte]int)
    left := 0
    maxLen := 0

    for right := 0; right < len(s); right++ {
        freq[s[right]]++

        for len(freq) > k {
            freq[s[left]]--
            if freq[s[left]] == 0 {
                delete(freq, s[left])
            }
            left++
        }

        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}

func main() {
    fmt.Println(lengthOfLongestSubstringKDistinct("eceba", 2))      // 3 → "ece"
    fmt.Println(lengthOfLongestSubstringKDistinct("aa", 1))         // 2
    fmt.Println(lengthOfLongestSubstringKDistinct("abcadcacacaca", 3)) // 11
}
```

**Textual Figure — Longest Substring with At Most K Distinct Characters:**

```
  Input: s = "eceba", k = 2

  Index:   0   1   2   3   4
  Char:    e   c   e   b   a

  Step  left right  Window   freq map        distinct  Valid?  maxLen
  ───── ──── ─────  ──────── ──────────────── ───────── ─────── ──────
   1     0    0     [e]      {e:1}            1        ✓       1
   2     0    1     [ec]     {e:1,c:1}        2        ✓       2
   3     0    2     [ece]    {e:2,c:1}        2        ✓       3
   4     0    3     [eceb]   {e:2,c:1,b:1}    3        ✗
         ┌─ shrink: remove s[0]='e' → {e:1,c:1,b:1} still 3
         │  shrink: remove s[1]='c' → {e:1,b:1}  now 2 ✓
         └─ left=2
   5     2    3     [eb]     {e:1,b:1}        2        ✓       3
   6     2    4     [eba]    {e:1,b:1,a:1}    3        ✗
         ┌─ shrink: remove s[2]='e' → {b:1,a:1}  now 2 ✓
         └─ left=3
   7     3    4     [ba]     {b:1,a:1}        2        ✓       3

  Pointer trace:
    e   c   e   b   a
   ├───────────┤              "ece"  distinct={e,c}≤2 ✓ maxLen=3
               ├───┤          "eb"   distinct={e,b}≤2 ✓
                   ├───┤      "ba"   distinct={b,a}≤2 ✓

  Result: 3 (→ "ece")
```

---

## Example 7: String Compression Using Sliding Window

```go
package main

import (
    "fmt"
    "strconv"
)

func compress(chars []byte) int {
    write := 0
    read := 0

    for read < len(chars) {
        ch := chars[read]
        count := 0

        // Count consecutive same characters
        for read < len(chars) && chars[read] == ch {
            read++
            count++
        }

        chars[write] = ch
        write++

        if count > 1 {
            for _, c := range strconv.Itoa(count) {
                chars[write] = byte(c)
                write++
            }
        }
    }
    return write
}

func main() {
    chars := []byte{'a', 'a', 'b', 'b', 'c', 'c', 'c'}
    k := compress(chars)
    fmt.Println(string(chars[:k])) // a2b2c3

    chars2 := []byte{'a'}
    k2 := compress(chars2)
    fmt.Println(string(chars2[:k2])) // a

    chars3 := []byte{'a', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b'}
    k3 := compress(chars3)
    fmt.Println(string(chars3[:k3])) // ab12
}
```

**Textual Figure — String Compression Using Sliding Window:**

```
  Input: chars = ['a','a','b','b','c','c','c']

  read/write pointer trace:

  Step   read  write  ch   count   chars (in-place)           Action
  ────── ───── ────── ──── ────── ────────────────────────── ──────────────
   1       0     0    'a'    ─     [a, a, b, b, c, c, c]     start group
           1          'a'    2     read 'a','a' → count=2
                                   write: chars[0]='a'
                                   write: chars[1]='2'
   2       2     2    'b'    ─     [a, 2, b, b, c, c, c]
           3          'b'    2     read 'b','b' → count=2
                                   write: chars[2]='b'
                                   write: chars[3]='2'
   3       4     4    'c'    ─     [a, 2, b, 2, c, c, c]
           5,6        'c'    3     read 'c','c','c' → count=3
                                   write: chars[4]='c'
                                   write: chars[5]='3'

  Final state:
  ┌───┬───┬───┬───┬───┬───┬───┐
  │ a │ 2 │ b │ 2 │ c │ 3 │ c │
  └───┴───┴───┴───┴───┴───┴───┘
   ←────  write=6  ────→  │ignore│

  Output: chars[:6] = "a2b2c3"   write = 6
```

---

## Example 8: Smallest Window Containing All Characters

```go
package main

import (
    "fmt"
    "math"
)

func smallestWindowAllChars(s string) string {
    // Find smallest substring containing all distinct characters
    distinct := make(map[byte]bool)
    for i := 0; i < len(s); i++ {
        distinct[s[i]] = true
    }
    required := len(distinct)

    freq := make(map[byte]int)
    formed := 0
    left := 0
    minLen := math.MaxInt64
    start := 0

    for right := 0; right < len(s); right++ {
        freq[s[right]]++
        if freq[s[right]] == 1 {
            formed++
        }

        for formed == required {
            if right-left+1 < minLen {
                minLen = right - left + 1
                start = left
            }
            freq[s[left]]--
            if freq[s[left]] == 0 {
                formed--
            }
            left++
        }
    }
    if minLen == math.MaxInt64 {
        return ""
    }
    return s[start : start+minLen]
}

func main() {
    fmt.Println(smallestWindowAllChars("aabcbcdbca")) // "dbca"
    fmt.Println(smallestWindowAllChars("ADOBECODEBANC")) // "BANC"
    fmt.Println(smallestWindowAllChars("aaaa"))       // "a"
}
```

**Textual Figure — Smallest Window Containing All Characters:**

```
  Input: s = "aabcbcdbca"
  Distinct chars: {a, b, c, d}   required = 4

  Index:   0   1   2   3   4   5   6   7   8   9
  Char:    a   a   b   c   b   c   d   b   c   a

  Step  left right  formed  Window          Action
  ───── ──── ─────  ──────  ──────────────  ─────────────────────
   1     0    0      1      [a]             +a  formed=1
   2     0    1      1      [aa]            +a
   3     0    2      2      [aab]           +b  formed=2
   4     0    3      3      [aabc]          +c  formed=3
   5     0    4      3      [aabcb]         +b
   6     0    5      3      [aabcbc]        +c
   7     0    6      4      [aabcbcd]       +d  formed=4!
         ┌─ shrink:
         │  save "aabcbcd" len=7, start=0
         │  left=1: remove a → freq[a]=1,  formed=4
         │  save "abcbcd" len=6, start=1
         │  left=2: remove a → freq[a]=0,  formed=3 → stop
   8     2    7      4      [bcbcdb]        +b  formed=4!
         ┌─ shrink:
         │  save "bcbcdb" len=6, start=2
         │  left=3: remove b → freq[b]=2,  formed=4
         │  save "cbcdb" len=5, start=3
         │  left=4: remove c → freq[c]=1,  formed=4
         │  save "bcdb" len=4, start=4
         │  left=5: remove b → freq[b]=1,  formed=4
         │  save "cdb" — wait, len=3 but formed drops
         │  left=5: remove c → freq[c]=1 still ≥ 1 → ok? No:
         │  Actually left=5: remove b(s[4]) → freq[b]=1  formed=4
         │  ...shrink until formed<4
         └─ Best seen so far ≈ "dbca" len=4
   9     6    8      4      […bca]          …
  10     6    9      4      [dbca]          save "dbca" len=4!
         ┌─ shrink: left=7 remove d → formed=3 → stop

  Candidate windows:
  ┌──────────────────────────────────────┐
  │  "aabcbcd"  len=7                      │
  │  "abcbcd"   len=6                      │
  │  "bcdb"     len=4                      │
  │  "dbca"     len=4   ← BEST (first)     │
  └──────────────────────────────────────┘

  Result: "dbca"
```

---

## Example 9: Count Distinct Characters in Every Window of Size K

```go
package main

import "fmt"

func countDistinctInWindows(s string, k int) []int {
    freq := make(map[byte]int)
    var result []int

    for i := 0; i < len(s); i++ {
        freq[s[i]]++

        if i >= k {
            freq[s[i-k]]--
            if freq[s[i-k]] == 0 {
                delete(freq, s[i-k])
            }
        }

        if i >= k-1 {
            result = append(result, len(freq))
        }
    }
    return result
}

func main() {
    fmt.Println(countDistinctInWindows("aabacbaa", 3))
    // Windows: aab=2, aba=2, bac=3, acb=3, cba=3, baa=2
    // [2 2 3 3 3 2]

    fmt.Println(countDistinctInWindows("aaaa", 2))
    // [1 1 1]
}
```

**Textual Figure — Count Distinct Characters in Every Window of Size K:**

```
  Input: s = "aabacbaa", k = 3

  Index:   0   1   2   3   4   5   6   7
  Char:    a   a   b   a   c   b   a   a

  i   Window    freq map          len(freq)   Result
  ─── ───────── ────────────────── ─────────── ─────────────
  0   [a]       {a:1}                          (size<3)
  1   [aa]      {a:2}                          (size<3)
  2   [aab]     {a:2,b:1}          2           [2]
  3   [aba]     {a:2,b:0→del,a:2}  → remove s[0]='a'
                {a:2,b:1}→{a:2,b:0}→del b
                → add s[3]='a': {a:2}   wait…
  ─── Let me retrace carefully:
  i=0: add 'a'           freq={a:1}                   (i<2)
  i=1: add 'a'           freq={a:2}                   (i<2)
  i=2: add 'b'           freq={a:2,b:1}   i>=2 → distinct=2   [2]
  i=3: add 'a', freq={a:3,b:1}; remove s[0]='a' freq={a:2,b:1}
       i>=2 → distinct=2                                      [2,2]
  i=4: add 'c', freq={a:2,b:1,c:1}; remove s[1]='a' freq={a:1,b:1,c:1}
       i>=2 → distinct=3                                      [2,2,3]
  i=5: add 'b', freq={a:1,b:2,c:1}; remove s[2]='b' freq={a:1,b:1,c:1}
       i>=2 → distinct=3                                      [2,2,3,3]
  i=6: add 'a', freq={a:2,b:1,c:1}; remove s[3]='a' freq={a:1,b:1,c:1}
       i>=2 → distinct=3                                      [2,2,3,3,3]
  i=7: add 'a', freq={a:2,b:1,c:1}; remove s[4]='c' freq={a:2,b:1}
       i>=2 → distinct=2                                      [2,2,3,3,3,2]

  Sliding window visualized:

    a   a   b   a   c   b   a   a
   ├─────────┤                          "aab" → 2 distinct
       ├─────────┤                      "aba" → 2 distinct
           ├─────────┤                  "bac" → 3 distinct
               ├─────────┤              "acb" → 3 distinct
                   ├─────────┤          "cba" → 3 distinct
                       ├─────────┤      "baa" → 2 distinct

  Result: [2, 2, 3, 3, 3, 2]
```

---

## Example 10: Longest Beautiful Substring (Vowels in Order)

```go
package main

import "fmt"

func longestBeautifulSubstring(word string) int {
    maxLen := 0
    curLen := 1
    distinct := 1

    for i := 1; i < len(word); i++ {
        if word[i] >= word[i-1] {
            curLen++
            if word[i] > word[i-1] {
                distinct++
            }
        } else {
            curLen = 1
            distinct = 1
        }

        if distinct == 5 && curLen > maxLen {
            maxLen = curLen
        }
    }
    return maxLen
}

func main() {
    fmt.Println(longestBeautifulSubstring("aeiaaioaaaaeiiiiouuuooaauuaeiu"))
    // 13 → "aaaaeiiiiouuu"
    fmt.Println(longestBeautifulSubstring("aeeeiiiioooauuuaeiou"))
    // 5 → "aeiou"
    fmt.Println(longestBeautifulSubstring("a"))
    // 0
}
```

**Textual Figure — Longest Beautiful Substring (Vowels in Order):**

```
  Vowel order required: a < e < i < o < u  (all 5 must appear)

  Input: word = "aeiaaioaaaaeiiiiouuuooaauuaeiu"

  Key rule:
  ┌──────────────────────────────────────────────────┐
  │  word[i] >= word[i-1]:  extend window           │
  │  word[i] >  word[i-1]:  new distinct vowel      │
  │  word[i] <  word[i-1]:  reset (break in order)  │
  │  "Beautiful" when distinct == 5                  │
  └──────────────────────────────────────────────────┘

  Scan through the word character by character:

  Pos  char  prev   relation  curLen  distinct  beautiful?  maxLen
  ──── ───── ─────  ────────  ──────  ────────  ─────────── ──────
   0    a     -                1       1
   1    e     a      e>a       2       2
   2    i     e      i>e       3       3
   3    a     i      a<i RESET 1       1
   4    a     a      a=a       2       1
   5    i     a      i>a       3       2         (skip e!)
   6    o     i      o>i       4       3
   7    a     o      a<o RESET 1       1
   8    a     a      a=a       2       1
   9    a     a      a=a       3       1
  10    a     a      a=a       4       1
  11    e     a      e>a       5       2
  12    i     e      i>e       6       3
  13    i     i      i=i       7       3
  14    i     i      i=i       8       3
  15    i     i      i=i       9       3
  16    o     i      o>i      10       4
  17    u     o      u>o      11       5         ✓ maxLen=11
  18    u     u      u=u      12       5         ✓ maxLen=12
  19    u     u      u=u      13       5         ✓ maxLen=13
  20    o     u      o<u RESET 1       1
  21    o     o      o=o       2       1
  22    a     o      a<o RESET 1       1
  23    a     a      a=a       2       1
  24    u     a      u>a       3       2
  25    u     u      u=u       4       2
  26    a     u      a<u RESET 1       1
  27    e     a      e>a       2       2
  28    i     e      i>e       3       3
  29    u     i      u>i       4       4         (only 4, no 'o')

  Best beautiful substring found:
  Positions 7–19: "aaaaeiiiiouuu"  (len=13, distinct=5)

       a  a  a  a  e  i  i  i  i  o  u  u  u
      │────────────────────────────────────┤
       a ≤ a ≤ a ≤ a ≤ e ≤ i ≤ i ≤ i ≤ i ≤ o ≤ u ≤ u ≤ u
       distinct vowels: {a, e, i, o, u} = 5  ✓

  Result: 13
```

---

## Key Takeaways

1. **Fixed-size window on strings**: use `[26]int` array for O(1) comparison
2. **Variable window**: expand right, shrink left when constraint breaks
3. **Frequency arrays** (`[26]int`) are faster than hash maps for lowercase letters
4. **Array equality** (`winCount == pCount`) works in Go for fixed-size arrays
5. **Most string sliding window problems** reduce to tracking character frequencies

> **Next up:** String Hashing →
