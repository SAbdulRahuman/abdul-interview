# Phase 11: Heap / Priority Queue — Heapify

## Overview

**Heapify** is the process of converting an unordered array into a valid heap. There are two approaches:

| Approach | Method | Time |
|----------|--------|------|
| Top-down | Insert elements one by one (sift up) | O(n log n) |
| Bottom-up | Start from last internal node, sift down | **O(n)** |

The bottom-up approach is O(n) because most nodes are near the bottom and sift down a small distance.

```
Unordered:  [4, 10, 3, 5, 1]
                4
              /   \
            10     3
           /  \
          5    1

After heapify (min heap):
                1
              /   \
            4      3
           /  \
          5   10
```

---

## Example 1: Bottom-Up Heapify (Min Heap)

```go
package main

import "fmt"

func heapify(arr []int) {
	n := len(arr)
	// Start from last internal node
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(arr, i, n)
	}
}

func siftDown(arr []int, i, n int) {
	for {
		smallest := i
		left, right := 2*i+1, 2*i+2
		if left < n && arr[left] < arr[smallest] {
			smallest = left
		}
		if right < n && arr[right] < arr[smallest] {
			smallest = right
		}
		if smallest == i {
			break
		}
		arr[i], arr[smallest] = arr[smallest], arr[i]
		i = smallest
	}
}

func main() {
	arr := []int{4, 10, 3, 5, 1, 8, 7}
	fmt.Println("Before:", arr)
	heapify(arr)
	fmt.Println("After:", arr) // [1 4 3 5 10 8 7] (valid min heap)

	// Verify: parent <= children
	for i := 0; i < len(arr); i++ {
		left, right := 2*i+1, 2*i+2
		if left < len(arr) && arr[i] > arr[left] {
			fmt.Println("INVALID at", i)
		}
		if right < len(arr) && arr[i] > arr[right] {
			fmt.Println("INVALID at", i)
		}
	}
	fmt.Println("Valid min heap!")
}
```

---

## Example 2: Bottom-Up Heapify (Max Heap)

```go
package main

import "fmt"

func heapifyMax(arr []int) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		siftDownMax(arr, i, n)
	}
}

func siftDownMax(arr []int, i, n int) {
	for {
		largest := i
		left, right := 2*i+1, 2*i+2
		if left < n && arr[left] > arr[largest] {
			largest = left
		}
		if right < n && arr[right] > arr[largest] {
			largest = right
		}
		if largest == i {
			break
		}
		arr[i], arr[largest] = arr[largest], arr[i]
		i = largest
	}
}

func main() {
	arr := []int{4, 10, 3, 5, 1, 8, 7}
	fmt.Println("Before:", arr)
	heapifyMax(arr)
	fmt.Println("After:", arr)
	fmt.Println("Root (max):", arr[0]) // 10
}
```

---

## Example 3: Using Go's heap.Init

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

func main() {
	// heap.Init does bottom-up heapify in O(n)
	h := &IntHeap{9, 5, 7, 1, 3, 8, 2, 6, 4}
	fmt.Println("Before Init:", *h)
	heap.Init(h)
	fmt.Println("After Init:", *h) // Valid min heap

	// Extract all in sorted order
	for h.Len() > 0 {
		fmt.Print(heap.Pop(h), " ") // 1 2 3 4 5 6 7 8 9
	}
	fmt.Println()
}
```

---

## Example 4: Top-Down vs Bottom-Up Performance

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func topDownHeapify(arr []int) []int {
	result := make([]int, 0, len(arr))
	for _, v := range arr {
		result = append(result, v)
		// sift up
		i := len(result) - 1
		for i > 0 {
			parent := (i - 1) / 2
			if result[i] < result[parent] {
				result[i], result[parent] = result[parent], result[i]
				i = parent
			} else {
				break
			}
		}
	}
	return result
}

func bottomUpHeapify(arr []int) {
	n := len(arr)
	for i := n/2 - 1; i >= 0; i-- {
		j := i
		for {
			smallest := j
			l, r := 2*j+1, 2*j+2
			if l < n && arr[l] < arr[smallest] { smallest = l }
			if r < n && arr[r] < arr[smallest] { smallest = r }
			if smallest == j { break }
			arr[j], arr[smallest] = arr[smallest], arr[j]
			j = smallest
		}
	}
}

func main() {
	n := 1_000_000
	arr := make([]int, n)
	for i := range arr {
		arr[i] = rand.Intn(n)
	}

	arr2 := make([]int, n)
	copy(arr2, arr)

	start := time.Now()
	topDownHeapify(arr)
	fmt.Printf("Top-down:  %v\n", time.Since(start))

	start = time.Now()
	bottomUpHeapify(arr2)
	fmt.Printf("Bottom-up: %v\n", time.Since(start))
	// Bottom-up is typically 2-3x faster
}
```

---

## Example 5: Heapify for Partial Array

```go
package main

import "fmt"

// Build heap only on first k elements
func partialHeapify(arr []int, k int) {
	if k > len(arr) {
		k = len(arr)
	}
	for i := k/2 - 1; i >= 0; i-- {
		siftDown(arr, i, k)
	}
}

func siftDown(arr []int, i, n int) {
	for {
		smallest := i
		l, r := 2*i+1, 2*i+2
		if l < n && arr[l] < arr[smallest] { smallest = l }
		if r < n && arr[r] < arr[smallest] { smallest = r }
		if smallest == i { break }
		arr[i], arr[smallest] = arr[smallest], arr[i]
		i = smallest
	}
}

func main() {
	arr := []int{9, 7, 5, 3, 1, 8, 6, 4, 2}
	fmt.Println("Original:", arr)
	
	partialHeapify(arr, 5) // heapify only first 5
	fmt.Println("First 5 heapified:", arr[:5])
	fmt.Println("Rest unchanged:", arr[5:])
}
```

---

## Example 6: Visualize Heapify Steps

```go
package main

import "fmt"

func heapifyWithTrace(arr []int) {
	n := len(arr)
	fmt.Printf("Start: %v\n", arr)
	for i := n/2 - 1; i >= 0; i-- {
		fmt.Printf("\nSift down index %d (value %d):\n", i, arr[i])
		siftDownTrace(arr, i, n)
		fmt.Printf("  Result: %v\n", arr)
	}
}

func siftDownTrace(arr []int, i, n int) {
	for {
		smallest := i
		l, r := 2*i+1, 2*i+2
		if l < n && arr[l] < arr[smallest] { smallest = l }
		if r < n && arr[r] < arr[smallest] { smallest = r }
		if smallest == i { break }
		fmt.Printf("  Swap arr[%d]=%d with arr[%d]=%d\n", i, arr[i], smallest, arr[smallest])
		arr[i], arr[smallest] = arr[smallest], arr[i]
		i = smallest
	}
}

func main() {
	arr := []int{8, 5, 3, 10, 2, 7, 1}
	heapifyWithTrace(arr)
}
```

---

## Example 7: Heapify to Find Top K

```go
package main

import "fmt"

// Build max heap, extract top k
func topK(arr []int, k int) []int {
	n := len(arr)
	// Build max heap
	for i := n/2 - 1; i >= 0; i-- {
		siftDownMax(arr, i, n)
	}
	// Extract k largest
	result := make([]int, k)
	size := n
	for i := 0; i < k; i++ {
		result[i] = arr[0]
		arr[0] = arr[size-1]
		size--
		siftDownMax(arr, 0, size)
	}
	return result
}

func siftDownMax(arr []int, i, n int) {
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
	arr := []int{3, 7, 2, 11, 5, 1, 9, 8}
	fmt.Println("Top 3:", topK(arr, 3)) // [11 9 8]
}
```

---

## Example 8: Verify Heap Property

```go
package main

import "fmt"

func isMinHeap(arr []int) bool {
	n := len(arr)
	for i := 0; i < n; i++ {
		left, right := 2*i+1, 2*i+2
		if left < n && arr[i] > arr[left] {
			return false
		}
		if right < n && arr[i] > arr[right] {
			return false
		}
	}
	return true
}

func isMaxHeap(arr []int) bool {
	n := len(arr)
	for i := 0; i < n; i++ {
		left, right := 2*i+1, 2*i+2
		if left < n && arr[i] < arr[left] {
			return false
		}
		if right < n && arr[i] < arr[right] {
			return false
		}
	}
	return true
}

func main() {
	fmt.Println(isMinHeap([]int{1, 3, 2, 7, 4}))  // true
	fmt.Println(isMinHeap([]int{1, 0, 2, 7, 4}))  // false
	fmt.Println(isMaxHeap([]int{9, 7, 8, 3, 5}))  // true
	fmt.Println(isMaxHeap([]int{9, 10, 8, 3, 5})) // false
}
```

---

## Example 9: Heapify with Custom Comparator

```go
package main

import (
	"container/heap"
	"fmt"
)

type Task struct {
	name     string
	priority int
}

type TaskHeap struct {
	tasks []Task
	less  func(a, b Task) bool
}

func (h TaskHeap) Len() int           { return len(h.tasks) }
func (h TaskHeap) Less(i, j int) bool { return h.less(h.tasks[i], h.tasks[j]) }
func (h TaskHeap) Swap(i, j int)      { h.tasks[i], h.tasks[j] = h.tasks[j], h.tasks[i] }
func (h *TaskHeap) Push(x interface{}) { h.tasks = append(h.tasks, x.(Task)) }
func (h *TaskHeap) Pop() interface{} {
	old := h.tasks
	n := len(old)
	x := old[n-1]
	h.tasks = old[:n-1]
	return x
}

func main() {
	// Min priority (lower number = higher priority)
	tasks := []Task{{"debug", 3}, {"deploy", 1}, {"test", 2}, {"review", 5}}
	h := &TaskHeap{
		tasks: tasks,
		less:  func(a, b Task) bool { return a.priority < b.priority },
	}
	heap.Init(h) // heapify with custom comparator

	for h.Len() > 0 {
		t := heap.Pop(h).(Task)
		fmt.Printf("Priority %d: %s\n", t.priority, t.name)
	}
}
```

---

## Example 10: Why Bottom-Up Heapify is O(n)

```go
package main

import "fmt"

func main() {
	/*
	 Analysis: Why bottom-up heapify is O(n)
	
	 In a heap of height h = log₂(n):
	 - Level 0 (root):     1 node,     sifts down h levels
	 - Level 1:            2 nodes,    sift down h-1 levels
	 - Level k:            2^k nodes,  sift down h-k levels
	 - Level h (leaves):   n/2 nodes,  sift down 0 levels
	
	 Total work = Σ(k=0 to h) 2^k × (h-k)
	            = Σ(j=0 to h) 2^(h-j) × j   (let j = h-k)
	            = 2^h × Σ(j=0 to h) j/2^j
	            ≤ 2^h × 2  (geometric series sum ≤ 2)
	            = 2n = O(n)
	
	 Key insight: Most nodes (n/2 leaves) do ZERO work!
	*/

	// Demonstrate: count total sift-down operations
	for _, n := range []int{7, 15, 31, 63, 127} {
		totalOps := 0
		h := 0
		for (1 << (h + 1)) <= n+1 {
			h++
		}
		for level := 0; level <= h; level++ {
			nodesAtLevel := 1 << level
			siftDistance := h - level
			ops := nodesAtLevel * siftDistance
			totalOps += ops
		}
		fmt.Printf("n=%3d h=%d totalOps=%3d ratio=%.2f\n",
			n, h, totalOps, float64(totalOps)/float64(n))
	}
	// ratio approaches ~1.0, confirming O(n)
}
```

---

## Key Takeaways

1. **Bottom-up heapify is O(n)** — start from last internal node `n/2-1`, sift down each
2. Top-down (repeated insert) is O(n log n) — always prefer bottom-up
3. `heap.Init()` in Go uses bottom-up heapify
4. Most nodes are leaves and require zero work — that's why it's linear
5. Always verify with `isMinHeap` / `isMaxHeap` during debugging

> **Next up:** Heap Operations →
