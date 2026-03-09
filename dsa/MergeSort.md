# Phase 16: Sorting Algorithms — Merge Sort

## Overview

**Merge Sort** divides the array in half, recursively sorts both halves, and merges them. It guarantees O(n log n) time and is **stable**.

| Aspect | Detail |
|--------|--------|
| **Time** | O(n log n) — always |
| **Space** | O(n) auxiliary |
| **Stable** | Yes |
| **Approach** | Divide and Conquer |

---

## Example 1: Classic Merge Sort

```go
package main

import "fmt"

func mergeSort(arr []int) []int {
	if len(arr) <= 1 { return arr }

	mid := len(arr) / 2
	left := mergeSort(arr[:mid])
	right := mergeSort(arr[mid:])
	return merge(left, right)
}

func merge(left, right []int) []int {
	result := make([]int, 0, len(left)+len(right))
	i, j := 0, 0

	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			result = append(result, left[i])
			i++
		} else {
			result = append(result, right[j])
			j++
		}
	}
	result = append(result, left[i:]...)
	result = append(result, right[j:]...)
	return result
}

func main() {
	arr := []int{38, 27, 43, 3, 9, 82, 10}
	fmt.Println(mergeSort(arr)) // [3 9 10 27 38 43 82]
}
```

---

## Example 2: In-Place Merge Sort (Minimal Extra Space)

```go
package main

import "fmt"

func mergeSortInPlace(arr []int, left, right int) {
	if left >= right { return }
	mid := (left + right) / 2
	mergeSortInPlace(arr, left, mid)
	mergeSortInPlace(arr, mid+1, right)
	mergeInPlace(arr, left, mid, right)
}

func mergeInPlace(arr []int, left, mid, right int) {
	temp := make([]int, right-left+1)
	i, j, k := left, mid+1, 0

	for i <= mid && j <= right {
		if arr[i] <= arr[j] {
			temp[k] = arr[i]; i++
		} else {
			temp[k] = arr[j]; j++
		}
		k++
	}
	for i <= mid { temp[k] = arr[i]; i++; k++ }
	for j <= right { temp[k] = arr[j]; j++; k++ }
	copy(arr[left:right+1], temp)
}

func main() {
	arr := []int{12, 11, 13, 5, 6, 7}
	mergeSortInPlace(arr, 0, len(arr)-1)
	fmt.Println(arr) // [5 6 7 11 12 13]
}
```

---

## Example 3: Bottom-Up Merge Sort (Iterative)

```go
package main

import "fmt"

func mergeSortBottomUp(arr []int) {
	n := len(arr)
	temp := make([]int, n)

	for width := 1; width < n; width *= 2 {
		for i := 0; i < n; i += 2 * width {
			left := i
			mid := min(i+width, n)
			right := min(i+2*width, n)

			// Merge arr[left:mid] and arr[mid:right]
			l, r, k := left, mid, left
			for l < mid && r < right {
				if arr[l] <= arr[r] {
					temp[k] = arr[l]; l++
				} else {
					temp[k] = arr[r]; r++
				}
				k++
			}
			for l < mid { temp[k] = arr[l]; l++; k++ }
			for r < right { temp[k] = arr[r]; r++; k++ }
		}
		copy(arr, temp)
	}
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	arr := []int{5, 2, 4, 7, 1, 3, 2, 6}
	mergeSortBottomUp(arr)
	fmt.Println(arr) // [1 2 2 3 4 5 6 7]
}
```

---

## Example 4: Sort Linked List (LeetCode 148)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func sortList(head *ListNode) *ListNode {
	if head == nil || head.Next == nil { return head }

	// Find middle using slow/fast pointers
	slow, fast := head, head.Next
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}

	mid := slow.Next
	slow.Next = nil

	left := sortList(head)
	right := sortList(mid)
	return mergeLists(left, right)
}

func mergeLists(a, b *ListNode) *ListNode {
	dummy := &ListNode{}
	curr := dummy
	for a != nil && b != nil {
		if a.Val <= b.Val {
			curr.Next = a; a = a.Next
		} else {
			curr.Next = b; b = b.Next
		}
		curr = curr.Next
	}
	if a != nil { curr.Next = a }
	if b != nil { curr.Next = b }
	return dummy.Next
}

func printList(head *ListNode) {
	for head != nil {
		fmt.Printf("%d ", head.Val)
		head = head.Next
	}
	fmt.Println()
}

func main() {
	head := &ListNode{4, &ListNode{2, &ListNode{1, &ListNode{3, nil}}}}
	sorted := sortList(head)
	printList(sorted) // 1 2 3 4
}
```

---

## Example 5: Count Inversions Using Merge Sort

```go
package main

import "fmt"

func mergeSortCount(arr []int) ([]int, int) {
	if len(arr) <= 1 { return arr, 0 }

	mid := len(arr) / 2
	left, lInv := mergeSortCount(arr[:mid])
	right, rInv := mergeSortCount(arr[mid:])
	merged, mInv := mergeCount(left, right)
	return merged, lInv + rInv + mInv
}

func mergeCount(left, right []int) ([]int, int) {
	result := make([]int, 0, len(left)+len(right))
	inversions := 0
	i, j := 0, 0

	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			result = append(result, left[i]); i++
		} else {
			result = append(result, right[j]); j++
			inversions += len(left) - i
		}
	}
	result = append(result, left[i:]...)
	result = append(result, right[j:]...)
	return result, inversions
}

func main() {
	arr := []int{2, 4, 1, 3, 5}
	_, inv := mergeSortCount(arr)
	fmt.Println("Inversions:", inv) // 3

	_, inv2 := mergeSortCount([]int{5, 4, 3, 2, 1})
	fmt.Println("Inversions (reversed):", inv2) // 10
}
```

---

## Example 6: Natural Merge Sort (Exploit Existing Runs)

```go
package main

import "fmt"

func naturalMergeSort(arr []int) {
	n := len(arr)
	if n <= 1 { return }

	for {
		merges := 0
		i := 0
		for i < n {
			// Find first run
			start1 := i
			for i < n-1 && arr[i] <= arr[i+1] { i++ }
			end1 := i
			i++

			if i >= n { break } // single run left

			// Find second run
			start2 := i
			for i < n-1 && arr[i] <= arr[i+1] { i++ }
			end2 := i
			i++

			// Merge two runs
			mergeRange(arr, start1, end1, start2, end2)
			merges++
		}
		if merges <= 0 { break } // single run = sorted
	}
}

func mergeRange(arr []int, s1, e1, s2, e2 int) {
	temp := make([]int, e2-s1+1)
	i, j, k := s1, s2, 0
	for i <= e1 && j <= e2 {
		if arr[i] <= arr[j] { temp[k] = arr[i]; i++ } else { temp[k] = arr[j]; j++ }
		k++
	}
	for i <= e1 { temp[k] = arr[i]; i++; k++ }
	for j <= e2 { temp[k] = arr[j]; j++; k++ }
	copy(arr[s1:e2+1], temp)
}

func main() {
	arr := []int{1, 3, 5, 2, 4, 6, 0, 7}
	naturalMergeSort(arr)
	fmt.Println(arr) // [0 1 2 3 4 5 6 7]
}
```

---

## Example 7: Merge K Sorted Arrays

```go
package main

import (
	"container/heap"
	"fmt"
)

type Item struct {
	val, arrIdx, elemIdx int
}
type MinHeap []Item

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].val < h[j].val }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(Item)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func mergeKSorted(arrays [][]int) []int {
	h := &MinHeap{}
	for i, arr := range arrays {
		if len(arr) > 0 {
			heap.Push(h, Item{arr[0], i, 0})
		}
	}

	result := []int{}
	for h.Len() > 0 {
		item := heap.Pop(h).(Item)
		result = append(result, item.val)
		if item.elemIdx+1 < len(arrays[item.arrIdx]) {
			next := arrays[item.arrIdx][item.elemIdx+1]
			heap.Push(h, Item{next, item.arrIdx, item.elemIdx + 1})
		}
	}
	return result
}

func main() {
	arrays := [][]int{{1, 4, 7}, {2, 5, 8}, {3, 6, 9}}
	fmt.Println(mergeKSorted(arrays)) // [1 2 3 4 5 6 7 8 9]
}
```

---

## Example 8: Merge Sort with Custom Comparator

```go
package main

import "fmt"

func mergeSortCustom(arr []string, less func(a, b string) bool) []string {
	if len(arr) <= 1 { return arr }
	mid := len(arr) / 2
	left := mergeSortCustom(arr[:mid], less)
	right := mergeSortCustom(arr[mid:], less)

	result := make([]string, 0, len(left)+len(right))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if less(left[i], right[j]) {
			result = append(result, left[i]); i++
		} else {
			result = append(result, right[j]); j++
		}
	}
	result = append(result, left[i:]...)
	result = append(result, right[j:]...)
	return result
}

func main() {
	words := []string{"banana", "apple", "cherry", "date"}

	// Sort by length
	byLength := mergeSortCustom(words, func(a, b string) bool {
		return len(a) < len(b)
	})
	fmt.Println("By length:", byLength)

	// Sort alphabetically
	byAlpha := mergeSortCustom(words, func(a, b string) bool {
		return a < b
	})
	fmt.Println("By alpha:", byAlpha)
}
```

---

## Example 9: External Merge Sort (Concept Demo)

```go
package main

import (
	"fmt"
	"sort"
)

// Simulate external merge sort for data too large to fit in memory
func externalMergeSort(data []int, memoryLimit int) []int {
	// Step 1: Create sorted chunks
	chunks := [][]int{}
	for i := 0; i < len(data); i += memoryLimit {
		end := i + memoryLimit
		if end > len(data) { end = len(data) }
		chunk := make([]int, end-i)
		copy(chunk, data[i:end])
		sort.Ints(chunk)
		chunks = append(chunks, chunk)
	}

	fmt.Printf("Created %d sorted chunks of max size %d\n", len(chunks), memoryLimit)

	// Step 2: K-way merge
	result := chunks[0]
	for i := 1; i < len(chunks); i++ {
		result = mergeTwoSorted(result, chunks[i])
	}
	return result
}

func mergeTwoSorted(a, b []int) []int {
	result := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		if a[i] <= b[j] { result = append(result, a[i]); i++ } else { result = append(result, b[j]); j++ }
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result
}

func main() {
	data := []int{38, 27, 43, 3, 9, 82, 10, 5, 1, 77, 44, 22}
	sorted := externalMergeSort(data, 4)
	fmt.Println("Sorted:", sorted)
}
```

---

## Example 10: Merge Sort Visualization

```go
package main

import "fmt"

func mergeSortViz(arr []int, depth int) []int {
	indent := ""
	for i := 0; i < depth; i++ { indent += "  " }

	fmt.Printf("%sSplit: %v\n", indent, arr)

	if len(arr) <= 1 { return arr }

	mid := len(arr) / 2
	left := mergeSortViz(arr[:mid], depth+1)
	right := mergeSortViz(arr[mid:], depth+1)

	result := make([]int, 0, len(left)+len(right))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i] <= right[j] { result = append(result, left[i]); i++ } else { result = append(result, right[j]); j++ }
	}
	result = append(result, left[i:]...)
	result = append(result, right[j:]...)

	fmt.Printf("%sMerge: %v\n", indent, result)
	return result
}

func main() {
	arr := []int{38, 27, 43, 3}
	fmt.Println("=== Merge Sort Visualization ===")
	result := mergeSortViz(arr, 0)
	fmt.Println("Result:", result)
}
```

---

## Key Takeaways

1. Merge sort guarantees O(n log n) in all cases — no worst case degradation
2. It's **stable** — equal elements maintain their relative order
3. Requires O(n) extra space — main disadvantage vs quicksort
4. Ideal for linked lists (O(1) extra space, no random access needed)
5. Applications: external sort, inversion count, merge k sorted arrays

> **Next up:** Quick Sort →
