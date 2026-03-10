# Phase 16: Sorting Algorithms — Quick Sort

## Overview

**Quick Sort** picks a pivot, partitions the array around it (elements < pivot go left, > pivot go right), then recursively sorts both sides.

| Aspect | Detail |
|--------|--------|
| **Average** | O(n log n) |
| **Worst** | O(n²) — bad pivot choice |
| **Space** | O(log n) recursion stack |
| **Stable** | No |
| **In-place** | Yes |

---

## Example 1: Lomuto Partition Quick Sort

```go
package main

import "fmt"

func quickSort(arr []int, low, high int) {
	if low < high {
		pi := lomutoPartition(arr, low, high)
		quickSort(arr, low, pi-1)
		quickSort(arr, pi+1, high)
	}
}

func lomutoPartition(arr []int, low, high int) int {
	pivot := arr[high]
	i := low - 1

	for j := low; j < high; j++ {
		if arr[j] <= pivot {
			i++
			arr[i], arr[j] = arr[j], arr[i]
		}
	}
	arr[i+1], arr[high] = arr[high], arr[i+1]
	return i + 1
}

func main() {
	arr := []int{10, 80, 30, 90, 40, 50, 70}
	quickSort(arr, 0, len(arr)-1)
	fmt.Println(arr) // [10 30 40 50 70 80 90]
}
```

**Textual Figure:**

```
Lomuto Partition Quick Sort: arr = [10, 80, 30, 90, 40, 50, 70]

  pivot = arr[high] = 70
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 10 │ 80 │ 30 │ 90 │ 40 │ 50 │[70]│  pivot=70
  └────┴────┴────┴────┴────┴────┴────┘
  i=-1
  j=0: 10≤70 → i=0, swap(0,0)
  j=1: 80>70  → skip
  j=2: 30≤70 → i=1, swap(1,2)  [10,30,80,90,40,50,70]
  j=3: 90>70  → skip
  j=4: 40≤70 → i=2, swap(2,4)  [10,30,40,90,80,50,70]
  j=5: 50≤70 → i=3, swap(3,5)  [10,30,40,50,80,90,70]
  swap pivot: arr[4]↔arr[6]    [10,30,40,50,70,90,80]

  After partition:
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 10 │ 30 │ 40 │ 50 │ 70 │ 90 │ 80 │
  └────┴────┴────┴────┴────┴────┴────┘
     ≤ 70           [P]    > 70
  Recurse: quickSort([10,30,40,50]) + quickSort([90,80])
  Final: [10, 30, 40, 50, 70, 80, 90]
```

---

## Example 2: Hoare Partition

```go
package main

import "fmt"

func quickSortHoare(arr []int, low, high int) {
	if low < high {
		p := hoarePartition(arr, low, high)
		quickSortHoare(arr, low, p)
		quickSortHoare(arr, p+1, high)
	}
}

func hoarePartition(arr []int, low, high int) int {
	pivot := arr[low+(high-low)/2]
	i, j := low-1, high+1

	for {
		i++
		for arr[i] < pivot { i++ }
		j--
		for arr[j] > pivot { j-- }
		if i >= j { return j }
		arr[i], arr[j] = arr[j], arr[i]
	}
}

func main() {
	arr := []int{13, 19, 9, 5, 12, 8, 7, 4, 21, 2, 6, 11}
	quickSortHoare(arr, 0, len(arr)-1)
	fmt.Println(arr)
}
```

**Textual Figure:**

```
Hoare Partition: arr = [13,19,9,5,12,8,7,4,21,2,6,11]

  pivot = arr[mid] = arr[5] = 8
  i starts at low-1, j starts at high+1

  ┌────┬────┬───┬───┬────┬───┬───┬───┬────┬───┬───┬────┐
  │ 13 │ 19 │ 9 │ 5 │ 12 │ 8 │ 7 │ 4 │ 21 │ 2 │ 6 │ 11 │
  └────┴────┴───┴───┴────┴───┴───┴───┴────┴───┴───┴────┘
  i→                                           ←j
  i: scan right until arr[i]≥8 → i=0 (13≥8)
  j: scan left until arr[j]≤8  → j=10 (6≤8)
  swap(13,6):
  ┌───┬────┬───┬───┬────┬───┬───┬───┬────┬───┬────┬────┐
  │ 6 │ 19 │ 9 │ 5 │ 12 │ 8 │ 7 │ 4 │ 21 │ 2 │ 13 │ 11 │
  └───┴────┴───┴───┴────┴───┴───┴───┴────┴───┴────┴────┘
  Continue: i scans to 19(i=1), j scans to 2(j=9)
  swap(19,2) → [6,2,9,5,12,8,7,4,21,19,13,11]
  Continue: i→9(i=2), j→4(j=7)
  swap(9,4) → [6,2,4,5,12,8,7,9,21,19,13,11]
  Continue: i→12(i=4), j→7(j=6)
  swap(12,7) → [6,2,4,5,7,8,12,9,21,19,13,11]
  Continue: i→8(i=5), j→8...5, i≥j → return j

  Partition point: [6,2,4,5,7,8 | 12,9,21,19,13,11]
  Recurse on both halves.
```

---

## Example 3: Randomized Quick Sort

```go
package main

import (
	"fmt"
	"math/rand"
)

func quickSortRandom(arr []int, low, high int) {
	if low < high {
		// Random pivot to avoid worst case
		randIdx := low + rand.Intn(high-low+1)
		arr[randIdx], arr[high] = arr[high], arr[randIdx]

		pi := partition(arr, low, high)
		quickSortRandom(arr, low, pi-1)
		quickSortRandom(arr, pi+1, high)
	}
}

func partition(arr []int, low, high int) int {
	pivot := arr[high]
	i := low - 1
	for j := low; j < high; j++ {
		if arr[j] <= pivot { i++; arr[i], arr[j] = arr[j], arr[i] }
	}
	arr[i+1], arr[high] = arr[high], arr[i+1]
	return i + 1
}

func main() {
	// Worst case for naive quicksort (already sorted)
	arr := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	quickSortRandom(arr, 0, len(arr)-1)
	fmt.Println(arr)
}
```

**Textual Figure:**

```
Randomized Quick Sort: arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

  Problem with naive quicksort on sorted input:
  ┌────────────────────────────────────────┐
  │ Fixed pivot (last element):                │
  │  pivot=10 → [1..9] | [10]  (n-1 vs 0)    │
  │  pivot=9  → [1..8] | [9]   (n-2 vs 0)    │
  │  ...                                       │
  │  n levels × O(n) work = O(n²) worst case! │
  └────────────────────────────────────────┘

  With random pivot:
  ┌────────────────────────────────────────┐
  │ Random pivot e.g. 5:                       │
  │  [1,2,3,4] | [5] | [6,7,8,9,10]           │
  │  Balanced split → O(n log n)               │
  │                                              │
  │ Expected: each element equally likely pivot │
  │ Average depth = O(log n)                    │
  └────────────────────────────────────────┘

  Random pivot eliminates adversarial worst cases.
```

---

## Example 4: Three-Way Partition (Dutch National Flag)

```go
package main

import "fmt"

// Handles many duplicate elements efficiently
func quickSort3Way(arr []int, low, high int) {
	if low >= high { return }

	pivot := arr[low]
	lt, gt := low, high // arr[low..lt-1] < pivot, arr[gt+1..high] > pivot
	i := low

	for i <= gt {
		if arr[i] < pivot {
			arr[lt], arr[i] = arr[i], arr[lt]
			lt++; i++
		} else if arr[i] > pivot {
			arr[gt], arr[i] = arr[i], arr[gt]
			gt--
		} else {
			i++
		}
	}

	quickSort3Way(arr, low, lt-1)
	quickSort3Way(arr, gt+1, high)
}

func main() {
	arr := []int{4, 2, 4, 1, 4, 3, 4, 2, 1, 4}
	quickSort3Way(arr, 0, len(arr)-1)
	fmt.Println(arr) // [1 1 2 2 3 4 4 4 4 4]
}
```

**Textual Figure:**

```
3-Way Partition: arr = [4, 2, 4, 1, 4, 3, 4, 2, 1, 4]
  pivot = arr[0] = 4

  Three regions: [< pivot | == pivot | unseen | > pivot]
  lt=0, i=0, gt=9

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 4 │ 2 │ 4 │ 1 │ 4 │ 3 │ 4 │ 2 │ 1 │ 4 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  lt                                          gt
   i

  Processing: arr[i]==4(pivot)→ i++
              arr[i]=2<4 → swap(lt,i), lt++, i++
              arr[i]=4 → i++
              arr[i]=1<4 → swap(lt,i), lt++, i++
              ... continue ...

  After 3-way partition:
  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 1 │ 2 │ 3 │ 4 │ 4 │ 4 │ 4 │ 4 │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
   < 4              lt  == 4 ==       gt  (no >4)

  Recurse ONLY on [1,2,1,2,3] — skip all 4s!
  Result: [1, 1, 2, 2, 3, 4, 4, 4, 4, 4]
```

---

## Example 5: QuickSelect — Kth Smallest Element

```go
package main

import (
	"fmt"
	"math/rand"
)

func quickSelect(arr []int, left, right, k int) int {
	if left == right { return arr[left] }

	pivotIdx := left + rand.Intn(right-left+1)
	arr[pivotIdx], arr[right] = arr[right], arr[pivotIdx]

	pi := partitionQS(arr, left, right)

	if pi == k {
		return arr[pi]
	} else if pi < k {
		return quickSelect(arr, pi+1, right, k)
	}
	return quickSelect(arr, left, pi-1, k)
}

func partitionQS(arr []int, left, right int) int {
	pivot := arr[right]
	i := left - 1
	for j := left; j < right; j++ {
		if arr[j] <= pivot { i++; arr[i], arr[j] = arr[j], arr[i] }
	}
	arr[i+1], arr[right] = arr[right], arr[i+1]
	return i + 1
}

func main() {
	arr := []int{3, 2, 1, 5, 6, 4}
	fmt.Println("2nd smallest:", quickSelect(arr, 0, len(arr)-1, 1)) // 2
	
	arr2 := []int{3, 2, 3, 1, 2, 4, 5, 5, 6}
	fmt.Println("4th smallest:", quickSelect(arr2, 0, len(arr2)-1, 3)) // 3
}
```

**Textual Figure:**

```
QuickSelect: arr = [3, 2, 1, 5, 6, 4], k=1 (2nd smallest)

  Random pivot, say pivot=4:
  Partition:
  ┌───┬───┬───┬───┬───┬───┐
  │ 3 │ 2 │ 1 │ 4 │ 6 │ 5 │  pivot=4 at idx=3
  └───┴───┴───┴───┴───┴───┘
    ≤ 4       [P]    > 4

  pi=3, k=1 → k < pi → recurse left [3,2,1]

  pivot=1, partition:
  ┌───┬───┬───┐
  │ 1 │ 2 │ 3 │  pivot=1 at idx=0
  └───┴───┴───┘
  pi=0, k=1 → k > pi → recurse right [2,3], k=0

  pivot=2 → pi=0, k=0 → found! answer = 2

  Only O(n) average — no need to fully sort!
```

---

## Example 6: Iterative Quick Sort (No Recursion)

```go
package main

import "fmt"

func quickSortIterative(arr []int) {
	stack := [][2]int{{0, len(arr) - 1}}

	for len(stack) > 0 {
		top := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		low, high := top[0], top[1]

		if low >= high { continue }

		pi := partitionIt(arr, low, high)
		stack = append(stack, [2]int{low, pi - 1})
		stack = append(stack, [2]int{pi + 1, high})
	}
}

func partitionIt(arr []int, low, high int) int {
	pivot := arr[high]
	i := low - 1
	for j := low; j < high; j++ {
		if arr[j] <= pivot { i++; arr[i], arr[j] = arr[j], arr[i] }
	}
	arr[i+1], arr[high] = arr[high], arr[i+1]
	return i + 1
}

func main() {
	arr := []int{4, 3, 5, 2, 1, 3, 2, 3}
	quickSortIterative(arr)
	fmt.Println(arr)
}
```

**Textual Figure:**

```
Iterative Quick Sort: arr = [4, 3, 5, 2, 1, 3, 2, 3]

  Uses explicit stack instead of recursion:
  stack = [(0,7)]

  ┌──────┬────────────┬───────────────────────────┐
  │ Pop  │ Partition  │ Push                      │
  ├──────┼────────────┼───────────────────────────┤
  │(0,7)│ pi=5 p=3   │ push (0,4), (6,7)         │
  │(6,7)│ pi=7 p=3   │ push (6,6) → skip         │
  │(0,4)│ pi=1 p=2   │ push (0,0),(2,4) → ...    │
  │ ...  │ ...        │ ...                       │
  └──────┴────────────┴───────────────────────────┘

  Same partitioning as recursive, just manages stack manually.
  Avoids stack overflow for very deep recursion.
  Result: [1, 2, 2, 3, 3, 3, 4, 5]
```

---

## Example 7: Median of Three Pivot Selection

```go
package main

import "fmt"

func medianOfThree(arr []int, low, high int) int {
	mid := low + (high-low)/2
	if arr[low] > arr[mid] { arr[low], arr[mid] = arr[mid], arr[low] }
	if arr[low] > arr[high] { arr[low], arr[high] = arr[high], arr[low] }
	if arr[mid] > arr[high] { arr[mid], arr[high] = arr[high], arr[mid] }
	// Now arr[low] <= arr[mid] <= arr[high]
	arr[mid], arr[high-1] = arr[high-1], arr[mid]
	return high - 1 // pivot is at high-1
}

func quickSortMOT(arr []int, low, high int) {
	if high-low < 10 {
		insertionSort(arr, low, high)
		return
	}

	pivotIdx := medianOfThree(arr, low, high)
	pivot := arr[pivotIdx]

	i, j := low, pivotIdx-1
	for {
		i++; for arr[i] < pivot { i++ }
		j--; for arr[j] > pivot { j-- }
		if i >= j { break }
		arr[i], arr[j] = arr[j], arr[i]
	}
	arr[i], arr[pivotIdx] = arr[pivotIdx], arr[i]

	quickSortMOT(arr, low, i-1)
	quickSortMOT(arr, i+1, high)
}

func insertionSort(arr []int, low, high int) {
	for i := low + 1; i <= high; i++ {
		key := arr[i]
		j := i - 1
		for j >= low && arr[j] > key { arr[j+1] = arr[j]; j-- }
		arr[j+1] = key
	}
}

func main() {
	arr := []int{8, 4, 2, 9, 5, 1, 7, 3, 6, 10}
	quickSortMOT(arr, 0, len(arr)-1)
	fmt.Println(arr)
}
```

**Textual Figure:**

```
Median of Three: arr = [8, 4, 2, 9, 5, 1, 7, 3, 6, 10]

  Choose pivot from low, mid, high:
  low=0 (8), mid=4 (5), high=9 (10)
  Sort these three: 5 ≤ 8 ≤ 10
  Median = 8 → good pivot!

  ┌───────────────────────────────────────┐
  │ Pivot selection strategies:               │
  ├─────────────────┬─────────────────────┤
  │ Strategy        │ Worst case?          │
  ├─────────────────┼─────────────────────┤
  │ First element   │ Yes (sorted input)   │
  │ Last element    │ Yes (sorted input)   │
  │ Random          │ Unlikely (expected)  │
  │ Median of 3     │ Very unlikely        │
  └─────────────────┴─────────────────────┘

  Also: switch to InsertionSort for size < 10.
  Result: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

---

## Example 8: Sort Colors (LeetCode 75) — Quick Sort Application

```go
package main

import "fmt"

// Dutch National Flag problem — one-pass partition
func sortColors(nums []int) {
	low, mid, high := 0, 0, len(nums)-1

	for mid <= high {
		switch nums[mid] {
		case 0:
			nums[low], nums[mid] = nums[mid], nums[low]
			low++; mid++
		case 1:
			mid++
		case 2:
			nums[mid], nums[high] = nums[high], nums[mid]
			high--
		}
	}
}

func main() {
	nums := []int{2, 0, 2, 1, 1, 0}
	sortColors(nums)
	fmt.Println(nums) // [0 0 1 1 2 2]
}
```

**Textual Figure:**

```
Sort Colors (Dutch National Flag): nums = [2, 0, 2, 1, 1, 0]

  Three pointers: low=0, mid=0, high=5
  ┌───┬───┬───┬───┬───┬───┐
  │ 2 │ 0 │ 2 │ 1 │ 1 │ 0 │  mid=0: 2 → swap(mid,high), high--
  └───┴───┴───┴───┴───┴───┘
  ┌───┬───┬───┬───┬───┬───┐
  │ 0 │ 0 │ 2 │ 1 │ 1 │ 2 │  mid=0: 0 → swap(low,mid), low++, mid++
  └───┴───┴───┴───┴───┴───┘
  ┌───┬───┬───┬───┬───┬───┐
  │ 0 │ 0 │ 2 │ 1 │ 1 │ 2 │  mid=2: 2 → swap(mid,high), high--
  └───┴───┴───┴───┴───┴───┘
  ┌───┬───┬───┬───┬───┬───┐
  │ 0 │ 0 │ 1 │ 1 │ 2 │ 2 │  mid=2: 1 → mid++
  └───┴───┴───┴───┴───┴───┘  mid=3: 1 → mid++
   0s       1s       2s         mid=4 > high=3 → DONE!

  Result: [0, 0, 1, 1, 2, 2]  — single pass O(n)!
```

---

## Example 9: Tail-Call Optimized Quick Sort

```go
package main

import "fmt"

// Optimize recursion: recurse on smaller half, iterate on larger
func quickSortTailOpt(arr []int, low, high int) {
	for low < high {
		pi := partitionTO(arr, low, high)

		// Recurse on smaller partition
		if pi-low < high-pi {
			quickSortTailOpt(arr, low, pi-1)
			low = pi + 1 // iterate on larger
		} else {
			quickSortTailOpt(arr, pi+1, high)
			high = pi - 1
		}
	}
}

func partitionTO(arr []int, low, high int) int {
	pivot := arr[high]
	i := low - 1
	for j := low; j < high; j++ {
		if arr[j] <= pivot { i++; arr[i], arr[j] = arr[j], arr[i] }
	}
	arr[i+1], arr[high] = arr[high], arr[i+1]
	return i + 1
}

func main() {
	arr := []int{9, 7, 5, 3, 1, 2, 4, 6, 8}
	quickSortTailOpt(arr, 0, len(arr)-1)
	fmt.Println(arr) // [1 2 3 4 5 6 7 8 9]
	fmt.Println("Worst-case stack depth: O(log n) with tail optimization")
}
```

**Textual Figure:**

```
Tail-Call Optimization:

  Standard Quick Sort (worst case stack depth O(n)):
  quickSort(0, n-1)
    quickSort(0, pi-1)     ← recurse
      quickSort(0, pi'-1)  ← recurse deeper
        ...                 (stack grows)
    quickSort(pi+1, n-1)   ← also recurse

  Tail-optimized (worst case stack depth O(log n)):
  ┌──────────────────────────────────────┐
  │  if pi-low < high-pi:              │
  │    recurse on SMALLER half         │
  │    iterate on LARGER half          │
  │  else:                             │
  │    recurse on SMALLER (right) half │
  │    iterate on LARGER (left) half   │
  └──────────────────────────────────────┘

  arr = [9,7,5,3,1,2,4,6,8], pi=4 (pivot=1):
  Left=[9,7,5,3](size 4), Right=[2,4,6,8](size 4)
  Recurse on smaller, iterate on larger.
  Max stack depth = O(log n) guaranteed.

  Result: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## Example 10: Quick Sort vs Merge Sort

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Quick Sort vs Merge Sort ===")
	fmt.Println()

	aspects := []struct{ aspect, quicksort, mergesort string }{
		{"Average time", "O(n log n)", "O(n log n)"},
		{"Worst time", "O(n²)", "O(n log n)"},
		{"Space", "O(log n)", "O(n)"},
		{"Stable", "No", "Yes"},
		{"In-place", "Yes", "No"},
		{"Cache friendly", "Yes (sequential)", "Less (copying)"},
		{"Linked lists", "Poor (no random access)", "Excellent"},
		{"Worst case fix", "Random/median pivot", "Always O(n log n)"},
		{"Practical speed", "Faster (cache, less copying)", "Slower overhead"},
		{"Parallelism", "Harder", "Naturally parallel"},
	}

	for _, a := range aspects {
		fmt.Printf("%-18s %-28s %s\n", a.aspect, a.quicksort, a.mergesort)
	}

	fmt.Println("\nWhen to use Quick Sort:")
	fmt.Println("  • In-memory sorting where space matters")
	fmt.Println("  • Arrays (cache-friendly sequential access)")
	fmt.Println("  • When average case matters more than worst case")
	fmt.Println("\nWhen to use Merge Sort:")
	fmt.Println("  • Need guaranteed O(n log n)")
	fmt.Println("  • Need stability")
	fmt.Println("  • Sorting linked lists or external data")
}
```

**Textual Figure:**

```
Quick Sort vs Merge Sort:

  ┌──────────────────┬───────────────────┬──────────────────┐
  │ Aspect           │ Quick Sort        │ Merge Sort       │
  ├──────────────────┼───────────────────┼──────────────────┤
  │ Average time     │ O(n log n)        │ O(n log n)       │
  │ Worst time       │ O(n²)             │ O(n log n)       │
  │ Space            │ O(log n)          │ O(n)             │
  │ Stable           │ No                │ Yes              │
  │ In-place         │ Yes               │ No               │
  │ Cache friendly   │ Yes (sequential)  │ Less (copying)   │
  │ Linked lists     │ Poor              │ Excellent        │
  │ Practical speed  │ Faster            │ Slower overhead  │
  │ Parallelism      │ Harder            │ Natural          │
  └──────────────────┴───────────────────┴──────────────────┘

  Use Quick Sort when:         Use Merge Sort when:
  • In-memory arrays            • Need guaranteed O(nlogn)
  • Space matters               • Need stability
  • Average case is fine        • Linked lists
```

---

## Key Takeaways

1. Quick sort is O(n log n) average, O(n²) worst — fixable with random pivot
2. In-place and cache-friendly — often faster than merge sort in practice
3. Three-way partition handles duplicates efficiently — O(n) for all-equal
4. QuickSelect finds kth element in O(n) average time
5. Median-of-three + insertion sort cutoff = production-quality quick sort

> **Next up:** Heap Sort →
