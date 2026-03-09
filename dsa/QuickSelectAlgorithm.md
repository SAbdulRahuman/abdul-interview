# Phase 24: Divide & Conquer — Quick Select Algorithm

## Overview

**Quick Select** finds the k-th smallest/largest element in O(n) **expected** time (vs O(n log n) for sorting). It uses the same partition scheme as Quick Sort but only recurses into one side.

| Variant | Expected | Worst Case | Space |
|---------|----------|------------|-------|
| Random pivot | O(n) | O(n²) | O(1) |
| Median of medians | O(n) | O(n) | O(n) |
| 3-way partition | O(n) | O(n²) | O(1) |

---

## Example 1: Basic Lomuto Quick Select

```go
package main

import (
	"fmt"
	"math/rand"
)

func partition(arr []int, lo, hi int) int {
	// Random pivot
	pivotIdx := lo + rand.Intn(hi-lo+1)
	arr[pivotIdx], arr[hi] = arr[hi], arr[pivotIdx]

	pivot := arr[hi]
	i := lo
	for j := lo; j < hi; j++ {
		if arr[j] < pivot {
			arr[i], arr[j] = arr[j], arr[i]
			i++
		}
	}
	arr[i], arr[hi] = arr[hi], arr[i]
	return i
}

func quickSelect(arr []int, lo, hi, k int) int {
	if lo == hi { return arr[lo] }

	pivotIdx := partition(arr, lo, hi)

	if k == pivotIdx { return arr[k] }
	if k < pivotIdx { return quickSelect(arr, lo, pivotIdx-1, k) }
	return quickSelect(arr, pivotIdx+1, hi, k)
}

func main() {
	arr := []int{7, 10, 4, 3, 20, 15, 8}
	n := len(arr)

	for k := 0; k < n; k++ {
		// Make a copy since partition modifies array
		tmp := append([]int{}, arr...)
		result := quickSelect(tmp, 0, n-1, k)
		fmt.Printf("  %d-th smallest = %d\n", k+1, result)
	}

	fmt.Println("\nAverage O(n): T(n) = T(n/2) + O(n) = O(n)")
}
```

---

## Example 2: Kth Largest Element (LeetCode 215)

```go
package main

import (
	"fmt"
	"math/rand"
)

func findKthLargest(nums []int, k int) int {
	// k-th largest = (n-k)-th smallest (0-indexed)
	target := len(nums) - k
	lo, hi := 0, len(nums)-1

	for lo < hi {
		pivotIdx := lo + rand.Intn(hi-lo+1)
		nums[pivotIdx], nums[hi] = nums[hi], nums[pivotIdx]

		pivot := nums[hi]
		i := lo
		for j := lo; j < hi; j++ {
			if nums[j] < pivot { nums[i], nums[j] = nums[j], nums[i]; i++ }
		}
		nums[i], nums[hi] = nums[hi], nums[i]

		if i == target { return nums[i] }
		if i < target { lo = i + 1 } else { hi = i - 1 }
	}
	return nums[lo]
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{3, 2, 1, 5, 6, 4}, 2},
		{[]int{3, 2, 3, 1, 2, 4, 5, 5, 6}, 4},
	}

	for _, t := range tests {
		arr := append([]int{}, t.nums...)
		result := findKthLargest(arr, t.k)
		fmt.Printf("nums=%v, k=%d → %d-th largest = %d\n", t.nums, t.k, t.k, result)
	}
}
```

---

## Example 3: Three-Way Partition (Dutch National Flag)

```go
package main

import (
	"fmt"
	"math/rand"
)

// Handles duplicates efficiently
// Returns [lo, hi] range of pivot values

func threeWayPartition(arr []int, lo, hi int) (int, int) {
	pivotIdx := lo + rand.Intn(hi-lo+1)
	pivot := arr[pivotIdx]

	lt, gt := lo, hi
	i := lo
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
	return lt, gt // arr[lt..gt] == pivot
}

func quickSelect3Way(arr []int, lo, hi, k int) int {
	if lo >= hi { return arr[lo] }

	lt, gt := threeWayPartition(arr, lo, hi)

	if k < lt { return quickSelect3Way(arr, lo, lt-1, k) }
	if k > gt { return quickSelect3Way(arr, gt+1, hi, k) }
	return arr[lt] // k is within pivot range
}

func main() {
	arr := []int{3, 3, 3, 1, 1, 2, 2, 3, 3}
	fmt.Println("Array:", arr)

	for k := 0; k < len(arr); k++ {
		tmp := append([]int{}, arr...)
		result := quickSelect3Way(tmp, 0, len(tmp)-1, k)
		fmt.Printf("  k=%d → %d\n", k, result)
	}

	fmt.Println("\n3-way partition: O(n) even with many duplicates")
}
```

---

## Example 4: Median of Medians (Worst-Case O(n))

```go
package main

import (
	"fmt"
	"sort"
)

func medianOfMedians(arr []int, k int) int {
	if len(arr) <= 5 {
		sort.Ints(arr)
		return arr[k]
	}

	// Step 1: Divide into groups of 5
	var medians []int
	for i := 0; i < len(arr); i += 5 {
		end := i + 5
		if end > len(arr) { end = len(arr) }
		group := append([]int{}, arr[i:end]...)
		sort.Ints(group)
		medians = append(medians, group[len(group)/2])
	}

	// Step 2: Recursively find median of medians
	pivot := medianOfMedians(medians, len(medians)/2)

	// Step 3: Partition around pivot
	var lo, eq, hi []int
	for _, v := range arr {
		if v < pivot { lo = append(lo, v) } else if v == pivot { eq = append(eq, v) } else { hi = append(hi, v) }
	}

	if k < len(lo) { return medianOfMedians(lo, k) }
	if k < len(lo)+len(eq) { return pivot }
	return medianOfMedians(hi, k-len(lo)-len(eq))
}

func main() {
	arr := []int{12, 3, 5, 7, 4, 19, 26, 23, 11, 2, 9, 15, 1, 8, 25}
	n := len(arr)

	fmt.Println("Array:", arr)
	for _, k := range []int{0, n / 2, n - 1} {
		result := medianOfMedians(append([]int{}, arr...), k)
		fmt.Printf("  k=%d → %d\n", k, result)
	}

	sorted := append([]int{}, arr...)
	sort.Ints(sorted)
	fmt.Println("\nSorted:", sorted)
	fmt.Println("\nMedian of medians: O(n) worst case (guaranteed)")
}
```

---

## Example 5: Top K Frequent Elements (LeetCode 347)

```go
package main

import (
	"fmt"
	"math/rand"
)

func topKFrequent(nums []int, k int) []int {
	// Count frequencies
	freq := map[int]int{}
	for _, v := range nums { freq[v]++ }

	// Build frequency pairs
	type pair struct{ val, cnt int }
	pairs := make([]pair, 0, len(freq))
	for v, c := range freq { pairs = append(pairs, pair{v, c}) }

	// Quick select to find k-th most frequent
	target := len(pairs) - k

	lo, hi := 0, len(pairs)-1
	for lo < hi {
		pivotIdx := lo + rand.Intn(hi-lo+1)
		pairs[pivotIdx], pairs[hi] = pairs[hi], pairs[pivotIdx]
		pivot := pairs[hi].cnt
		i := lo
		for j := lo; j < hi; j++ {
			if pairs[j].cnt < pivot { pairs[i], pairs[j] = pairs[j], pairs[i]; i++ }
		}
		pairs[i], pairs[hi] = pairs[hi], pairs[i]

		if i == target { break }
		if i < target { lo = i + 1 } else { hi = i - 1 }
	}

	result := make([]int, k)
	for i := 0; i < k; i++ { result[i] = pairs[target+i].val }
	return result
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{1, 1, 1, 2, 2, 3}, 2},
		{[]int{1}, 1},
		{[]int{4, 1, -1, 2, -1, 2, 3}, 2},
	}

	for _, t := range tests {
		result := topKFrequent(t.nums, t.k)
		fmt.Printf("nums=%v, k=%d → top %d = %v\n", t.nums, t.k, t.k, result)
	}

	fmt.Println("\nO(n) average with quick select on frequencies")
}
```

---

## Example 6: K Closest Points to Origin (LeetCode 973)

```go
package main

import (
	"fmt"
	"math/rand"
)

func kClosest(points [][]int, k int) [][]int {
	dist := func(p []int) int { return p[0]*p[0] + p[1]*p[1] }

	lo, hi := 0, len(points)-1
	for lo < hi {
		pivotIdx := lo + rand.Intn(hi-lo+1)
		points[pivotIdx], points[hi] = points[hi], points[pivotIdx]
		pivotDist := dist(points[hi])
		i := lo
		for j := lo; j < hi; j++ {
			if dist(points[j]) < pivotDist {
				points[i], points[j] = points[j], points[i]; i++
			}
		}
		points[i], points[hi] = points[hi], points[i]

		if i == k-1 { break }
		if i < k-1 { lo = i + 1 } else { hi = i - 1 }
	}

	return points[:k]
}

func main() {
	tests := []struct {
		points [][]int
		k      int
	}{
		{[][]int{{1, 3}, {-2, 2}}, 1},
		{[][]int{{3, 3}, {5, -1}, {-2, 4}}, 2},
	}

	for _, t := range tests {
		pts := make([][]int, len(t.points))
		for i := range t.points { pts[i] = append([]int{}, t.points[i]...) }
		result := kClosest(pts, t.k)
		fmt.Printf("points=%v, k=%d → %v\n", t.points, t.k, result)
	}
}
```

---

## Example 7: Wiggle Sort II (LeetCode 324)

```go
package main

import (
	"fmt"
	"math/rand"
)

// nums[0] < nums[1] > nums[2] < nums[3] > nums[4] ...
// Use quick select to find median, then 3-way partition with virtual indexing

func wiggleSort(nums []int) {
	n := len(nums)
	median := quickSelectMedian(append([]int{}, nums...), n/2)

	// Virtual index mapping for interleaving
	newIdx := func(i int) int { return (1 + 2*i) % (n | 1) }

	// 3-way partition with virtual indexing
	lo, hi, i := 0, n-1, 0
	for i <= hi {
		if nums[newIdx(i)] > median {
			nums[newIdx(i)], nums[newIdx(lo)] = nums[newIdx(lo)], nums[newIdx(i)]
			lo++; i++
		} else if nums[newIdx(i)] < median {
			nums[newIdx(i)], nums[newIdx(hi)] = nums[newIdx(hi)], nums[newIdx(i)]
			hi--
		} else {
			i++
		}
	}
}

func quickSelectMedian(arr []int, k int) int {
	lo, hi := 0, len(arr)-1
	for lo < hi {
		pivotIdx := lo + rand.Intn(hi-lo+1)
		arr[pivotIdx], arr[hi] = arr[hi], arr[pivotIdx]
		pivot := arr[hi]
		i := lo
		for j := lo; j < hi; j++ {
			if arr[j] < pivot { arr[i], arr[j] = arr[j], arr[i]; i++ }
		}
		arr[i], arr[hi] = arr[hi], arr[i]
		if i == k { return arr[i] }
		if i < k { lo = i + 1 } else { hi = i - 1 }
	}
	return arr[lo]
}

func main() {
	tests := [][]int{
		{1, 5, 1, 1, 6, 4},
		{1, 3, 2, 2, 3, 1},
	}

	for _, nums := range tests {
		arr := append([]int{}, nums...)
		wiggleSort(arr)
		fmt.Printf("Input: %v → Wiggle: %v\n", nums, arr)

		// Verify
		valid := true
		for i := 1; i < len(arr); i++ {
			if i%2 == 1 && arr[i] <= arr[i-1] { valid = false }
			if i%2 == 0 && arr[i] >= arr[i-1] { valid = false }
		}
		fmt.Printf("  Valid: %v\n", valid)
	}
}
```

---

## Example 8: Quickselect for Weighted Median

```go
package main

import (
	"fmt"
	"math/rand"
)

// Weighted median: find element where sum of weights on each side ≤ totalWeight/2
// Meeting point problem: minimize weighted distance

type Item struct {
	val    int
	weight float64
}

func weightedMedian(items []Item) Item {
	total := 0.0
	for _, it := range items { total += it.weight }

	lo, hi := 0, len(items)-1
	for lo < hi {
		// Partition
		pivotIdx := lo + rand.Intn(hi-lo+1)
		items[pivotIdx], items[hi] = items[hi], items[pivotIdx]
		pivot := items[hi].val
		i := lo
		for j := lo; j < hi; j++ {
			if items[j].val < pivot { items[i], items[j] = items[j], items[i]; i++ }
		}
		items[i], items[hi] = items[hi], items[i]

		// Calculate left weight
		leftWeight := 0.0
		for j := lo; j < i; j++ { leftWeight += items[j].weight }
		rightWeight := 0.0
		for j := i + 1; j <= hi; j++ { rightWeight += items[j].weight }

		if leftWeight > total/2 {
			hi = i - 1
		} else if rightWeight > total/2 {
			lo = i + 1
		} else {
			return items[i]
		}
	}
	return items[lo]
}

func main() {
	items := []Item{
		{1, 0.1}, {3, 0.2}, {5, 0.3}, {7, 0.15}, {9, 0.25},
	}

	result := weightedMedian(append([]Item{}, items...))
	fmt.Printf("Items: "); for _, it := range items { fmt.Printf("(%d,%.2f) ", it.val, it.weight) }
	fmt.Printf("\nWeighted median: val=%d, weight=%.2f\n", result.val, result.weight)

	fmt.Println("\nApplication: optimal meeting point, 1D facility location")
}
```

---

## Example 9: Introselect (Hybrid Quick Select)

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
)

// Introselect: quickselect with fallback to median-of-medians
// Guarantees O(n) worst case while being fast in practice

func introSelect(arr []int, k int) int {
	return introHelper(arr, 0, len(arr)-1, k, 2*log2(len(arr)))
}

func log2(n int) int {
	l := 0
	for n > 1 { n /= 2; l++ }
	return l
}

func introHelper(arr []int, lo, hi, k, depth int) int {
	if lo == hi { return arr[lo] }

	if depth == 0 {
		// Fallback to median of medians
		return selectMOM(arr, lo, hi, k)
	}

	// Random partition
	pivotIdx := lo + rand.Intn(hi-lo+1)
	arr[pivotIdx], arr[hi] = arr[hi], arr[pivotIdx]
	pivot := arr[hi]
	i := lo
	for j := lo; j < hi; j++ {
		if arr[j] < pivot { arr[i], arr[j] = arr[j], arr[i]; i++ }
	}
	arr[i], arr[hi] = arr[hi], arr[i]

	if k == i { return arr[i] }
	if k < i { return introHelper(arr, lo, i-1, k, depth-1) }
	return introHelper(arr, i+1, hi, k, depth-1)
}

func selectMOM(arr []int, lo, hi, k int) int {
	sub := append([]int{}, arr[lo:hi+1]...)
	sort.Ints(sub) // simplified: full sort for small ranges
	return sub[k-lo]
}

func main() {
	arr := []int{12, 3, 5, 7, 4, 19, 26, 23, 11, 2, 9, 15}
	n := len(arr)

	fmt.Println("Array:", arr)
	for _, k := range []int{0, n/4, n/2, 3*n/4, n - 1} {
		result := introSelect(append([]int{}, arr...), k)
		fmt.Printf("  k=%d → %d\n", k, result)
	}

	fmt.Println("\nIntroselect: O(n) average, O(n) worst case")
	fmt.Println("Go's pdqsort uses a similar hybrid approach")
}
```

---

## Example 10: Quick Select Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Quick Select Patterns ===\n")

	patterns := []struct {
		problem, approach, complexity string
	}{
		{"K-th smallest (LC 215)", "Quick select on array", "O(n) avg"},
		{"Top K frequent (LC 347)", "Count freq → quick select on freq", "O(n) avg"},
		{"K closest points (LC 973)", "Quick select on distance", "O(n) avg"},
		{"Wiggle sort II (LC 324)", "Find median → 3-way partition", "O(n) avg"},
		{"Weighted median", "Partition + track side weights", "O(n) avg"},
		{"Median of medians", "Groups of 5 → deterministic pivot", "O(n) worst"},
		{"Introselect", "Quick select + MOM fallback", "O(n) worst"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-30s %s  [%s]\n", p.problem, p.approach, p.complexity)
	}

	fmt.Println("\n--- Quick Select vs Sorting ---")
	fmt.Println("Sort: O(n log n) — finds ALL positions")
	fmt.Println("Quick Select: O(n) — finds ONE position")
	fmt.Println("If you only need k-th element, don't sort!")

	fmt.Println("\n--- Pivot Selection ---")
	fmt.Println("Random pivot: simple, O(n) expected, O(n²) worst")
	fmt.Println("Median of 3: slightly better distribution")
	fmt.Println("Median of medians: O(n) guaranteed")
	fmt.Println("3-way partition: handles duplicates efficiently")
}
```

---

## Key Takeaways

1. **Quick Select** = Quick Sort that only recurses into one side → O(n) average
2. **K-th largest** = (n-k)-th smallest → same algorithm
3. **Three-way partition**: essential when array has many duplicates
4. **Median of medians**: O(n) worst case but large constant
5. **Top K problems**: count frequencies, then quick select on frequencies
6. **Introselect**: practical hybrid — random pivot with deterministic fallback
7. **Don't sort** when you only need the k-th element!

> **Next up:** Closest Pair of Points →
