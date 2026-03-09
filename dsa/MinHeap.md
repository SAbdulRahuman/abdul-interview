# Phase 11: Heap / Priority Queue — Min Heap

## Overview

A **min heap** is a complete binary tree where every parent is ≤ its children. The root always holds the **minimum** element.

```
       1
      / \
     3   2
    / \
   7   4

Property: parent ≤ children at every level
```

**Operations:**
| Operation | Time |
|-----------|------|
| Insert | O(log n) |
| Extract Min | O(log n) |
| Peek Min | O(1) |
| Build Heap | O(n) |

Go's `container/heap` package provides the heap interface.

---

## Example 1: Min Heap from Scratch

```go
package main

import "fmt"

type MinHeap struct {
	data []int
}

func (h *MinHeap) Push(val int) {
	h.data = append(h.data, val)
	h.siftUp(len(h.data) - 1)
}

func (h *MinHeap) Pop() int {
	n := len(h.data)
	min := h.data[0]
	h.data[0] = h.data[n-1]
	h.data = h.data[:n-1]
	if len(h.data) > 0 {
		h.siftDown(0)
	}
	return min
}

func (h *MinHeap) Peek() int {
	return h.data[0]
}

func (h *MinHeap) siftUp(i int) {
	for i > 0 {
		parent := (i - 1) / 2
		if h.data[i] < h.data[parent] {
			h.data[i], h.data[parent] = h.data[parent], h.data[i]
			i = parent
		} else {
			break
		}
	}
}

func (h *MinHeap) siftDown(i int) {
	n := len(h.data)
	for {
		smallest := i
		left, right := 2*i+1, 2*i+2
		if left < n && h.data[left] < h.data[smallest] {
			smallest = left
		}
		if right < n && h.data[right] < h.data[smallest] {
			smallest = right
		}
		if smallest == i {
			break
		}
		h.data[i], h.data[smallest] = h.data[smallest], h.data[i]
		i = smallest
	}
}

func main() {
	h := &MinHeap{}
	for _, v := range []int{5, 3, 8, 1, 2, 7} {
		h.Push(v)
	}
	for len(h.data) > 0 {
		fmt.Print(h.Pop(), " ") // 1 2 3 5 7 8
	}
	fmt.Println()
}
```

---

## Example 2: Min Heap Using container/heap

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

func (h *IntHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[:n-1]
	return x
}

func main() {
	h := &IntHeap{5, 3, 8, 1, 2, 7}
	heap.Init(h)

	heap.Push(h, 0)
	fmt.Println("Min:", (*h)[0]) // 0

	for h.Len() > 0 {
		fmt.Print(heap.Pop(h), " ") // 0 1 2 3 5 7 8
	}
	fmt.Println()
}
```

---

## Example 3: Kth Smallest Element

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

func kthSmallest(nums []int, k int) int {
	h := &IntHeap{}
	heap.Init(h)
	for _, n := range nums {
		heap.Push(h, n)
	}
	var result int
	for i := 0; i < k; i++ {
		result = heap.Pop(h).(int)
	}
	return result
}

func main() {
	nums := []int{7, 10, 4, 3, 20, 15}
	fmt.Println("3rd smallest:", kthSmallest(nums, 3)) // 7
	fmt.Println("1st smallest:", kthSmallest(nums, 1)) // 3
}
```

---

## Example 4: Merge K Sorted Lists (LeetCode 23)

```go
package main

import (
	"container/heap"
	"fmt"
)

type ListNode struct {
	Val  int
	Next *ListNode
}

type NodeHeap []*ListNode

func (h NodeHeap) Len() int           { return len(h) }
func (h NodeHeap) Less(i, j int) bool { return h[i].Val < h[j].Val }
func (h NodeHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *NodeHeap) Push(x interface{}) { *h = append(*h, x.(*ListNode)) }
func (h *NodeHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func mergeKLists(lists []*ListNode) *ListNode {
	h := &NodeHeap{}
	heap.Init(h)
	for _, l := range lists {
		if l != nil {
			heap.Push(h, l)
		}
	}
	dummy := &ListNode{}
	cur := dummy
	for h.Len() > 0 {
		node := heap.Pop(h).(*ListNode)
		cur.Next = node
		cur = cur.Next
		if node.Next != nil {
			heap.Push(h, node.Next)
		}
	}
	return dummy.Next
}

func main() {
	// [1->4->5, 1->3->4, 2->6]
	l1 := &ListNode{1, &ListNode{4, &ListNode{5, nil}}}
	l2 := &ListNode{1, &ListNode{3, &ListNode{4, nil}}}
	l3 := &ListNode{2, &ListNode{6, nil}}

	merged := mergeKLists([]*ListNode{l1, l2, l3})
	for merged != nil {
		fmt.Print(merged.Val, " ") // 1 1 2 3 4 4 5 6
		merged = merged.Next
	}
	fmt.Println()
}
```

---

## Example 5: Last Stone Weight (LeetCode 1046) Using Min Heap Trick

```go
package main

import (
	"container/heap"
	"fmt"
)

// Use min heap with negated values to simulate max heap
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func lastStoneWeight(stones []int) int {
	h := &IntHeap{}
	for _, s := range stones {
		heap.Push(h, -s) // negate for max heap behavior
	}
	for h.Len() > 1 {
		y := -heap.Pop(h).(int)
		x := -heap.Pop(h).(int)
		if y != x {
			heap.Push(h, -(y - x))
		}
	}
	if h.Len() == 0 {
		return 0
	}
	return -heap.Pop(h).(int)
}

func main() {
	fmt.Println(lastStoneWeight([]int{2, 7, 4, 1, 8, 1})) // 1
}
```

---

## Example 6: Sort Nearly Sorted Array (K-Sorted)

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
	arr := []int{6, 5, 3, 2, 8, 10, 9}
	fmt.Println(sortKSorted(arr, 3)) // [2 3 5 6 8 9 10]
}
```

---

## Example 7: Meeting Rooms II — Min Rooms (LeetCode 253)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minMeetingRooms(intervals [][]int) int {
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})
	h := &IntHeap{}
	heap.Init(h)
	for _, iv := range intervals {
		if h.Len() > 0 && (*h)[0] <= iv[0] {
			heap.Pop(h) // free the room
		}
		heap.Push(h, iv[1])
	}
	return h.Len()
}

func main() {
	fmt.Println(minMeetingRooms([][]int{{0, 30}, {5, 10}, {15, 20}})) // 2
	fmt.Println(minMeetingRooms([][]int{{7, 10}, {2, 4}}))             // 1
}
```

---

## Example 8: Sliding Window Maximum Using Deque (Heap-based)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Item struct {
	val, idx int
}
type MaxHeap []Item
func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i].val > h[j].val }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MaxHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func maxSlidingWindow(nums []int, k int) []int {
	h := &MaxHeap{}
	result := []int{}
	for i, v := range nums {
		heap.Push(h, Item{v, i})
		if i >= k-1 {
			for (*h)[0].idx <= i-k {
				heap.Pop(h)
			}
			result = append(result, (*h)[0].val)
		}
	}
	return result
}

func main() {
	fmt.Println(maxSlidingWindow([]int{1, 3, -1, -3, 5, 3, 6, 7}, 3))
	// [3 3 5 5 6 7]
}
```

---

## Example 9: K Closest Points to Origin (LeetCode 973)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Point struct {
	x, y, dist int
}
type PointHeap []Point
func (h PointHeap) Len() int           { return len(h) }
func (h PointHeap) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h PointHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PointHeap) Push(x interface{}) { *h = append(*h, x.(Point)) }
func (h *PointHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func kClosest(points [][]int, k int) [][]int {
	h := &PointHeap{}
	for _, p := range points {
		heap.Push(h, Point{p[0], p[1], p[0]*p[0] + p[1]*p[1]})
	}
	result := make([][]int, k)
	for i := 0; i < k; i++ {
		p := heap.Pop(h).(Point)
		result[i] = []int{p.x, p.y}
	}
	return result
}

func main() {
	fmt.Println(kClosest([][]int{{1, 3}, {-2, 2}}, 1))                 // [[-2 2]]
	fmt.Println(kClosest([][]int{{3, 3}, {5, -1}, {-2, 4}}, 2)) // [[3 3] [-2 4]]
}
```

---

## Example 10: Continuous Median (Running Median with Min Heap)

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
	// Use min heap to find k smallest after each insertion
	stream := []int{5, 15, 1, 3, 2, 8, 7, 9, 10, 6, 11, 4}
	h := &IntHeap{}
	
	for _, v := range stream {
		heap.Push(h, v)
		// Current minimum
		fmt.Printf("Added %2d → min = %2d, size = %d\n", v, (*h)[0], h.Len())
	}
}
```

---

## Key Takeaways

1. Min heap: parent ≤ children; root = minimum element
2. Go's `container/heap` requires implementing 5 methods: Len, Less, Swap, Push, Pop
3. `heap.Init` builds a heap in O(n); Push/Pop are O(log n)
4. Array representation: parent at `(i-1)/2`, children at `2i+1`, `2i+2`
5. Negate values in min heap to simulate max heap behavior

> **Next up:** Max Heap →
