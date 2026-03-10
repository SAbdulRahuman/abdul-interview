# Phase 16: Sorting Algorithms — In-Place Sorting

## Overview

An **in-place** sorting algorithm sorts using O(1) extra memory (or O(log n) for the call stack). It rearranges elements within the original array without allocating a proportional auxiliary array.

| In-Place (O(1) extra) | Not In-Place |
|------------------------|--------------|
| Bubble, Selection, Insertion | Merge Sort (standard) |
| Quick Sort (O(log n) stack) | Counting Sort |
| Heap Sort | Radix Sort |
| Shell Sort | Bucket Sort |

---

## Example 1: In-Place Swap-Based Sorting (Bubble Sort)

```go
package main

import "fmt"

func bubbleSort(arr []int) {
	n := len(arr)
	for i := 0; i < n-1; i++ {
		swapped := false
		for j := 0; j < n-1-i; j++ {
			if arr[j] > arr[j+1] {
				arr[j], arr[j+1] = arr[j+1], arr[j]
				swapped = true
			}
		}
		if !swapped { break }
	}
}

func main() {
	arr := []int{64, 34, 25, 12, 22, 11, 90}
	bubbleSort(arr) // O(1) extra space
	fmt.Println(arr) // [11 12 22 25 34 64 90]
}
```

**Textual Figure:**

```
In-Place Bubble Sort: arr = [64, 34, 25, 12, 22, 11, 90]
Only swaps adjacent elements — O(1) extra space.

  Pass 0: compare & swap neighbors:
  [64,34,25,12,22,11,90]
    64>34 swap → [34,64,25,12,22,11,90]
    64>25 swap → [34,25,64,12,22,11,90]
    64>12 swap → [34,25,12,64,22,11,90]
    64>22 swap → [34,25,12,22,64,11,90]
    64>11 swap → [34,25,12,22,11,64,90]
    64<90 ok    → 90 in place

  Pass 1: [25,12,22,11,34,|64,90]  (64,90 settled)
  Pass 2: [12,22,11,25,|34,64,90]
  ...

  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 11 │ 12 │ 22 │ 25 │ 34 │ 64 │ 90 │
  └────┴────┴────┴────┴────┴────┴────┘
  Extra space: only 1 temp for swap = O(1)
```

---

## Example 2: In-Place Partition (Quick Sort Core)

```go
package main

import "fmt"

func partition(arr []int, lo, hi int) int {
	pivot := arr[hi]
	i := lo
	for j := lo; j < hi; j++ {
		if arr[j] <= pivot {
			arr[i], arr[j] = arr[j], arr[i]
			i++
		}
	}
	arr[i], arr[hi] = arr[hi], arr[i]
	return i
}

func quickSort(arr []int, lo, hi int) {
	if lo < hi {
		p := partition(arr, lo, hi)
		quickSort(arr, lo, p-1)
		quickSort(arr, p+1, hi)
	}
}

func main() {
	arr := []int{10, 80, 30, 90, 40, 50, 70}
	quickSort(arr, 0, len(arr)-1) // O(log n) stack space
	fmt.Println(arr) // [10 30 40 50 70 80 90]
}
```

**Textual Figure:**

```
In-Place Partition: arr = [10, 80, 30, 90, 40, 50, 70]
pivot = arr[hi] = 70, i = lo = 0

  j=0: 10≤70 → swap(0,0), i=1  [10,80,30,90,40,50,70]
  j=1: 80>70 → skip            [10,80,30,90,40,50,70]
  j=2: 30≤70 → swap(1,2), i=2  [10,30,80,90,40,50,70]
  j=3: 90>70 → skip            [10,30,80,90,40,50,70]
  j=4: 40≤70 → swap(2,4), i=3  [10,30,40,90,80,50,70]
  j=5: 50≤70 → swap(3,5), i=4  [10,30,40,50,80,90,70]
  Place pivot: swap(4,6)        [10,30,40,50,70,90,80]
                                           ↑
                                         pivot at i=4

  ┌─────────────────┬────┬──────────┐
  │ ≤ pivot          │ 70 │ > pivot  │
  │ [10,30,40,50]   │    │ [90,80]  │
  └─────────────────┴────┴──────────┘
  O(log n) stack space for recursion.
```

---

## Example 3: In-Place Reversal

```go
package main

import "fmt"

func reverseInPlace(arr []int) {
	left, right := 0, len(arr)-1
	for left < right {
		arr[left], arr[right] = arr[right], arr[left]
		left++
		right--
	}
}

func main() {
	arr := []int{1, 2, 3, 4, 5}
	reverseInPlace(arr)
	fmt.Println(arr) // [5 4 3 2 1]
}
```

**Textual Figure:**

```
In-Place Reversal: arr = [1, 2, 3, 4, 5]
Two pointers from both ends, swap toward center.

  Step 1: left=0, right=4
  ┌───┬───┬───┬───┬───┐
  │ 5 │ 2 │ 3 │ 4 │ 1 │  swap(1,5)
  └───┴───┴───┴───┴───┘
   ↑                   ↑

  Step 2: left=1, right=3
  ┌───┬───┬───┬───┬───┐
  │ 5 │ 4 │ 3 │ 2 │ 1 │  swap(2,4)
  └───┴───┴───┴───┴───┘
       ↑       ↑

  Step 3: left=2, right=2 → done (left ≥ right)
  ┌───┬───┬───┬───┬───┐
  │ 5 │ 4 │ 3 │ 2 │ 1 │  O(1) space, O(n) time
  └───┴───┴───┴───┴───┘
```

---

## Example 4: In-Place Dutch National Flag (3-Way Partition)

```go
package main

import "fmt"

func sortColors(nums []int) {
	lo, mid, hi := 0, 0, len(nums)-1

	for mid <= hi {
		switch nums[mid] {
		case 0:
			nums[lo], nums[mid] = nums[mid], nums[lo]
			lo++
			mid++
		case 1:
			mid++
		case 2:
			nums[mid], nums[hi] = nums[hi], nums[mid]
			hi--
		}
	}
}

func main() {
	nums := []int{2, 0, 2, 1, 1, 0}
	sortColors(nums) // O(1) space, single pass
	fmt.Println(nums) // [0 0 1 1 2 2]
}
```

**Textual Figure:**

```
Dutch National Flag: nums = [2, 0, 2, 1, 1, 0]
Three pointers: lo=0, mid=0, hi=5

  mid=0: nums[0]=2 → swap(mid,hi), hi--
  [0, 0, 2, 1, 1, 2]  lo=0,mid=0,hi=4
   ↑                ↑

  mid=0: nums[0]=0 → swap(lo,mid), lo++, mid++
  [0, 0, 2, 1, 1, 2]  lo=1,mid=1,hi=4

  mid=1: nums[1]=0 → swap(lo,mid), lo++, mid++
  [0, 0, 2, 1, 1, 2]  lo=2,mid=2,hi=4

  mid=2: nums[2]=2 → swap(mid,hi), hi--
  [0, 0, 1, 1, 2, 2]  lo=2,mid=2,hi=3

  mid=2: nums[2]=1 → mid++
  [0, 0, 1, 1, 2, 2]  lo=2,mid=3,hi=3

  mid=3: nums[3]=1 → mid++
  mid=4 > hi=3 → done!

  ┌───┬───┬───┬───┬───┬───┐
  │ 0 │ 0 │ 1 │ 1 │ 2 │ 2 │  O(n) time, O(1) space
  └───┴───┴───┴───┴───┴───┘
```

---

## Example 5: In-Place Merge (O(1) Space Merge Sort)

```go
package main

import "fmt"

// In-place merge using rotation — O(n²) but O(1) space
func inPlaceMerge(arr []int, lo, mid, hi int) {
	i, j := lo, mid+1
	for i <= mid && j <= hi {
		if arr[i] <= arr[j] {
			i++
		} else {
			val := arr[j]
			// Shift elements right
			for k := j; k > i; k-- {
				arr[k] = arr[k-1]
			}
			arr[i] = val
			i++
			mid++
			j++
		}
	}
}

func inPlaceMergeSort(arr []int, lo, hi int) {
	if lo < hi {
		mid := lo + (hi-lo)/2
		inPlaceMergeSort(arr, lo, mid)
		inPlaceMergeSort(arr, mid+1, hi)
		inPlaceMerge(arr, lo, mid, hi)
	}
}

func main() {
	arr := []int{12, 11, 13, 5, 6, 7}
	inPlaceMergeSort(arr, 0, len(arr)-1)
	fmt.Println(arr) // [5 6 7 11 12 13]
}
```

**Textual Figure:**

```
In-Place Merge Sort: arr = [12, 11, 13, 5, 6, 7]

  Split: [12,11,13] | [5,6,7]
  Recurse left:  [11,12,13]
  Recurse right: [5,6,7]

  In-place merge [11,12,13,5,6,7]:
  i=0,j=3: 11>5 → shift right, insert 5
  [5,11,12,13,6,7]  i=1,j=4
  i=1,j=4: 11>6 → shift right, insert 6
  [5,6,11,12,13,7]  i=2,j=5
  i=2,j=5: 11>7 → shift right, insert 7
  [5,6,7,11,12,13]  done!

  ┌───┬───┬───┬────┬────┬────┐
  │ 5 │ 6 │ 7 │ 11 │ 12 │ 13 │
  └───┴───┴───┴────┴────┴────┘

  Trade-off: O(1) space but O(n²) merge (shift is O(n)).
  Standard merge sort: O(n) space, O(n) merge.
```

---

## Example 6: In-Place Array Rotation (for In-Place Algorithms)

```go
package main

import "fmt"

func reverse(arr []int, lo, hi int) {
	for lo < hi {
		arr[lo], arr[hi] = arr[hi], arr[lo]
		lo++
		hi--
	}
}

func rotateLeft(arr []int, k int) {
	n := len(arr)
	k %= n
	// Three reversals — O(1) space
	reverse(arr, 0, k-1)
	reverse(arr, k, n-1)
	reverse(arr, 0, n-1)
}

func rotateRight(arr []int, k int) {
	n := len(arr)
	k %= n
	reverse(arr, 0, n-1)
	reverse(arr, 0, k-1)
	reverse(arr, k, n-1)
}

func main() {
	arr := []int{1, 2, 3, 4, 5, 6, 7}
	rotateLeft(arr, 2)
	fmt.Println("Left by 2:", arr) // [3 4 5 6 7 1 2]

	arr2 := []int{1, 2, 3, 4, 5, 6, 7}
	rotateRight(arr2, 3)
	fmt.Println("Right by 3:", arr2) // [5 6 7 1 2 3 4]
}
```

**Textual Figure:**

```
In-Place Array Rotation: arr = [1,2,3,4,5,6,7], rotate left by 2
Three-reversal approach — O(1) space:

  Step 1: Reverse first k=2 elements:
  [1,2|3,4,5,6,7] → [2,1|3,4,5,6,7]

  Step 2: Reverse remaining n-k=5 elements:
  [2,1|3,4,5,6,7] → [2,1|7,6,5,4,3]

  Step 3: Reverse entire array:
  [2,1,7,6,5,4,3] → [3,4,5,6,7,1,2] ✓

  Before: ┌─┬─┬─┬─┬─┬─┬─┐
          │1│2│3│4│5│6│7│
          └─┴─┴─┴─┴─┴─┴─┘
  After:  ┌─┬─┬─┬─┬─┬─┬─┐
          │3│4│5│6│7│1│2│
          └─┴─┴─┴─┴─┴─┴─┘

  No extra array needed — just swaps.
```

---

## Example 7: In-Place Move Zeroes to End

```go
package main

import "fmt"

func moveZeroes(nums []int) {
	writeIdx := 0
	for _, v := range nums {
		if v != 0 {
			nums[writeIdx] = v
			writeIdx++
		}
	}
	for writeIdx < len(nums) {
		nums[writeIdx] = 0
		writeIdx++
	}
}

func main() {
	nums := []int{0, 1, 0, 3, 12}
	moveZeroes(nums)
	fmt.Println(nums) // [1 3 12 0 0]
}
```

**Textual Figure:**

```
Move Zeroes: nums = [0, 1, 0, 3, 12]
Read-write pointer pattern, O(1) space.

  Phase 1 — Write non-zeros:
  writeIdx=0
  ┌───┬───┬───┬───┬────┐
  │ 0 │ 1 │ 0 │ 3 │ 12 │  v=0 skip, v=1 write[0], v=0 skip,
  └───┴───┴───┴───┴────┘  v=3 write[1], v=12 write[2]
   r→→→→→
  ┌───┬───┬────┬───┬────┐
  │ 1 │ 3 │ 12 │ 3 │ 12 │  writeIdx=3
  └───┴───┴────┴───┴────┘
   w→→→         (stale values remain)

  Phase 2 — Fill rest with 0:
  ┌───┬───┬────┬───┬───┐
  │ 1 │ 3 │ 12 │ 0 │ 0 │  ✓
  └───┴───┴────┴───┴───┘
```

---

## Example 8: In-Place Remove Duplicates from Sorted Array

```go
package main

import "fmt"

func removeDuplicates(nums []int) int {
	if len(nums) == 0 { return 0 }
	writeIdx := 1
	for i := 1; i < len(nums); i++ {
		if nums[i] != nums[i-1] {
			nums[writeIdx] = nums[i]
			writeIdx++
		}
	}
	return writeIdx
}

func main() {
	nums := []int{0, 0, 1, 1, 1, 2, 2, 3, 3, 4}
	n := removeDuplicates(nums)
	fmt.Println(nums[:n]) // [0 1 2 3 4]
}
```

**Textual Figure:**

```
Remove Duplicates: nums = [0,0,1,1,1,2,2,3,3,4]
Sorted array → duplicates are adjacent.
writeIdx=1, scan with i=1..9

  ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
  │0│0│1│1│1│2│2│3│3│4│  wr=1
  └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
   w i

  i=1: 0==0 skip
  i=2: 1!=0 → nums[1]=1, wr=2
  i=3: 1==1 skip
  i=4: 1==1 skip
  i=5: 2!=1 → nums[2]=2, wr=3
  i=6: 2==2 skip
  i=7: 3!=2 → nums[3]=3, wr=4
  i=8: 3==3 skip
  i=9: 4!=3 → nums[4]=4, wr=5

  ┌─┬─┬─┬─┬─┬─────────┐
  │0│1│2│3│4│  ...     │  return 5
  └─┴─┴─┴─┴─┴─────────┘
  Only writeIdx (1 variable) = O(1) space.
```

---

## Example 9: Shell Sort (In-Place, Better Than Insertion)

```go
package main

import "fmt"

func shellSort(arr []int) {
	n := len(arr)
	// Start with large gap, reduce
	for gap := n / 2; gap > 0; gap /= 2 {
		for i := gap; i < n; i++ {
			temp := arr[i]
			j := i
			for j >= gap && arr[j-gap] > temp {
				arr[j] = arr[j-gap]
				j -= gap
			}
			arr[j] = temp
		}
	}
}

func main() {
	arr := []int{12, 34, 54, 2, 3}
	shellSort(arr) // O(1) extra space
	fmt.Println(arr) // [2 3 12 34 54]
}
```

**Textual Figure:**

```
Shell Sort: arr = [12, 34, 54, 2, 3], n=5
Diminishing gap insertion sort — O(1) space.

  gap=2: compare elements 2 apart
  ┌────┬────┬────┬───┬───┐
  │ 12 │ 34 │ 54 │ 2 │ 3 │
  └────┴────┴────┴───┴───┘
   ╰────────╯  ╰────╯  groups: {12,54,3} {34,2}
  i=2: 54>12 ok
  i=3: 2<34 → swap → [12,2,54,34,3]
  i=4: 3<54 → swap → [12,2,3,34,54], 3<12 → swap → nope (gap=2)
  After gap=2: [12, 2, 3, 34, 54]

  gap=1: standard insertion sort on nearly-sorted array
  ┌───┬───┬────┬────┬────┐
  │ 2 │ 3 │ 12 │ 34 │ 54 │  ✓ sorted
  └───┴───┴────┴────┴────┘

  Larger gaps move elements far quickly.
  Gap=1 pass finishes with few swaps.
```

---

## Example 10: In-Place Sorting Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== In-Place Sorting Summary ===")
	fmt.Println()

	fmt.Println("Definition:")
	fmt.Println("  O(1) extra memory (excluding input & call stack)")
	fmt.Println("  Some count O(log n) stack as 'in-place' (quicksort)")
	fmt.Println()

	fmt.Println("In-place algorithms:")
	fmt.Println("  ┌──────────────────┬───────────┬────────────┬────────┐")
	fmt.Println("  │ Algorithm        │ Time      │ Space      │ Stable │")
	fmt.Println("  ├──────────────────┼───────────┼────────────┼────────┤")
	fmt.Println("  │ Bubble Sort      │ O(n²)     │ O(1)       │ Yes    │")
	fmt.Println("  │ Selection Sort   │ O(n²)     │ O(1)       │ No     │")
	fmt.Println("  │ Insertion Sort   │ O(n²)     │ O(1)       │ Yes    │")
	fmt.Println("  │ Shell Sort       │ O(n^1.5)  │ O(1)       │ No     │")
	fmt.Println("  │ Heap Sort        │ O(n lg n) │ O(1)       │ No     │")
	fmt.Println("  │ Quick Sort       │ O(n lg n) │ O(lg n)*   │ No     │")
	fmt.Println("  └──────────────────┴───────────┴────────────┴────────┘")
	fmt.Println("  * Quick sort uses O(log n) stack space")
	fmt.Println()

	fmt.Println("Key techniques for in-place operations:")
	fmt.Println("  • Swap: exchange two elements")
	fmt.Println("  • Partition: rearrange around pivot")
	fmt.Println("  • Two pointers: read/write pointers")
	fmt.Println("  • Reversal: three-reversal rotation")
	fmt.Println("  • Overwrite: use negative/sentinel values")
}
```

**Textual Figure:**

```
In-Place Sorting Summary:

  ┌──────────────────┬───────────┬────────────┬────────┐
  │ Algorithm        │ Time      │ Space      │ Stable │
  ├──────────────────┼───────────┼────────────┼────────┤
  │ Bubble Sort      │ O(n²)     │ O(1)       │ Yes    │
  │ Selection Sort   │ O(n²)     │ O(1)       │ No     │
  │ Insertion Sort   │ O(n²)     │ O(1)       │ Yes    │
  │ Shell Sort       │ O(n^1.5)  │ O(1)       │ No     │
  │ Heap Sort        │ O(n lg n) │ O(1)       │ No     │
  │ Quick Sort       │ O(n lg n) │ O(lg n)*   │ No     │
  └──────────────────┴───────────┴────────────┴────────┘
  * O(log n) call stack

  Key in-place techniques:
  • Swap          ── exchange two elements
  • Two pointers  ── read/write pattern
  • Partition     ── rearrange around pivot
  • Reversal      ── three-reversal rotation
  • Overwrite     ── sentinel/negative markers
```

---

## Key Takeaways

1. In-place = O(1) extra space (input array modified directly)
2. All O(n²) simple sorts are in-place
3. Heap sort is the only O(n log n) in-place sort that's always O(n log n)
4. In-place merge sort exists but trades time O(n²) for space O(1)
5. Key pattern: two-pointer / read-write pointer approach

> **Next up:** Divide and Conquer in Sorting →
