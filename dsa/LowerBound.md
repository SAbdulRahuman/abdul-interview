# Phase 10: Binary Search — Lower Bound

## Overview

**Lower bound** finds the **first position** where `arr[i] >= target`. It's the most fundamental binary search variant.

```
sorted:  [1, 2, 4, 4, 4, 7, 9]
lower_bound(4) → index 2 (first 4)
lower_bound(5) → index 5 (first value ≥ 5, which is 7)
```

Template: `lo < hi`, shrink by setting `hi = mid` when condition met.

---

## Example 1: Basic Lower Bound

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
	return lo // first index where nums[i] >= target
}

func main() {
	nums := []int{1, 2, 4, 4, 4, 7, 9}
	fmt.Println(lowerBound(nums, 4)) // 2
	fmt.Println(lowerBound(nums, 5)) // 5
	fmt.Println(lowerBound(nums, 1)) // 0
	fmt.Println(lowerBound(nums, 10)) // 7 (past end)
}
```

---

## Example 2: Lower Bound Using sort.Search

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{1, 3, 3, 5, 5, 5, 8, 10}

	tests := []int{0, 1, 3, 4, 5, 6, 8, 10, 11}
	for _, t := range tests {
		idx := sort.Search(len(nums), func(i int) bool {
			return nums[i] >= t
		})
		if idx < len(nums) {
			fmt.Printf("LB(%2d) = index %d (val=%d)\n", t, idx, nums[idx])
		} else {
			fmt.Printf("LB(%2d) = index %d (past end)\n", t, idx)
		}
	}
}
```

---

## Example 3: First Occurrence of Target

```go
package main

import "fmt"

func firstOccurrence(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	if lo < len(nums) && nums[lo] == target {
		return lo
	}
	return -1
}

func main() {
	nums := []int{1, 2, 2, 3, 3, 3, 4, 5}
	fmt.Println(firstOccurrence(nums, 3)) // 3
	fmt.Println(firstOccurrence(nums, 6)) // -1
	fmt.Println(firstOccurrence(nums, 1)) // 0
}
```

---

## Example 4: Count Elements Less Than Target

```go
package main

import "fmt"

func countLessThan(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo // lower bound index = count of elements < target
}

func main() {
	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	fmt.Println(countLessThan(nums, 5))  // 4
	fmt.Println(countLessThan(nums, 1))  // 0
	fmt.Println(countLessThan(nums, 11)) // 10
}
```

---

## Example 5: Lower Bound on Strings

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	words := []string{"apple", "banana", "cherry", "date", "fig", "grape"}
	// Already sorted

	targets := []string{"cherry", "coconut", "a", "z"}
	for _, t := range targets {
		idx := sort.Search(len(words), func(i int) bool {
			return words[i] >= t
		})
		if idx < len(words) {
			fmt.Printf("LB(%q) = index %d (%q)\n", t, idx, words[idx])
		} else {
			fmt.Printf("LB(%q) = index %d (past end)\n", t, idx)
		}
	}
}
```

---

## Example 6: Lower Bound on Custom Struct

```go
package main

import (
	"fmt"
	"sort"
)

type Event struct {
	Time int
	Name string
}

func main() {
	events := []Event{
		{100, "login"}, {200, "click"}, {200, "scroll"},
		{300, "submit"}, {400, "logout"},
	}

	// Find first event at or after time 200
	idx := sort.Search(len(events), func(i int) bool {
		return events[i].Time >= 200
	})
	fmt.Println("Events from time 200:")
	for i := idx; i < len(events); i++ {
		fmt.Printf("  t=%d %s\n", events[i].Time, events[i].Name)
	}
}
```

---

## Example 7: Floor Value (Largest Element ≤ Target)

```go
package main

import "fmt"

func floor(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	// lo is upper bound (first > target)
	// lo-1 is floor (last <= target)
	if lo == 0 { return -1 }
	return nums[lo-1]
}

func main() {
	nums := []int{1, 3, 5, 7, 9, 11}
	fmt.Println("Floor(6):", floor(nums, 6))   // 5
	fmt.Println("Floor(7):", floor(nums, 7))   // 7
	fmt.Println("Floor(0):", floor(nums, 0))   // -1
	fmt.Println("Floor(12):", floor(nums, 12)) // 11
}
```

---

## Example 8: Rank Query — How Many Elements ≤ Target

```go
package main

import "fmt"

func rank(nums []int, target int) int {
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
	return lo // count of elements <= target
}

func main() {
	nums := []int{1, 2, 2, 3, 3, 3, 4, 5}
	fmt.Println("Rank(3):", rank(nums, 3)) // 6 (six elements ≤ 3)
	fmt.Println("Rank(0):", rank(nums, 0)) // 0
	fmt.Println("Rank(5):", rank(nums, 5)) // 8
}
```

---

## Example 9: Lower Bound to Solve "H-Index" (LeetCode 275)

```go
package main

import "fmt"

// Given sorted citations, find h-index
func hIndex(citations []int) int {
	n := len(citations)
	lo, hi := 0, n
	for lo < hi {
		mid := lo + (hi-lo)/2
		if citations[mid] >= n-mid {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return n - lo
}

func main() {
	fmt.Println(hIndex([]int{0, 1, 3, 5, 6})) // 3
	fmt.Println(hIndex([]int{1, 2, 100}))       // 2
}
```

---

## Example 10: Lower Bound for Range Count [lo, hi]

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

func rangeCount(nums []int, lo, hi int) int {
	left := lowerBound(nums, lo)      // first >= lo
	right := lowerBound(nums, hi+1)   // first > hi
	return right - left
}

func main() {
	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	fmt.Println("Count in [3,7]:", rangeCount(nums, 3, 7))   // 5
	fmt.Println("Count in [5,5]:", rangeCount(nums, 5, 5))   // 1
	fmt.Println("Count in [11,20]:", rangeCount(nums, 11, 20)) // 0

	nums2 := []int{1, 1, 2, 2, 2, 3, 3, 4}
	fmt.Println("Count of 2s:", rangeCount(nums2, 2, 2)) // 3
}
```

---

## Key Takeaways

1. Lower bound = first index where `arr[i] >= target` — the most versatile binary search
2. Template: `lo, hi = 0, n` with `lo < hi`, set `hi = mid` when condition holds
3. `sort.Search` in Go is exactly a lower-bound search
4. Lower bound - 1 gives the **floor** (largest ≤ target)
5. `lowerBound(target+1) - lowerBound(target)` = count of exact matches

> **Next up:** Upper Bound →
