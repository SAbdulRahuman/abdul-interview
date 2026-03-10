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

**Textual Figure: lowerBound([1, 2, 4, 4, 4, 7, 9], target=4)**

```
Index:   0   1   2   3   4   5   6
       ┌───┬───┬───┬───┬───┬───┬───┐
Array: │ 1 │ 2 │ 4 │ 4 │ 4 │ 7 │ 9 │
       └───┴───┴───┴───┴───┴───┴───┘

Iter 1: lo=0                hi=7   mid=3
        nums[3]=4 ≥ 4 → hi=3
Iter 2: lo=0      hi=3      mid=1
        nums[1]=2 < 4 → lo=2
Iter 3: lo=2  hi=3  mid=2
        nums[2]=4 ≥ 4 → hi=2
        lo == hi == 2 → return 2 ✓

       ┌───┬───┬───┬───┬───┬───┬───┐
       │ 1 │ 2 │►4 │ 4 │ 4 │ 7 │ 9 │  first ≥ 4
       └───┴───┴───┴───┴───┴───┴───┘
            ↑
            LB = 2
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

**Textual Figure: sort.Search lower bound on [1, 3, 3, 5, 5, 5, 8, 10]**

```
Index:   0   1   2   3   4   5   6    7
       ┌───┬───┬───┬───┬───┬───┬───┬────┐
Array: │ 1 │ 3 │ 3 │ 5 │ 5 │ 5 │ 8 │ 10 │
       └───┴───┴───┴───┴───┴───┴───┴────┘

LB(0) =0  LB(1) =0  LB(3) =1  LB(4) =3
LB(5) =3  LB(6) =6  LB(8) =6  LB(10)=7  LB(11)=8(past)

Example trace for LB(5):
  lo=0 hi=8 mid=4 → nums[4]=5 ≥ 5 → hi=4
  lo=0 hi=4 mid=2 → nums[2]=3 < 5 → lo=3
  lo=3 hi=4 mid=3 → nums[3]=5 ≥ 5 → hi=3
  → index=3  val=5 ✓

       │ 1 │ 3 │ 3 │►5 │ 5 │ 5 │ 8 │ 10 │
                    ↑
                    first ≥ 5
```

---

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

**Textual Figure: firstOccurrence([1,2,2,3,3,3,4,5], target=3)**

```
Index:   0   1   2   3   4   5   6   7
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
Array: │ 1 │ 2 │ 2 │ 3 │ 3 │ 3 │ 4 │ 5 │
       └───┴───┴───┴───┴───┴───┴───┴───┘

Iter 1: lo=0                    hi=8   mid=4
        nums[4]=3 ≥ 3 → hi=4
Iter 2: lo=0         hi=4       mid=2
        nums[2]=2 < 3 → lo=3
Iter 3: lo=3  hi=4  mid=3
        nums[3]=3 ≥ 3 → hi=3
        lo == hi == 3 → nums[3]=3 == target → return 3 ✓

       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 1 │ 2 │ 2 │►3 │ 3 │ 3 │ 4 │ 5 │  first occurrence
       └───┴───┴───┴───┴───┴───┴───┴───┘
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

**Textual Figure: countLessThan([1..10], target=5)**

```
Index:  0   1   2   3   4   5   6   7   8   9
      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬────┐
      │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │ 10 │
      └───┴───┴───┴───┴───┴───┴───┴───┴───┴────┘
      |< 5 elements < 5 >|  ↑
                            LB = 4

Iter 1: lo=0 hi=10 mid=5 → nums[5]=6 ≥ 5 → hi=5
Iter 2: lo=0 hi=5  mid=2 → nums[2]=3 < 5 → lo=3
Iter 3: lo=3 hi=5  mid=4 → nums[4]=5 ≥ 5 → hi=4
Iter 4: lo=3 hi=4  mid=3 → nums[3]=4 < 5 → lo=4
        lo == hi == 4 → return 4 (count of elements < 5) ✓
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

**Textual Figure: Lower bound on strings**

```
Index:     0        1        2       3      4       5
       ┌───────┬────────┬────────┬──────┬─────┬───────┐
Array: │ apple │ banana │ cherry │ date │ fig │ grape │
       └───────┴────────┴────────┴──────┴─────┴───────┘

LB("cherry")  → idx=2 (exact match)
LB("coconut") → idx=3 ("date", first ≥ "coconut")
LB("a")       → idx=0 ("apple", first ≥ "a")
LB("z")       → idx=6 (past end, no word ≥ "z")

Trace for "coconut":
  lo=0 hi=6 mid=3 → "date" ≥ "coconut" → hi=3
  lo=0 hi=3 mid=1 → "banana" < "coconut" → lo=2
  lo=2 hi=3 mid=2 → "cherry" < "coconut" → lo=3
  → index=3 ("date") ✓
```

---

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

**Textual Figure: Lower bound on custom struct (Events, target time=200)**

```
Index:    0          1          2          3          4
       ┌────────┬────────┬────────┬────────┬────────┐
Time:  │  100   │  200   │  200   │  300   │  400   │
Name:  │ login  │ click  │ scroll │ submit │ logout │
       └────────┴────────┴────────┴────────┴────────┘

Search: first event where Time ≥ 200
  lo=0 hi=5 mid=2 → events[2].Time=200 ≥ 200 → hi=2
  lo=0 hi=2 mid=1 → events[1].Time=200 ≥ 200 → hi=1
  lo=0 hi=1 mid=0 → events[0].Time=100 < 200 → lo=1
  → index=1  ✓

  Events from index 1 onward:
        idx=1: t=200 click
        idx=2: t=200 scroll     ← all ≥ 200
        idx=3: t=300 submit
        idx=4: t=400 logout
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

**Textual Figure: floor([1, 3, 5, 7, 9, 11], target=6)**

```
Index:   0   1   2   3   4    5
       ┌───┬───┬───┬───┬───┬────┐
Array: │ 1 │ 3 │ 5 │ 7 │ 9 │ 11 │
       └───┴───┴───┴───┴───┴────┘

Upper bound (first > 6):
  lo=0 hi=6 mid=3 → nums[3]=7 > 6 → hi=3
  lo=0 hi=3 mid=1 → nums[1]=3 ≤ 6 → lo=2
  lo=2 hi=3 mid=2 → nums[2]=5 ≤ 6 → lo=3
  → upper bound = 3
  → floor = nums[3-1] = nums[2] = 5 ✓

       ┌───┬───┬───┬───┬───┬────┐
       │ 1 │ 3 │►5 │ 7 │ 9 │ 11 │  floor(6) = 5
       └───┴───┴───┴───┴───┴────┘
               ↑   ↑
            UB-1  UB=3
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

**Textual Figure: rank([1,2,2,3,3,3,4,5], target=3)**

```
Index:   0   1   2   3   4   5   6   7
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
Array: │ 1 │ 2 │ 2 │ 3 │ 3 │ 3 │ 4 │ 5 │
       └───┴───┴───┴───┴───┴───┴───┴───┘
       |───── 6 elements ≤ 3 ─────|

Upper bound (first > 3):
  lo=0 hi=8 mid=4 → nums[4]=3 ≤ 3 → lo=5
  lo=5 hi=8 mid=6 → nums[6]=4 > 3 → hi=6
  lo=5 hi=6 mid=5 → nums[5]=3 ≤ 3 → lo=6
  → upper bound = 6 = rank(3) = 6 ✓
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

**Textual Figure: hIndex([0, 1, 3, 5, 6])**

```
Index:     0    1    2    3    4
         ┌────┬────┬────┬────┬────┐
Cites:   │  0 │  1 │  3 │  5 │  6 │   n=5
         └────┴────┴────┴────┴────┘
n-mid:     5    4    3    2    1

Condition: citations[mid] ≥ n-mid
  mid=0: 0 ≥ 5? No    mid=1: 1 ≥ 4? No
  mid=2: 3 ≥ 3? Yes   mid=3: 5 ≥ 2? Yes

Binary search for first mid where cites[mid] ≥ n-mid:
  lo=0 hi=5 mid=2 → 3 ≥ 3 → hi=2
  lo=0 hi=2 mid=1 → 1 < 4 → lo=2
  → lo=2, h-index = n - lo = 5 - 2 = 3 ✓

         ┌────┬────┬────┬────┬────┐
         │  0 │  1 │ ►3 │  5 │  6 │  3 papers with ≥ 3 cites
         └────┴────┴────┴────┴────┘
                       ↑
                       lo=2  h=n-lo=3
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

**Textual Figure: rangeCount([1..10], lo=3, hi=7)**

```
Index:  0   1   2   3   4   5   6   7   8   9
      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬────┐
      │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │ 10 │
      └───┴───┴───┴───┴───┴───┴───┴───┴───┴────┘
              ↑               ↑
          left=LB(3)=2   right=LB(8)=7

  lowerBound(3) → first ≥ 3 → index 2
  lowerBound(8) → first ≥ 8 → index 7   (= lowerBound(hi+1))

      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬────┐
      │ 1 │ 2 │[3 │ 4 │ 5 │ 6 │ 7]│ 8 │ 9 │ 10 │
      └───┴───┴───┴───┴───┴───┴───┴───┴───┴────┘
       count = 7 - 2 = 5 ✓
```

---

## Key Takeaways

1. Lower bound = first index where `arr[i] >= target` — the most versatile binary search
2. Template: `lo, hi = 0, n` with `lo < hi`, set `hi = mid` when condition holds
3. `sort.Search` in Go is exactly a lower-bound search
4. Lower bound - 1 gives the **floor** (largest ≤ target)
5. `lowerBound(target+1) - lowerBound(target)` = count of exact matches

> **Next up:** Upper Bound →
