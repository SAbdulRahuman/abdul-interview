# Phase 25: Advanced Sliding Window — Two Pointer for Sorted Arrays

## Overview

Two-pointer technique on sorted arrays uses the sorted property to move pointers intelligently. Unlike sliding windows on unsorted data, sorted order lets us decide which pointer to move based on comparison with target.

### Common Patterns
1. **Opposite ends**: left=0, right=n-1, move toward each other
2. **Same direction**: both start at 0, one advances faster (merge-style)
3. **Multiple arrays**: one pointer per array

---

## Example 1: Two Sum — Sorted Array (LC 167)

```go
package main

import "fmt"

func twoSum(nums []int, target int) (int, int) {
	left, right := 0, len(nums)-1

	for left < right {
		sum := nums[left] + nums[right]
		if sum == target {
			return left, right
		} else if sum < target {
			left++ // need larger sum
		} else {
			right-- // need smaller sum
		}
	}
	return -1, -1
}

func main() {
	nums := []int{2, 7, 11, 15}
	target := 9
	i, j := twoSum(nums, target)
	fmt.Printf("nums=%v target=%d → indices (%d, %d)\n", nums, target, i, j)

	nums2 := []int{1, 2, 3, 4, 5, 6, 7}
	target2 := 10
	i2, j2 := twoSum(nums2, target2)
	fmt.Printf("nums=%v target=%d → indices (%d, %d) = %d+%d\n",
		nums2, target2, i2, j2, nums2[i2], nums2[j2])
}
```

---

## Example 2: Three Sum (LC 15)

```go
package main

import (
	"fmt"
	"sort"
)

func threeSum(nums []int) [][]int {
	sort.Ints(nums)
	var result [][]int

	for i := 0; i < len(nums)-2; i++ {
		if i > 0 && nums[i] == nums[i-1] { continue } // skip duplicates

		left, right := i+1, len(nums)-1
		target := -nums[i]

		for left < right {
			sum := nums[left] + nums[right]
			if sum == target {
				result = append(result, []int{nums[i], nums[left], nums[right]})
				for left < right && nums[left] == nums[left+1] { left++ }
				for left < right && nums[right] == nums[right-1] { right-- }
				left++
				right--
			} else if sum < target {
				left++
			} else {
				right--
			}
		}
	}
	return result
}

func main() {
	nums := []int{-1, 0, 1, 2, -1, -4}
	fmt.Printf("nums=%v\n", nums)
	fmt.Printf("3Sum: %v\n", threeSum(nums))
}
```

---

## Example 3: Three Sum Closest (LC 16)

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

func threeSumClosest(nums []int, target int) int {
	sort.Ints(nums)
	closest := nums[0] + nums[1] + nums[2]

	for i := 0; i < len(nums)-2; i++ {
		left, right := i+1, len(nums)-1

		for left < right {
			sum := nums[i] + nums[left] + nums[right]
			if math.Abs(float64(sum-target)) < math.Abs(float64(closest-target)) {
				closest = sum
			}
			if sum < target {
				left++
			} else if sum > target {
				right--
			} else {
				return target // exact match
			}
		}
	}
	return closest
}

func main() {
	nums := []int{-1, 2, 1, -4}
	target := 1
	fmt.Printf("nums=%v target=%d → closest=%d\n",
		nums, target, threeSumClosest(nums, target))
}
```

---

## Example 4: Container With Most Water (LC 11)

```go
package main

import "fmt"

func maxArea(height []int) int {
	left, right := 0, len(height)-1
	maxWater := 0

	for left < right {
		h := height[left]
		if height[right] < h { h = height[right] }
		area := h * (right - left)
		if area > maxWater { maxWater = area }

		// Move the shorter side (greedy: shorter side limits area)
		if height[left] < height[right] {
			left++
		} else {
			right--
		}
	}
	return maxWater
}

func main() {
	height := []int{1, 8, 6, 2, 5, 4, 8, 3, 7}
	fmt.Printf("height=%v\n", height)
	fmt.Printf("Max water: %d\n", maxArea(height))
}
```

---

## Example 5: Merge Two Sorted Arrays (LC 88)

```go
package main

import "fmt"

// Two pointers from same direction: merge pattern

func merge(nums1 []int, m int, nums2 []int, n int) {
	// Start from the end to avoid overwriting
	p1, p2, p := m-1, n-1, m+n-1

	for p1 >= 0 && p2 >= 0 {
		if nums1[p1] > nums2[p2] {
			nums1[p] = nums1[p1]
			p1--
		} else {
			nums1[p] = nums2[p2]
			p2--
		}
		p--
	}

	// Copy remaining from nums2
	for p2 >= 0 {
		nums1[p] = nums2[p2]
		p2--
		p--
	}
}

func main() {
	nums1 := []int{1, 2, 3, 0, 0, 0}
	nums2 := []int{2, 5, 6}
	m, n := 3, 3

	fmt.Printf("Before: nums1=%v nums2=%v\n", nums1, nums2)
	merge(nums1, m, nums2, n)
	fmt.Printf("After:  nums1=%v\n", nums1)
}
```

---

## Example 6: Square of Sorted Array (LC 977)

```go
package main

import "fmt"

func sortedSquares(nums []int) []int {
	n := len(nums)
	result := make([]int, n)
	left, right := 0, n-1

	// Fill from the end (largest squares at edges)
	for i := n - 1; i >= 0; i-- {
		leftSq := nums[left] * nums[left]
		rightSq := nums[right] * nums[right]

		if leftSq > rightSq {
			result[i] = leftSq
			left++
		} else {
			result[i] = rightSq
			right--
		}
	}
	return result
}

func main() {
	tests := [][]int{
		{-4, -1, 0, 3, 10},
		{-7, -3, 2, 3, 11},
	}

	for _, nums := range tests {
		fmt.Printf("nums=%v → squares=%v\n", nums, sortedSquares(nums))
	}
}
```

---

## Example 7: Remove Duplicates from Sorted Array (LC 26)

```go
package main

import "fmt"

// Slow-fast two pointer: slow marks the write position

func removeDuplicates(nums []int) int {
	if len(nums) == 0 { return 0 }

	slow := 0
	for fast := 1; fast < len(nums); fast++ {
		if nums[fast] != nums[slow] {
			slow++
			nums[slow] = nums[fast]
		}
	}
	return slow + 1
}

// Allow at most 2 duplicates (LC 80)
func removeDuplicatesII(nums []int) int {
	if len(nums) <= 2 { return len(nums) }

	slow := 2
	for fast := 2; fast < len(nums); fast++ {
		if nums[fast] != nums[slow-2] {
			nums[slow] = nums[fast]
			slow++
		}
	}
	return slow
}

func main() {
	nums1 := []int{1, 1, 2}
	k := removeDuplicates(nums1)
	fmt.Printf("Remove dups: %v → k=%d, result=%v\n", []int{1, 1, 2}, k, nums1[:k])

	nums2 := []int{1, 1, 1, 2, 2, 3}
	k2 := removeDuplicatesII(nums2)
	fmt.Printf("Allow 2 dups: %v → k=%d, result=%v\n", []int{1, 1, 1, 2, 2, 3}, k2, nums2[:k2])
}
```

---

## Example 8: Pair with Target Difference

```go
package main

import (
	"fmt"
	"sort"
)

// Find pair in sorted array with difference = target
// Two pointers, same direction

func findPairWithDiff(arr []int, target int) (int, int, bool) {
	sort.Ints(arr)
	left, right := 0, 1

	for right < len(arr) {
		if left == right { right++; continue }
		diff := arr[right] - arr[left]

		if diff == target {
			return arr[left], arr[right], true
		} else if diff < target {
			right++ // need larger difference
		} else {
			left++ // need smaller difference
		}
	}
	return 0, 0, false
}

func main() {
	arr := []int{1, 8, 5, 2, 3, 10}
	target := 3

	a, b, found := findPairWithDiff(arr, target)
	fmt.Printf("arr=%v target=%d\n", arr, target)
	if found {
		fmt.Printf("Pair: (%d, %d), diff=%d\n", a, b, b-a)
	}

	fmt.Println("\nOpposite direction: sum target")
	fmt.Println("Same direction: difference target")
}
```

---

## Example 9: Four Sum (LC 18)

```go
package main

import (
	"fmt"
	"sort"
)

func fourSum(nums []int, target int) [][]int {
	sort.Ints(nums)
	n := len(nums)
	var result [][]int

	for i := 0; i < n-3; i++ {
		if i > 0 && nums[i] == nums[i-1] { continue }

		for j := i + 1; j < n-2; j++ {
			if j > i+1 && nums[j] == nums[j-1] { continue }

			left, right := j+1, n-1
			rem := target - nums[i] - nums[j]

			for left < right {
				sum := nums[left] + nums[right]
				if sum == rem {
					result = append(result, []int{nums[i], nums[j], nums[left], nums[right]})
					for left < right && nums[left] == nums[left+1] { left++ }
					for left < right && nums[right] == nums[right-1] { right-- }
					left++; right--
				} else if sum < rem {
					left++
				} else {
					right--
				}
			}
		}
	}
	return result
}

func main() {
	nums := []int{1, 0, -1, 0, -2, 2}
	target := 0
	fmt.Printf("nums=%v target=%d\n", nums, target)
	fmt.Printf("4Sum: %v\n", fourSum(nums, target))
}
```

---

## Example 10: Patterns & Complexity Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Two Pointer on Sorted Arrays ===\n")

	fmt.Println("--- Pattern 1: Opposite Ends ---")
	fmt.Println("  left=0, right=n-1; converge inward")
	fmt.Println("  Use for: sum targets, container water, palindrome")
	fmt.Println()

	fmt.Println("--- Pattern 2: Same Direction ---")
	fmt.Println("  slow, fast both start at 0")
	fmt.Println("  Use for: remove duplicates, merge, difference target")
	fmt.Println()

	fmt.Println("--- Pattern 3: Multiple Arrays ---")
	fmt.Println("  One pointer per array")
	fmt.Println("  Use for: merge sorted arrays/lists, intersection")
	fmt.Println()

	fmt.Println("--- Common Problems ---")
	problems := []struct{ lc, name, pattern string }{
		{"LC 1", "Two Sum (sorted)", "Opposite ends"},
		{"LC 11", "Container With Water", "Opposite ends"},
		{"LC 15", "Three Sum", "Fix one + opposite ends"},
		{"LC 16", "3Sum Closest", "Fix one + opposite ends"},
		{"LC 18", "Four Sum", "Fix two + opposite ends"},
		{"LC 26", "Remove Duplicates", "Same direction"},
		{"LC 88", "Merge Sorted Array", "Same direction (reverse)"},
		{"LC 977", "Squares Sorted Array", "Opposite ends"},
	}
	for _, p := range problems {
		fmt.Printf("  %-7s %-25s %s\n", p.lc, p.name, p.pattern)
	}

	fmt.Println("\n--- Key Insight ---")
	fmt.Println("  Sorted order → can determine which pointer to move")
	fmt.Println("  Sum too small → move left pointer right (get bigger)")
	fmt.Println("  Sum too big → move right pointer left (get smaller)")
	fmt.Println("  Time: O(n) with sorted data; O(n log n) if sort needed")
}
```

---

## Key Takeaways

1. **Sorted order is essential**: it tells you which pointer to move
2. **Opposite ends** for sum/pair problems: left++ if too small, right-- if too big
3. **Same direction** for merge/deduplicate: fast scans, slow writes
4. **k-Sum reduction**: fix k-2 elements, then use 2-pointer on remainder → O(n^(k-1))
5. **Container water**: move the shorter side — greedy reasoning with two pointers
6. **Always O(n)** per two-pointer pass; sorting adds O(n log n) if not already sorted

> **Next up:** Fast & Slow Pointers →
