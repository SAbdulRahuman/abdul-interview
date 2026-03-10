# Phase 11: Heap / Priority Queue — Heap Sort

## Overview

**Heap Sort** sorts an array in-place using a binary heap:

1. **Build a max heap** from the array — O(n)
2. **Repeatedly extract max**, swapping it to the end — O(n log n)

```
[4, 10, 3, 5, 1]  →  Build max heap  →  [10, 5, 3, 4, 1]
Extract 10 → [5, 4, 3, 1, | 10]
Extract 5  → [4, 1, 3, | 5, 10]
Extract 4  → [3, 1, | 4, 5, 10]
Extract 3  → [1, | 3, 4, 5, 10]
Done       → [1, 3, 4, 5, 10]
```

| Property | Value |
|----------|-------|
| Time | O(n log n) — always |
| Space | O(1) — in-place |
| Stable | No |
| In-place | Yes |

---

## Example 1: Heap Sort from Scratch

```go
package main

import "fmt"

func heapSort(arr []int) {
	n := len(arr)

	// Build max heap (bottom-up)
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, n, i)
	}

	// Extract max one by one
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0] // move max to end
		siftDown(arr, i, 0)             // restore heap on reduced array
	}
}

func siftDown(arr []int, size, root int) {
	largest := root
	left := 2*root + 1
	right := 2*root + 2

	if left < size && arr[left] > arr[largest] {
		largest = left
	}
	if right < size && arr[right] > arr[largest] {
		largest = right
	}
	if largest != root {
		arr[root], arr[largest] = arr[largest], arr[root]
		siftDown(arr, size, largest)
	}
}

func main() {
	arr := []int{12, 11, 13, 5, 6, 7}
	fmt.Println("Before:", arr)
	heapSort(arr)
	fmt.Println("After: ", arr)
	// Before: [12 11 13 5 6 7]
	// After:  [5 6 7 11 12 13]
}
```

**Textual Figure:**

```
Input: [12, 11, 13, 5, 6, 7]

  Phase 1: Build MAX heap (bottom-up)
  ┌─────────────────────────────────────────────┐
  │  Initial:        12                          │
  │                 /  \                          │
  │               11    13                        │
  │              / \   /                          │
  │             5   6  7                          │
  │                                               │
  │  i=2: 13>7 → no swap                         │
  │  i=1: 11<5,6? no, 11>6>5 → no swap           │
  │  i=0: 12<13 → swap 12↔13                      │
  │                                               │
  │  Max heap:       13                           │
  │                 /  \                          │
  │               11    12                        │
  │              / \   /                          │
  │             5   6  7                          │
  └─────────────────────────────────────────────┘

  Phase 2: Extract max + siftDown
  ┌──────┬────────────────┬─────────────────┐
  │ Swap │ Heap portion   │ Sorted portion │
  ├──────┼────────────────┼─────────────────┤
  │ 13↔7 │ [12,11,7,5,6] │ [13]           │
  │ 12↔6 │ [11,6,7,5]    │ [12, 13]       │
  │ 11↔5 │ [7,6,5]       │ [11,12,13]     │
  │  7↔5 │ [6,5]         │ [7,11,12,13]   │
  │  6↔5 │ [5]           │ [6,7,11,12,13] │
  └──────┴────────────────┴─────────────────┘

  Result: [5, 6, 7, 11, 12, 13]
```

---

## Example 2: Iterative Sift Down

```go
package main

import "fmt"

func siftDownIterative(arr []int, size, root int) {
	for {
		largest := root
		left := 2*root + 1
		right := 2*root + 2

		if left < size && arr[left] > arr[largest] {
			largest = left
		}
		if right < size && arr[right] > arr[largest] {
			largest = right
		}
		if largest == root {
			break
		}
		arr[root], arr[largest] = arr[largest], arr[root]
		root = largest
	}
}

func heapSort(arr []int) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		siftDownIterative(arr, n, i)
	}
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		siftDownIterative(arr, i, 0)
	}
}

func main() {
	arr := []int{4, 10, 3, 5, 1, 8, 7, 2, 9, 6}
	heapSort(arr)
	fmt.Println(arr) // [1 2 3 4 5 6 7 8 9 10]
}
```

**Textual Figure:**

```
Input: [4, 10, 3, 5, 1, 8, 7, 2, 9, 6]

  Iterative siftDown vs Recursive:
  ┌────────────────────────────────────────────┐
  │ Recursive: uses call stack O(log n)      │
  │ Iterative: loop, root = largest, O(1)    │
  │ Both: O(log n) per sift, same result     │
  └────────────────────────────────────────────┘

  Build max heap (iterative siftDown):
           10(0)
          /      \
        9(1)      8(2)
       /    \    /    \
      5(3)  6(4) 3(5) 7(6)
     /  \   /
    2(7) 4(8) 1(9)

  Extract all 10 → sort ascending:
  Result: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

**Why iterative?** Avoids stack overflow on deep heaps; same O(log n) per sift.

---

## Example 3: Min Heap Sort (Descending Order)

```go
package main

import "fmt"

func siftDownMin(arr []int, size, root int) {
	smallest := root
	left := 2*root + 1
	right := 2*root + 2

	if left < size && arr[left] < arr[smallest] {
		smallest = left
	}
	if right < size && arr[right] < arr[smallest] {
		smallest = right
	}
	if smallest != root {
		arr[root], arr[smallest] = arr[smallest], arr[root]
		siftDownMin(arr, size, smallest)
	}
}

func heapSortDescending(arr []int) {
	n := len(arr)
	// Build min heap
	for i := n/2 - 1; i >= 0; i-- {
		siftDownMin(arr, n, i)
	}
	// Extract min to end → descending order
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		siftDownMin(arr, i, 0)
	}
}

func main() {
	arr := []int{3, 1, 4, 1, 5, 9, 2, 6}
	heapSortDescending(arr)
	fmt.Println(arr) // [9 6 5 4 3 2 1 1]
}
```

**Textual Figure:**

```
Input: [3, 1, 4, 1, 5, 9, 2, 6]

  Build MIN heap (siftDownMin uses < instead of >):
           1(0)
          /     \
        1(1)     2(2)
       /   \    /   \
      3(3) 5(4) 9(5) 4(6)
     /
    6(7)

  Extract min to end → descending order:
  ┌──────┬────────────────┬─────────────────┐
  │ Swap │ Heap portion   │ Sorted portion │
  ├──────┼────────────────┼─────────────────┤
  │ 1↔6 │ [1,3,2,6,5,9,4]│ [1]            │
  │ 1↔4 │ [2,3,4,6,5,9]  │ [1,1]          │
  │ 2↔9 │ [3,5,4,6,9]    │ [2,1,1]        │
  │  ...                  ...              │
  └──────┴────────────────┴─────────────────┘

  Result: [9, 6, 5, 4, 3, 2, 1, 1]  (descending!)

  Key: min heap + extract-min-to-end = descending sort
       max heap + extract-max-to-end = ascending sort
```

---

## Example 4: Using Go's container/heap for Sort

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func heapSortViaLib(arr []int) []int {
	h := IntHeap(arr)
	heap.Init(&h) // O(n) heapify

	result := make([]int, 0, len(arr))
	for h.Len() > 0 {
		result = append(result, heap.Pop(&h).(int))
	}
	return result
}

func main() {
	arr := []int{38, 27, 43, 3, 9, 82, 10}
	sorted := heapSortViaLib(arr)
	fmt.Println(sorted) // [3 9 10 27 38 43 82]
}
```

**Textual Figure:**

```
Input: [38, 27, 43, 3, 9, 82, 10]

  heap.Init → build min heap O(n):
          3(0)
        /      \
      9(1)     10(2)
     /   \     /   \
   27(3) 38(4) 82(5) 43(6)

  Pop all into result (O(n log n)):
  ┌─────┬───────┬──────────────────────────┐
  │ Pop │  Val  │ result so far             │
  ├─────┼───────┼──────────────────────────┤
  │  1  │   3   │ [3]                      │
  │  2  │   9   │ [3, 9]                   │
  │  3  │  10   │ [3, 9, 10]               │
  │  4  │  27   │ [3, 9, 10, 27]           │
  │  5  │  38   │ [3, 9, 10, 27, 38]       │
  │  6  │  43   │ [3, 9, 10, 27, 38, 43]   │
  │  7  │  82   │ [3, 9, 10, 27, 38, 43, 82]│
  └─────┴───────┴──────────────────────────┘

  Note: uses O(n) extra space (result array)
  True in-place heap sort doesn't need extra array.
```

**Note:** This uses O(n) extra space. True heap sort is in-place.

---

## Example 5: Partial Heap Sort — Top K Elements

```go
package main

import "fmt"

func siftDown(arr []int, size, root int) {
	largest := root
	left := 2*root + 1
	right := 2*root + 2
	if left < size && arr[left] > arr[largest] { largest = left }
	if right < size && arr[right] > arr[largest] { largest = right }
	if largest != root {
		arr[root], arr[largest] = arr[largest], arr[root]
		siftDown(arr, size, largest)
	}
}

func topK(arr []int, k int) []int {
	n := len(arr)
	// Build max heap
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, n, i)
	}
	// Extract only k elements — O(k log n) instead of O(n log n)
	result := make([]int, k)
	for i := 0; i < k; i++ {
		result[i] = arr[0]
		arr[0] = arr[n-1-i]
		siftDown(arr, n-1-i, 0)
	}
	return result
}

func main() {
	arr := []int{3, 1, 4, 1, 5, 9, 2, 6, 5, 3}
	fmt.Println("Top 3:", topK(arr, 3)) // [9 6 5]
}
```

**Textual Figure:**

```
Input: [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]    k=3

  Build max heap:
             9(0)
           /      \
         6(1)      4(2)
        /    \    /    \
      5(3)  5(4) 3(5)  2(6)
     / \   /
    1   1  3

  Extract only k=3 (NOT full sort):
  ┌─────────┬───────┬──────────────────────┐
  │ Extract │ Value │ Heap after siftDown  │
  ├─────────┼───────┼──────────────────────┤
  │    1    │   9   │ [6, 5, 4, 1, 5, ...]  │
  │    2    │   6   │ [5, 5, 4, 1, 3, ...]  │
  │    3    │   5   │ (stop here)           │
  └─────────┴───────┴──────────────────────┘

  Top 3: [9, 6, 5]
  Time: O(n + k log n)  ← much better than O(n log n) when k << n
```

**Why?** When k << n, partial heap sort is faster than full sort.

---

## Example 6: Visualize Heap Sort Steps

```go
package main

import "fmt"

func siftDown(arr []int, size, root int) {
	largest := root
	left := 2*root + 1
	right := 2*root + 2
	if left < size && arr[left] > arr[largest] { largest = left }
	if right < size && arr[right] > arr[largest] { largest = right }
	if largest != root {
		arr[root], arr[largest] = arr[largest], arr[root]
		siftDown(arr, size, largest)
	}
}

func main() {
	arr := []int{4, 10, 3, 5, 1}
	n := len(arr)

	fmt.Println("Original:", arr)

	// Phase 1: Build max heap
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, n, i)
		fmt.Printf("Heapify i=%d: %v\n", i, arr)
	}

	// Phase 2: Extract
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		fmt.Printf("Swap top to pos %d: %v (sorted: %v)\n", i, arr[:i], arr[i:])
		siftDown(arr, i, 0)
		fmt.Printf("After sift:        %v (sorted: %v)\n", arr[:i], arr[i:])
	}
	fmt.Println("Result:", arr)
}
```

**Textual Figure:**

```
Input: [4, 10, 3, 5, 1]

  Phase 1: Build max heap
  ┌─────────────────────────────────────────────┐
  │  Original:       4                           │
  │                /   \                          │
  │              10     3                         │
  │             /  \                              │
  │            5    1                             │
  │                                               │
  │  i=1: 10>5,1 → no swap                       │
  │  i=0: 4<10 → swap 4↔10, then 4<5 → swap 4↔5  │
  │                                               │
  │  Max heap:      10        [10, 5, 3, 4, 1]   │
  │                /   \                          │
  │               5     3                         │
  │              / \                              │
  │             4   1                             │
  └─────────────────────────────────────────────┘

  Phase 2: Extract max, swap to end
  ┌─────────┬─────────────────┬─────────────────┐
  │ Swap    │ Heap portion    │ Sorted portion │
  ├─────────┼─────────────────┼─────────────────┤
  │ 10↔1   │ [5, 4, 3, 1]    │ [10]           │
  │  5↔1   │ [4, 1, 3]       │ [5, 10]        │
  │  4↔3   │ [3, 1]          │ [4, 5, 10]     │
  │  3↔1   │ [1]             │ [3, 4, 5, 10]  │
  └─────────┴─────────────────┴─────────────────┘

  Result: [1, 3, 4, 5, 10]
```

---

## Example 7: Heap Sort Strings

```go
package main

import "fmt"

func siftDownStr(arr []string, size, root int) {
	largest := root
	left := 2*root + 1
	right := 2*root + 2
	if left < size && arr[left] > arr[largest] { largest = left }
	if right < size && arr[right] > arr[largest] { largest = right }
	if largest != root {
		arr[root], arr[largest] = arr[largest], arr[root]
		siftDownStr(arr, size, largest)
	}
}

func heapSortStrings(arr []string) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		siftDownStr(arr, n, i)
	}
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		siftDownStr(arr, i, 0)
	}
}

func main() {
	words := []string{"banana", "apple", "cherry", "date", "elderberry"}
	heapSortStrings(words)
	fmt.Println(words) // [apple banana cherry date elderberry]
}
```

**Textual Figure:**

```
Input: ["banana", "apple", "cherry", "date", "elderberry"]

  Build max heap (lexicographic comparison):
          "elderberry"(0)
            /           \
      "date"(1)      "cherry"(2)
       /       \
  "banana"(3) "apple"(4)

  Extract max to end:
  ┌──────┬───────────────┬───────────────────────────┐
  │ Step │ Extract       │ Sorted (end)              │
  ├──────┼───────────────┼───────────────────────────┤
  │   1  │ "elderberry"  │ [elderberry]              │
  │   2  │ "date"        │ [date, elderberry]        │
  │   3  │ "cherry"      │ [cherry, date, elderberry]│
  │   4  │ "banana"      │ [...banana...]            │
  └──────┴───────────────┴───────────────────────────┘

  Result: [apple, banana, cherry, date, elderberry]
```

---

## Example 8: Heap Sort Stability Test

```go
package main

import "fmt"

type Pair struct {
	key   int
	label string
}

func siftDown(arr []Pair, size, root int) {
	largest := root
	left := 2*root + 1
	right := 2*root + 2
	if left < size && arr[left].key > arr[largest].key { largest = left }
	if right < size && arr[right].key > arr[largest].key { largest = right }
	if largest != root {
		arr[root], arr[largest] = arr[largest], arr[root]
		siftDown(arr, size, largest)
	}
}

func main() {
	// Demonstrate that heap sort is NOT stable
	arr := []Pair{
		{3, "A"}, {1, "B"}, {3, "C"}, {1, "D"}, {2, "E"},
	}
	fmt.Println("Before:", arr)

	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, n, i)
	}
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		siftDown(arr, i, 0)
	}

	fmt.Println("After: ", arr)
	fmt.Println()
	fmt.Println("Notice: Elements with key=3 may swap order (A↔C)")
	fmt.Println("This proves heap sort is NOT stable.")
	fmt.Println("For stable sorting, use merge sort or Go's sort.Stable().")
}
```

**Textual Figure:**

```
Input: [(3,A), (1,B), (3,C), (1,D), (2,E)]

  Build max heap (by key):
          (3,A)(0)
          /       \
       (1,B)(1)  (3,C)(2)
       /     \
    (1,D)(3) (2,E)(4)

  After heapify:
          (3,?)(0)            Keys 3: A and C may swap!
          /       \
       (2,E)(1)  (3,?)(2)
       /     \
    (1,D)(3) (1,B)(4)

  Heap sort extracts:
  ┌──────────────────────────────────────────┐
  │  Stable sort: (1,B)(1,D)(2,E)(3,A)(3,C)  │
  │  Heap sort:   (1,D)(1,B)(2,E)(3,C)(3,A)  │
  │                  ↑                 ↑      │
  │             B↔D swapped!      A↔C swapped!│
  │                                            │
  │  ∴ Heap sort is NOT stable                │
  └──────────────────────────────────────────┘
```

---

## Example 9: Heap Sort vs Quick Sort vs Merge Sort

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"time"
)

func siftDown(arr []int, size, root int) {
	for {
		largest := root
		l, r := 2*root+1, 2*root+2
		if l < size && arr[l] > arr[largest] { largest = l }
		if r < size && arr[r] > arr[largest] { largest = r }
		if largest == root { break }
		arr[root], arr[largest] = arr[largest], arr[root]
		root = largest
	}
}

func heapSort(arr []int) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- { siftDown(arr, n, i) }
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		siftDown(arr, i, 0)
	}
}

func benchmark(name string, sortFn func([]int), data []int) {
	arr := make([]int, len(data))
	copy(arr, data)
	start := time.Now()
	sortFn(arr)
	fmt.Printf("%-15s %v\n", name, time.Since(start))
}

func main() {
	n := 1_000_000
	data := make([]int, n)
	for i := range data { data[i] = rand.Intn(n) }

	benchmark("Heap Sort", heapSort, data)
	benchmark("Go sort.Ints", func(a []int) { sort.Ints(a) }, data)
	benchmark("sort.Slice", func(a []int) {
		sort.Slice(a, func(i, j int) bool { return a[i] < a[j] })
	}, data)

	fmt.Println()
	fmt.Println("=== Comparison ===")
	fmt.Println("| Algorithm    | Best    | Average  | Worst    | Space | Stable |")
	fmt.Println("|-------------|---------|----------|----------|-------|--------|")
	fmt.Println("| Heap Sort   | O(nlogn)| O(nlogn) | O(nlogn) | O(1)  | No     |")
	fmt.Println("| Quick Sort  | O(nlogn)| O(nlogn) | O(n²)    | O(logn)| No    |")
	fmt.Println("| Merge Sort  | O(nlogn)| O(nlogn) | O(nlogn) | O(n)  | Yes    |")
}
```

**Textual Figure:**

```
Sorting Algorithm Comparison (n=1,000,000 random ints):

  ┌──────────────┬─────────┬──────────┬──────────┬─────────┬────────┐
  │  Algorithm  │  Best   │ Average  │  Worst   │  Space  │ Stable │
  ├──────────────┼─────────┼──────────┼──────────┼─────────┼────────┤
  │ Heap Sort   │ O(nlogn)│ O(nlogn) │ O(nlogn) │ O(1)    │   No   │
  │ Quick Sort  │ O(nlogn)│ O(nlogn) │ O(n²)    │ O(logn) │   No   │
  │ Merge Sort  │ O(nlogn)│ O(nlogn) │ O(nlogn) │ O(n)    │   Yes  │
  └──────────────┴─────────┴──────────┴──────────┴─────────┴────────┘

  Trade-offs:
  • Heap Sort:  guaranteed O(nlogn), in-place, but poor cache locality
  • Quick Sort: fastest in practice (cache-friendly), O(n²) worst case
  • Merge Sort: stable + guaranteed O(nlogn), but O(n) extra space

  In practice: Quick Sort ≈ Go sort.Ints > Heap Sort
  (due to cache effects and constant factors)
```

---

## Example 10: Intro Sort — Heap Sort as Fallback

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
)

// IntroSort: QuickSort with HeapSort fallback to guarantee O(n log n)
func introSort(arr []int) {
	maxDepth := 2 * int(math.Floor(math.Log2(float64(len(arr)))))
	introSortHelper(arr, 0, len(arr)-1, maxDepth)
}

func introSortHelper(arr []int, lo, hi, depth int) {
	if hi-lo < 16 {
		insertionSort(arr, lo, hi)
		return
	}
	if depth == 0 {
		// Fallback to heap sort
		heapSortRange(arr, lo, hi)
		return
	}
	pivot := partition(arr, lo, hi)
	introSortHelper(arr, lo, pivot-1, depth-1)
	introSortHelper(arr, pivot+1, hi, depth-1)
}

func partition(arr []int, lo, hi int) int {
	pivot := arr[hi]
	i := lo - 1
	for j := lo; j < hi; j++ {
		if arr[j] <= pivot {
			i++
			arr[i], arr[j] = arr[j], arr[i]
		}
	}
	arr[i+1], arr[hi] = arr[hi], arr[i+1]
	return i + 1
}

func insertionSort(arr []int, lo, hi int) {
	for i := lo + 1; i <= hi; i++ {
		key := arr[i]
		j := i - 1
		for j >= lo && arr[j] > key {
			arr[j+1] = arr[j]
			j--
		}
		arr[j+1] = key
	}
}

func siftDown(arr []int, lo, size, root int) {
	for {
		largest := root
		l := 2*(root-lo) + 1 + lo
		r := 2*(root-lo) + 2 + lo
		if l <= lo+size-1 && arr[l] > arr[largest] { largest = l }
		if r <= lo+size-1 && arr[r] > arr[largest] { largest = r }
		if largest == root { break }
		arr[root], arr[largest] = arr[largest], arr[root]
		root = largest
	}
}

func heapSortRange(arr []int, lo, hi int) {
	n := hi - lo + 1
	sub := arr[lo : hi+1]
	for i := n/2 - 1; i >= 0; i-- {
		siftDownSub(sub, n, i)
	}
	for i := n - 1; i > 0; i-- {
		sub[0], sub[i] = sub[i], sub[0]
		siftDownSub(sub, i, 0)
	}
}

func siftDownSub(arr []int, size, root int) {
	for {
		largest := root
		l, r := 2*root+1, 2*root+2
		if l < size && arr[l] > arr[largest] { largest = l }
		if r < size && arr[r] > arr[largest] { largest = r }
		if largest == root { break }
		arr[root], arr[largest] = arr[largest], arr[root]
		root = largest
	}
}

func main() {
	_ = bits.Len // for bit operations
	arr := []int{38, 27, 43, 3, 9, 82, 10, 55, 1, 99, 42, 7}
	fmt.Println("Before:", arr)
	introSort(arr)
	fmt.Println("After: ", arr)
	// After:  [1 3 7 9 10 27 38 42 43 55 82 99]

	fmt.Println()
	fmt.Println("IntroSort = QuickSort + HeapSort fallback")
	fmt.Println("  - Uses QuickSort for average O(n log n) performance")
	fmt.Println("  - Falls back to HeapSort if recursion too deep → guaranteed O(n log n)")
	fmt.Println("  - Uses InsertionSort for small subarrays (< 16 elements)")
	fmt.Println("  - This is what C++ std::sort uses!")
}
```

**Textual Figure:**

```
IntroSort: Hybrid sorting algorithm

  Decision tree:
                    introSort(arr)
                         │
                ┌────────┴────────┐
                │                 │
          size < 16?        depth == 0?
           │    │             │     │
          Yes   No           Yes    No
           │    │             │     │
     InsertionSort     HeapSort   QuickSort
      O(n²) but       O(nlogn)   O(nlogn) avg
      fast for        guaranteed  fastest in
      tiny arrays     worst case  practice
                                      │
                              recurse with
                              depth - 1

  maxDepth = 2 × floor(log₂(n))

  Example: arr = [38, 27, 43, 3, 9, 82, 10, 55, 1, 99, 42, 7]
  n=12, maxDepth = 2 × 3 = 6

  ┌─────────────────────────────────────┐
  │ Uses QuickSort partitions normally  │
  │ If depth exhausted → HeapSort       │
  │ If subarray < 16 → InsertionSort    │
  │                                     │
  │ Result: guaranteed O(n log n)       │
  │ This is what C++ std::sort uses!    │
  └─────────────────────────────────────┘

  Output: [1, 3, 7, 9, 10, 27, 38, 42, 43, 55, 82, 99]
```

---

## Key Takeaways

1. **Heap sort** = build max heap + repeatedly extract max → O(n log n) always
2. **In-place** (O(1) space) but **not stable** — elements with equal keys may reorder
3. Build heap is O(n); extraction phase is O(n log n) — total O(n log n)
4. Worse cache performance than quicksort (random access pattern)
5. Used as a **fallback** in IntroSort to guarantee worst-case O(n log n)

> **Phase 11 Complete!** Next up: Phase 12 — Graphs →
