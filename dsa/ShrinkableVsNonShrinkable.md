# Phase 25: Advanced Sliding Window — Shrinkable vs Non-Shrinkable

## Overview

Variable-size sliding windows come in two flavors:

| Type | Shrink condition | Window state | Answer update |
|------|-----------------|--------------|---------------|
| **Shrinkable** | `while` invalid → shrink as much as possible | Always valid after shrink loop | After shrink loop |
| **Non-Shrinkable** | `if` invalid → shrink exactly once | Window size never decreases | Window size = answer candidate |

**Shrinkable** finds the exact optimal. **Non-shrinkable** is simpler and sufficient for "find maximum length" problems.

---

## Example 1: Shrinkable — Longest Substring Without Repeats (LC 3)

```go
package main

import "fmt"

// Shrinkable: use `for` (while) to shrink until valid

func longestUniqueShrinkable(s string) int {
	freq := make(map[byte]int)
	left, maxLen := 0, 0

	for right := 0; right < len(s); right++ {
		freq[s[right]]++

		// WHILE invalid: shrink until no duplicates
		for freq[s[right]] > 1 {
			freq[s[left]]--
			left++
		}

		// Window is valid → update answer
		if right-left+1 > maxLen {
			maxLen = right - left + 1
		}
	}
	return maxLen
}

func main() {
	tests := []string{"abcabcbb", "bbbbb", "pwwkew"}
	for _, s := range tests {
		fmt.Printf("Shrinkable: %q → %d\n", s, longestUniqueShrinkable(s))
	}
}
```

**Textual Figure — Shrinkable: Longest Substring Without Repeats:**

```
  s = "abcabcbb"    (shrink with `for`/while)

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ b │ c │ a │ b │ c │ b │ b │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    L→                          R→

  r=0: [a]       valid  max=1
  r=1: [ab]      valid  max=2
  r=2: [abc]     valid  max=3
  r=3: [abca]    'a' dup → WHILE shrink:
       L=1 [bca]  valid  max=3
  r=4: [bcab]    'b' dup → WHILE shrink:
       L=2 [cab]  valid  max=3
  r=5: [cabc]    'c' dup → WHILE shrink:
       L=3 [abc]  valid  max=3
  r=6: [abcb]    'b' dup → WHILE shrink:
       L=4 [cb] → L=5 [b]  max=3
  r=7: [bb]     'b' dup → WHILE shrink:
       L=7 [b]            max=3

  Result: 3
  Window always valid after shrink loop ✓
```

---

## Example 2: Non-Shrinkable — Same Problem

```go
package main

import "fmt"

// Non-shrinkable: use `if` to shrink at most once
// Window size never decreases, acts as "high water mark"

func longestUniqueNonShrinkable(s string) int {
	freq := make(map[byte]int)
	left := 0

	for right := 0; right < len(s); right++ {
		freq[s[right]]++

		// IF invalid: shrink exactly once
		if freq[s[right]] > 1 {
			freq[s[left]]--
			left++
		}
		// NOTE: window might still be invalid!
		// But it never shrinks below the max valid size found so far
	}

	// Final answer = window size (right reached end, left stopped)
	return len(s) - left
}

func main() {
	tests := []string{"abcabcbb", "bbbbb", "pwwkew"}
	for _, s := range tests {
		fmt.Printf("Non-shrinkable: %q → %d\n", s, longestUniqueNonShrinkable(s))
	}
}
```

**Textual Figure — Non-Shrinkable: Same Problem:**

```
  s = "abcabcbb"    (shrink with `if`, at most once)

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ b │ c │ a │ b │ c │ b │ b │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    L→                          R→

  r=0: [a]       valid       window=1
  r=1: [ab]      valid       window=2
  r=2: [abc]     valid       window=3
  r=3: [abca]    invalid → IF shrink once:
       L=1 [bca]             window=3 (kept!)
  r=4: [bcab]    invalid → IF shrink once:
       L=2 [cab]             window=3 (kept!)
  r=5: [cabc]    invalid → IF shrink once:
       L=3 [abc]             window=3 (kept!)
  r=6: [abcb]    invalid → IF shrink once:
       L=4 [bcb]             window=3 (kept!)
  r=7: [bcbb]    invalid → IF shrink once:
       L=5 [cbb]             window=3 (kept!)

  Final: len(s) - left = 8 - 5 = 3

  Window may be INVALID, but size never shrinks
  below the maximum valid size found ✓
```

---

## Example 3: Side-by-Side Comparison

```go
package main

import "fmt"

// Both approaches for: longest subarray with at most k distinct

func shrinkable(nums []int, k int) int {
	freq := make(map[int]int)
	left, maxLen := 0, 0

	for right := 0; right < len(nums); right++ {
		freq[nums[right]]++

		for len(freq) > k { // WHILE
			freq[nums[left]]--
			if freq[nums[left]] == 0 { delete(freq, nums[left]) }
			left++
		}

		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func nonShrinkable(nums []int, k int) int {
	freq := make(map[int]int)
	left := 0

	for right := 0; right < len(nums); right++ {
		freq[nums[right]]++

		if len(freq) > k { // IF (not while!)
			freq[nums[left]]--
			if freq[nums[left]] == 0 { delete(freq, nums[left]) }
			left++
		}
	}
	return len(nums) - left
}

func main() {
	nums := []int{1, 2, 1, 2, 3, 3, 4, 1, 2}
	k := 2

	fmt.Printf("nums=%v k=%d\n", nums, k)
	fmt.Printf("Shrinkable:     %d\n", shrinkable(nums, k))
	fmt.Printf("Non-shrinkable: %d\n", nonShrinkable(nums, k))

	fmt.Println("\nBoth give same answer for 'max length' problems!")
}
```

**Textual Figure — Side-by-Side: Shrinkable vs Non-Shrinkable:**

```
  nums = [1, 2, 1, 2, 3, 3, 4, 1, 2]    k = 2

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 1 │ 2 │ 3 │ 3 │ 4 │ 1 │ 2 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┘

  SHRINKABLE (for):            NON-SHRINKABLE (if):
  r=3: [1,2,1,2] valid len=4   [1,2,1,2] valid win=4
  r=4: [1,2,1,2,3] 3 dist!     [1,2,1,2,3] 3 dist!
       for: L=2 [1,2,3] still! for: L=1 (once)
       for: L=3 [2,3]   ok     [2,1,2,3] win=4(kept)
       len=2  max=4             win=4
  r=5: [2,3,3] ok len=3        [...] win=4
  r=6: 3 dist! shrink...       IF shrink once
       max stays 4              win stays 4

  Both return: 4  (subarray [1,2,1,2])

  ┌────────────────────────────────────┐
  │ for (while): window always valid  │
  │ if:          window may be invalid │
  │ Same answer for MAX LENGTH         │
  └────────────────────────────────────┘
```

---

## Example 4: When Shrinkable is Required

```go
package main

import (
	"fmt"
	"math"
)

// For MINIMUM length problems, must use shrinkable!
// Non-shrinkable only works for maximum length

func minSubarrayLen(target int, nums []int) int {
	left, sum := 0, 0
	minLen := math.MaxInt64

	for right := 0; right < len(nums); right++ {
		sum += nums[right]

		// MUST use while (shrinkable) to find minimum
		for sum >= target {
			if right-left+1 < minLen { minLen = right - left + 1 }
			sum -= nums[left]
			left++
		}
	}

	if minLen == math.MaxInt64 { return 0 }
	return minLen
}

func main() {
	target := 7
	nums := []int{2, 3, 1, 2, 4, 3}
	fmt.Printf("target=%d nums=%v\n", target, nums)
	fmt.Printf("Min subarray length: %d\n", minSubarrayLen(target, nums))

	fmt.Println("\n⚠️ Non-shrinkable CANNOT find minimum length!")
	fmt.Println("   It only works for maximum length problems")
}
```

**Textual Figure — Shrinkable Required for Minimum Length:**

```
  nums = [2, 3, 1, 2, 4, 3]    target = 7

  ┌───┬───┬───┬───┬───┬───┐
  │ 2 │ 3 │ 1 │ 2 │ 4 │ 3 │
  └───┴───┴───┴───┴───┴───┘
    L→                  R→

  MUST use `for` (while) to shrink as much as possible:

  r=3: sum=8 ≥ 7  min=4   shrink!
       L=1 sum=6         stop
  r=4: sum=10 ≥ 7 min=3  shrink!
       L=2 sum=7 ≥ 7 min=3  shrink!
       L=3 sum=6         stop
  r=5: sum=9 ≥ 7  min=3  shrink!
       L=4 sum=7 ≥ 7 min=2 ←  shrink!
       L=5 sum=3         stop

  Result: 2  (subarray [4,3])

  ⚠️ Non-shrinkable (`if`) would only shrink once
     and miss the optimal shorter window!
```

---

## Example 5: Non-Shrinkable — Max Consecutive Ones III (LC 1004)

```go
package main

import "fmt"

// Non-shrinkable version: cleaner for max length

func longestOnesShrinkable(nums []int, k int) int {
	left, zeros, maxLen := 0, 0, 0

	for right := 0; right < len(nums); right++ {
		if nums[right] == 0 { zeros++ }

		for zeros > k { // WHILE
			if nums[left] == 0 { zeros-- }
			left++
		}
		if right-left+1 > maxLen { maxLen = right - left + 1 }
	}
	return maxLen
}

func longestOnesNonShrinkable(nums []int, k int) int {
	left, zeros := 0, 0

	for right := 0; right < len(nums); right++ {
		if nums[right] == 0 { zeros++ }

		if zeros > k { // IF
			if nums[left] == 0 { zeros-- }
			left++
		}
	}
	return len(nums) - left
}

func main() {
	nums := []int{1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 0}
	k := 2

	fmt.Printf("nums=%v k=%d\n", nums, k)
	fmt.Printf("Shrinkable:     %d\n", longestOnesShrinkable(nums, k))
	fmt.Printf("Non-shrinkable: %d\n", longestOnesNonShrinkable(nums, k))
}
```

**Textual Figure — Max Consecutive Ones III (Both Approaches):**

```
  nums = [1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 0]    k = 2

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 1 │ 1 │ 0 │ 0 │ 0 │ 1 │ 1 │ 1 │ 1 │ 0 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7   8   9  10

  SHRINKABLE (for):
  r=4: zeros=2 ≤ k  window=5  max=5
  r=5: zeros=3 > k  → for: shrink until zeros ≤ 2
       L goes to 4  window=2  max=5
  r=9: [0,0,1,1,1,1] zeros=2  window=6  max=6 ←

  NON-SHRINKABLE (if):
  r=5: zeros=3 > k  → if: shrink once, L=1
       window=5 (kept)
  r=9: window grows to 6  max=6
  Final: 11 - 5 = 6

  Both return: 6  ✓
```

---

## Example 6: Non-Shrinkable — Character Replacement (LC 424)

```go
package main

import "fmt"

// Non-shrinkable is natural for this problem

func characterReplacement(s string, k int) int {
	var freq [26]int
	left, maxFreq := 0, 0

	for right := 0; right < len(s); right++ {
		freq[s[right]-'A']++
		if freq[s[right]-'A'] > maxFreq {
			maxFreq = freq[s[right]-'A']
		}

		windowLen := right - left + 1
		if windowLen-maxFreq > k { // IF (non-shrinkable)
			freq[s[left]-'A']--
			left++
		}
	}
	return len(s) - left
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

	fmt.Println("\nNote: maxFreq is never decremented in non-shrinkable!")
	fmt.Println("This is correct because window can only grow or stay same size")
}
```

**Textual Figure — Non-Shrinkable Character Replacement:**

```
  s = "AABABBA"    k = 1

  ┌───┬───┬───┬───┬───┬───┬───┐
  │ A │ A │ B │ A │ B │ B │ A │
  └───┴───┴───┴───┴───┴───┴───┘
    L→                      R→

  windowLen - maxFreq ≤ k?  (non-shrinkable: `if`)

  r=0: [A]       mxF=1  1-1=0 ≤ 1  win=1
  r=1: [AA]      mxF=2  2-2=0 ≤ 1  win=2
  r=2: [AAB]     mxF=2  3-2=1 ≤ 1  win=3
  r=3: [AABA]    mxF=3  4-3=1 ≤ 1  win=4
  r=4: [AABAB]   mxF=3  5-3=2 > 1  IF: L++
       L=1 [ABAB]                   win=4
  r=5: [ABABB]   mxF=3  5-3=2 > 1  IF: L++
       L=2 [BABB]                   win=4
  r=6: [BABBA]   mxF=3  5-3=2 > 1  IF: L++
       L=3 [ABBA]                   win=4

  Final: len(s) - left = 7 - 3 = 4

  maxFreq never decreases in non-shrinkable →
  window only grows or stays same ✓
```

---

## Example 7: Understanding Why Non-Shrinkable Works

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Why Non-Shrinkable Works for Max Length ===\n")

	fmt.Println("Key insight: we only care about the MAXIMUM valid window\n")

	fmt.Println("Shrinkable trace for 'abcba' (no repeats):")
	fmt.Println("  r=0: [a]     → left=0, window=1, maxLen=1")
	fmt.Println("  r=1: [ab]    → left=0, window=2, maxLen=2")
	fmt.Println("  r=2: [abc]   → left=0, window=3, maxLen=3")
	fmt.Println("  r=3: [abcb]  → SHRINK to [cb], left=2, window=2, maxLen=3")
	fmt.Println("  r=4: [cba]   → left=2, window=3, maxLen=3")

	fmt.Println("\nNon-shrinkable trace for 'abcba' (no repeats):")
	fmt.Println("  r=0: [a]     → left=0, window=1")
	fmt.Println("  r=1: [ab]    → left=0, window=2")
	fmt.Println("  r=2: [abc]   → left=0, window=3")
	fmt.Println("  r=3: [abcb]  → INVALID, left=1, window=3 (kept!)")
	fmt.Println("  r=4: [bcba]  → INVALID, left=2, window=3 (kept!)")
	fmt.Println("  Final: 5-2 = 3 ✓")

	fmt.Println("\n--- The Rule ---")
	fmt.Println("Non-shrinkable: window only grows.")
	fmt.Println("  • Right always advances by 1")
	fmt.Println("  • Left advances by 0 or 1")
	fmt.Println("  • Window size = right-left+1 ≥ previous max")
	fmt.Println("  • Window might be INVALID, but that's OK!")
	fmt.Println("  • We only need the final window size as the answer")
}
```

**Textual Figure — Why Non-Shrinkable Works:**

```
  s = "abcba"    (longest substring without repeats)

  SHRINKABLE:                  NON-SHRINKABLE:
  ┌───┬───┬───┬───┬───┐      ┌───┬───┬───┬───┬───┐
  │ a │ b │ c │ b │ a │      │ a │ b │ c │ b │ a │
  └───┴───┴───┴───┴───┘      └───┴───┴───┴───┴───┘

  r=2: [abc] valid  max=3    r=2: [abc] valid   win=3
  r=3: [abcb] dup!           r=3: [abcb] dup!
       shrink→ [cb] max=3         IF: L=1 [bcb]  win=3
  r=4: [cba] valid  max=3   r=4: [bcba] dup!
                                  IF: L=2 [cba]  win=3

  Shrinkable: max = 3        Non-shrinkable: 5-2 = 3

  ┌──────────────────────────────────────┐
  │ Non-shrinkable key insight:          │
  │ Window size = "high water mark"       │
  │ Once valid window of size k found,    │
  │ we only look for k+1 or bigger.       │
  │ Invalid windows are OK — they just    │
  │ slide along without growing.          │
  └──────────────────────────────────────┘
```

---

## Example 8: Shrinkable for Counting Problems

```go
package main

import "fmt"

// For COUNTING (not just max length), must use shrinkable

// Count subarrays where max - min <= k using shrinkable window

func countSubarrays(nums []int, k int) int {
	left, count := 0, 0
	// Use two monotonic deques for min/max
	var maxDeq, minDeq []int // store indices

	for right := 0; right < len(nums); right++ {
		for len(maxDeq) > 0 && nums[maxDeq[len(maxDeq)-1]] <= nums[right] {
			maxDeq = maxDeq[:len(maxDeq)-1]
		}
		maxDeq = append(maxDeq, right)

		for len(minDeq) > 0 && nums[minDeq[len(minDeq)-1]] >= nums[right] {
			minDeq = minDeq[:len(minDeq)-1]
		}
		minDeq = append(minDeq, right)

		// WHILE invalid (must shrink to count valid subarrays)
		for nums[maxDeq[0]]-nums[minDeq[0]] > k {
			left++
			if maxDeq[0] < left { maxDeq = maxDeq[1:] }
			if minDeq[0] < left { minDeq = minDeq[1:] }
		}

		// All subarrays [left..right], [left+1..right], ..., [right..right]
		count += right - left + 1
	}
	return count
}

func main() {
	nums := []int{1, 3, 2, 4, 1}
	k := 2

	fmt.Printf("nums=%v k=%d\n", nums, k)
	fmt.Printf("Subarrays with max-min ≤ k: %d\n", countSubarrays(nums, k))

	fmt.Println("\n⚠️ Counting requires shrinkable (while loop)")
	fmt.Println("   Non-shrinkable doesn't give correct counts")
}
```

**Textual Figure — Shrinkable for Counting (Max−Min ≤ k):**

```
  nums = [1, 3, 2, 4, 1]    k = 2

  ┌───┬───┬───┬───┬───┐
  │ 1 │ 3 │ 2 │ 4 │ 1 │
  └───┴───┴───┴───┴───┘
    L→              R→

  Must shrink fully to count ALL valid subarrays:

  r=0: max=1 min=1 diff=0 ≤ 2
       valid subarrays ending at 0: [1]           count+=1 =1
  r=1: max=3 min=1 diff=2 ≤ 2
       valid: [3],[1,3]                           count+=2 =3
  r=2: max=3 min=1 diff=2 ≤ 2
       valid: [2],[3,2],[1,3,2]                   count+=3 =6
  r=3: max=4 min=1 diff=3 > 2 → WHILE shrink!
       L=1: max=4 min=2 diff=2 ≤ 2
       valid: [4],[2,4],[3,2,4]                   count+=3 =9
  r=4: max=4 min=1 diff=3 > 2 → WHILE shrink!
       L=2: max=4 min=1... L=3: max=4 min=1...
       L=4: max=1 min=1 diff=0 ≤ 2
       valid: [1]                                 count+=1 =10

  Result: 10 valid subarrays
```

---

## Example 9: Shrinkable — Minimum Window Substring (LC 76)

```go
package main

import (
	"fmt"
	"math"
)

// Classic problem that REQUIRES shrinkable

func minWindow(s, t string) string {
	need := make(map[byte]int)
	for i := 0; i < len(t); i++ { need[t[i]]++ }

	have := make(map[byte]int)
	formed, required := 0, len(need)
	left, minLen, start := 0, math.MaxInt64, 0

	for right := 0; right < len(s); right++ {
		c := s[right]
		have[c]++
		if have[c] == need[c] { formed++ }

		// WHILE valid → shrink to find minimum (shrinkable!)
		for formed == required {
			if right-left+1 < minLen {
				minLen = right - left + 1
				start = left
			}
			have[s[left]]--
			if have[s[left]] < need[s[left]] { formed-- }
			left++
		}
	}

	if minLen == math.MaxInt64 { return "" }
	return s[start : start+minLen]
}

func main() {
	s, t := "ADOBECODEBANC", "ABC"
	fmt.Printf("s=%q t=%q → %q\n", s, t, minWindow(s, t))

	fmt.Println("\nMinimum → must use shrinkable (while)")
	fmt.Println("Cannot use non-shrinkable for minimum problems")
}
```

**Textual Figure — Shrinkable Minimum Window Substring:**

```
  s = "ADOBECODEBANC"    t = "ABC"
  need: {A:1, B:1, C:1}

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ A │ D │ O │ B │ E │ C │ O │ D │ E │ B │ A │ N │ C │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7   8   9  10  11  12
    L→                                             R→

  Shrinkable (WHILE valid → shrink to minimize):

  r=5: ADOBEC  → formed=3 (all ABC found)
       WHILE valid: min=6  shrink L→1
       DOBEC → lost A, formed=2  stop

  r=10: expanded to find A again
        WHILE valid: keep shrinking...
        CODEBA  min=6

  r=12: ...BANC  formed=3
        WHILE valid: shrink!
        BANC  min=4 ← best!
        ANC  lost B, formed=2  stop

  Result: "BANC" (length 4)

  Must shrink FULLY (while) to find minimum!
```

---

## Example 10: Decision Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Shrinkable vs Non-Shrinkable Decision Guide ===\n")

	fmt.Println("--- When to Use Which ---")
	fmt.Println()

	fmt.Println("SHRINKABLE (while loop):")
	fmt.Println("  ✓ Finding MINIMUM length → REQUIRED")
	fmt.Println("  ✓ COUNTING valid subarrays → REQUIRED")
	fmt.Println("  ✓ Finding MAXIMUM length → works (but overkill)")
	fmt.Println("  ✓ Need exact window boundaries → REQUIRED")
	fmt.Println("  • Uses: for/while to shrink as much as possible")
	fmt.Println("  • Window is always valid after shrink loop")
	fmt.Println()

	fmt.Println("NON-SHRINKABLE (if statement):")
	fmt.Println("  ✓ Finding MAXIMUM length → simpler code")
	fmt.Println("  ✗ Finding MINIMUM length → WRONG")
	fmt.Println("  ✗ COUNTING subarrays → WRONG")
	fmt.Println("  • Uses: if to shrink at most once")
	fmt.Println("  • Window might be invalid, but size tracks max")
	fmt.Println()

	fmt.Println("--- Quick Decision ---")
	decisions := []struct{ question, answer string }{
		{"Want max length?", "Either works; non-shrinkable is simpler"},
		{"Want min length?", "Must use shrinkable (while)"},
		{"Need to count?", "Must use shrinkable (while)"},
		{"Need exact window?", "Must use shrinkable (while)"},
		{"Prefer simpler code?", "Non-shrinkable (if) for max length"},
	}
	for _, d := range decisions {
		fmt.Printf("  Q: %-25s A: %s\n", d.question, d.answer)
	}

	fmt.Println("\n--- Code Difference ---")
	fmt.Println("  Shrinkable:     for CONDITION { shrink }")
	fmt.Println("  Non-shrinkable: if CONDITION { shrink }")
	fmt.Println("  That's it! Just `for` vs `if`")
}
```

---

## Key Takeaways

1. **Shrinkable** (`for`/`while`): shrinks as much as possible — window always valid afterward
2. **Non-shrinkable** (`if`): shrinks at most once — window may be invalid but size tracks maximum
3. **Maximum length**: either works; non-shrinkable is simpler
4. **Minimum length / counting**: **must** use shrinkable
5. **The only code difference**: `for condition` vs `if condition`
6. **Non-shrinkable insight**: window size never decreases — once you find a valid window of size k, you only care about k+1 or larger

> **Next up:** Two Pointer for Sorted Arrays →
