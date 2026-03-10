# Phase 10: Binary Search — Binary Search Basics

## Overview

Binary search finds a target in a **sorted** collection in O(log n) by repeatedly halving the search space.

| Variant | Returns |
|---------|---------|
| Exact match | Index of target, or -1 |
| Lower bound | First index where `arr[i] >= target` |
| Upper bound | First index where `arr[i] > target` |

---

## Example 1: Classic Binary Search (LeetCode 704)

```go
package main

import "fmt"

func search(nums []int, target int) int {
	lo, hi := 0, len(nums)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2 // avoid overflow
		if nums[mid] == target {
			return mid
		} else if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return -1
}

func main() {
	nums := []int{-1, 0, 3, 5, 9, 12}
	fmt.Println(search(nums, 9))  // 4
	fmt.Println(search(nums, 2))  // -1
}
```

**Textual Figure: search([-1, 0, 3, 5, 9, 12], target=9)**

```
Index:    0    1    2    3    4    5
        ┌────┬────┬────┬────┬────┬────┐
Array:  │ -1 │  0 │  3 │  5 │  9 │ 12 │
        └────┴────┴────┴────┴────┴────┘

Iter 1: lo=0                      hi=5   mid=2
        [-1,  0, [3],  5,  9,  12]
         lo       mid              hi
        nums[2]=3 < 9 → lo = mid+1 = 3

Iter 2:               lo=3        hi=5   mid=4
        [-1,  0,  3,  5, [9],  12]
                       lo  mid     hi
        nums[4]=9 == 9 → return 4 ✓

        ┌────┬────┬────┬────┬────┬────┐
        │ -1 │  0 │  3 │  5 │ ►9 │ 12 │  Found at index 4
        └────┴────┴────┴────┴────┴────┘
```

---

## Example 2: Search Insert Position (LeetCode 35)

```go
package main

import "fmt"

func searchInsert(nums []int, target int) int {
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
	nums := []int{1, 3, 5, 6}
	fmt.Println(searchInsert(nums, 5)) // 2
	fmt.Println(searchInsert(nums, 2)) // 1
	fmt.Println(searchInsert(nums, 7)) // 4
	fmt.Println(searchInsert(nums, 0)) // 0
}
```

**Textual Figure: searchInsert([1, 3, 5, 6], target=5)**

```
Index:   0    1    2    3    (4)
       ┌────┬────┬────┬────┐
Array: │  1 │  3 │  5 │  6 │
       └────┴────┴────┴────┘

Iter 1: lo=0            hi=4   mid=2
        [ 1,  3, [5],  6]
         lo      mid        hi
        nums[2]=5 ≥ 5 → hi = mid = 2

Iter 2: lo=0   hi=2   mid=1
        [ 1, [3],  5,  6]
         lo  mid  hi
        nums[1]=3 < 5 → lo = mid+1 = 2

        lo == hi == 2 → return 2 ✓

       ┌────┬────┬────┬────┐
       │  1 │  3 │ ►5 │  6 │  Insert pos = 2
       └────┴────┴────┴────┘
```

---

## Example 3: Recursive Binary Search

```go
package main

import "fmt"

func binarySearchRecursive(nums []int, target, lo, hi int) int {
	if lo > hi { return -1 }
	mid := lo + (hi-lo)/2
	if nums[mid] == target { return mid }
	if nums[mid] < target {
		return binarySearchRecursive(nums, target, mid+1, hi)
	}
	return binarySearchRecursive(nums, target, lo, mid-1)
}

func main() {
	nums := []int{1, 3, 5, 7, 9, 11, 13, 15}
	fmt.Println(binarySearchRecursive(nums, 7, 0, len(nums)-1))  // 3
	fmt.Println(binarySearchRecursive(nums, 6, 0, len(nums)-1))  // -1
}
```

**Textual Figure: binarySearchRecursive([1,3,5,7,9,11,13,15], target=7)**

```
Index:   0   1   2   3   4    5    6    7
       ┌───┬───┬───┬───┬───┬────┬────┬────┐
Array: │ 1 │ 3 │ 5 │ 7 │ 9 │ 11 │ 13 │ 15 │
       └───┴───┴───┴───┴───┴────┴────┴────┘

Call 1: lo=0                            hi=7   mid=3
        [ 1,  3,  5, [7],  9, 11, 13, 15]
         lo           mid                hi
        nums[3]=7 == 7 → return 3 ✓

       ┌───┬───┬───┬───┬───┬────┬────┬────┐
       │ 1 │ 3 │ 5 │►7 │ 9 │ 11 │ 13 │ 15 │  Found at index 3
       └───┴───┴───┴───┴───┴────┴────┴────┘

For target=6 (not found):
  Call 1: lo=0 hi=7 mid=3 → nums[3]=7 > 6 → go left
  Call 2: lo=0 hi=2 mid=1 → nums[1]=3 < 6 → go right
  Call 3: lo=2 hi=2 mid=2 → nums[2]=5 < 6 → go right
  Call 4: lo=3 hi=2 → lo > hi → return -1
```

---

## Example 4: First and Last Position (LeetCode 34)

```go
package main

import "fmt"

func searchRange(nums []int, target int) [2]int {
	return [2]int{findFirst(nums, target), findLast(nums, target)}
}

func findFirst(nums []int, target int) int {
	lo, hi := 0, len(nums)-1
	result := -1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if nums[mid] == target {
			result = mid
			hi = mid - 1 // keep searching left
		} else if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return result
}

func findLast(nums []int, target int) int {
	lo, hi := 0, len(nums)-1
	result := -1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if nums[mid] == target {
			result = mid
			lo = mid + 1 // keep searching right
		} else if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return result
}

func main() {
	nums := []int{5, 7, 7, 8, 8, 10}
	fmt.Println(searchRange(nums, 8)) // [3 4]
	fmt.Println(searchRange(nums, 6)) // [-1 -1]
}
```

**Textual Figure: searchRange([5, 7, 7, 8, 8, 10], target=8)**

```
Index:   0   1   2   3   4    5
       ┌───┬───┬───┬───┬───┬────┐
Array: │ 5 │ 7 │ 7 │ 8 │ 8 │ 10 │
       └───┴───┴───┴───┴───┴────┘

findFirst(8):
  Iter 1: lo=0  hi=5  mid=2 → nums[2]=7  < 8  → lo=3
  Iter 2: lo=3  hi=5  mid=4 → nums[4]=8 == 8  → result=4, hi=3
  Iter 3: lo=3  hi=3  mid=3 → nums[3]=8 == 8  → result=3, hi=2
  lo > hi → return result = 3

findLast(8):
  Iter 1: lo=0  hi=5  mid=2 → nums[2]=7  < 8  → lo=3
  Iter 2: lo=3  hi=5  mid=4 → nums[4]=8 == 8  → result=4, lo=5
  lo > hi → return result = 4

       ┌───┬───┬───┬───┬───┬────┐
       │ 5 │ 7 │ 7 │►8 │►8 │ 10 │
       └───┴───┴───┴───┴───┴────┘
                    ↑first ↑last

Result: [3, 4]
```

---

## Example 5: Search a 2D Matrix (LeetCode 74)

```go
package main

import "fmt"

func searchMatrix(matrix [][]int, target int) bool {
	m, n := len(matrix), len(matrix[0])
	lo, hi := 0, m*n-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		val := matrix[mid/n][mid%n]
		if val == target {
			return true
		} else if val < target {
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return false
}

func main() {
	matrix := [][]int{
		{1, 3, 5, 7},
		{10, 11, 16, 20},
		{23, 30, 34, 60},
	}
	fmt.Println(searchMatrix(matrix, 3))  // true
	fmt.Println(searchMatrix(matrix, 13)) // false
}
```

**Textual Figure: searchMatrix(matrix, target=3)**

```
Matrix (rows sorted, each row starts > previous row's end):
         col:  0    1    2    3
        ┌────┬────┬────┬────┐
row 0   │  1 │  3 │  5 │  7 │  flat: 0─3
        ├────┼────┼────┼────┤
row 1   │ 10 │ 11 │ 16 │ 20 │  flat: 4─7
        ├────┼────┼────┼────┤
row 2   │ 23 │ 30 │ 34 │ 60 │  flat: 8─11
        └────┴────┴────┴────┘
        n=4  →  row = mid/4, col = mid%4

Iter 1: lo=0   hi=11  mid=5  → mat[1][1]=11 > 3  → hi=4
Iter 2: lo=0   hi=4   mid=2  → mat[0][2]=5  > 3  → hi=1
Iter 3: lo=0   hi=1   mid=0  → mat[0][0]=1  < 3  → lo=1
Iter 4: lo=1   hi=1   mid=1  → mat[0][1]=3 == 3  → return true ✓

        ┌────┬────┬────┬────┐
row 0   │  1 │ ►3 │  5 │  7 │  ← Found at [0][1]
        ├────┼────┼────┼────┤
row 1   │ 10 │ 11 │ 16 │ 20 │
        ├────┼────┼────┼────┤
row 2   │ 23 │ 30 │ 34 │ 60 │
        └────┴────┴────┴────┘
```

---

## Example 6: Search in Rotated Sorted Array (LeetCode 33)

```go
package main

import "fmt"

func search(nums []int, target int) int {
	lo, hi := 0, len(nums)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if nums[mid] == target { return mid }

		if nums[lo] <= nums[mid] {
			// Left half is sorted
			if nums[lo] <= target && target < nums[mid] {
				hi = mid - 1
			} else {
				lo = mid + 1
			}
		} else {
			// Right half is sorted
			if nums[mid] < target && target <= nums[hi] {
				lo = mid + 1
			} else {
				hi = mid - 1
			}
		}
	}
	return -1
}

func main() {
	fmt.Println(search([]int{4, 5, 6, 7, 0, 1, 2}, 0)) // 4
	fmt.Println(search([]int{4, 5, 6, 7, 0, 1, 2}, 3)) // -1
	fmt.Println(search([]int{1}, 0))                      // -1
}
```

**Textual Figure: search([4,5,6,7,0,1,2], target=0) — Rotated Array**

```
Index:   0   1   2   3   4   5   6
       ┌───┬───┬───┬───┬───┬───┬───┐
Array: │ 4 │ 5 │ 6 │ 7 │ 0 │ 1 │ 2 │  (rotated at index 4)
       └───┴───┴───┴───┴───┴───┴───┘

Iter 1: lo=0               hi=6   mid=3
        [4, 5, 6, [7], 0, 1, 2]
        lo         mid         hi
        nums[0]=4 ≤ nums[3]=7 → left half [4..7] sorted
        target=0 not in [4,7) → lo = 4

Iter 2: lo=4        hi=6   mid=5
        [4, 5, 6, 7, 0, [1], 2]
                      lo  mid  hi
        nums[4]=0 ≤ nums[5]=1 → left half [0..1] sorted
        target=0 in [0,1) → hi = 4

Iter 3: lo=4  hi=4  mid=4
        nums[4]=0 == 0 → return 4 ✓

       ┌───┬───┬───┬───┬───┬───┬───┐
       │ 4 │ 5 │ 6 │ 7 │►0 │ 1 │ 2 │  Found at index 4
       └───┴───┴───┴───┴───┴───┴───┘
```

---

## Example 7: Find Peak Element (LeetCode 162)

```go
package main

import "fmt"

func findPeakElement(nums []int) int {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] > nums[mid+1] {
			hi = mid // peak is at mid or left
		} else {
			lo = mid + 1 // peak is to the right
		}
	}
	return lo
}

func main() {
	fmt.Println(findPeakElement([]int{1, 2, 3, 1}))    // 2
	fmt.Println(findPeakElement([]int{1, 2, 1, 3, 5, 6, 4})) // 5 (or 1)
}
```

**Textual Figure: findPeakElement([1, 2, 3, 1])**

```
Index:   0   1   2   3
       ┌───┬───┬───┬───┐
Array: │ 1 │ 2 │ 3 │ 1 │
       └───┴───┴───┴───┘
            ╱     ╲         3 is the peak
           ╱       ╲
          2         1
         ╱
        1

Iter 1: lo=0         hi=3   mid=1
        nums[1]=2 < nums[2]=3 → peak is right → lo=2

Iter 2: lo=2  hi=3   mid=2
        nums[2]=3 > nums[3]=1 → peak at mid or left → hi=2

        lo == hi == 2 → return 2 ✓

       ┌───┬───┬───┬───┐
       │ 1 │ 2 │►3 │ 1 │  Peak at index 2
       └───┴───┴───┴───┘
```

---

## Example 8: Find Minimum in Rotated Sorted Array (LeetCode 153)

```go
package main

import "fmt"

func findMin(nums []int) int {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] > nums[hi] {
			lo = mid + 1 // min in right half
		} else {
			hi = mid // min at mid or left
		}
	}
	return nums[lo]
}

func main() {
	fmt.Println(findMin([]int{3, 4, 5, 1, 2}))    // 1
	fmt.Println(findMin([]int{4, 5, 6, 7, 0, 1, 2})) // 0
	fmt.Println(findMin([]int{11, 13, 15, 17}))    // 11
}
```

**Textual Figure: findMin([3, 4, 5, 1, 2])**

```
Index:   0   1   2   3   4
       ┌───┬───┬───┬───┬───┐
Array: │ 3 │ 4 │ 5 │ 1 │ 2 │  (rotated, min at index 3)
       └───┴───┴───┴───┴───┘

        5
       ╱ ╲          Rotation point
      4   ╲
     ╱     2
    3     ╱
         1  ← minimum

Iter 1: lo=0         hi=4   mid=2
        nums[2]=5 > nums[4]=2 → min in right half → lo=3

Iter 2: lo=3    hi=4   mid=3
        nums[3]=1 < nums[4]=2 → min at mid or left → hi=3

        lo == hi == 3 → return nums[3] = 1 ✓

       ┌───┬───┬───┬───┬───┐
       │ 3 │ 4 │ 5 │►1 │ 2 │  Min at index 3
       └───┴───┴───┴───┴───┘
```

---

## Example 9: Count Occurrences in Sorted Array

```go
package main

import "fmt"

func countOccurrences(nums []int, target int) int {
	first := lowerBound(nums, target)
	if first == len(nums) || nums[first] != target {
		return 0
	}
	last := lowerBound(nums, target+1)
	return last - first
}

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

func main() {
	nums := []int{1, 2, 2, 2, 3, 4, 4, 5}
	fmt.Println("Count of 2:", countOccurrences(nums, 2)) // 3
	fmt.Println("Count of 4:", countOccurrences(nums, 4)) // 2
	fmt.Println("Count of 6:", countOccurrences(nums, 6)) // 0
}
```

**Textual Figure: countOccurrences([1,2,2,2,3,4,4,5], target=2)**

```
Index:   0   1   2   3   4   5   6   7
       ┌───┬───┬───┬───┬───┬───┬───┬───┐
Array: │ 1 │ 2 │ 2 │ 2 │ 3 │ 4 │ 4 │ 5 │
       └───┴───┴───┴───┴───┴───┴───┴───┘

lowerBound(target=2):  first index where nums[i] >= 2
  lo=0 hi=8 mid=4 → nums[4]=3 ≥ 2 → hi=4
  lo=0 hi=4 mid=2 → nums[2]=2 ≥ 2 → hi=2
  lo=0 hi=2 mid=1 → nums[1]=2 ≥ 2 → hi=1
  lo=0 hi=1 mid=0 → nums[0]=1 < 2 → lo=1
  → lowerBound = 1

lowerBound(target=3):  first index where nums[i] >= 3
  lo=0 hi=8 mid=4 → nums[4]=3 ≥ 3 → hi=4
  lo=0 hi=4 mid=2 → nums[2]=2 < 3 → lo=3
  lo=3 hi=4 mid=3 → nums[3]=2 < 3 → lo=4
  → lowerBound = 4

       ┌───┬───┬───┬───┬───┬───┬───┬───┐
       │ 1 │[2 │ 2 │ 2]│ 3 │ 4 │ 4 │ 5 │
       └───┴───┴───┴───┴───┴───┴───┴───┘
            ↑           ↑
            LB(2)=1     LB(3)=4

       count = 4 - 1 = 3 ✓
```

---

## Example 10: Go's sort.Search — Built-in Binary Search

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{1, 3, 5, 7, 9, 11, 13}

	// sort.Search finds the smallest index i where f(i) is true
	// Equivalent to lower bound
	idx := sort.Search(len(nums), func(i int) bool {
		return nums[i] >= 7
	})
	fmt.Printf("Lower bound of 7: index=%d val=%d\n", idx, nums[idx]) // 3, 7

	// Upper bound
	idx = sort.Search(len(nums), func(i int) bool {
		return nums[i] > 7
	})
	fmt.Printf("Upper bound of 7: index=%d\n", idx) // 4

	// Search for non-existent
	idx = sort.Search(len(nums), func(i int) bool {
		return nums[i] >= 6
	})
	fmt.Printf("Lower bound of 6: index=%d val=%d\n", idx, nums[idx]) // 3, 7

	// sort.SearchInts shortcut
	idx = sort.SearchInts(nums, 9)
	fmt.Printf("SearchInts(9): index=%d\n", idx) // 4
}
```

**Textual Figure: sort.Search on [1, 3, 5, 7, 9, 11, 13]**

```
Index:   0   1   2   3   4    5    6
       ┌───┬───┬───┬───┬───┬────┬────┐
Array: │ 1 │ 3 │ 5 │ 7 │ 9 │ 11 │ 13 │
       └───┴───┴───┴───┴───┴────┴────┘

Lower bound of 7 (first i where nums[i] >= 7):
  lo=0 hi=7 mid=3 → nums[3]=7 ≥ 7 → hi=3
  lo=0 hi=3 mid=1 → nums[1]=3 < 7 → lo=2
  lo=2 hi=3 mid=2 → nums[2]=5 < 7 → lo=3
  → index=3  val=7 ✓

Upper bound of 7 (first i where nums[i] > 7):
  lo=0 hi=7 mid=3 → nums[3]=7 ≤ 7 → lo=4
  lo=4 hi=7 mid=5 → nums[5]=11 > 7 → hi=5
  lo=4 hi=5 mid=4 → nums[4]=9  > 7 → hi=4
  → index=4 ✓

       ┌───┬───┬───┬───┬───┬────┬────┐
       │ 1 │ 3 │ 5 │ 7 │ 9 │ 11 │ 13 │
       └───┴───┴───┴───┴───┴────┴────┘
                    ↑   ↑
                    LB  UB
                    =3  =4
```

---

## Key Takeaways

1. **Always use `lo + (hi-lo)/2`** to avoid integer overflow
2. **`lo <= hi`** for exact match; **`lo < hi`** for boundary-finding
3. Binary search on rotated arrays: identify which half is sorted first
4. Go provides `sort.Search` — use it for lower/upper bound in practice
5. Binary search requires **sorted** data (or a monotonic predicate)

> **Next up:** Lower Bound →
