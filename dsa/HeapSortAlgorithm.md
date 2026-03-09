# Phase 16: Sorting Algorithms — Heap Sort

## Overview

**Heap Sort** builds a max-heap from the array, then repeatedly extracts the maximum to sort. It guarantees O(n log n) with O(1) extra space.

| Aspect | Detail |
|--------|--------|
| **Time** | O(n log n) — always |
| **Space** | O(1) — in-place |
| **Stable** | No |
| **Key operation** | Heapify (sift down) |

---

## Example 1: Classic Heap Sort

```go
package main

import "fmt"

func heapSort(arr []int) {
	n := len(arr)

	// Build max heap
	for i := n/2 - 1; i >= 0; i-- {
		heapify(arr, n, i)
	}

	// Extract max one by one
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		heapify(arr, i, 0)
	}
}

func heapify(arr []int, n, i int) {
	largest := i
	left := 2*i + 1
	right := 2*i + 2

	if left < n && arr[left] > arr[largest] { largest = left }
	if right < n && arr[right] > arr[largest] { largest = right }

	if largest != i {
		arr[i], arr[largest] = arr[largest], arr[i]
		heapify(arr, n, largest)
	}
}

func main() {
	arr := []int{12, 11, 13, 5, 6, 7}
	heapSort(arr)
	fmt.Println(arr) // [5 6 7 11 12 13]
}
```

---

## Example 2: Iterative Heapify

```go
package main

import "fmt"

func heapifyIterative(arr []int, n, i int) {
	for {
		largest := i
		left := 2*i + 1
		right := 2*i + 2

		if left < n && arr[left] > arr[largest] { largest = left }
		if right < n && arr[right] > arr[largest] { largest = right }

		if largest == i { break }
		arr[i], arr[largest] = arr[largest], arr[i]
		i = largest
	}
}

func heapSortIterative(arr []int) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		heapifyIterative(arr, n, i)
	}
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		heapifyIterative(arr, i, 0)
	}
}

func main() {
	arr := []int{4, 10, 3, 5, 1}
	heapSortIterative(arr)
	fmt.Println(arr) // [1 3 4 5 10]
}
```

---

## Example 3: Min Heap Sort (Descending Order)

```go
package main

import "fmt"

func heapSortDescending(arr []int) {
	n := len(arr)

	// Build MIN heap
	for i := n/2 - 1; i >= 0; i-- {
		minHeapify(arr, n, i)
	}

	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		minHeapify(arr, i, 0)
	}
}

func minHeapify(arr []int, n, i int) {
	smallest := i
	left := 2*i + 1
	right := 2*i + 2

	if left < n && arr[left] < arr[smallest] { smallest = left }
	if right < n && arr[right] < arr[smallest] { smallest = right }

	if smallest != i {
		arr[i], arr[smallest] = arr[smallest], arr[i]
		minHeapify(arr, n, smallest)
	}
}

func main() {
	arr := []int{5, 3, 8, 1, 9, 2}
	heapSortDescending(arr)
	fmt.Println(arr) // [9 8 5 3 2 1] — descending
}
```

---

## Example 4: K Largest Elements Using Partial Heap Sort

```go
package main

import "fmt"

func kLargest(arr []int, k int) []int {
	n := len(arr)

	// Build max heap
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, n, i)
	}

	result := make([]int, k)
	for i := 0; i < k; i++ {
		result[i] = arr[0]
		arr[0] = arr[n-1-i]
		siftDown(arr, n-1-i, 0)
	}
	return result
}

func siftDown(arr []int, n, i int) {
	for {
		largest := i
		l, r := 2*i+1, 2*i+2
		if l < n && arr[l] > arr[largest] { largest = l }
		if r < n && arr[r] > arr[largest] { largest = r }
		if largest == i { break }
		arr[i], arr[largest] = arr[largest], arr[i]
		i = largest
	}
}

func main() {
	arr := []int{3, 2, 1, 5, 6, 4}
	fmt.Println("3 largest:", kLargest(arr, 3)) // [6 5 4]
}
```

---

## Example 5: Heap Sort Visualization

```go
package main

import "fmt"

func heapSortVisualize(arr []int) {
	n := len(arr)
	fmt.Println("Initial:", arr)

	// Build max heap
	for i := n/2 - 1; i >= 0; i-- {
		heapifyViz(arr, n, i)
	}
	fmt.Println("Max heap built:", arr)

	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		fmt.Printf("Extract max %d: %v (heap size %d)\n", arr[i], arr, i)
		heapifyViz(arr, i, 0)
	}
	fmt.Println("Sorted:", arr)
}

func heapifyViz(arr []int, n, i int) {
	largest := i
	l, r := 2*i+1, 2*i+2
	if l < n && arr[l] > arr[largest] { largest = l }
	if r < n && arr[r] > arr[largest] { largest = r }
	if largest != i {
		arr[i], arr[largest] = arr[largest], arr[i]
		heapifyViz(arr, n, largest)
	}
}

func main() {
	arr := []int{4, 10, 3, 5, 1}
	heapSortVisualize(arr)
}
```

---

## Example 6: Build Heap Time Complexity — O(n) Proof

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println("=== Build Heap is O(n), not O(n log n) ===")
	fmt.Println()
	fmt.Println("Key insight: most nodes are near the bottom of the heap")
	fmt.Println("and need very few sift-down operations.")
	fmt.Println()

	for _, n := range []int{15, 31, 63, 127, 1023} {
		h := int(math.Log2(float64(n + 1)))
		totalWork := 0
		for level := 0; level < h; level++ {
			nodesAtLevel := 1 << level
			siftDownSteps := h - level
			totalWork += nodesAtLevel * siftDownSteps
		}

		fmt.Printf("n=%4d, height=%d:\n", n, h)
		fmt.Printf("  Naive bound (n log n): %d\n", n*h)
		fmt.Printf("  Actual work (sum):     %d\n", totalWork)
		fmt.Printf("  Ratio work/n:          %.2f\n\n", float64(totalWork)/float64(n))
	}
}
```

---

## Example 7: Partial Sort (Sort First K Elements)

```go
package main

import "fmt"

// Sort only first k elements — O(n + k log n)
func partialSort(arr []int, k int) []int {
	n := len(arr)
	if k > n { k = n }

	// Build max heap: O(n)
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, n, i)
	}

	// Extract k elements: O(k log n)
	result := make([]int, k)
	size := n
	for i := 0; i < k; i++ {
		result[k-1-i] = arr[0] // store in reverse for ascending order
		arr[0] = arr[size-1]
		size--
		siftDown(arr, size, 0)
	}
	return result
}

func siftDown(arr []int, n, i int) {
	for {
		largest := i
		l, r := 2*i+1, 2*i+2
		if l < n && arr[l] > arr[largest] { largest = l }
		if r < n && arr[r] > arr[largest] { largest = r }
		if largest == i { break }
		arr[i], arr[largest] = arr[largest], arr[i]
		i = largest
	}
}

func main() {
	arr := []int{9, 3, 7, 1, 5, 8, 2, 6, 4, 10}
	top5 := partialSort(arr, 5)
	fmt.Println("Top 5 sorted:", top5) // [6 7 8 9 10]
}
```

---

## Example 8: Heap Sort for Strings

```go
package main

import "fmt"

func heapSortStrings(arr []string) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		heapifyStr(arr, n, i)
	}
	for i := n - 1; i > 0; i-- {
		arr[0], arr[i] = arr[i], arr[0]
		heapifyStr(arr, i, 0)
	}
}

func heapifyStr(arr []string, n, i int) {
	largest := i
	l, r := 2*i+1, 2*i+2
	if l < n && arr[l] > arr[largest] { largest = l }
	if r < n && arr[r] > arr[largest] { largest = r }
	if largest != i {
		arr[i], arr[largest] = arr[largest], arr[i]
		heapifyStr(arr, n, largest)
	}
}

func main() {
	words := []string{"banana", "apple", "cherry", "date", "elderberry"}
	heapSortStrings(words)
	fmt.Println(words)
}
```

---

## Example 9: Sort Nearly Sorted Array (K-sorted)

```go
package main

import (
	"container/heap"
	"fmt"
)

// Each element is at most k positions from its sorted position
type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func sortKSorted(arr []int, k int) []int {
	h := &IntHeap{}
	result := make([]int, 0, len(arr))

	for i, v := range arr {
		heap.Push(h, v)
		if i >= k {
			result = append(result, heap.Pop(h).(int))
		}
	}
	for h.Len() > 0 {
		result = append(result, heap.Pop(h).(int))
	}
	return result
}

func main() {
	// Each element at most 3 away from final position
	arr := []int{2, 6, 3, 12, 56, 8}
	fmt.Println(sortKSorted(arr, 3)) // [2 3 6 8 12 56]
}
```

---

## Example 10: Heap Sort Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Heap Sort Summary ===")
	fmt.Println()

	fmt.Println("Properties:")
	fmt.Println("  Time:     O(n log n) always")
	fmt.Println("  Space:    O(1) in-place")
	fmt.Println("  Stable:   No")
	fmt.Println("  Adaptive: No")
	fmt.Println()

	fmt.Println("Algorithm:")
	fmt.Println("  1. Build max-heap from array:     O(n)")
	fmt.Println("  2. Swap max with last element")
	fmt.Println("  3. Reduce heap size by 1")
	fmt.Println("  4. Heapify root:                  O(log n)")
	fmt.Println("  5. Repeat steps 2-4:              n-1 times")
	fmt.Println("  Total:                            O(n log n)")
	fmt.Println()

	fmt.Println("vs Quick Sort:")
	fmt.Println("  • Heap sort: guaranteed O(n log n), but poor cache locality")
	fmt.Println("  • Quick sort: O(n²) worst case, but better cache performance")
	fmt.Println("  • In practice, quick sort is usually faster")
	fmt.Println()

	fmt.Println("Best for:")
	fmt.Println("  • Guaranteed O(n log n) with O(1) space")
	fmt.Println("  • Partial sorting (top-K elements)")
	fmt.Println("  • Priority queue operations")
}
```

---

## Key Takeaways

1. Build heap is O(n), total heap sort is O(n log n) — always
2. O(1) extra space — the only O(n log n) in-place sort that's not unstable worst-case like quicksort
3. Not stable — relative order of equal elements may change
4. Excellent for partial sorting (extract top-K in O(n + k log n))
5. Poor cache locality compared to quicksort — generally slower in practice

> **Next up:** Counting Sort →
