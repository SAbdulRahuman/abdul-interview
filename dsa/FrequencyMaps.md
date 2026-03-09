# Phase 4: Hash Tables — Frequency Maps

## What is a Frequency Map?

A **frequency map** (also called a **counter** or **histogram**) uses a hash map to count occurrences of each element. In Go: `map[T]int`.

**Core pattern:**
```go
freq := make(map[T]int)
for _, item := range collection {
    freq[item]++
}
```

Used extensively in problems involving counting, anagrams, majority elements, and sliding windows.

---

## Example 1: Basic Frequency Counter

```go
package main

import "fmt"

func charFrequency(s string) map[rune]int {
    freq := make(map[rune]int)
    for _, c := range s {
        freq[c]++
    }
    return freq
}

func intFrequency(arr []int) map[int]int {
    freq := make(map[int]int)
    for _, v := range arr {
        freq[v]++
    }
    return freq
}

func main() {
    s := "abracadabra"
    fmt.Println("Character frequencies:")
    for ch, count := range charFrequency(s) {
        fmt.Printf("  '%c': %d\n", ch, count)
    }

    nums := []int{1, 2, 3, 2, 1, 3, 3, 4}
    fmt.Println("\nNumber frequencies:")
    for v, count := range intFrequency(nums) {
        fmt.Printf("  %d: %d\n", v, count)
    }
}
```

---

## Example 2: Most Frequent Element (Mode)

```go
package main

import "fmt"

func mostFrequent(arr []int) (int, int) {
    freq := make(map[int]int)
    for _, v := range arr {
        freq[v]++
    }

    maxVal, maxCount := 0, 0
    for v, c := range freq {
        if c > maxCount {
            maxVal, maxCount = v, c
        }
    }
    return maxVal, maxCount
}

func topK(arr []int, k int) []int {
    freq := make(map[int]int)
    for _, v := range arr {
        freq[v]++
    }

    // Bucket sort by frequency
    maxFreq := 0
    for _, c := range freq {
        if c > maxFreq {
            maxFreq = c
        }
    }

    buckets := make([][]int, maxFreq+1)
    for v, c := range freq {
        buckets[c] = append(buckets[c], v)
    }

    result := make([]int, 0, k)
    for i := maxFreq; i >= 0 && len(result) < k; i-- {
        for _, v := range buckets[i] {
            if len(result) >= k {
                break
            }
            result = append(result, v)
        }
    }
    return result
}

func main() {
    arr := []int{1, 1, 1, 2, 2, 3, 3, 3, 3, 4}
    val, count := mostFrequent(arr)
    fmt.Printf("Most frequent: %d (count=%d)\n", val, count)

    top := topK([]int{1, 1, 1, 2, 2, 3}, 2)
    fmt.Println("Top 2 frequent:", top)
}
```

---

## Example 3: Check Anagram

```go
package main

import "fmt"

func isAnagram(s1, s2 string) bool {
    if len(s1) != len(s2) {
        return false
    }
    freq := make(map[rune]int)
    for _, c := range s1 {
        freq[c]++
    }
    for _, c := range s2 {
        freq[c]--
        if freq[c] < 0 {
            return false
        }
    }
    return true
}

func groupAnagrams(words []string) [][]string {
    groups := make(map[[26]int][]string)
    for _, w := range words {
        var key [26]int
        for _, c := range w {
            key[c-'a']++
        }
        groups[key] = append(groups[key], w)
    }

    result := make([][]string, 0, len(groups))
    for _, g := range groups {
        result = append(result, g)
    }
    return result
}

func main() {
    fmt.Println("listen/silent:", isAnagram("listen", "silent"))
    fmt.Println("hello/world:", isAnagram("hello", "world"))

    words := []string{"eat", "tea", "tan", "ate", "nat", "bat"}
    fmt.Println("\nGrouped anagrams:")
    for _, group := range groupAnagrams(words) {
        fmt.Println(" ", group)
    }
}
```

---

## Example 4: Majority Element (Boyer-Moore + Verify)

```go
package main

import "fmt"

// Boyer-Moore Voting Algorithm: O(1) space
func majorityBoyerMoore(nums []int) (int, bool) {
    candidate, count := 0, 0
    for _, n := range nums {
        if count == 0 {
            candidate = n
        }
        if n == candidate {
            count++
        } else {
            count--
        }
    }

    // Verify
    count = 0
    for _, n := range nums {
        if n == candidate {
            count++
        }
    }
    return candidate, count > len(nums)/2
}

// Frequency map approach
func majorityFreqMap(nums []int) (int, bool) {
    freq := make(map[int]int)
    half := len(nums) / 2
    for _, n := range nums {
        freq[n]++
        if freq[n] > half {
            return n, true
        }
    }
    return 0, false
}

func main() {
    tests := [][]int{
        {3, 2, 3},
        {2, 2, 1, 1, 1, 2, 2},
        {1, 2, 3, 4, 5},
    }

    for _, nums := range tests {
        v1, ok1 := majorityBoyerMoore(nums)
        v2, ok2 := majorityFreqMap(nums)
        fmt.Printf("nums=%v → BM=(%d,%v), freq=(%d,%v)\n", nums, v1, ok1, v2, ok2)
    }
}
```

---

## Example 5: Frequency-Based Sorting

```go
package main

import (
    "fmt"
    "sort"
)

func sortByFrequency(arr []int) []int {
    freq := make(map[int]int)
    for _, v := range arr {
        freq[v]++
    }

    result := make([]int, len(arr))
    copy(result, arr)

    sort.Slice(result, func(i, j int) bool {
        fi, fj := freq[result[i]], freq[result[j]]
        if fi != fj {
            return fi > fj // higher frequency first
        }
        return result[i] < result[j] // tie-break: smaller value first
    })

    return result
}

func sortStringByFrequency(s string) string {
    freq := make(map[byte]int)
    for i := 0; i < len(s); i++ {
        freq[s[i]]++
    }

    type charFreq struct {
        ch    byte
        count int
    }
    pairs := make([]charFreq, 0, len(freq))
    for ch, c := range freq {
        pairs = append(pairs, charFreq{ch, c})
    }
    sort.Slice(pairs, func(i, j int) bool {
        return pairs[i].count > pairs[j].count
    })

    result := make([]byte, 0, len(s))
    for _, p := range pairs {
        for i := 0; i < p.count; i++ {
            result = append(result, p.ch)
        }
    }
    return string(result)
}

func main() {
    nums := []int{1, 1, 2, 2, 2, 3}
    fmt.Println("Sort by frequency:", sortByFrequency(nums))
    // [2 2 2 1 1 3]

    s := "tree"
    fmt.Println("String freq sort:", sortStringByFrequency(s))
    // "eert" or "eetr"
}
```

---

## Example 6: Sliding Window with Frequency Map

```go
package main

import "fmt"

// Find all anagrams of pattern in text
// LeetCode 438
func findAnagrams(s, p string) []int {
    if len(s) < len(p) {
        return nil
    }

    pFreq := make(map[byte]int)
    wFreq := make(map[byte]int)
    for i := 0; i < len(p); i++ {
        pFreq[p[i]]++
    }

    matches := func() bool {
        for k, v := range pFreq {
            if wFreq[k] != v {
                return false
            }
        }
        return true
    }

    var result []int
    for i := 0; i < len(s); i++ {
        wFreq[s[i]]++

        if i >= len(p) {
            wFreq[s[i-len(p)]]--
            if wFreq[s[i-len(p)]] == 0 {
                delete(wFreq, s[i-len(p)])
            }
        }

        if i >= len(p)-1 && matches() {
            result = append(result, i-len(p)+1)
        }
    }
    return result
}

func main() {
    s := "cbaebabacd"
    p := "abc"
    fmt.Printf("Anagram positions of %q in %q: %v\n", p, s, findAnagrams(s, p))
    // [0, 6]

    s2 := "abab"
    p2 := "ab"
    fmt.Printf("Anagram positions of %q in %q: %v\n", p2, s2, findAnagrams(s2, p2))
    // [0, 1, 2]
}
```

---

## Example 7: Minimum Window Substring

```go
package main

import "fmt"

// LeetCode 76: Find smallest window in s containing all chars of t
func minWindow(s, t string) string {
    if len(t) > len(s) {
        return ""
    }

    need := make(map[byte]int)
    for i := 0; i < len(t); i++ {
        need[t[i]]++
    }

    window := make(map[byte]int)
    have, total := 0, len(need)
    bestLen := len(s) + 1
    bestStart := 0
    left := 0

    for right := 0; right < len(s); right++ {
        c := s[right]
        window[c]++

        if need[c] > 0 && window[c] == need[c] {
            have++
        }

        for have == total {
            // Update best
            winLen := right - left + 1
            if winLen < bestLen {
                bestLen = winLen
                bestStart = left
            }

            // Shrink from left
            lc := s[left]
            window[lc]--
            if need[lc] > 0 && window[lc] < need[lc] {
                have--
            }
            left++
        }
    }

    if bestLen > len(s) {
        return ""
    }
    return s[bestStart : bestStart+bestLen]
}

func main() {
    tests := [][2]string{
        {"ADOBECODEBANC", "ABC"},
        {"a", "a"},
        {"a", "aa"},
    }

    for _, t := range tests {
        fmt.Printf("minWindow(%q, %q) = %q\n", t[0], t[1], minWindow(t[0], t[1]))
    }
}
```

---

## Example 8: First Non-Repeating Character

```go
package main

import "fmt"

func firstUniqChar(s string) int {
    freq := make(map[byte]int)
    for i := 0; i < len(s); i++ {
        freq[s[i]]++
    }
    for i := 0; i < len(s); i++ {
        if freq[s[i]] == 1 {
            return i
        }
    }
    return -1
}

// Streaming version: first unique in a stream
type FirstUnique struct {
    freq  map[int]int
    order []int
}

func NewFirstUnique(nums []int) *FirstUnique {
    fu := &FirstUnique{
        freq: make(map[int]int),
    }
    for _, n := range nums {
        fu.Add(n)
    }
    return fu
}

func (fu *FirstUnique) ShowFirstUnique() int {
    for _, n := range fu.order {
        if fu.freq[n] == 1 {
            return n
        }
    }
    return -1
}

func (fu *FirstUnique) Add(val int) {
    fu.freq[val]++
    if fu.freq[val] == 1 {
        fu.order = append(fu.order, val)
    }
}

func main() {
    s := "leetcode"
    idx := firstUniqChar(s)
    fmt.Printf("First unique char in %q: index %d ('%c')\n", s, idx, s[idx])

    s2 := "aabb"
    idx2 := firstUniqChar(s2)
    fmt.Printf("First unique char in %q: index %d\n", s2, idx2)

    fu := NewFirstUnique([]int{2, 3, 5})
    fmt.Println("\nStream first unique:", fu.ShowFirstUnique()) // 2
    fu.Add(5)
    fmt.Println("After add 5:", fu.ShowFirstUnique()) // 2
    fu.Add(2)
    fmt.Println("After add 2:", fu.ShowFirstUnique()) // 3
    fu.Add(3)
    fmt.Println("After add 3:", fu.ShowFirstUnique()) // -1
}
```

---

## Example 9: Subarray Sum Equals K (Prefix Sum + Freq Map)

```go
package main

import "fmt"

// LeetCode 560: Count subarrays summing to k
func subarraySum(nums []int, k int) int {
    prefixFreq := make(map[int]int)
    prefixFreq[0] = 1 // empty prefix

    sum := 0
    count := 0
    for _, n := range nums {
        sum += n
        // If (sum - k) was seen before, those prefixes form valid subarrays
        if freq, ok := prefixFreq[sum-k]; ok {
            count += freq
        }
        prefixFreq[sum]++
    }
    return count
}

func main() {
    tests := []struct {
        nums []int
        k    int
    }{
        {[]int{1, 1, 1}, 2},          // → 2
        {[]int{1, 2, 3}, 3},          // → 2 ([1,2] and [3])
        {[]int{1, -1, 0}, 0},         // → 3
        {[]int{3, 4, 7, 2, -3, 1, 4, 2}, 7}, // → 4
    }

    for _, t := range tests {
        fmt.Printf("subarraySum(%v, %d) = %d\n", t.nums, t.k, subarraySum(t.nums, t.k))
    }
}
```

---

## Example 10: Ransom Note (Multi-set Comparison)

```go
package main

import "fmt"

// Can we construct ransomNote from characters in magazine?
func canConstruct(ransomNote, magazine string) bool {
    freq := make(map[byte]int)
    for i := 0; i < len(magazine); i++ {
        freq[magazine[i]]++
    }
    for i := 0; i < len(ransomNote); i++ {
        freq[ransomNote[i]]--
        if freq[ransomNote[i]] < 0 {
            return false
        }
    }
    return true
}

// Check if two strings are permutations
func isPermutation(a, b string) bool {
    if len(a) != len(b) {
        return false
    }
    freq := make(map[byte]int)
    for i := 0; i < len(a); i++ {
        freq[a[i]]++
        freq[b[i]]--
    }
    for _, v := range freq {
        if v != 0 {
            return false
        }
    }
    return true
}

// Multiset intersection: min of each element
func multisetIntersection(a, b []int) map[int]int {
    freqA := make(map[int]int)
    freqB := make(map[int]int)
    for _, v := range a {
        freqA[v]++
    }
    for _, v := range b {
        freqB[v]++
    }

    result := make(map[int]int)
    for k, countA := range freqA {
        if countB, ok := freqB[k]; ok {
            minCount := countA
            if countB < minCount {
                minCount = countB
            }
            result[k] = minCount
        }
    }
    return result
}

func main() {
    fmt.Println("Ransom note 'a' from 'b':", canConstruct("a", "b"))
    fmt.Println("Ransom note 'aa' from 'aab':", canConstruct("aa", "aab"))

    fmt.Println("\nPermutation 'abc'/'bca':", isPermutation("abc", "bca"))
    fmt.Println("Permutation 'abc'/'abd':", isPermutation("abc", "abd"))

    a := []int{1, 2, 2, 3, 3, 3}
    b := []int{2, 2, 2, 3, 3, 4}
    fmt.Println("\nMultiset intersection:", multisetIntersection(a, b))
    // {2:2, 3:2}
}
```

---

## Example 11: Word Frequency Counter

```go
package main

import (
    "fmt"
    "sort"
    "strings"
)

func wordFrequency(text string) map[string]int {
    text = strings.ToLower(text)
    words := strings.Fields(text)
    freq := make(map[string]int)
    for _, w := range words {
        // Strip punctuation
        w = strings.Trim(w, ".,!?;:\"'()-")
        if w != "" {
            freq[w]++
        }
    }
    return freq
}

func topNWords(freq map[string]int, n int) []struct {
    Word  string
    Count int
} {
    type wc struct {
        Word  string
        Count int
    }
    pairs := make([]wc, 0, len(freq))
    for w, c := range freq {
        pairs = append(pairs, wc{w, c})
    }
    sort.Slice(pairs, func(i, j int) bool {
        return pairs[i].Count > pairs[j].Count
    })
    if n > len(pairs) {
        n = len(pairs)
    }
    return pairs[:n]
}

func main() {
    text := `Go is an open-source programming language. Go makes it easy to build 
    simple, reliable, and efficient software. Go is designed at Google. 
    Go supports concurrency. Go has garbage collection. Go is fast.`

    freq := wordFrequency(text)

    fmt.Println("Top 5 words:")
    for _, wc := range topNWords(freq, 5) {
        fmt.Printf("  %-15s %d\n", wc.Word, wc.Count)
    }

    fmt.Printf("\nTotal unique words: %d\n", len(freq))
}
```

---

## Frequency Map Patterns Summary

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| `map[T]int` counting | Count occurrences | O(n) |
| Two-map comparison | Anagram check | O(n) |
| Prefix sum + freq map | Subarray sum = K | O(n) |
| Sliding window + freq | Find all anagrams | O(n) |
| Bucket sort by freq | Top-K frequent | O(n) |
| Decrement counter | Ransom note / multiset | O(n) |

## Key Takeaways

1. **`map[T]int`** is the Go idiom for frequency counting
2. **Zero-value trick**: `freq[key]++` works without initialization (Go zero-value = 0)
3. **Anagram check**: build freq for one string, decrement for other
4. **Prefix sum + freq map**: solve subarray sum problems in O(n)
5. **Sliding window + freq map**: find all anagram positions in O(n)
6. **Top-K frequent**: bucket sort gives O(n) solution
7. **Two-pass pattern**: first count, then query — cleaner than single pass

> **Phase 4 Complete!** Next up: Phase 5 — Linked Lists →
