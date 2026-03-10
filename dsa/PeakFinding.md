# Peak Finding

Peak finding is a classic application of binary search where we locate an element that is greater than (or equal to) its neighbors. Instead of scanning linearly in O(n), binary search narrows the search space by comparing the middle element with its neighbors and moving toward the "uphill" side, achieving O(log n).

**Core Insight:** If `nums[mid] < nums[mid+1]`, then a peak must exist to the right (by a "mountain" argument). Conversely, if `nums[mid] < nums[mid-1]`, a peak exists to the left. This guarantees convergence.

---

## Example 1: Find Peak Element (LeetCode 162)

A peak element is strictly greater than its neighbors. `nums[-1] = nums[n] = -∞`. Return index of any peak.

```go
package main

import "fmt"

func findPeakElement(nums []int) int {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < nums[mid+1] {
			lo = mid + 1 // peak is to the right
		} else {
			hi = mid // peak is at mid or to the left
		}
	}
	return lo
}

func main() {
	fmt.Println(findPeakElement([]int{1, 2, 3, 1}))          // 2
	fmt.Println(findPeakElement([]int{1, 2, 1, 3, 5, 6, 4})) // 5
}
```

**Textual Figure: findPeakElement([1, 2, 3, 1])**

```
    Array: [1, 2, 3, 1]
    Index:  0  1  2  3

    Iteration 1:  lo=0, hi=3, mid=1
    ┌───┬───┬───┬───┐
    │ 1 │ 2 │ 3 │ 1 │
    └───┴───┴───┴───┘
      lo  mid      hi
           │
           ├─ nums[1]=2 < nums[2]=3 → peak is RIGHT
           └─ lo = mid+1 = 2

    Iteration 2:  lo=2, hi=3, mid=2
    ┌───┬───┬───┬───┐
    │ 1 │ 2 │ 3 │ 1 │
    └───┴───┴───┴───┘
              lo  hi
             mid
              │
              ├─ nums[2]=3 > nums[3]=1 → peak at mid or LEFT
              └─ hi = mid = 2

    lo == hi == 2  →  STOP

    Element comparison flow:
        nums[1]=2 ──→ nums[2]=3   (go right)
                          │
                          ↓
                   nums[2]=3 > nums[3]=1  (peak found!)

    Result: index 2, value 3
    ┌───┬───┬───┬───┐
    │ 1 │ 2 │►3◄│ 1 │   Peak at index 2
    └───┴───┴───┴───┘
```

---

## Example 2: Peak in Bitonic Array

A bitonic array first strictly increases then strictly decreases. Find the maximum (peak).

```go
package main

import "fmt"

func bitonicPeak(nums []int) int {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < nums[mid+1] {
			lo = mid + 1 // still ascending
		} else {
			hi = mid // descending or at peak
		}
	}
	return lo // index of peak
}

func main() {
	arr := []int{1, 3, 8, 12, 9, 5, 2}
	idx := bitonicPeak(arr)
	fmt.Printf("Peak at index %d, value %d\n", idx, arr[idx]) // Peak at index 3, value 12
}
```

**Textual Figure: bitonicPeak([1, 3, 8, 12, 9, 5, 2])**

```
    Bitonic array (increases then decreases):

              12
             ╱  ╲
            8    9
           ╱      ╲
          3        5
         ╱          ╲
        1             2
    idx: 0  1  2  3  4  5  6

    Iteration 1:  lo=0, hi=6, mid=3
    ┌───┬───┬───┬────┬───┬───┬───┐
    │ 1 │ 3 │ 8 │ 12 │ 9 │ 5 │ 2 │
    └───┴───┴───┴────┴───┴───┴───┘
      lo          mid            hi
                   │
                   ├─ nums[3]=12 > nums[4]=9 → at peak or descending
                   └─ hi = mid = 3

    Iteration 2:  lo=0, hi=3, mid=1
    ┌───┬───┬───┬────┬───┬───┬───┐
    │ 1 │ 3 │ 8 │ 12 │ 9 │ 5 │ 2 │
    └───┴───┴───┴────┴───┴───┴───┘
      lo  mid      hi
           │
           ├─ nums[1]=3 < nums[2]=8 → still ascending
           └─ lo = mid+1 = 2

    Iteration 3:  lo=2, hi=3, mid=2
    ┌───┬───┬───┬────┬───┬───┬───┐
    │ 1 │ 3 │ 8 │ 12 │ 9 │ 5 │ 2 │
    └───┴───┴───┴────┴───┴───┴───┘
              lo   hi
             mid
              │
              ├─ nums[2]=8 < nums[3]=12 → still ascending
              └─ lo = mid+1 = 3

    lo == hi == 3  →  STOP

    Result: index 3, value 12
    ┌───┬───┬───┬─────┬───┬───┬───┐
    │ 1 │ 3 │ 8 │►12◄ │ 9 │ 5 │ 2 │   Peak at index 3
    └───┴───┴───┴─────┴───┴───┴───┘
```

---

## Example 3: Find Minimum in Rotated Sorted Array (LeetCode 153)

A rotated sorted array has a single "valley" (minimum). This is the inverse of peak finding — we search for the dip.

```go
package main

import "fmt"

func findMin(nums []int) int {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] > nums[hi] {
			lo = mid + 1 // min is in right half
		} else {
			hi = mid // min is at mid or in left half
		}
	}
	return nums[lo]
}

func main() {
	fmt.Println(findMin([]int{3, 4, 5, 1, 2})) // 1
	fmt.Println(findMin([]int{4, 5, 6, 7, 0, 1, 2})) // 0
}
```

**Textual Figure: findMin([3, 4, 5, 1, 2])**

```
    Rotated sorted array — finding the valley (minimum):

          5
         ╱ ╲
        4    ╲      Rotation break
       ╱      ╲
      3        1
                ╲
                 2
    idx: 0  1  2  3  4

    Iteration 1:  lo=0, hi=4, mid=2
    ┌───┬───┬───┬───┬───┐
    │ 3 │ 4 │ 5 │ 1 │ 2 │
    └───┴───┴───┴───┴───┘
      lo      mid      hi
               │
               ├─ nums[2]=5 > nums[4]=2 → min is in RIGHT half
               └─ lo = mid+1 = 3

    Iteration 2:  lo=3, hi=4, mid=3
    ┌───┬───┬───┬───┬───┐
    │ 3 │ 4 │ 5 │ 1 │ 2 │
    └───┴───┴───┴───┴───┘
                  lo  hi
                 mid
                  │
                  ├─ nums[3]=1 ≤ nums[4]=2 → min at mid or LEFT
                  └─ hi = mid = 3

    lo == hi == 3  →  STOP

    Result: nums[3] = 1
    ┌───┬───┬───┬───┬───┐
    │ 3 │ 4 │ 5 │►1◄│ 2 │   Minimum at index 3
    └───┴───┴───┴───┴───┘
```

---

## Example 4: Find Peak in Mountain Array (LeetCode 852)

Given an array guaranteed to be a mountain (increases then decreases), return index of the mountain peak.

```go
package main

import "fmt"

func peakIndexInMountainArray(arr []int) int {
	lo, hi := 0, len(arr)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if arr[mid] < arr[mid+1] {
			lo = mid + 1 // ascending side
		} else {
			hi = mid // descending side or peak
		}
	}
	return lo
}

func main() {
	fmt.Println(peakIndexInMountainArray([]int{0, 2, 1, 0}))       // 1
	fmt.Println(peakIndexInMountainArray([]int{0, 10, 5, 2}))      // 1
	fmt.Println(peakIndexInMountainArray([]int{3, 5, 3, 2, 0}))    // 1
	fmt.Println(peakIndexInMountainArray([]int{0, 1, 2, 3, 4, 3, 1})) // 4
}
```

**Textual Figure: peakIndexInMountainArray([0, 1, 2, 3, 4, 3, 1])**

```
    Mountain shape:
                 4
                ╱ ╲
               3   3
              ╱     ╲
             2       ╲
            ╱         1
           1
          ╱
         0
    idx: 0  1  2  3  4  5  6

    Iteration 1:  lo=0, hi=6, mid=3
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 0 │ 1 │ 2 │ 3 │ 4 │ 3 │ 1 │
    └───┴───┴───┴───┴───┴───┴───┘
      lo          mid          hi
                   │
                   ├─ arr[3]=3 < arr[4]=4 → ascending side
                   └─ lo = mid+1 = 4

    Iteration 2:  lo=4, hi=6, mid=5
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 0 │ 1 │ 2 │ 3 │ 4 │ 3 │ 1 │
    └───┴───┴───┴───┴───┴───┴───┘
                      lo  mid  hi
                           │
                           ├─ arr[5]=3 > arr[6]=1 → descending side or peak
                           └─ hi = mid = 5

    Iteration 3:  lo=4, hi=5, mid=4
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 0 │ 1 │ 2 │ 3 │ 4 │ 3 │ 1 │
    └───┴───┴───┴───┴───┴───┴───┘
                      lo  hi
                     mid
                      │
                      ├─ arr[4]=4 > arr[5]=3 → descending side or peak
                      └─ hi = mid = 4

    lo == hi == 4  →  STOP

    Result: index 4, value 4
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 0 │ 1 │ 2 │ 3 │►4◄│ 3 │ 1 │   Peak at index 4
    └───┴───┴───┴───┴───┴───┴───┘
```

---

## Example 5: Find Peak Element with Plateau Handling

Variation where array may have equal adjacent elements. We find any local maximum.

```go
package main

import "fmt"

func findPeakWithPlateau(nums []int) int {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < nums[mid+1] {
			lo = mid + 1
		} else if nums[mid] > nums[mid+1] {
			hi = mid
		} else {
			// nums[mid] == nums[mid+1]: shrink from right
			hi--
		}
	}
	return lo
}

func main() {
	fmt.Println(findPeakWithPlateau([]int{1, 2, 2, 3, 5, 5, 3})) // 4 or 5
	fmt.Println(findPeakWithPlateau([]int{1, 1, 1, 2, 1}))        // 3
}
```

**Textual Figure: findPeakWithPlateau([1, 2, 2, 3, 5, 5, 3])**

```
    Array with plateaus (equal adjacent elements):
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 1 │ 2 │ 2 │ 3 │ 5 │ 5 │ 3 │
    └───┴───┴───┴───┴───┴───┴───┘
      0   1   2   3   4   5   6

    Iteration 1:  lo=0, hi=6, mid=3
      nums[3]=3 < nums[4]=5 → go RIGHT → lo=4

    Iteration 2:  lo=4, hi=6, mid=5
      nums[5]=5 > nums[6]=3 → peak at mid or LEFT → hi=5

    Iteration 3:  lo=4, hi=5, mid=4
      nums[4]=5 == nums[5]=5 → PLATEAU! → hi-- → hi=4

    lo == hi == 4  →  STOP

    Step-by-step pointer trace:
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 1 │ 2 │ 2 │ 3 │ 5 │ 5 │ 3 │  Step 1: mid=3, go right
    └───┴───┴───┴───┴───┴───┴───┘
      lo          mid↑         hi
                      ─────→
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 1 │ 2 │ 2 │ 3 │ 5 │ 5 │ 3 │  Step 2: mid=5, go left
    └───┴───┴───┴───┴───┴───┴───┘
                      lo mid↑ hi
                         ←───
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 1 │ 2 │ 2 │ 3 │ 5 │ 5 │ 3 │  Step 3: plateau, hi--
    └───┴───┴───┴───┴───┴───┴───┘
                     mid↑    hi←

    Result: index 4, value 5
    ┌───┬───┬───┬───┬───┬───┬───┐
    │ 1 │ 2 │ 2 │ 3 │►5◄│ 5 │ 3 │   Peak at index 4
    └───┴───┴───┴───┴───┴───┴───┘
```

---

## Key Takeaways

| Pattern | Condition to go right | Condition to go left | Time |
|---|---|---|---|
| Peak Element | `nums[mid] < nums[mid+1]` | `nums[mid] >= nums[mid+1]` | O(log n) |
| Bitonic Max | `nums[mid] < nums[mid+1]` | `nums[mid] >= nums[mid+1]` | O(log n) |
| Rotated Min | `nums[mid] > nums[hi]` | `nums[mid] <= nums[hi]` | O(log n) |
| Mountain Peak | `arr[mid] < arr[mid+1]` | `arr[mid] >= arr[mid+1]` | O(log n) |
| Plateau Peak | `nums[mid] < nums[mid+1]` | `nums[mid] > nums[mid+1]` | O(n) worst |

---
