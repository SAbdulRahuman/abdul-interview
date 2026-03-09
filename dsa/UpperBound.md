# Phase 10: Binary Search — Upper Bound

## Overview

**Upper bound** finds the **first position** where `arr[i] > target`. Combined with lower bound, you can solve almost any binary search problem.

```
sorted:  [1, 2, 4, 4, 4, 7, 9]
upper_bound(4) → index 5 (first element > 4, which is 7)
upper_bound(5) → index 5 (first element > 5, which is 7)
```

Key relationships:
- **Count of target** = upperBound(target) − lowerBound(target)
- **Last occurrence** = upperBound(target) − 1
- **Ceiling(target)** = lowerBound(target)
- **Floor(target)** = upperBound(target) − 1

---

## Example 1: Basic Upper Bound

```go
package main

import "fmt"

func upperBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo // first index where nums[i] > target
}

func main() {
	nums := []int{1, 2, 4, 4, 4, 7, 9}
	fmt.Println(upperBound(nums, 4))  // 5
	fmt.Println(upperBound(nums, 5))  // 5
	fmt.Println(upperBound(nums, 9))  // 7 (past end)
	fmt.Println(upperBound(nums, 0))  // 0
}
```

---

## Example 2: Last Occurrence of Target

```go
package main

import "fmt"

func lastOccurrence(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	// lo = upper bound = first > target
	// lo-1 = last <= target
	if lo > 0 && nums[lo-1] == target {
		return lo - 1
	}
	return -1
}

func main() {
	nums := []int{1, 2, 3, 3, 3, 4, 5}
	fmt.Println(lastOccurrence(nums, 3)) // 4
	fmt.Println(lastOccurrence(nums, 6)) // -1
	fmt.Println(lastOccurrence(nums, 5)) // 6
}
```

---

## Example 3: Count of Target (LeetCode 34 Related)

```go
package main

import "fmt"

func lowerBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func upperBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func countTarget(nums []int, target int) int {
	return upperBound(nums, target) - lowerBound(nums, target)
}

func main() {
	nums := []int{1, 2, 2, 3, 3, 3, 3, 4, 5}
	fmt.Println("Count of 3:", countTarget(nums, 3)) // 4
	fmt.Println("Count of 2:", countTarget(nums, 2)) // 2
	fmt.Println("Count of 6:", countTarget(nums, 6)) // 0
}
```

---

## Example 4: First and Last Position (LeetCode 34)

```go
package main

import "fmt"

func searchRange(nums []int, target int) [2]int {
	first := lowerBound34(nums, target)
	if first == len(nums) || nums[first] != target {
		return [2]int{-1, -1}
	}
	last := upperBound34(nums, target) - 1
	return [2]int{first, last}
}

func lowerBound34(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func upperBound34(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func main() {
	fmt.Println(searchRange([]int{5, 7, 7, 8, 8, 10}, 8))  // [3, 4]
	fmt.Println(searchRange([]int{5, 7, 7, 8, 8, 10}, 6))  // [-1, -1]
	fmt.Println(searchRange([]int{}, 0))                      // [-1, -1]
}
```

---

## Example 5: Upper Bound with sort.SearchInts

```go
package main

import (
	"fmt"
	"sort"
)

func upperBound(nums []int, target int) int {
	// sort.SearchInts finds first >= target
	// For upper bound, search for target+1
	return sort.SearchInts(nums, target+1)
}

func main() {
	nums := []int{1, 2, 4, 4, 4, 7, 9}
	fmt.Println("UB(4):", upperBound(nums, 4)) // 5
	fmt.Println("UB(5):", upperBound(nums, 5)) // 5
	fmt.Println("UB(7):", upperBound(nums, 7)) // 6
	fmt.Println("UB(9):", upperBound(nums, 9)) // 7
}
```

---

## Example 6: Insert Position to Keep Sorted Order

```go
package main

import "fmt"

// If duplicates, insert AFTER all existing copies
func insertAfter(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

// If duplicates, insert BEFORE all existing copies
func insertBefore(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func main() {
	nums := []int{1, 3, 3, 3, 5, 7}
	fmt.Println("Insert 3 after:", insertAfter(nums, 3))   // 4
	fmt.Println("Insert 3 before:", insertBefore(nums, 3))  // 1
}
```

---

## Example 7: Closest Element to Target

```go
package main

import "fmt"

func closestElement(nums []int, target int) int {
	if len(nums) == 0 {
		return -1
	}
	// Upper bound
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	// Candidates: lo-1 (floor) and lo (ceiling)
	if lo == 0 {
		return nums[0]
	}
	if lo == len(nums) {
		return nums[len(nums)-1]
	}
	if target-nums[lo-1] <= nums[lo]-target {
		return nums[lo-1]
	}
	return nums[lo]
}

func main() {
	nums := []int{1, 3, 5, 7, 9}
	fmt.Println("Closest to 6:", closestElement(nums, 6)) // 5 (tie-break left)
	fmt.Println("Closest to 4:", closestElement(nums, 4)) // 3
	fmt.Println("Closest to 9:", closestElement(nums, 9)) // 9
	fmt.Println("Closest to 0:", closestElement(nums, 0)) // 1
}
```

---

## Example 8: Count Elements Strictly Greater Than Target

```go
package main

import "fmt"

func countGreater(nums []int, target int) int {
	// upper bound = first index > target
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return len(nums) - lo
}

func main() {
	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	fmt.Println("Greater than 5:", countGreater(nums, 5))   // 5
	fmt.Println("Greater than 0:", countGreater(nums, 0))   // 10
	fmt.Println("Greater than 10:", countGreater(nums, 10)) // 0
}
```

---

## Example 9: Merge Intervals Using Upper Bound

```go
package main

import (
	"fmt"
	"sort"
)

// Given sorted non-overlapping intervals, find which ones overlap with [lo,hi]
func overlappingIntervals(intervals [][2]int, lo, hi int) [][2]int {
	// Find first interval whose start > hi → upper bound
	right := sort.Search(len(intervals), func(i int) bool {
		return intervals[i][0] > hi
	})
	// Find first interval whose end >= lo → lower bound
	left := sort.Search(len(intervals), func(i int) bool {
		return intervals[i][1] >= lo
	})

	if left >= right {
		return nil
	}
	return intervals[left:right]
}

func main() {
	intervals := [][2]int{{1, 3}, {5, 7}, {9, 11}, {13, 15}}
	result := overlappingIntervals(intervals, 4, 10)
	fmt.Println(result) // [[5 7] [9 11]]
}
```

---

## Example 10: Lower & Upper Bound Combined — Remove Duplicates Count

```go
package main

import "fmt"

func lowerBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func upperBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func countDistinct(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	count := 0
	i := 0
	for i < len(nums) {
		count++
		// Jump to upper bound of current value
		i = upperBound(nums, nums[i])
	}
	return count
}

func main() {
	nums := []int{1, 1, 2, 2, 2, 3, 4, 4, 5}
	fmt.Println("Distinct count:", countDistinct(nums)) // 5
	fmt.Println("LB(2):", lowerBound(nums, 2))  // 2
	fmt.Println("UB(2):", upperBound(nums, 2))  // 5
	fmt.Println("Count 2:", upperBound(nums, 2)-lowerBound(nums, 2)) // 3
}
```

---

## Lower Bound vs Upper Bound Summary

| Aspect | Lower Bound | Upper Bound |
|--------|-------------|-------------|
| Returns | First `i` where `a[i] >= target` | First `i` where `a[i] > target` |
| Condition | `a[mid] < target → lo = mid+1` | `a[mid] <= target → lo = mid+1` |
| First occurrence | `lowerBound(target)` | — |
| Last occurrence | — | `upperBound(target) - 1` |
| Count of target | `upperBound - lowerBound` | — |
| Go stdlib | `sort.SearchInts(a, target)` | `sort.SearchInts(a, target+1)` |

## Key Takeaways

1. Upper bound = first index where `arr[i] > target`
2. The only difference from lower bound: `<=` instead of `<`
3. `upperBound(target) - 1` = last occurrence
4. `upperBound(target) - lowerBound(target)` = exact count
5. Go's `sort.SearchInts(a, target+1)` acts as upper bound

> **Next up:** Search on Answer →
