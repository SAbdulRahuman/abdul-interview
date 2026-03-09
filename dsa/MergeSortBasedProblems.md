# Phase 24: Divide & Conquer — Merge Sort Based Problems

## Overview

Merge sort's **divide-and-merge** structure can be adapted to solve many counting and ordering problems. The key insight: during the merge step, elements from the left half and right half have a known relative ordering that enables efficient counting.

| Problem | What to Count During Merge |
|---------|---------------------------|
| Count inversions | Right element placed before remaining left elements |
| Count range sum | Number of valid (i,j) pairs across halves |
| Count smaller after self | Count of right elements already merged |
| Reverse pairs | Count pairs where A[i] > 2*A[j] |

---

## Example 1: Count Inversions (Classic)

```go
package main

import "fmt"

func mergeSortCount(arr []int) ([]int, int) {
	if len(arr) <= 1 { return arr, 0 }
	mid := len(arr) / 2
	left, lc := mergeSortCount(append([]int{}, arr[:mid]...))
	right, rc := mergeSortCount(append([]int{}, arr[mid:]...))

	merged := make([]int, 0, len(arr))
	count := lc + rc
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			merged = append(merged, left[i]); i++
		} else {
			merged = append(merged, right[j]); j++
			count += len(left) - i // all remaining left elements > right[j]
		}
	}
	merged = append(merged, left[i:]...)
	merged = append(merged, right[j:]...)
	return merged, count
}

func main() {
	tests := [][]int{
		{5, 4, 3, 2, 1},
		{1, 2, 3, 4, 5},
		{2, 4, 1, 3, 5},
		{1, 3, 5, 2, 4, 6},
	}

	for _, arr := range tests {
		_, count := mergeSortCount(append([]int{}, arr...))
		fmt.Printf("%v → %d inversions\n", arr, count)
	}
}
```

---

## Example 2: Count of Smaller Numbers After Self (LeetCode 315)

```go
package main

import "fmt"

func countSmaller(nums []int) []int {
	n := len(nums)
	counts := make([]int, n)
	indices := make([]int, n)
	for i := range indices { indices[i] = i }

	var mergeSort func(idxs []int) []int
	mergeSort = func(idxs []int) []int {
		if len(idxs) <= 1 { return idxs }
		mid := len(idxs) / 2
		left := mergeSort(idxs[:mid])
		right := mergeSort(idxs[mid:])

		merged := make([]int, 0, len(idxs))
		i, j := 0, 0
		rightCount := 0
		for i < len(left) && j < len(right) {
			if nums[left[i]] > nums[right[j]] {
				rightCount++
				merged = append(merged, right[j]); j++
			} else {
				counts[left[i]] += rightCount
				merged = append(merged, left[i]); i++
			}
		}
		for ; i < len(left); i++ {
			counts[left[i]] += rightCount
			merged = append(merged, left[i])
		}
		merged = append(merged, right[j:]...)
		return merged
	}

	idx := make([]int, n)
	copy(idx, indices)
	mergeSort(idx)
	return counts
}

func main() {
	tests := [][]int{
		{5, 2, 6, 1},
		{-1, -1},
		{2, 0, 1},
	}

	for _, nums := range tests {
		result := countSmaller(append([]int{}, nums...))
		fmt.Printf("nums=%v → counts=%v\n", nums, result)
	}
	// [5,2,6,1] → [2,1,1,0]
}
```

---

## Example 3: Reverse Pairs (LeetCode 493)

```go
package main

import "fmt"

// Count pairs (i,j) where i < j and nums[i] > 2 * nums[j]

func reversePairs(nums []int) int {
	return mergeCount(nums, 0, len(nums)-1)
}

func mergeCount(arr []int, lo, hi int) int {
	if lo >= hi { return 0 }
	mid := (lo + hi) / 2
	count := mergeCount(arr, lo, mid) + mergeCount(arr, mid+1, hi)

	// Count reverse pairs across halves
	j := mid + 1
	for i := lo; i <= mid; i++ {
		for j <= hi && int64(arr[i]) > 2*int64(arr[j]) { j++ }
		count += j - (mid + 1)
	}

	// Standard merge
	temp := make([]int, 0, hi-lo+1)
	l, r := lo, mid+1
	for l <= mid && r <= hi {
		if arr[l] <= arr[r] { temp = append(temp, arr[l]); l++ } else { temp = append(temp, arr[r]); r++ }
	}
	for l <= mid { temp = append(temp, arr[l]); l++ }
	for r <= hi { temp = append(temp, arr[r]); r++ }
	copy(arr[lo:], temp)

	return count
}

func main() {
	tests := []struct {
		nums     []int
		expected int
	}{
		{[]int{1, 3, 2, 3, 1}, 2},
		{[]int{2, 4, 3, 5, 1}, 3},
	}

	for _, t := range tests {
		arr := append([]int{}, t.nums...)
		result := reversePairs(arr)
		fmt.Printf("nums=%v → reverse pairs=%d (expected=%d)\n", t.nums, result, t.expected)
	}
}
```

---

## Example 4: Count of Range Sum (LeetCode 327)

```go
package main

import "fmt"

// Count (i,j) where lower <= sum(arr[i..j]) <= upper
// Use prefix sums: need lower <= prefix[j+1] - prefix[i] <= upper

func countRangeSum(nums []int, lower, upper int) int {
	n := len(nums)
	prefix := make([]int64, n+1)
	for i, v := range nums { prefix[i+1] = prefix[i] + int64(v) }

	var mergeCount func(arr []int64) int
	mergeCount = func(arr []int64) int {
		if len(arr) <= 1 { return 0 }
		mid := len(arr) / 2
		left := append([]int64{}, arr[:mid]...)
		right := append([]int64{}, arr[mid:]...)
		count := mergeCount(left) + mergeCount(right)

		// Count valid pairs: for each left[i], count right[j] where
		// lower <= right[j] - left[i] <= upper
		lo, hi := 0, 0
		for _, lv := range left {
			for lo < len(right) && right[lo]-lv < int64(lower) { lo++ }
			for hi < len(right) && right[hi]-lv <= int64(upper) { hi++ }
			count += hi - lo
		}

		// Standard merge
		merged := make([]int64, 0, len(arr))
		i, j := 0, 0
		for i < len(left) && j < len(right) {
			if left[i] <= right[j] { merged = append(merged, left[i]); i++ } else { merged = append(merged, right[j]); j++ }
		}
		merged = append(merged, left[i:]...)
		merged = append(merged, right[j:]...)
		copy(arr, merged)
		return count
	}

	return mergeCount(prefix)
}

func main() {
	tests := []struct {
		nums         []int
		lower, upper int
		expected     int
	}{
		{[]int{-2, 5, -1}, -2, 2, 3},
		{[]int{0}, 0, 0, 1},
		{[]int{-2147483647, 0, -2147483647, 2147483647}, -564, 3864, 3},
	}

	for _, t := range tests {
		result := countRangeSum(t.nums, t.lower, t.upper)
		fmt.Printf("nums=%v, [%d,%d] → count=%d (expected=%d)\n", t.nums, t.lower, t.upper, result, t.expected)
	}
}
```

---

## Example 5: Sort Linked List (Merge Sort)

```go
package main

import "fmt"

// LeetCode 148: O(n log n) sort with O(1) space (for linked list)

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
	slow.Next = nil // cut

	left := sortList(head)
	right := sortList(mid)
	return mergeLists(left, right)
}

func mergeLists(a, b *ListNode) *ListNode {
	dummy := &ListNode{}
	curr := dummy
	for a != nil && b != nil {
		if a.Val <= b.Val { curr.Next = a; a = a.Next } else { curr.Next = b; b = b.Next }
		curr = curr.Next
	}
	if a != nil { curr.Next = a } else { curr.Next = b }
	return dummy.Next
}

func buildList(arr []int) *ListNode {
	dummy := &ListNode{}
	curr := dummy
	for _, v := range arr { curr.Next = &ListNode{Val: v}; curr = curr.Next }
	return dummy.Next
}

func printList(head *ListNode) {
	for head != nil { fmt.Printf("%d→", head.Val); head = head.Next }
	fmt.Println("nil")
}

func main() {
	tests := [][]int{
		{4, 2, 1, 3},
		{-1, 5, 3, 4, 0},
	}

	for _, arr := range tests {
		head := buildList(arr)
		fmt.Print("Before: "); printList(head)
		sorted := sortList(head)
		fmt.Print("After:  "); printList(sorted)
		fmt.Println()
	}

	fmt.Println("Linked list merge sort: O(n log n) time, O(log n) stack space")
}
```

---

## Example 6: Merge k Sorted Arrays

```go
package main

import (
	"container/heap"
	"fmt"
)

type Item struct {
	val, listIdx, elemIdx int
}
type MinHeap []Item
func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i].val < h[j].val }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)         { *h = append(*h, x.(Item)) }
func (h *MinHeap) Pop() any           { old := *h; x := old[len(old)-1]; *h = old[:len(old)-1]; return x }

func mergeKSorted(lists [][]int) []int {
	h := &MinHeap{}
	for i, l := range lists {
		if len(l) > 0 { heap.Push(h, Item{l[0], i, 0}) }
	}

	var result []int
	for h.Len() > 0 {
		top := heap.Pop(h).(Item)
		result = append(result, top.val)
		if top.elemIdx+1 < len(lists[top.listIdx]) {
			heap.Push(h, Item{lists[top.listIdx][top.elemIdx+1], top.listIdx, top.elemIdx + 1})
		}
	}
	return result
}

// D&C approach: merge pairwise like merge sort
func mergeKDC(lists [][]int) []int {
	if len(lists) == 0 { return nil }
	if len(lists) == 1 { return lists[0] }
	mid := len(lists) / 2
	left := mergeKDC(lists[:mid])
	right := mergeKDC(lists[mid:])
	return merge2(left, right)
}

func merge2(a, b []int) []int {
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
	lists := [][]int{
		{1, 4, 7},
		{2, 5, 8},
		{3, 6, 9},
		{0, 10, 11},
	}

	fmt.Println("Lists:", lists)
	fmt.Println("Heap merge: ", mergeKSorted(lists))
	fmt.Println("D&C merge:  ", mergeKDC(lists))

	fmt.Println("\nHeap: O(N log k), D&C: O(N log k) — same complexity")
}
```

---

## Example 7: Count of Range Sum Variants — Maximum Sum with Limit

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

// Find max subarray sum ≤ k
// Use sorted prefix sums + binary search (TreeSet-like)

func maxSumAtMostK(arr []int, k int) int {
	// For subarray sum[i..j] = prefix[j+1] - prefix[i]
	// We want max (prefix[j+1] - prefix[i]) ≤ k
	// i.e., min prefix[i] ≥ prefix[j+1] - k

	sorted := []int{0} // sorted prefix sums seen so far
	prefix := 0
	result := math.MinInt64

	for _, v := range arr {
		prefix += v

		// Binary search for smallest prefix[i] ≥ prefix - k
		target := prefix - k
		idx := sort.SearchInts(sorted, target)
		if idx < len(sorted) {
			sum := prefix - sorted[idx]
			if sum > result { result = sum }
		}

		// Insert current prefix
		pos := sort.SearchInts(sorted, prefix)
		sorted = append(sorted, 0)
		copy(sorted[pos+1:], sorted[pos:])
		sorted[pos] = prefix
	}

	return result
}

func main() {
	tests := []struct {
		arr []int
		k   int
	}{
		{[]int{2, 1, -1, 3, -2}, 4},
		{[]int{-2, -3, 4, -1, -2, 1, 5, -3}, 6},
		{[]int{5, -2, 3, 1}, 3},
	}

	for _, t := range tests {
		result := maxSumAtMostK(t.arr, t.k)
		fmt.Printf("arr=%v, k=%d → max sum ≤ k = %d\n", t.arr, t.k, result)
	}
}
```

---

## Example 8: Merge Sort for Stability Demonstration

```go
package main

import (
	"fmt"
	"sort"
)

type Student struct {
	Name  string
	Grade int
}

// Merge sort is STABLE: equal elements preserve original order
func mergeSortStable(arr []Student) []Student {
	if len(arr) <= 1 { return arr }
	mid := len(arr) / 2
	left := mergeSortStable(append([]Student{}, arr[:mid]...))
	right := mergeSortStable(append([]Student{}, arr[mid:]...))

	result := make([]Student, 0, len(arr))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i].Grade <= right[j].Grade { // <= ensures stability
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
	students := []Student{
		{"Alice", 90}, {"Bob", 85}, {"Charlie", 90},
		{"Dave", 85}, {"Eve", 95}, {"Frank", 90},
	}

	fmt.Println("Original order:")
	for _, s := range students { fmt.Printf("  %s(%d)\n", s.Name, s.Grade) }

	sorted := mergeSortStable(append([]Student{}, students...))
	fmt.Println("\nMerge sorted (stable):")
	for _, s := range sorted { fmt.Printf("  %s(%d)\n", s.Name, s.Grade) }
	// Alice, Charlie, Frank all have 90 and stay in original relative order

	// Go's sort.SliceStable is also stable
	sort.SliceStable(students, func(i, j int) bool { return students[i].Grade < students[j].Grade })
	fmt.Println("\nGo sort.SliceStable:")
	for _, s := range students { fmt.Printf("  %s(%d)\n", s.Name, s.Grade) }
}
```

---

## Example 9: Merge Sort — Count of Important Reverse Pairs

```go
package main

import "fmt"

// Generalized: count pairs where nums[i] > k * nums[j] for i < j
// Same merge sort pattern, just change the comparison

func countImportantPairs(nums []int, k int) int {
	arr := append([]int{}, nums...)
	return solve(arr, 0, len(arr)-1, k)
}

func solve(arr []int, lo, hi, k int) int {
	if lo >= hi { return 0 }
	mid := (lo + hi) / 2
	count := solve(arr, lo, mid, k) + solve(arr, mid+1, hi, k)

	// Count pairs across halves
	j := mid + 1
	for i := lo; i <= mid; i++ {
		for j <= hi && int64(arr[i]) > int64(k)*int64(arr[j]) { j++ }
		count += j - (mid + 1)
	}

	// Merge
	temp := make([]int, 0, hi-lo+1)
	l, r := lo, mid+1
	for l <= mid && r <= hi {
		if arr[l] <= arr[r] { temp = append(temp, arr[l]); l++ } else { temp = append(temp, arr[r]); r++ }
	}
	for l <= mid { temp = append(temp, arr[l]); l++ }
	for r <= hi { temp = append(temp, arr[r]); r++ }
	copy(arr[lo:], temp)

	return count
}

func main() {
	nums := []int{1, 3, 2, 3, 1}

	// k=2: same as LeetCode 493 reverse pairs
	fmt.Printf("nums=%v, k=2: %d pairs\n", nums, countImportantPairs(nums, 2))

	// k=1: same as count inversions
	nums2 := []int{5, 4, 3, 2, 1}
	fmt.Printf("nums=%v, k=1: %d pairs (inversions)\n", nums2, countImportantPairs(nums2, 1))

	// k=3
	nums3 := []int{10, 1, 2, 3}
	fmt.Printf("nums=%v, k=3: %d pairs (>3x)\n", nums3, countImportantPairs(nums3, 3))
}
```

---

## Example 10: Merge Sort Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Merge Sort Based Problem Patterns ===\n")

	patterns := []struct {
		problem, merge, complexity string
	}{
		{"Count inversions", "Count right elements placed before left", "O(n log n)"},
		{"Count smaller after self (LC 315)", "Track original indices, count right merged", "O(n log n)"},
		{"Reverse pairs (LC 493)", "Two-pointer count before standard merge", "O(n log n)"},
		{"Count range sum (LC 327)", "Prefix sums + two-pointer in merge", "O(n log n)"},
		{"Sort linked list (LC 148)", "Find mid with slow/fast, merge halves", "O(n log n)"},
		{"Merge k sorted lists (LC 23)", "D&C pairwise merge or min-heap", "O(N log k)"},
		{"Stability proof", "Use <= to preserve relative order", "O(n log n)"},
		{"Max subarray sum ≤ k", "Sorted prefix + binary search", "O(n log n)"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-32s %-45s %s\n", p.problem, p.merge, p.complexity)
	}

	fmt.Println("\n--- Template ---")
	fmt.Println("1. mergeSort(arr, lo, hi)")
	fmt.Println("2.   if lo >= hi: return")
	fmt.Println("3.   mid = (lo+hi)/2")
	fmt.Println("4.   mergeSort(lo, mid)")
	fmt.Println("5.   mergeSort(mid+1, hi)")
	fmt.Println("6.   *** COUNT/PROCESS pairs across halves ***")
	fmt.Println("7.   standard merge")
}
```

---

## Key Takeaways

1. **During merge**, left and right halves are individually sorted — exploit this!
2. **Count inversions**: when right[j] is placed, `len(left) - i` inversions
3. **Track original indices**: use index arrays for "count smaller after self"
4. **Two-pass on halves**: count pairs first, then standard merge
5. **Prefix sums + merge sort** = powerful range sum counting
6. **Linked list merge sort** achieves O(1) extra space (no random access needed)
7. **All these problems**: O(n log n) — same as merge sort itself

> **Next up:** Quick Select Algorithm →
