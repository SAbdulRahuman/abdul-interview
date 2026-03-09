# Phase 11: Heap / Priority Queue — K-Largest Problems

## Overview

A family of problems asking for the **K-th largest**, **top K**, or **K-th smallest** elements. Two main approaches:

| Approach | Time | Space | Best When |
|----------|------|-------|-----------|
| Min heap of size k | O(n log k) | O(k) | Streaming data, k << n |
| Max heap (full) | O(n + k log n) | O(n) | k is close to n |
| QuickSelect | O(n) avg | O(1) | One-shot, no streaming |

---

## Example 1: Kth Largest Element (LeetCode 215)

```go
package main

import (
	"container/heap"
	"fmt"
)

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func findKthLargest(nums []int, k int) int {
	h := &MinHeap{}
	for _, n := range nums {
		heap.Push(h, n)
		if h.Len() > k {
			heap.Pop(h) // evict smallest
		}
	}
	return (*h)[0] // min of top-k = kth largest
}

func main() {
	fmt.Println(findKthLargest([]int{3, 2, 1, 5, 6, 4}, 2))    // 5
	fmt.Println(findKthLargest([]int{3, 2, 3, 1, 2, 4, 5, 5, 6}, 4)) // 4
}
```

---

## Example 2: Kth Largest Element in a Stream (LeetCode 703)

```go
package main

import (
	"container/heap"
	"fmt"
)

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

type KthLargest struct {
	k    int
	heap *MinHeap
}

func Constructor(k int, nums []int) KthLargest {
	h := &MinHeap{}
	kl := KthLargest{k: k, heap: h}
	for _, n := range nums {
		kl.Add(n)
	}
	return kl
}

func (kl *KthLargest) Add(val int) int {
	heap.Push(kl.heap, val)
	if kl.heap.Len() > kl.k {
		heap.Pop(kl.heap)
	}
	return (*kl.heap)[0]
}

func main() {
	kl := Constructor(3, []int{4, 5, 8, 2})
	fmt.Println(kl.Add(3))  // 4
	fmt.Println(kl.Add(5))  // 5
	fmt.Println(kl.Add(10)) // 5
	fmt.Println(kl.Add(9))  // 8
	fmt.Println(kl.Add(4))  // 8
}
```

---

## Example 3: Top K Frequent Elements (LeetCode 347)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Pair struct{ val, freq int }
type MinFreqHeap []Pair
func (h MinFreqHeap) Len() int           { return len(h) }
func (h MinFreqHeap) Less(i, j int) bool { return h[i].freq < h[j].freq }
func (h MinFreqHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinFreqHeap) Push(x interface{}) { *h = append(*h, x.(Pair)) }
func (h *MinFreqHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func topKFrequent(nums []int, k int) []int {
	freq := map[int]int{}
	for _, n := range nums { freq[n]++ }

	h := &MinFreqHeap{}
	for v, f := range freq {
		heap.Push(h, Pair{v, f})
		if h.Len() > k {
			heap.Pop(h)
		}
	}

	result := make([]int, h.Len())
	for i := h.Len() - 1; i >= 0; i-- {
		result[i] = heap.Pop(h).(Pair).val
	}
	return result
}

func main() {
	fmt.Println(topKFrequent([]int{1, 1, 1, 2, 2, 3}, 2))       // [2 1]
	fmt.Println(topKFrequent([]int{1}, 1))                         // [1]
	fmt.Println(topKFrequent([]int{4, 1, -1, 2, -1, 2, 3}, 2))   // [-1 2]
}
```

---

## Example 4: K Closest Points to Origin (LeetCode 973)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Point struct{ x, y, dist int }
type MaxDistHeap []Point
func (h MaxDistHeap) Len() int           { return len(h) }
func (h MaxDistHeap) Less(i, j int) bool { return h[i].dist > h[j].dist } // max heap
func (h MaxDistHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxDistHeap) Push(x interface{}) { *h = append(*h, x.(Point)) }
func (h *MaxDistHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func kClosest(points [][]int, k int) [][]int {
	h := &MaxDistHeap{}
	for _, p := range points {
		d := p[0]*p[0] + p[1]*p[1]
		heap.Push(h, Point{p[0], p[1], d})
		if h.Len() > k {
			heap.Pop(h) // remove farthest
		}
	}
	result := make([][]int, h.Len())
	for i := h.Len() - 1; i >= 0; i-- {
		p := heap.Pop(h).(Point)
		result[i] = []int{p.x, p.y}
	}
	return result
}

func main() {
	fmt.Println(kClosest([][]int{{1, 3}, {-2, 2}}, 1))           // [[-2 2]]
	fmt.Println(kClosest([][]int{{3, 3}, {5, -1}, {-2, 4}}, 2))  // [[3 3] [-2 4]]
}
```

---

## Example 5: K Closest Elements (LeetCode 658)

```go
package main

import (
	"container/heap"
	"fmt"
	"sort"
)

type Item struct{ val, diff int }
type MaxDiffHeap []Item
func (h MaxDiffHeap) Len() int           { return len(h) }
func (h MaxDiffHeap) Less(i, j int) bool {
	if h[i].diff != h[j].diff { return h[i].diff > h[j].diff }
	return h[i].val > h[j].val // larger val removed first on tie
}
func (h MaxDiffHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxDiffHeap) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MaxDiffHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func abs(a int) int { if a < 0 { return -a }; return a }

func findClosestElements(arr []int, k int, x int) []int {
	h := &MaxDiffHeap{}
	for _, v := range arr {
		heap.Push(h, Item{v, abs(v - x)})
		if h.Len() > k {
			heap.Pop(h)
		}
	}
	result := make([]int, h.Len())
	for i := h.Len() - 1; i >= 0; i-- {
		result[i] = heap.Pop(h).(Item).val
	}
	sort.Ints(result)
	return result
}

func main() {
	fmt.Println(findClosestElements([]int{1, 2, 3, 4, 5}, 4, 3))      // [1 2 3 4]
	fmt.Println(findClosestElements([]int{1, 2, 3, 4, 5}, 4, -1))     // [1 2 3 4]
}
```

---

## Example 6: Top K Frequent Words (LeetCode 692)

```go
package main

import (
	"container/heap"
	"fmt"
)

type WordFreq struct{ word string; freq int }
type WFHeap []WordFreq
func (h WFHeap) Len() int { return len(h) }
func (h WFHeap) Less(i, j int) bool {
	if h[i].freq != h[j].freq { return h[i].freq < h[j].freq }
	return h[i].word > h[j].word // higher lexicographic = lower priority
}
func (h WFHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *WFHeap) Push(x interface{}) { *h = append(*h, x.(WordFreq)) }
func (h *WFHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func topKFrequentWords(words []string, k int) []string {
	freq := map[string]int{}
	for _, w := range words { freq[w]++ }

	h := &WFHeap{}
	for w, f := range freq {
		heap.Push(h, WordFreq{w, f})
		if h.Len() > k {
			heap.Pop(h)
		}
	}
	
	result := make([]string, k)
	for i := k - 1; i >= 0; i-- {
		result[i] = heap.Pop(h).(WordFreq).word
	}
	return result
}

func main() {
	fmt.Println(topKFrequentWords([]string{"i", "love", "leetcode", "i", "love", "coding"}, 2))
	// ["i", "love"]
	fmt.Println(topKFrequentWords([]string{"the", "day", "is", "sunny", "the", "the", "the", "sunny", "is", "is"}, 4))
	// ["the", "is", "sunny", "day"]
}
```

---

## Example 7: K Largest in Each Subarray

```go
package main

import (
	"container/heap"
	"fmt"
)

type MaxHeap []int
func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func kLargestInSubarrays(nums []int, windowSize, k int) [][]int {
	var result [][]int
	for i := 0; i+windowSize <= len(nums); i++ {
		h := &MaxHeap{}
		for j := i; j < i+windowSize; j++ {
			heap.Push(h, nums[j])
		}
		topK := make([]int, k)
		for j := 0; j < k; j++ {
			topK[j] = heap.Pop(h).(int)
		}
		result = append(result, topK)
	}
	return result
}

func main() {
	nums := []int{1, 5, 2, 8, 3, 7, 4, 6}
	fmt.Println(kLargestInSubarrays(nums, 4, 2))
	// [[5 2] [8 5] [8 7] [8 7] [7 6]]
}
```

---

## Example 8: K Smallest Pairs Sum (LeetCode 373)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Pair struct{ i, j, sum int }
type PairHeap []Pair
func (h PairHeap) Len() int           { return len(h) }
func (h PairHeap) Less(a, b int) bool { return h[a].sum < h[b].sum }
func (h PairHeap) Swap(a, b int)      { h[a], h[b] = h[b], h[a] }
func (h *PairHeap) Push(x interface{}) { *h = append(*h, x.(Pair)) }
func (h *PairHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func kSmallestPairs(nums1, nums2 []int, k int) [][]int {
	if len(nums1) == 0 || len(nums2) == 0 {
		return nil
	}
	h := &PairHeap{}
	for i := 0; i < len(nums1) && i < k; i++ {
		heap.Push(h, Pair{i, 0, nums1[i] + nums2[0]})
	}
	var result [][]int
	for h.Len() > 0 && len(result) < k {
		p := heap.Pop(h).(Pair)
		result = append(result, []int{nums1[p.i], nums2[p.j]})
		if p.j+1 < len(nums2) {
			heap.Push(h, Pair{p.i, p.j + 1, nums1[p.i] + nums2[p.j+1]})
		}
	}
	return result
}

func main() {
	fmt.Println(kSmallestPairs([]int{1, 7, 11}, []int{2, 4, 6}, 3))
	// [[1 2] [1 4] [1 6]]
	fmt.Println(kSmallestPairs([]int{1, 1, 2}, []int{1, 2, 3}, 2))
	// [[1 1] [1 1]]
}
```

---

## Example 9: K-th Largest XOR Coordinate Value (LeetCode 1738)

```go
package main

import (
	"container/heap"
	"fmt"
)

type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func kthLargestValue(matrix [][]int, k int) int {
	m, n := len(matrix), len(matrix[0])
	pre := make([][]int, m+1)
	for i := range pre { pre[i] = make([]int, n+1) }

	h := &MinHeap{}
	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			pre[i][j] = pre[i-1][j] ^ pre[i][j-1] ^ pre[i-1][j-1] ^ matrix[i-1][j-1]
			heap.Push(h, pre[i][j])
			if h.Len() > k {
				heap.Pop(h)
			}
		}
	}
	return (*h)[0]
}

func main() {
	matrix := [][]int{{5, 2}, {1, 6}}
	fmt.Println(kthLargestValue(matrix, 1)) // 7
	fmt.Println(kthLargestValue(matrix, 2)) // 5
}
```

---

## Example 10: Pattern Summary — When to Use Which Heap

```go
package main

import "fmt"

func main() {
	fmt.Println("=== K-Largest Problem Patterns ===")
	fmt.Println()

	patterns := []struct {
		problem    string
		heapType   string
		heapSize   string
		rationale  string
	}{
		{"Kth largest", "Min heap", "k", "Root = kth largest; evict smaller"},
		{"Kth smallest", "Max heap", "k", "Root = kth smallest; evict larger"},
		{"Top K frequent", "Min heap (freq)", "k", "Evict least frequent"},
		{"K closest", "Max heap (dist)", "k", "Evict farthest"},
		{"K smallest pairs", "Min heap (sum)", "varies", "BFS-like expansion"},
		{"Streaming Kth", "Min heap", "k", "Add + evict per element"},
	}

	for _, p := range patterns {
		fmt.Printf("%-20s → %-18s size=%-6s %s\n",
			p.problem, p.heapType, p.heapSize, p.rationale)
	}

	fmt.Println()
	fmt.Println("Rule of thumb:")
	fmt.Println("  For K LARGEST  → use MIN heap of size k (evict the smallest)")
	fmt.Println("  For K SMALLEST → use MAX heap of size k (evict the largest)")
	fmt.Println("  This is counterintuitive but correct!")
}
```

---

## Key Takeaways

1. **K largest → min heap of size k**: root is the kth largest; evict smaller elements
2. **K smallest → max heap of size k**: root is the kth smallest; evict larger elements
3. Time: O(n log k) — better than sorting O(n log n) when k << n
4. For streaming data, bounded heap is the only viable approach
5. Custom `Less()` handles multi-key sorting (frequency + lexicographic, etc.)

> **Next up:** Two Heap Pattern →
