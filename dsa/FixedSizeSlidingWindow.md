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
