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

---

## Key Takeaways

1. **Fixed-size window on strings**: use `[26]int` array for O(1) comparison
2. **Variable window**: expand right, shrink left when constraint breaks
3. **Frequency arrays** (`[26]int`) are faster than hash maps for lowercase letters
4. **Array equality** (`winCount == pCount`) works in Go for fixed-size arrays
5. **Most string sliding window problems** reduce to tracking character frequencies

> **Next up:** String Hashing →
