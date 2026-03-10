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

**Textual Figure:**

```
Input: [4, 10, 3, 5, 1, 8, 7]

  Initial tree:                 Array: [4, 10, 3, 5, 1, 8, 7]
          4(0)                  Index:  0   1  2  3  4  5  6
        /      \
      10(1)     3(2)
     /   \     /   \
    5(3) 1(4) 8(5) 7(6)

  Step 1: siftDown(i=2, val=3)
  ┌─────────────────────────────────────────────┐
  │ Children: 8(5), 7(6)                        │
  │ 3 < 7 < 8 → already smallest → no swap     │
  └─────────────────────────────────────────────┘

  Step 2: siftDown(i=1, val=10)
  ┌─────────────────────────────────────────────┐
  │ Children: 5(3), 1(4)                        │
  │ smallest = 1 at index 4                     │
  │ Swap 10 ↔ 1                                 │
  └─────────────────────────────────────────────┘
          4                     Array: [4, 1, 3, 5, 10, 8, 7]
        /      \
       1        3
     /   \     /   \
    5    10   8     7

  Step 3: siftDown(i=0, val=4)
  ┌─────────────────────────────────────────────┐
  │ Children: 1(1), 3(2)                        │
  │ smallest = 1 at index 1 → Swap 4 ↔ 1       │
  │ Continue at i=1: children 5(3), 10(4)       │
  │ 4 < 5 and 4 < 10 → done                    │
  └─────────────────────────────────────────────┘

  Final min heap:               Array: [1, 4, 3, 5, 10, 8, 7]
          1(0)
        /      \
       4(1)     3(2)
     /   \     /   \
    5(3) 10(4) 8(5) 7(6)

  Verify: 1≤4 ✓  1≤3 ✓  4≤5 ✓  4≤10 ✓  3≤8 ✓  3≤7 ✓
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

**Textual Figure:**

```
Input: [4, 10, 3, 5, 1, 8, 7]

  Initial tree:                 Array: [4, 10, 3, 5, 1, 8, 7]
          4(0)
        /      \
      10(1)     3(2)
     /   \     /   \
    5(3) 1(4) 8(5) 7(6)

  Step 1: siftDown(i=2, val=3)
  ┌─────────────────────────────────────────────┐
  │ Children: 8(5), 7(6) → largest = 8          │
  │ Swap 3 ↔ 8                                  │
  └─────────────────────────────────────────────┘
          4                     Array: [4, 10, 8, 5, 1, 3, 7]
        /      \
      10        8
     /   \     /   \
    5     1   3     7

  Step 2: siftDown(i=1, val=10)
  ┌─────────────────────────────────────────────┐
  │ Children: 5(3), 1(4)                        │
  │ 10 > 5 and 10 > 1 → already largest → done  │
  └─────────────────────────────────────────────┘

  Step 3: siftDown(i=0, val=4)
  ┌─────────────────────────────────────────────┐
  │ Children: 10(1), 8(2) → largest = 10        │
  │ Swap 4 ↔ 10                                 │
  │ Continue at i=1: children 5(3), 1(4)        │
  │ 4 < 5 → Swap 4 ↔ 5                         │
  └─────────────────────────────────────────────┘

  Final max heap:               Array: [10, 5, 8, 4, 1, 3, 7]
         10(0)
        /      \
       5(1)     8(2)
     /   \     /   \
    4(3) 1(4) 3(5) 7(6)

  Verify: 10≥5 ✓  10≥8 ✓  5≥4 ✓  5≥1 ✓  8≥3 ✓  8≥7 ✓
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

**Textual Figure:**

```
Input: [9, 5, 7, 1, 3, 8, 2, 6, 4]

  Before heap.Init:             Array: [9, 5, 7, 1, 3, 8, 2, 6, 4]
            9(0)
          /       \
        5(1)       7(2)
       /    \     /    \
      1(3)  3(4) 8(5)  2(6)
     /   \
    6(7) 4(8)

  heap.Init → bottom-up heapify (O(n)):
  ┌──────────────────────────────────────┐
  │ Process nodes from i=3 down to i=0  │
  │ i=3: val=1, children 6,4 → no swap  │
  │ i=2: val=7, children 8,2 → swap 7↔2 │
  │ i=1: val=5, children 1,3 → swap 5↔1 │
  │ i=0: val=9, children 1,2 → swap 9↔1 │
  │       then 9 at i=1 → swap 9↔3      │
  └──────────────────────────────────────┘

  After heap.Init (min heap):   Array: [1, 3, 2, 4, 5, 8, 7, 6, 9]
            1(0)
          /       \
        3(1)       2(2)
       /    \     /    \
      4(3)  5(4) 8(5)  7(6)
     /   \
    6(7) 9(8)

  Pop order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9
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

**Textual Figure:**

```
Top-Down vs Bottom-Up Heapify Comparison

  Top-Down (insert one by one, sift UP):
  ┌─────────────────────────────────────────────┐
  │ Insert 4:    [4]                             │
  │ Insert 10:   [4,10]     → sift up, no swap  │
  │ Insert 3:    [3,10,4]   → sift up, swap 3↔4 │
  │ Insert 5:    [3,5,4,10] → sift up, swap 5↔10│
  │ Insert 1:    sift up through multiple levels │
  │ ...each insert = O(log n) → total O(n log n)│
  └─────────────────────────────────────────────┘

  Bottom-Up (start from last internal, sift DOWN):
  ┌─────────────────────────────────────────────┐
  │ Place all into tree, then fix from bottom:   │
  │   Level 3 (leaves): 0 work    ← n/2 nodes!  │
  │   Level 2: sift down 1 level  ← n/4 nodes   │
  │   Level 1: sift down 2 levels ← n/8 nodes   │
  │   Level 0: sift down 3 levels ← 1 node      │
  │ Total = O(n) — most nodes do minimal work    │
  └─────────────────────────────────────────────┘

  Work distribution (n=1,000,000):
  ┌──────────┬───────────┬──────────┐
  │  Level   │   Nodes   │ Sift Dist│
  ├──────────┼───────────┼──────────┤
  │ Leaves   │  500,000  │    0     │
  │ Level h-1│  250,000  │    1     │
  │ Level h-2│  125,000  │    2     │
  │   ...    │    ...    │   ...    │
  │ Root     │     1     │    20    │
  └──────────┴───────────┴──────────┘
  Bottom-up is typically 2-3x faster in practice.
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

**Textual Figure:**

```
Input: [9, 7, 5, 3, 1, 8, 6, 4, 2]    k=5 (heapify first 5 only)

  Full array as tree:
            9(0)
          /       \
        7(1)       5(2)
       /    \     /    \
      3(3)  1(4) 8(5)  6(6)
     /   \
    4(7) 2(8)

  Partial heapify: only indices 0..4
  ┌─────────────────────────────────────────┐
  │ k=5, last internal = k/2-1 = 1         │
  │                                         │
  │ i=1: val=7, children 3(3),1(4)          │
  │   smallest=1 → Swap 7↔1                │
  │                                         │
  │ i=0: val=9, children 1(1),5(2)          │
  │   smallest=1 → Swap 9↔1                │
  │   continue i=1: children 7(3),9→no 4<5 │
  │   smallest=3(val=3)? → Swap 9↔3         │
  └─────────────────────────────────────────┘

  Result:   [1, 3, 5, 9, 7,│ 8, 6, 4, 2]
             ├─heapified──┤  ├─unchanged─┤

  Heapified portion as tree:
       1(0)
      /     \
    3(1)    5(2)
   /    \
  9(3)  7(4)

  Use case: When you only need top-k from first k elements.
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

**Textual Figure:**

```
Input: [8, 5, 3, 10, 2, 7, 1]

  Initial tree:                 Array: [8, 5, 3, 10, 2, 7, 1]
          8(0)
        /      \
       5(1)     3(2)
      /   \    /   \
    10(3) 2(4) 7(5) 1(6)

  siftDown i=2 (val=3):
  ┌──────────────────────────────────────┐
  │  3(2)       children: 7(5), 1(6)   │
  │   \                                │
  │    → Swap 3 ↔ 1 (smallest child)   │
  │  Result: [8, 5, 1, 10, 2, 7, 3]    │
  └──────────────────────────────────────┘

  siftDown i=1 (val=5):
  ┌──────────────────────────────────────┐
  │  5(1)       children: 10(3), 2(4)  │
  │   \                                │
  │    → Swap 5 ↔ 2 (smallest child)   │
  │  Result: [8, 2, 1, 10, 5, 7, 3]    │
  └──────────────────────────────────────┘

  siftDown i=0 (val=8):
  ┌──────────────────────────────────────┐
  │  8(0)       children: 2(1), 1(2)   │
  │   → Swap 8 ↔ 1                     │
  │  Now 8 at i=2: children 7(5),3(6)  │
  │   → Swap 8 ↔ 3                     │
  │  Result: [1, 2, 3, 10, 5, 7, 8]    │
  └──────────────────────────────────────┘

  Final min heap:               Array: [1, 2, 3, 10, 5, 7, 8]
          1(0)
        /      \
       2(1)     3(2)
      /   \    /   \
    10(3) 5(4) 7(5) 8(6)
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

**Textual Figure:**

```
Input: [3, 7, 2, 11, 5, 1, 9, 8]    k=3

  Step 1: Build max heap
           11(0)
         /       \
        8(1)      9(2)
       /   \     /   \
      7(3) 5(4) 1(5) 2(6)
     /
    3(7)
  Array: [11, 8, 9, 7, 5, 1, 2, 3]

  Step 2: Extract top k=3
  ┌────────────────────────────────────────────┐
  │ Extract 1: root=11                        │
  │   Move last(3) to root → siftDown         │
  │       9                                    │
  │      / \                                   │
  │     8   3  → [9, 8, 3, 7, 5, 1, 2]        │
  │    / \  /                                  │
  │   7  5 1                                   │
  │                                            │
  │ Extract 2: root=9                          │
  │   Move last(2) → siftDown                  │
  │   [8, 7, 3, 2, 5, 1]                       │
  │                                            │
  │ Extract 3: root=8                          │
  └────────────────────────────────────────────┘

  Result: [11, 9, 8]   (top 3 largest)
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

**Textual Figure:**

```
Verify Heap Property — check parent vs children at every node:

  Test: isMinHeap([1, 3, 2, 7, 4]) → true
       1(0)             1≤3 ✓  1≤2 ✓
      / \               3≤7 ✓  3≤4 ✓
     3   2
    / \
   7   4

  Test: isMinHeap([1, 0, 2, 7, 4]) → false
       1(0)             1≤0? ✘ (0 < 1)
      / \
     0   2              ← child < parent!
    / \
   7   4

  Test: isMaxHeap([9, 7, 8, 3, 5]) → true
       9(0)             9≥7 ✓  9≥8 ✓
      / \               7≥3 ✓  7≥5 ✓
     7   8
    / \
   3   5

  Test: isMaxHeap([9, 10, 8, 3, 5]) → false
       9(0)             9≥10? ✘ (10 > 9)
      / \
    10   8              ← child > parent!
    / \
   3   5
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

**Textual Figure:**

```
Input tasks: [("debug",3), ("deploy",1), ("test",2), ("review",5)]
Comparator: a.priority < b.priority (min-priority first)

  After heap.Init (min-priority heap):

         (deploy,1)
        /          \
    (debug,3)    (test,2)
       /
  (review,5)

  Array: [(deploy,1), (debug,3), (test,2), (review,5)]

  Pop order:
  ┌─────┬───────────┬──────────┐
  │ Pop │   Task    │ Priority │
  ├─────┼───────────┼──────────┤
  │  1  │ deploy    │    1     │
  │  2  │ test      │    2     │
  │  3  │ debug     │    3     │
  │  4  │ review    │    5     │
  └─────┴───────────┴──────────┘
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

**Textual Figure:**

```
Why Bottom-Up Heapify is O(n):

  Heap of height h = log₂(n):

  Level 0 (root):     ○                    1 node  × sift h levels
                     / \
  Level 1:          ○   ○                  2 nodes × sift h-1 levels
                   /\ /\
  Level 2:        ○ ○ ○ ○               4 nodes × sift h-2 levels
                  ...   ...
  Level h-1:    ○○○○○○○○             n/4 nodes × sift 1 level
  Level h:      ●●●●●●●●●●●●●●●●   n/2 nodes × sift 0 levels!

  Total work = Σ(k=0→h) 2ᵏ × (h-k)
  ┌───────────┬────────┬─────────────┬─────────┐
  │   n       │   h    │  total ops  │  ratio  │
  ├───────────┼────────┼─────────────┼─────────┤
  │     7     │   2    │      4      │  0.57   │
  │    15     │   3    │     11      │  0.73   │
  │    31     │   4    │     26      │  0.84   │
  │    63     │   5    │     57      │  0.90   │
  │   127     │   6    │    120      │  0.94   │
  └───────────┴────────┴─────────────┴─────────┘
  ratio → 1.0 as n → ∞  ∴ Total = O(n)

  Key: n/2 leaves do ZERO work!
```

---

## Key Takeaways

1. **Bottom-up heapify is O(n)** — start from last internal node `n/2-1`, sift down each
2. Top-down (repeated insert) is O(n log n) — always prefer bottom-up
3. `heap.Init()` in Go uses bottom-up heapify
4. Most nodes are leaves and require zero work — that's why it's linear
5. Always verify with `isMinHeap` / `isMaxHeap` during debugging

> **Next up:** Heap Operations →
