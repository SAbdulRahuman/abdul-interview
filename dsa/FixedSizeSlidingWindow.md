# Phase 25: Advanced Sliding Window — Fixed Size Sliding Window

## Overview

A **fixed-size sliding window** maintains a window of exactly `k` elements as it slides across an array/string. At each step, one element enters and one leaves.

### Template
```
1. Initialize window with first k elements
2. Record answer for first window
3. For i = k to n-1:
   a. Add element arr[i]         (enters window)
   b. Remove element arr[i-k]    (leaves window)
   c. Update answer
```

**Time Complexity**: O(n) — each element is added and removed exactly once.

---

## Example 1: Maximum Sum Subarray of Size K

```go
package main

import "fmt"

func maxSumSubarray(arr []int, k int) int {
	n := len(arr)
	if n < k { return 0 }

	// Window: sum of first k elements
	windowSum := 0
	for i := 0; i < k; i++ { windowSum += arr[i] }
	maxSum := windowSum

	// Slide
	for i := k; i < n; i++ {
		windowSum += arr[i] - arr[i-k]  // add new, remove old
		if windowSum > maxSum { maxSum = windowSum }
	}

	return maxSum
}

func main() {
	arr := []int{2, 1, 5, 1, 3, 2}
	k := 3
	fmt.Printf("Array: %v, k=%d\n", arr, k)
	fmt.Printf("Max sum subarray of size %d: %d\n", k, maxSumSubarray(arr, k))
	// Window: [2,1,5]=8, [1,5,1]=7, [5,1,3]=9, [1,3,2]=6 → max=9
}
```

**Textual Figure — Maximum Sum Subarray of Size K:**

```
  arr = [2, 1, 5, 1, 3, 2]    k = 3

  ┌───┬───┬───┬───┬───┬───┐
  │ 2 │ 1 │ 5 │ 1 │ 3 │ 2 │
  └───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5

  Window slides → one element enters, one leaves:

  Step 1: [2, 1, 5]           sum = 8
          ├─────────┤
  Step 2:    [1, 5, 1]        sum = 7   (−2, +1)
             ├─────────┤
  Step 3:       [5, 1, 3]     sum = 9   (−1, +3)  ← max
                ├─────────┤
  Step 4:          [1, 3, 2]  sum = 6   (−5, +2)
                   ├─────────┤

  Result: max sum = 9  (window [5, 1, 3])
```

---

## Example 2: Maximum of All Subarrays of Size K (LC 239)

```go
package main

import "fmt"

// Sliding window maximum using monotonic deque

func maxSlidingWindow(nums []int, k int) []int {
	var result []int
	var deque []int // stores indices, front = max

	for i := 0; i < len(nums); i++ {
		// Remove elements outside window
		for len(deque) > 0 && deque[0] <= i-k {
			deque = deque[1:]
		}

		// Remove smaller elements from back (maintain decreasing order)
		for len(deque) > 0 && nums[deque[len(deque)-1]] <= nums[i] {
			deque = deque[:len(deque)-1]
		}

		deque = append(deque, i)

		// Window is complete (i >= k-1)
		if i >= k-1 {
			result = append(result, nums[deque[0]])
		}
	}
	return result
}

func main() {
	nums := []int{1, 3, -1, -3, 5, 3, 6, 7}
	k := 3
	fmt.Printf("nums: %v, k=%d\n", nums, k)
	fmt.Printf("Sliding window max: %v\n", maxSlidingWindow(nums, k))
	// [3,3,5,5,6,7]
}
```

**Textual Figure — Sliding Window Maximum with Monotonic Deque:**

```
  nums = [1, 3, -1, -3, 5, 3, 6, 7]    k = 3

  ┌───┬───┬────┬────┬───┬───┬───┬───┐
  │ 1 │ 3 │ -1 │ -3 │ 5 │ 3 │ 6 │ 7 │
  └───┴───┴────┴────┴───┴───┴───┴───┘
    0   1    2    3   4   5   6   7

  Window          Deque (vals)   Max
  ├───────┤
  [1, 3,-1]       [3, -1]        3
     ├───────┤
  [3,-1,-3]       [3, -1, -3]    3
        ├────────┤
  [-1,-3, 5]      [5]            5
           ├────────┤
  [-3, 5, 3]      [5, 3]         5
              ├────────┤
  [5, 3, 6]       [6]            6
                 ├────────┤
  [3, 6, 7]       [7]            7

  Result: [3, 3, 5, 5, 6, 7]

  Deque invariant: front = max of current window
  Smaller elements removed from back (never useful)
```

---

## Example 3: Average of Subarrays of Size K

```go
package main

import "fmt"

func averages(arr []float64, k int) []float64 {
	n := len(arr)
	if n < k { return nil }

	result := make([]float64, 0, n-k+1)

	windowSum := 0.0
	for i := 0; i < k; i++ { windowSum += arr[i] }
	result = append(result, windowSum/float64(k))

	for i := k; i < n; i++ {
		windowSum += arr[i] - arr[i-k]
		result = append(result, windowSum/float64(k))
	}
	return result
}

func main() {
	arr := []float64{1, 3, 2, 6, -1, 4, 1, 8, 2}
	k := 5
	fmt.Printf("Array: %v, k=%d\n", arr, k)
	fmt.Printf("Averages: %v\n", averages(arr, k))
}
```

**Textual Figure — Average of Subarrays of Size K:**

```
  arr = [1, 3, 2, 6, -1, 4, 1, 8, 2]    k = 5

  ┌───┬───┬───┬───┬────┬───┬───┬───┬───┐
  │ 1 │ 3 │ 2 │ 6 │ -1 │ 4 │ 1 │ 8 │ 2 │
  └───┴───┴───┴───┴────┴───┴───┴───┴───┘
    0   1   2   3    4   5   6   7   8

  Window (size 5)             Sum    Avg
  ├───────────────────┤
  [1, 3, 2, 6, -1]           11     2.2
     ├───────────────────┤
  [3, 2, 6, -1, 4]           14     2.8
        ├───────────────────┤
  [2, 6, -1, 4, 1]           12     2.4
           ├───────────────────┤
  [6, -1, 4, 1, 8]           18     3.6
              ├───────────────────┤
  [-1, 4, 1, 8, 2]           14     2.8

  Averages: [2.2, 2.8, 2.4, 3.6, 2.8]
```

---

## Example 4: Contains Duplicate II (LC 219)

```go
package main

import "fmt"

// Check if there are two equal elements within distance k

func containsNearbyDuplicate(nums []int, k int) bool {
	window := make(map[int]bool)

	for i, num := range nums {
		if window[num] { return true }
		window[num] = true

		// Maintain window size k
		if i >= k {
			delete(window, nums[i-k])
		}
	}
	return false
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{1, 2, 3, 1}, 3},
		{[]int{1, 0, 1, 1}, 1},
		{[]int{1, 2, 3, 1, 2, 3}, 2},
	}

	for _, t := range tests {
		fmt.Printf("nums=%v k=%d → %v\n", t.nums, t.k, containsNearbyDuplicate(t.nums, t.k))
	}
}
```

**Textual Figure — Contains Duplicate II (Hash Set Window):**

```
  nums = [1, 2, 3, 1]    k = 3

  ┌───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 1 │
  └───┴───┴───┴───┘
    0   1   2   3

  Window (hash set) of at most k+1 elements:

  i=0: window={1}         │ 1 not seen → add
  i=1: window={1,2}       │ 2 not seen → add
  i=2: window={1,2,3}     │ 3 not seen → add
  i=3: window={2,3,1}     │ 1 seen? No (removed at i-k=0)
       Wait — check: i=3, i-k=0 → remove nums[0]=1
       Actually: add 1, check first → 1 IS in {1,2,3} → true!

  ┌────────────────────────────────────────┐
  │ Maintain a set of size k as window     │
  │ If new element already in set → true   │
  │ Remove element leaving window (i−k)    │
  └────────────────────────────────────────┘

  Result: true (nums[0]=1 and nums[3]=1, distance=3 ≤ k=3)
```

---

## Example 5: Maximum Number of Vowels in Substring of Size K (LC 1456)

```go
package main

import "fmt"

func maxVowels(s string, k int) int {
	isVowel := func(c byte) bool {
		return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u'
	}

	count := 0
	for i := 0; i < k; i++ {
		if isVowel(s[i]) { count++ }
	}
	maxCount := count

	for i := k; i < len(s); i++ {
		if isVowel(s[i]) { count++ }
		if isVowel(s[i-k]) { count-- }
		if count > maxCount { maxCount = count }
	}
	return maxCount
}

func main() {
	tests := []struct {
		s string
		k int
	}{
		{"abciiidef", 3},
		{"aeiou", 2},
		{"leetcode", 3},
	}

	for _, t := range tests {
		fmt.Printf("s=%q k=%d → %d\n", t.s, t.k, maxVowels(t.s, t.k))
	}
}
```

**Textual Figure — Maximum Vowels in Substring of Size K:**

```
  s = "abciiidef"    k = 3

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ a │ b │ c │ i │ i │ i │ d │ e │ f │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7   8
    V           V   V   V       V
  (V = vowel)

  Window (size 3)       Vowels
  ├───────┤
  [a,b,c]               1 (a)
     ├───────┤
  [b,c,i]               1 (i)         −a, +i
        ├───────┤
  [c,i,i]               2 (i,i)       −b, +i
           ├───────┤
  [i,i,i]               3 (i,i,i)     −c, +i  ← max
              ├───────┤
  [i,i,d]               2 (i,i)       −i, +d
                 ├───────┤
  [i,d,e]               2 (i,e)       −i, +e
                    ├───────┤
  [d,e,f]               1 (e)         −i, +f

  Result: max vowels = 3
```

---

## Example 6: Number of Sub-arrays of Size K with Average ≥ Threshold (LC 1343)

```go
package main

import "fmt"

func numOfSubarrays(arr []int, k int, threshold int) int {
	target := threshold * k // Compare sum to avoid float division
	count := 0

	windowSum := 0
	for i := 0; i < k; i++ { windowSum += arr[i] }
	if windowSum >= target { count++ }

	for i := k; i < len(arr); i++ {
		windowSum += arr[i] - arr[i-k]
		if windowSum >= target { count++ }
	}
	return count
}

func main() {
	arr := []int{2, 2, 2, 2, 5, 5, 5, 8}
	k, threshold := 3, 4
	fmt.Printf("arr=%v k=%d threshold=%d\n", arr, k, threshold)
	fmt.Printf("Count: %d\n", numOfSubarrays(arr, k, threshold))
	// [2,5,5]=12≥12, [5,5,5]=15≥12, [5,5,8]=18≥12 → 3 (actually more...)
}
```

**Textual Figure — Subarrays with Average ≥ Threshold:**

```
  arr = [2, 2, 2, 2, 5, 5, 5, 8]   k = 3   threshold = 4
  target = threshold × k = 12  (compare sum directly)

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 2 │ 2 │ 2 │ 2 │ 5 │ 5 │ 5 │ 8 │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7

  Window        Sum   ≥ 12?
  [2,2,2]        6     ✗
  [2,2,2]        6     ✗
  [2,2,5]        9     ✗
  [2,5,5]       12     ✓  count=1
  [5,5,5]       15     ✓  count=2
  [5,5,8]       18     ✓  count=3

  Result: count = 3
```

---

## Example 7: Grumpy Bookstore Owner (LC 1052)

```go
package main

import "fmt"

func maxSatisfied(customers []int, grumpy []int, minutes int) int {
	n := len(customers)

	// Base: customers when owner is not grumpy
	base := 0
	for i := 0; i < n; i++ {
		if grumpy[i] == 0 { base += customers[i] }
	}

	// Fixed window of size 'minutes': extra customers saved
	extra := 0
	for i := 0; i < minutes; i++ {
		if grumpy[i] == 1 { extra += customers[i] }
	}
	maxExtra := extra

	for i := minutes; i < n; i++ {
		if grumpy[i] == 1 { extra += customers[i] }
		if grumpy[i-minutes] == 1 { extra -= customers[i-minutes] }
		if extra > maxExtra { maxExtra = extra }
	}

	return base + maxExtra
}

func main() {
	customers := []int{1, 0, 1, 2, 1, 1, 7, 5}
	grumpy    := []int{0, 1, 0, 1, 0, 1, 0, 1}
	minutes := 3

	fmt.Printf("Customers: %v\n", customers)
	fmt.Printf("Grumpy:    %v\n", grumpy)
	fmt.Printf("Minutes:   %d\n", minutes)
	fmt.Printf("Max satisfied: %d\n", maxSatisfied(customers, grumpy, minutes))
}
```

**Textual Figure — Grumpy Bookstore Owner:**

```
  customers = [1, 0, 1, 2, 1, 1, 7, 5]
  grumpy    = [0, 1, 0, 1, 0, 1, 0, 1]
  minutes   = 3

  Base satisfied (grumpy=0):  1 + 1 + 1 + 7 = 10

  Index:      0   1   2   3   4   5   6   7
  Customers: [1] [0] [1] [2] [1] [1] [7] [5]
  Grumpy:     0   1   0   1   0   1   0   1
              ✓   ✗   ✓   ✗   ✓   ✗   ✓   ✗

  Slide window (size 3) → extra saved from grumpy=1:

  Window [0..2]: grumpy[1]=1 → extra=0     total=10+0=10
  Window [1..3]: grumpy[1,3]=1 → extra=0+2=2  total=10+2=12
  Window [2..4]: grumpy[3]=1 → extra=2     total=10+2=12
  Window [3..5]: grumpy[3,5]=1 → extra=2+1=3 total=10+3=13
  Window [4..6]: grumpy[5]=1 → extra=1     total=10+1=11
  Window [5..7]: grumpy[5,7]=1 → extra=1+5=6 total=10+6=16 ← max

  Result: 16
```

---

## Example 8: K Radius Subarray Averages (LC 2090)

```go
package main

import "fmt"

func getAverages(nums []int, k int) []int {
	n := len(nums)
	result := make([]int, n)
	for i := range result { result[i] = -1 }

	windowSize := 2*k + 1
	if windowSize > n { return result }

	windowSum := 0
	for i := 0; i < windowSize; i++ { windowSum += nums[i] }
	result[k] = windowSum / windowSize

	for i := windowSize; i < n; i++ {
		windowSum += nums[i] - nums[i-windowSize]
		result[i-k] = windowSum / windowSize
	}
	return result
}

func main() {
	nums := []int{7, 4, 3, 9, 1, 8, 5, 2, 6}
	k := 3
	fmt.Printf("nums=%v k=%d\n", nums, k)
	fmt.Printf("K-radius averages: %v\n", getAverages(nums, k))
}
```

**Textual Figure — K Radius Subarray Averages:**

```
  nums = [7, 4, 3, 9, 1, 8, 5, 2, 6]    k = 3
  windowSize = 2×3+1 = 7

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 7 │ 4 │ 3 │ 9 │ 1 │ 8 │ 5 │ 2 │ 6 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7   8

  Index 0,1,2: not enough left neighbors → -1
  Index 6,7,8: not enough right neighbors → -1

  i=3 (center): window [0..6]
  ├─────────────────────────┤
  [7,4,3,9,1,8,5] sum=37 avg=37/7=5

  i=4 (center): window [1..7]
     ├─────────────────────────┤
  [4,3,9,1,8,5,2] sum=32 avg=32/7=4

  i=5 (center): window [2..8]
        ├─────────────────────────┤
  [3,9,1,8,5,2,6] sum=34 avg=34/7=4

  Result: [-1,-1,-1, 5, 4, 4, -1,-1,-1]
```

---

## Example 9: Fixed Window with HashMap — Permutation in String (LC 567)

```go
package main

import "fmt"

func checkInclusion(s1, s2 string) bool {
	if len(s1) > len(s2) { return false }

	var need, have [26]int
	for _, c := range s1 { need[c-'a']++ }

	k := len(s1)

	// Build initial window
	for i := 0; i < k; i++ { have[s2[i]-'a']++ }
	if need == have { return true }

	// Slide
	for i := k; i < len(s2); i++ {
		have[s2[i]-'a']++     // add new
		have[s2[i-k]-'a']--   // remove old
		if need == have { return true }
	}
	return false
}

func main() {
	tests := []struct{ s1, s2 string }{
		{"ab", "eidbaooo"},
		{"ab", "eidboaoo"},
		{"adc", "dcda"},
	}

	for _, t := range tests {
		fmt.Printf("s1=%q s2=%q → %v\n", t.s1, t.s2, checkInclusion(t.s1, t.s2))
	}
}
```

**Textual Figure — Permutation in String (Char Frequency Window):**

```
  s1 = "ab"   s2 = "eidbaooo"   k = len(s1) = 2

  need: [a=1, b=1]

  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ e │ i │ d │ b │ a │ o │ o │ o │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    0   1   2   3   4   5   6   7

  Window (size 2)    have          Match?
  ├───┤
  [e,i]             [e=1,i=1]      ✗
     ├───┤
  [i,d]             [i=1,d=1]      ✗
        ├───┤
  [d,b]             [d=1,b=1]      ✗
           ├───┤
  [b,a]             [a=1,b=1]      ✓ == need!

  Result: true  ("ba" is a permutation of "ab")

  ┌───────────────────────────────────────┐
  │ Compare frequency arrays directly:   │
  │ need == have → permutation found     │
  └───────────────────────────────────────┘
```

---

## Example 10: Fixed Window Pattern Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Fixed Size Sliding Window Patterns ===\n")

	fmt.Println("--- Template ---")
	fmt.Println("  windowState := initialize(arr[:k])")
	fmt.Println("  answer := evaluate(windowState)")
	fmt.Println("  for i := k; i < n; i++ {")
	fmt.Println("      add(arr[i])          // element enters")
	fmt.Println("      remove(arr[i-k])     // element leaves")
	fmt.Println("      answer = better(answer, evaluate(windowState))")
	fmt.Println("  }\n")

	fmt.Println("--- Window State Data Structures ---")
	states := []struct{ ds, useCase string }{
		{"int (sum)", "Sum / average queries"},
		{"int (count)", "Count of elements matching condition"},
		{"[26]int", "Character frequency (anagram/permutation)"},
		{"map[int]bool", "Set membership (contains duplicate)"},
		{"Deque (monotonic)", "Min/Max in window"},
		{"Sorted container", "Median in window"},
	}
	for _, s := range states {
		fmt.Printf("  %-22s → %s\n", s.ds, s.useCase)
	}

	fmt.Println("\n--- Complexity ---")
	fmt.Println("  Time:  O(n) — each element processed exactly twice")
	fmt.Println("  Space: O(k) — window state")

	fmt.Println("\n--- Common Problems ---")
	problems := []string{
		"LC 239:  Sliding Window Maximum (monotonic deque)",
		"LC 567:  Permutation in String (char freq array)",
		"LC 219:  Contains Duplicate II (hash set window)",
		"LC 1456: Max Vowels in Substring (count)",
		"LC 1052: Grumpy Bookstore Owner (extra saved)",
		"LC 2090: K Radius Subarray Averages (sum)",
		"LC 1343: Subarrays with Average ≥ Threshold (sum)",
	}
	for _, p := range problems { fmt.Println("  " + p) }
}
```

---

## Key Takeaways

1. **Fixed window = constant size**: exactly one element enters and one leaves per step
2. **Template**: init window → slide by add/remove → O(n)
3. **State tracking**: sum, count, hashmap, monotonic deque depending on the query
4. **Monotonic deque** for min/max queries in O(1) per operation
5. **Char frequency array** comparison for anagram/permutation checks
6. **Avoid recomputation**: maintain running state instead of recalculating from scratch

> **Next up:** Variable Size Sliding Window →
