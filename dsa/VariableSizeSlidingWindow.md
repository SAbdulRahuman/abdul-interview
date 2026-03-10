# Phase 25: Advanced Sliding Window — Variable Size Sliding Window

## Overview

A **variable-size sliding window** expands and contracts dynamically to find an optimal subarray/substring. The window size is not predetermined — it grows until a condition breaks, then shrinks from the left.

### Template
```
left := 0
for right := 0; right < n; right++ {
    add arr[right] to window
    while window is invalid:
        remove arr[left] from window
        left++
    update answer with current window [left..right]
}
```

**Time Complexity**: O(n) — each element is added/removed at most once (amortized).

---

## Example 1: Longest Substring Without Repeating Characters (LC 3)

```go
package main

import "fmt"

func lengthOfLongestSubstring(s string) int {
	seen := make(map[byte]int) // char → last index
	maxLen := 0
	left := 0

	for right := 0; right < len(s); right++ {
		if idx, ok := seen[s[right]]; ok && idx >= left {
			left = idx + 1 // shrink window past the duplicate
		}
		seen[s[right]] = right
		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func main() {
	tests := []string{"abcabcbb", "bbbbb", "pwwkew", ""}
	for _, s := range tests {
		fmt.Printf("%q → %d\n", s, lengthOfLongestSubstring(s))
	}
	// "abcabcbb" → 3, "bbbbb" → 1, "pwwkew" → 3
}
```

**Textual Figure — Longest Substring Without Repeating Characters:**

```
  s = "abcabcbb"

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ b │ c │ a │ b │ c │ b │ b │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7
    L               R

  r=0: [a]          L=0  len=1  max=1
  r=1: [ab]         L=0  len=2  max=2
  r=2: [abc]        L=0  len=3  max=3
  r=3: [a]bcA       'a' seen at 0 → L=1
       [bca]        L=1  len=3  max=3
  r=4: [b]ca[b]     'b' seen at 1 → L=2
       [cab]        L=2  len=3  max=3
  r=5: [c]ab[c]     'c' seen at 2 → L=3
       [abc]        L=3  len=3  max=3
  r=6: a[b]c[b]     'b' seen at 4 → L=5
       [cb]         L=5  len=2  max=3
  r=7: c[b][b]      'b' seen at 6 → L=7
       [b]          L=7  len=1  max=3

  Result: 3  ("abc")
```

---

## Example 2: Minimum Size Subarray Sum (LC 209)

```go
package main

import "fmt"

func minSubArrayLen(target int, nums []int) int {
	left, sum := 0, 0
	minLen := len(nums) + 1

	for right := 0; right < len(nums); right++ {
		sum += nums[right]

		// Shrink while condition is met → find minimum
		for sum >= target {
			if right-left+1 < minLen { minLen = right - left + 1 }
			sum -= nums[left]
			left++
		}
	}

	if minLen > len(nums) { return 0 }
	return minLen
}

func main() {
	tests := []struct {
		target int
		nums   []int
	}{
		{7, []int{2, 3, 1, 2, 4, 3}},
		{4, []int{1, 4, 4}},
		{11, []int{1, 1, 1, 1, 1}},
	}

	for _, t := range tests {
		fmt.Printf("target=%d nums=%v → %d\n", t.target, t.nums, minSubArrayLen(t.target, t.nums))
	}
}
```

**Textual Figure — Minimum Size Subarray Sum:**

```
  nums = [2, 3, 1, 2, 4, 3]    target = 7

  ┌───┬───┬───┬───┬───┬───┐
  │ 2 │ 3 │ 1 │ 2 │ 4 │ 3 │
  └───┴───┴───┴───┴───┴───┘
    L→                     R→

  Expand right, shrink left while sum ≥ target:

  r=0: sum=2                      < 7
  r=1: sum=5                      < 7
  r=2: sum=6                      < 7
  r=3: sum=8  ≥ 7  → min=4        shrink
       L++ sum=6                  < 7
  r=4: sum=10 ≥ 7  → min=3       shrink
       L++ sum=7 ≥ 7 → min=3     shrink
       L++ sum=6                  < 7
  r=5: sum=9  ≥ 7  → min=3       shrink
       L++ sum=7 ≥ 7 → min=2 ←   shrink!
       L++ sum=3                  < 7

  Result: min length = 2  (subarray [4,3])
```

---

## Example 3: Longest Substring with At Most K Distinct Characters (LC 340)

```go
package main

import "fmt"

func longestKDistinct(s string, k int) int {
	freq := make(map[byte]int)
	left := 0
	maxLen := 0

	for right := 0; right < len(s); right++ {
		freq[s[right]]++

		// Shrink when too many distinct characters
		for len(freq) > k {
			freq[s[left]]--
			if freq[s[left]] == 0 { delete(freq, s[left]) }
			left++
		}

		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func main() {
	tests := []struct {
		s string
		k int
	}{
		{"eceba", 2},
		{"aa", 1},
		{"aabacbebebe", 3},
	}

	for _, t := range tests {
		fmt.Printf("s=%q k=%d → %d\n", t.s, t.k, longestKDistinct(t.s, t.k))
	}
}
```

**Textual Figure — Longest Substring with At Most K Distinct:**

```
  s = "eceba"    k = 2

  ┌───┬───┬───┬───┬───┐
  │ e │ c │ e │ b │ a │
  └───┴───┴───┴───┴───┘
    0   1   2   3   4
    L→              R→

  Expand right, shrink when distinct > k:

  r=0: [e]       freq={e:1}       dist=1 ≤ 2  len=1  max=1
  r=1: [ec]      freq={e:1,c:1}   dist=2 ≤ 2  len=2  max=2
  r=2: [ece]     freq={e:2,c:1}   dist=2 ≤ 2  len=3  max=3
  r=3: [eceb]    freq={e:2,c:1,b:1} dist=3 > 2  → shrink!
       L=1 [ceb] freq={e:1,c:1,b:1} dist=3 > 2  → shrink!
       L=2 [eb]  freq={e:1,b:1}     dist=2 ≤ 2  len=2  max=3
  r=4: [eba]     freq={e:1,b:1,a:1} dist=3 > 2  → shrink!
       L=3 [ba]  freq={b:1,a:1}     dist=2 ≤ 2  len=2  max=3

  Result: 3  ("ece")
```

---

## Example 4: Minimum Window Substring (LC 76)

```go
package main

import (
	"fmt"
	"math"
)

func minWindow(s, t string) string {
	if len(s) < len(t) { return "" }

	need := make(map[byte]int)
	for i := 0; i < len(t); i++ { need[t[i]]++ }

	have := make(map[byte]int)
	formed := 0
	required := len(need)

	left := 0
	minLen := math.MaxInt64
	start := 0

	for right := 0; right < len(s); right++ {
		c := s[right]
		have[c]++

		if need[c] > 0 && have[c] == need[c] { formed++ }

		// Shrink
		for formed == required {
			windowLen := right - left + 1
			if windowLen < minLen {
				minLen = windowLen
				start = left
			}

			leftC := s[left]
			have[leftC]--
			if need[leftC] > 0 && have[leftC] < need[leftC] { formed-- }
			left++
		}
	}

	if minLen == math.MaxInt64 { return "" }
	return s[start : start+minLen]
}

func main() {
	tests := []struct{ s, t string }{
		{"ADOBECODEBANC", "ABC"},
		{"a", "a"},
		{"a", "aa"},
	}

	for _, test := range tests {
		fmt.Printf("s=%q t=%q → %q\n", test.s, test.t, minWindow(test.s, test.t))
	}
}
```

**Textual Figure — Minimum Window Substring:**

```
  s = "ADOBECODEBANC"    t = "ABC"
  need: {A:1, B:1, C:1}   required = 3 chars

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ A │ D │ O │ B │ E │ C │ O │ D │ E │ B │ A │ N │ C │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7   8   9  10  11  12
    L→                                             R→

  Expand R until all chars found, then shrink L:

  r=5:  [ADOBEC] formed=3  → valid! min=6
        shrink: L=1 [DOBEC] formed=2 → invalid
  r=9:  [DOBECODEB] ... expand more
  r=10: [DOBECOD EBA] formed=3  → valid! min=6
        shrink: L=5 [CODEBA] min=6
        shrink: L=6 [ODEBA] ... still shrinking
  r=12: [...BANC]  found valid window
        shrink to: [BANC] min=4  ← best!

  Result: "BANC"  (length 4)

  ┌─────────────────────────────────────┐
  │ Expand R → until all chars covered  │
  │ Shrink L → while still valid       │
  │ Track minimum valid window          │
  └─────────────────────────────────────┘
```

---

## Example 5: Longest Repeating Character Replacement (LC 424)

```go
package main

import "fmt"

func characterReplacement(s string, k int) int {
	var freq [26]int
	left, maxFreq, maxLen := 0, 0, 0

	for right := 0; right < len(s); right++ {
		freq[s[right]-'A']++
		if freq[s[right]-'A'] > maxFreq {
			maxFreq = freq[s[right]-'A']
		}

		// Window size - max freq char count = chars to replace
		// If replacements needed > k, shrink
		windowLen := right - left + 1
		if windowLen-maxFreq > k {
			freq[s[left]-'A']--
			left++
		}

		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func main() {
	tests := []struct {
		s string
		k int
	}{
		{"ABAB", 2},
		{"AABABBA", 1},
	}

	for _, t := range tests {
		fmt.Printf("s=%q k=%d → %d\n", t.s, t.k, characterReplacement(t.s, t.k))
	}
}
```

**Textual Figure — Longest Repeating Character Replacement:**

```
  s = "AABABBA"    k = 1

  ┌───┬───┬───┬───┬───┬───┬───┐
  │ A │ A │ B │ A │ B │ B │ A │
  └───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6
    L→                      R→

  windowLen - maxFreq = chars to replace (≤ k?)

  r=0: [A]       maxF=1 replace=0 ≤ 1  len=1
  r=1: [AA]      maxF=2 replace=0 ≤ 1  len=2
  r=2: [AAB]     maxF=2 replace=1 ≤ 1  len=3
  r=3: [AABA]    maxF=3 replace=1 ≤ 1  len=4  ← max
  r=4: [AABAB]   maxF=3 replace=2 > 1  → shrink!
       L++ [ABAB]  len=4
  r=5: [ABABB]   maxF=3 replace=2 > 1  → shrink!
       L++ [BABB]  len=4
  r=6: [BABBA]   maxF=3 replace=2 > 1  → shrink!
       L++ [ABBA]  len=4

  Result: 4  ("AABA" → replace B→A = "AAAA")
```

---

## Example 6: Subarrays with K Different Integers (LC 992)

```go
package main

import "fmt"

// Exactly K = atMost(K) - atMost(K-1)

func subarraysWithKDistinct(nums []int, k int) int {
	return atMost(nums, k) - atMost(nums, k-1)
}

func atMost(nums []int, k int) int {
	freq := make(map[int]int)
	left, count := 0, 0

	for right := 0; right < len(nums); right++ {
		freq[nums[right]]++

		for len(freq) > k {
			freq[nums[left]]--
			if freq[nums[left]] == 0 { delete(freq, nums[left]) }
			left++
		}

		count += right - left + 1 // All subarrays ending at right
	}
	return count
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{1, 2, 1, 2, 3}, 2},
		{[]int{1, 2, 1, 3, 4}, 3},
	}

	for _, t := range tests {
		fmt.Printf("nums=%v k=%d → %d\n", t.nums, t.k, subarraysWithKDistinct(t.nums, t.k))
	}
}
```

**Textual Figure — Subarrays with K Different Integers:**

```
  nums = [1, 2, 1, 2, 3]    k = 2

  ┌───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 1 │ 2 │ 3 │
  └───┴───┴───┴───┴───┘
    0   1   2   3   4

  exactlyK(2) = atMost(2) - atMost(1)

  atMost(2):                   atMost(1):
  r=0: [1]       cnt=1         [1]       cnt=1
  r=1: [1,2]     cnt=2+1=3     L++ [2]   cnt=1
  r=2: [1,2,1]   cnt=3+3=6     [2,1]→L++ [1] cnt=1+1=2
  r=3: [1,2,1,2] cnt=4+6=10    [1,2]→L++ [2] cnt=1+2=3
  r=4: L++..     cnt=2+10=12   [2,3]→L++ [3] cnt=1+3=4
       Total: 12               Total: 5

  exactlyK(2) = 12 - 5 = 7

  The 7 subarrays: [1,2] [2,1] [1,2,1] [2,1,2]
                   [1,2,1,2] [2,3] [1,2,3]
```

---

## Example 7: Max Consecutive Ones III (LC 1004)

```go
package main

import "fmt"

// Longest subarray of 1s after flipping at most k zeros

func longestOnes(nums []int, k int) int {
	left, zeros, maxLen := 0, 0, 0

	for right := 0; right < len(nums); right++ {
		if nums[right] == 0 { zeros++ }

		// Shrink when too many zeros
		for zeros > k {
			if nums[left] == 0 { zeros-- }
			left++
		}

		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 0}, 2},
		{[]int{0, 0, 1, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 1, 1, 1, 1}, 3},
	}

	for _, t := range tests {
		fmt.Printf("nums=%v k=%d → %d\n", t.nums, t.k, longestOnes(t.nums, t.k))
	}
}
```

**Textual Figure — Max Consecutive Ones III:**

```
  nums = [1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 0]    k = 2

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 1 │ 1 │ 0 │ 0 │ 0 │ 1 │ 1 │ 1 │ 1 │ 0 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7   8   9  10
    L→                                     R→

  Expand R, count zeros, shrink when zeros > k:

  r=0..2: [1,1,1]       zeros=0           len=3
  r=3:    [1,1,1,0]     zeros=1 ≤ 2       len=4
  r=4:    [1,1,1,0,0]   zeros=2 ≤ 2       len=5
  r=5:    [1,1,1,0,0,0] zeros=3 > 2 → shrink!
          L=1 [1,1,0,0,0] zeros=3 > 2 → shrink!
          L=2 [1,0,0,0]   zeros=3 > 2 → shrink!
          L=3 [0,0,0]     zeros=3 > 2 → shrink!
          L=4 [0,0]       zeros=2 ≤ 2     len=2
  r=6..9: [0,0,1,1,1,1] zeros=2 ≤ 2       len=6 ← max
  r=10:   [...,1,1,1,1,0] zeros=3 > 2 → shrink!
          L=5  [...] zeros=2              len=6

  Result: 6  (flip two 0s at indices 3,4 → [1,1,1,1,1,1])
```

---

## Example 8: Fruit Into Baskets (LC 904)

```go
package main

import "fmt"

// Max subarray with at most 2 distinct elements

func totalFruit(fruits []int) int {
	freq := make(map[int]int)
	left, maxLen := 0, 0

	for right := 0; right < len(fruits); right++ {
		freq[fruits[right]]++

		for len(freq) > 2 {
			freq[fruits[left]]--
			if freq[fruits[left]] == 0 { delete(freq, fruits[left]) }
			left++
		}

		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func main() {
	tests := [][]int{
		{1, 2, 1},
		{0, 1, 2, 2},
		{1, 2, 3, 2, 2},
		{3, 3, 3, 1, 2, 1, 1, 2, 3, 3, 4},
	}

	for _, t := range tests {
		fmt.Printf("fruits=%v → %d\n", t, totalFruit(t))
	}
}
```

**Textual Figure — Fruit Into Baskets (At Most 2 Distinct):**

```
  fruits = [1, 2, 3, 2, 2]

  ┌───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 2 │ 2 │
  └───┴───┴───┴───┴───┘
    0   1   2   3   4
    L→              R→

  2 baskets = at most 2 distinct fruit types.

  r=0: [1]       freq={1:1}       dist=1 ≤ 2  len=1
  r=1: [1,2]     freq={1:1,2:1}   dist=2 ≤ 2  len=2
  r=2: [1,2,3]   freq={1:1,2:1,3:1} dist=3 > 2  → shrink!
       L=1 [2,3] freq={2:1,3:1}    dist=2 ≤ 2  len=2
  r=3: [2,3,2]   freq={2:2,3:1}   dist=2 ≤ 2  len=3
  r=4: [2,3,2,2] freq={2:3,3:1}   dist=2 ≤ 2  len=4 ← max

  Result: 4  ([2,3,2,2] → types {2,3})
```

---

## Example 9: Longest Substring with At Most Two Distinct Characters (LC 159)

```go
package main

import "fmt"

func lengthOfLongestSubstringTwoDistinct(s string) int {
	freq := make(map[byte]int)
	left, maxLen := 0, 0

	for right := 0; right < len(s); right++ {
		freq[s[right]]++

		for len(freq) > 2 {
			freq[s[left]]--
			if freq[s[left]] == 0 { delete(freq, s[left]) }
			left++
		}

		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func main() {
	tests := []string{"eceba", "ccaabbb", "abcabcabc"}
	for _, s := range tests {
		fmt.Printf("s=%q → %d\n", s, lengthOfLongestSubstringTwoDistinct(s))
	}
}
```

**Textual Figure — Longest Substring with At Most 2 Distinct:**

```
  s = "ccaabbb"

  ┌───┬───┬───┬───┬───┬───┬───┐
  │ c │ c │ a │ a │ b │ b │ b │
  └───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6
    L→                      R→

  r=0: [c]        freq={c:1}       dist=1  len=1
  r=1: [cc]       freq={c:2}       dist=1  len=2
  r=2: [cca]      freq={c:2,a:1}   dist=2  len=3
  r=3: [ccaa]     freq={c:2,a:2}   dist=2  len=4
  r=4: [ccaab]    freq={c:2,a:2,b:1} dist=3 > 2 → shrink!
       L=1 [caab] freq={c:1,a:2,b:1} dist=3 > 2 → shrink!
       L=2 [aab]  freq={a:2,b:1}     dist=2 ≤ 2  len=3
  r=5: [aabb]     freq={a:2,b:2}   dist=2  len=4
  r=6: [aabbb]    freq={a:2,b:3}   dist=2  len=5 ← max

  Result: 5  ("aabbb")
```

---

## Example 10: Variable Window Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Variable Size Sliding Window Patterns ===\n")

	fmt.Println("--- Two Types ---")
	fmt.Println("  1. LONGEST/MAX: expand right, shrink left when INVALID")
	fmt.Println("     → Answer updated after shrinking (valid window)")
	fmt.Println("  2. SHORTEST/MIN: expand right, shrink left while VALID")
	fmt.Println("     → Answer updated during shrinking (before becoming invalid)\n")

	fmt.Println("--- Template for LONGEST ---")
	fmt.Println("  left := 0")
	fmt.Println("  for right := range arr {")
	fmt.Println("      add arr[right]")
	fmt.Println("      for INVALID: remove arr[left]; left++")
	fmt.Println("      update maxLen = max(maxLen, right-left+1)")
	fmt.Println("  }\n")

	fmt.Println("--- Template for SHORTEST ---")
	fmt.Println("  left := 0")
	fmt.Println("  for right := range arr {")
	fmt.Println("      add arr[right]")
	fmt.Println("      for VALID: update minLen; remove arr[left]; left++")
	fmt.Println("  }\n")

	fmt.Println("--- Exactly K Trick ---")
	fmt.Println("  exactlyK(K) = atMost(K) - atMost(K-1)")
	fmt.Println("  Converts exact count to two at-most problems\n")

	fmt.Println("--- Common Problems ---")
	problems := []struct{ lc, name, typ string }{
		{"LC 3", "Longest Substring No Repeats", "Longest"},
		{"LC 76", "Minimum Window Substring", "Shortest"},
		{"LC 209", "Min Size Subarray Sum", "Shortest"},
		{"LC 340", "Longest K Distinct Chars", "Longest"},
		{"LC 424", "Longest Repeating Replacement", "Longest"},
		{"LC 904", "Fruit Into Baskets", "Longest"},
		{"LC 992", "Subarrays K Different", "Exactly K"},
		{"LC 1004", "Max Consecutive Ones III", "Longest"},
	}
	for _, p := range problems {
		fmt.Printf("  %-7s %-35s %s\n", p.lc, p.name, p.typ)
	}
}
```

---

## Key Takeaways

1. **Two patterns**: "find longest valid" (shrink when invalid) vs "find shortest valid" (shrink while valid)
2. **O(n) time**: each pointer moves at most n times → total 2n operations
3. **Exactly K = atMost(K) - atMost(K-1)**: powerful reduction technique
4. **State tracking**: hashmap for distinct counts, sum for target-based problems
5. **Window never moves backward**: left pointer only advances, guaranteeing O(n)
6. **Minimum Window Substring** is the classic hard problem — mastering it covers most patterns

> **Next up:** Shrinkable vs Non-Shrinkable Window →
