# Phase 16: Sorting Algorithms — Divide and Conquer in Sorting

## Overview

**Divide and Conquer** splits a problem into smaller subproblems, solves them recursively, and combines results. In sorting, this gives O(n log n) algorithms — merge sort divides evenly then merges; quicksort partitions then recurses.

| Phase | Merge Sort | Quick Sort |
|-------|-----------|------------|
| **Divide** | Split at midpoint | Partition around pivot |
| **Conquer** | Recurse on halves | Recurse on partitions |
| **Combine** | Merge two sorted halves | No combine needed |

---

## Example 1: Classic Divide and Conquer — Merge Sort

```go
package main

import "fmt"

func mergeSort(arr []int) []int {
	if len(arr) <= 1 { return arr }

	// DIVIDE
	mid := len(arr) / 2
	left := mergeSort(arr[:mid])
	right := mergeSort(arr[mid:])

	// COMBINE
	return merge(left, right)
}

func merge(a, b []int) []int {
	result := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		if a[i] <= b[j] {
			result = append(result, a[i]); i++
		} else {
			result = append(result, b[j]); j++
		}
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result
}

func main() {
	arr := []int{38, 27, 43, 3, 9, 82, 10}
	fmt.Println(mergeSort(arr)) // [3 9 10 27 38 43 82]
}
```

---

## Example 2: Count Inversions (D&C Application)

```go
package main

import "fmt"

func countInversions(arr []int) ([]int, int) {
	if len(arr) <= 1 { return arr, 0 }

	mid := len(arr) / 2
	left, leftInv := countInversions(append([]int{}, arr[:mid]...))
	right, rightInv := countInversions(append([]int{}, arr[mid:]...))

	merged, splitInv := mergeCount(left, right)
	return merged, leftInv + rightInv + splitInv
}

func mergeCount(a, b []int) ([]int, int) {
	result := make([]int, 0, len(a)+len(b))
	inversions := 0
	i, j := 0, 0

	for i < len(a) && j < len(b) {
		if a[i] <= b[j] {
			result = append(result, a[i]); i++
		} else {
			result = append(result, b[j]); j++
			inversions += len(a) - i // all remaining in left are inversions
		}
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result, inversions
}

func main() {
	arr := []int{2, 4, 1, 3, 5}
	sorted, inv := countInversions(arr)
	fmt.Println("Sorted:", sorted)     // [1 2 3 4 5]
	fmt.Println("Inversions:", inv)    // 3 — (2,1), (4,1), (4,3)
}
```

---

## Example 3: Quick Select — Kth Smallest (D&C without full sort)

```go
package main

import (
	"fmt"
	"math/rand"
)

func quickSelect(arr []int, k int) int {
	if len(arr) == 1 { return arr[0] }

	pivotIdx := rand.Intn(len(arr))
	arr[pivotIdx], arr[len(arr)-1] = arr[len(arr)-1], arr[pivotIdx]
	pivot := arr[len(arr)-1]

	i := 0
	for j := 0; j < len(arr)-1; j++ {
		if arr[j] <= pivot {
			arr[i], arr[j] = arr[j], arr[i]
			i++
		}
	}
	arr[i], arr[len(arr)-1] = arr[len(arr)-1], arr[i]

	if i == k {
		return arr[i]
	} else if k < i {
		return quickSelect(arr[:i], k)
	}
	return quickSelect(arr[i+1:], k-i-1)
}

func main() {
	arr := []int{7, 10, 4, 3, 20, 15}
	fmt.Println("3rd smallest:", quickSelect(arr, 2)) // 7
}
```

---

## Example 4: Merge Sort for Linked List

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

func mergeSortList(head *ListNode) *ListNode {
	if head == nil || head.Next == nil { return head }

	// Find middle
	slow, fast := head, head.Next
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}

	// Split
	right := slow.Next
	slow.Next = nil

	// Recurse
	left := mergeSortList(head)
	right = mergeSortList(right)

	// Merge
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
	sorted := mergeSortList(head)
	printList(sorted) // 1 2 3 4
}
```

---

## Example 5: Closest Pair of Points (D&C)

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Point struct{ X, Y float64 }

func dist(a, b Point) float64 {
	dx, dy := a.X-b.X, a.Y-b.Y
	return math.Sqrt(dx*dx + dy*dy)
}

func closestPair(points []Point) float64 {
	sort.Slice(points, func(i, j int) bool { return points[i].X < points[j].X })
	return closestRec(points)
}

func closestRec(pts []Point) float64 {
	n := len(pts)
	if n <= 3 {
		minD := math.Inf(1)
		for i := 0; i < n; i++ {
			for j := i + 1; j < n; j++ {
				if d := dist(pts[i], pts[j]); d < minD { minD = d }
			}
		}
		return minD
	}

	mid := n / 2
	midX := pts[mid].X

	dl := closestRec(pts[:mid])
	dr := closestRec(pts[mid:])
	d := math.Min(dl, dr)

	// Check strip
	var strip []Point
	for _, p := range pts {
		if math.Abs(p.X-midX) < d {
			strip = append(strip, p)
		}
	}
	sort.Slice(strip, func(i, j int) bool { return strip[i].Y < strip[j].Y })

	for i := 0; i < len(strip); i++ {
		for j := i + 1; j < len(strip) && strip[j].Y-strip[i].Y < d; j++ {
			if dd := dist(strip[i], strip[j]); dd < d { d = dd }
		}
	}
	return d
}

func main() {
	points := []Point{{2, 3}, {12, 30}, {40, 50}, {5, 1}, {12, 10}, {3, 4}}
	fmt.Printf("Closest pair distance: %.4f\n", closestPair(points))
}
```

---

## Example 6: Merge K Sorted Arrays (D&C)

```go
package main

import "fmt"

func mergeKArrays(arrays [][]int) []int {
	if len(arrays) == 0 { return nil }
	if len(arrays) == 1 { return arrays[0] }

	mid := len(arrays) / 2
	left := mergeKArrays(arrays[:mid])
	right := mergeKArrays(arrays[mid:])
	return mergeTwo(left, right)
}

func mergeTwo(a, b []int) []int {
	result := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		if a[i] <= b[j] {
			result = append(result, a[i]); i++
		} else {
			result = append(result, b[j]); j++
		}
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result
}

func main() {
	arrays := [][]int{
		{1, 4, 7},
		{2, 5, 8},
		{3, 6, 9},
		{0, 10, 11},
	}
	fmt.Println(mergeKArrays(arrays)) // [0 1 2 3 4 5 6 7 8 9 10 11]
}
```

---

## Example 7: Power Function (D&C — Not Sorting but Core Pattern)

```go
package main

import "fmt"

// O(log n) — classic divide and conquer
func power(base float64, exp int) float64 {
	if exp == 0 { return 1 }
	if exp < 0 { return 1 / power(base, -exp) }

	half := power(base, exp/2)
	if exp%2 == 0 {
		return half * half
	}
	return half * half * base
}

func main() {
	fmt.Println(power(2, 10))   // 1024
	fmt.Println(power(2, -2))   // 0.25
	fmt.Println(power(3, 5))    // 243
}
```

---

## Example 8: Maximum Subarray (D&C Approach)

```go
package main

import (
	"fmt"
	"math"
)

func maxSubArray(arr []int) int {
	return maxSubHelper(arr, 0, len(arr)-1)
}

func maxSubHelper(arr []int, lo, hi int) int {
	if lo == hi { return arr[lo] }

	mid := lo + (hi-lo)/2

	leftMax := maxSubHelper(arr, lo, mid)
	rightMax := maxSubHelper(arr, mid+1, hi)
	crossMax := maxCrossingSum(arr, lo, mid, hi)

	return max3(leftMax, rightMax, crossMax)
}

func maxCrossingSum(arr []int, lo, mid, hi int) int {
	leftSum := math.MinInt64
	sum := 0
	for i := mid; i >= lo; i-- {
		sum += arr[i]
		if sum > leftSum { leftSum = sum }
	}

	rightSum := math.MinInt64
	sum = 0
	for i := mid + 1; i <= hi; i++ {
		sum += arr[i]
		if sum > rightSum { rightSum = sum }
	}
	return leftSum + rightSum
}

func max3(a, b, c int) int {
	m := a
	if b > m { m = b }
	if c > m { m = c }
	return m
}

func main() {
	arr := []int{-2, 1, -3, 4, -1, 2, 1, -5, 4}
	fmt.Println("Max subarray sum:", maxSubArray(arr)) // 6
}
```

---

## Example 9: Recurrence Relations for D&C Sorting

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Recurrence Relations for D&C Sorts ===")
	fmt.Println()

	fmt.Println("Merge Sort: T(n) = 2T(n/2) + O(n)")
	fmt.Println("  • Always splits evenly")
	fmt.Println("  • Master theorem: a=2, b=2, f(n)=n")
	fmt.Println("  • Case 2: T(n) = O(n log n)")
	fmt.Println()

	fmt.Println("Quick Sort (best/avg): T(n) = 2T(n/2) + O(n)")
	fmt.Println("  • Average partition is near middle")
	fmt.Println("  • T(n) = O(n log n)")
	fmt.Println()

	fmt.Println("Quick Sort (worst): T(n) = T(n-1) + O(n)")
	fmt.Println("  • Already sorted + first/last pivot")
	fmt.Println("  • T(n) = O(n²)")
	fmt.Println()

	fmt.Println("Quick Select (avg): T(n) = T(n/2) + O(n)")
	fmt.Println("  • Only recurse on one side")
	fmt.Println("  • T(n) = O(n)")
	fmt.Println()

	fmt.Println("Master Theorem: T(n) = aT(n/b) + O(n^d)")
	fmt.Println("  Case 1: d < log_b(a) → O(n^log_b(a))")
	fmt.Println("  Case 2: d = log_b(a) → O(n^d log n)")
	fmt.Println("  Case 3: d > log_b(a) → O(n^d)")
}
```

---

## Example 10: D&C Sorting Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Divide and Conquer Sorting Summary ===")
	fmt.Println()

	fmt.Println("┌───────────────┬──────────────┬──────────────┬─────────────┐")
	fmt.Println("│ Property      │ Merge Sort   │ Quick Sort   │ Quick Select│")
	fmt.Println("├───────────────┼──────────────┼──────────────┼─────────────┤")
	fmt.Println("│ Divide        │ Split at mid │ Partition    │ Partition   │")
	fmt.Println("│ Conquer       │ 2 halves     │ 2 partitions │ 1 partition │")
	fmt.Println("│ Combine       │ Merge O(n)   │ None         │ None        │")
	fmt.Println("│ Best Time     │ O(n log n)   │ O(n log n)   │ O(n)        │")
	fmt.Println("│ Worst Time    │ O(n log n)   │ O(n²)        │ O(n²)       │")
	fmt.Println("│ Avg Time      │ O(n log n)   │ O(n log n)   │ O(n)        │")
	fmt.Println("│ Space         │ O(n)         │ O(log n)     │ O(log n)    │")
	fmt.Println("│ Stable        │ Yes          │ No           │ N/A         │")
	fmt.Println("│ In-Place      │ No           │ Yes          │ Yes         │")
	fmt.Println("└───────────────┴──────────────┴──────────────┴─────────────┘")
	fmt.Println()

	fmt.Println("D&C pattern in sorting:")
	fmt.Println("  1. DIVIDE: split the problem (by index or by value)")
	fmt.Println("  2. CONQUER: recursively solve subproblems")
	fmt.Println("  3. COMBINE: merge results (may be trivial)")
	fmt.Println()

	fmt.Println("Applications beyond sorting:")
	fmt.Println("  • Count inversions (merge sort variant)")
	fmt.Println("  • Closest pair of points")
	fmt.Println("  • Strassen's matrix multiplication")
	fmt.Println("  • Karatsuba multiplication")
}
```

---

## Key Takeaways

1. D&C splits problems into subproblems, solves recursively, combines results
2. Merge sort: divide evenly, heavy combine (merge) — guaranteed O(n log n)
3. Quick sort: heavy divide (partition), trivial combine — O(n log n) average
4. Master theorem gives time complexity from recurrence relation
5. D&C extends beyond sorting: inversions, closest pair, selection

> **Phase 16 Complete!** Next up: Phase 17 — Greedy Algorithms →
