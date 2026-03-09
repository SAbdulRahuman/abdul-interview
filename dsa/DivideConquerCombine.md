# Phase 24: Divide & Conquer — Divide, Conquer, Combine

## Overview

The **Divide & Conquer** paradigm:
1. **Divide**: Break the problem into smaller subproblems
2. **Conquer**: Solve subproblems recursively
3. **Combine**: Merge solutions into the final answer

The key insight is that the **combine step** determines the overall complexity.

| Algorithm | Divide | Conquer | Combine | Time |
|-----------|--------|---------|---------|------|
| Merge Sort | Split in half | Sort halves | Merge O(n) | O(n log n) |
| Quick Sort | Partition by pivot | Sort partitions | None | O(n log n) avg |
| Binary Search | Eliminate half | Search one half | None | O(log n) |
| Closest Pair | Split by x | Solve halves | Check strip O(n) | O(n log n) |

---

## Example 1: Master Theorem Illustration

```go
package main

import "fmt"

// T(n) = a*T(n/b) + O(n^d)
// Case 1: d < log_b(a) → O(n^(log_b(a)))
// Case 2: d = log_b(a) → O(n^d * log n)
// Case 3: d > log_b(a) → O(n^d)

func main() {
	fmt.Println("=== Master Theorem Examples ===\n")

	cases := []struct {
		name     string
		a, b, d  int
		result   string
	}{
		{"Binary Search", 1, 2, 0, "T(n)=T(n/2)+O(1) → O(log n) [Case 2]"},
		{"Merge Sort", 2, 2, 1, "T(n)=2T(n/2)+O(n) → O(n log n) [Case 2]"},
		{"Karatsuba", 3, 2, 1, "T(n)=3T(n/2)+O(n) → O(n^1.585) [Case 1]"},
		{"Strassen", 7, 2, 2, "T(n)=7T(n/2)+O(n²) → O(n^2.807) [Case 1]"},
		{"Naive multiply", 4, 2, 1, "T(n)=4T(n/2)+O(n) → O(n²) [Case 1]"},
		{"Median of medians", 1, 1, 1, "T(n)=T(n/5)+T(7n/10)+O(n) → O(n) [special]"},
	}

	for _, c := range cases {
		fmt.Printf("  %-20s %s\n", c.name+":", c.result)
	}

	fmt.Println("\nThe combine step cost (d) vs recursive branching (log_b(a)) determines complexity")
}
```

---

## Example 2: Maximum Subarray (D&C Approach)

```go
package main

import (
	"fmt"
	"math"
)

// Kadane's is O(n), but D&C illustrates the paradigm
// LeetCode 53

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

func maxSubarrayDC(arr []int, lo, hi int) int {
	if lo == hi { return arr[lo] }

	mid := (lo + hi) / 2
	leftMax := maxSubarrayDC(arr, lo, mid)      // Conquer left
	rightMax := maxSubarrayDC(arr, mid+1, hi)    // Conquer right
	crossMax := maxCrossingSum(arr, lo, mid, hi) // Combine

	return max(leftMax, max(rightMax, crossMax))
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	arr := []int{-2, 1, -3, 4, -1, 2, 1, -5, 4}
	fmt.Println("Array:", arr)
	fmt.Println("Max subarray sum (D&C):", maxSubarrayDC(arr, 0, len(arr)-1))
	// Answer: 6 ([4,-1,2,1])

	fmt.Println("\nDivide: split array in half")
	fmt.Println("Conquer: find max subarray in each half")
	fmt.Println("Combine: find max crossing subarray")
	fmt.Println("T(n) = 2T(n/2) + O(n) = O(n log n)")
}
```

---

## Example 3: Count Inversions

```go
package main

import "fmt"

// Count pairs (i,j) where i < j but arr[i] > arr[j]
// Modified merge sort: O(n log n)

func countInversions(arr []int) ([]int, int) {
	n := len(arr)
	if n <= 1 { return arr, 0 }

	mid := n / 2
	left, lInv := countInversions(arr[:mid])
	right, rInv := countInversions(arr[mid:])
	merged, splitInv := mergeCount(left, right)
	return merged, lInv + rInv + splitInv
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
			inversions += len(a) - i // all remaining in left > b[j]
		}
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result, inversions
}

func main() {
	tests := [][]int{
		{2, 4, 1, 3, 5},
		{5, 4, 3, 2, 1},
		{1, 2, 3, 4, 5},
		{1, 3, 5, 2, 4, 6},
	}

	for _, arr := range tests {
		sorted, inv := countInversions(append([]int{}, arr...))
		fmt.Printf("Array: %v → inversions=%d, sorted=%v\n", arr, inv, sorted)
	}

	fmt.Println("\nKey insight: during merge, when right element is placed,")
	fmt.Println("all remaining left elements form inversions with it")
}
```

---

## Example 4: Power Function (Fast Exponentiation)

```go
package main

import "fmt"

// D&C: x^n = (x^(n/2))^2 * (x if n is odd)
// LeetCode 50

func myPow(x float64, n int) float64 {
	if n < 0 { return 1.0 / myPow(x, -n) }
	if n == 0 { return 1.0 }

	half := myPow(x, n/2) // Conquer
	if n%2 == 0 {
		return half * half // Combine (even)
	}
	return half * half * x // Combine (odd)
}

// Iterative version
func myPowIter(x float64, n int) float64 {
	if n < 0 { x = 1.0 / x; n = -n }
	result := 1.0
	for n > 0 {
		if n%2 == 1 { result *= x }
		x *= x
		n /= 2
	}
	return result
}

// Modular exponentiation
func modPow(base, exp, mod int64) int64 {
	result := int64(1)
	base %= mod
	for exp > 0 {
		if exp%2 == 1 { result = result * base % mod }
		base = base * base % mod
		exp /= 2
	}
	return result
}

func main() {
	fmt.Printf("2^10 = %.0f\n", myPow(2, 10))
	fmt.Printf("2^(-2) = %.4f\n", myPow(2, -2))
	fmt.Printf("3^13 = %.0f\n", myPow(3, 13))

	fmt.Printf("\nIterative: 2^10 = %.0f\n", myPowIter(2, 10))

	mod := int64(1_000_000_007)
	fmt.Printf("2^1000000 mod 10^9+7 = %d\n", modPow(2, 1000000, mod))

	fmt.Println("\nO(log n) multiplications instead of O(n)")
}
```

---

## Example 5: Majority Element (D&C)

```go
package main

import "fmt"

// LeetCode 169: Element appearing > n/2 times
// D&C approach: the majority of the whole must be majority of at least one half

func majorityDC(arr []int, lo, hi int) int {
	if lo == hi { return arr[lo] }

	mid := (lo + hi) / 2
	leftMaj := majorityDC(arr, lo, mid)
	rightMaj := majorityDC(arr, mid+1, hi)

	if leftMaj == rightMaj { return leftMaj }

	// Count occurrences of each candidate in [lo, hi]
	leftCount, rightCount := 0, 0
	for i := lo; i <= hi; i++ {
		if arr[i] == leftMaj { leftCount++ }
		if arr[i] == rightMaj { rightCount++ }
	}

	if leftCount > rightCount { return leftMaj }
	return rightMaj
}

// Boyer-Moore for comparison: O(n) time, O(1) space
func majorityBM(arr []int) int {
	candidate, count := 0, 0
	for _, v := range arr {
		if count == 0 { candidate = v }
		if v == candidate { count++ } else { count-- }
	}
	return candidate
}

func main() {
	tests := [][]int{
		{3, 2, 3},
		{2, 2, 1, 1, 1, 2, 2},
		{1, 1, 1, 2, 2},
	}

	for _, arr := range tests {
		dc := majorityDC(arr, 0, len(arr)-1)
		bm := majorityBM(arr)
		fmt.Printf("Array: %v → D&C=%d, Boyer-Moore=%d\n", arr, dc, bm)
	}

	fmt.Println("\nD&C: O(n log n), Boyer-Moore: O(n)")
	fmt.Println("D&C insight: majority of whole must be majority of at least one half")
}
```

---

## Example 6: Search in Rotated Sorted Array (D&C)

```go
package main

import "fmt"

// LeetCode 33

func searchRotated(nums []int, target int) int {
	lo, hi := 0, len(nums)-1
	for lo <= hi {
		mid := (lo + hi) / 2
		if nums[mid] == target { return mid }

		// Determine which half is sorted
		if nums[lo] <= nums[mid] {
			// Left half sorted
			if nums[lo] <= target && target < nums[mid] {
				hi = mid - 1
			} else {
				lo = mid + 1
			}
		} else {
			// Right half sorted
			if nums[mid] < target && target <= nums[hi] {
				lo = mid + 1
			} else {
				hi = mid - 1
			}
		}
	}
	return -1
}

func main() {
	tests := []struct {
		nums   []int
		target int
	}{
		{[]int{4, 5, 6, 7, 0, 1, 2}, 0},
		{[]int{4, 5, 6, 7, 0, 1, 2}, 3},
		{[]int{1}, 0},
		{[]int{3, 1}, 1},
	}

	for _, t := range tests {
		result := searchRotated(t.nums, t.target)
		fmt.Printf("nums=%v, target=%d → index=%d\n", t.nums, t.target, result)
	}

	fmt.Println("\nDivide: pick midpoint")
	fmt.Println("Conquer: search in the correct sorted half")
	fmt.Println("O(log n) — eliminate half each time")
}
```

---

## Example 7: Tournament Method — Find Min and Max

```go
package main

import (
	"fmt"
	"math"
)

// Find both min and max with ~1.5n comparisons instead of 2n

func minMaxDC(arr []int, lo, hi int) (int, int) {
	if lo == hi { return arr[lo], arr[lo] }
	if hi == lo+1 {
		if arr[lo] < arr[hi] { return arr[lo], arr[hi] }
		return arr[hi], arr[lo]
	}

	mid := (lo + hi) / 2
	min1, max1 := minMaxDC(arr, lo, mid)
	min2, max2 := minMaxDC(arr, mid+1, hi)

	mn := min1; if min2 < mn { mn = min2 }
	mx := max1; if max2 > mx { mx = max2 }
	return mn, mx
}

// Pair comparison approach: also ~1.5n comparisons
func minMaxPairs(arr []int) (int, int) {
	mn, mx := math.MaxInt64, math.MinInt64
	i := 0
	if len(arr)%2 == 1 {
		mn, mx = arr[0], arr[0]
		i = 1
	}
	for ; i+1 < len(arr); i += 2 {
		small, large := arr[i], arr[i+1]
		if small > large { small, large = large, small }
		if small < mn { mn = small }
		if large > mx { mx = large }
	}
	return mn, mx
}

func main() {
	arr := []int{3, 5, 1, 8, 2, 9, 4, 7, 6}
	min1, max1 := minMaxDC(arr, 0, len(arr)-1)
	min2, max2 := minMaxPairs(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("D&C:   min=%d, max=%d\n", min1, max1)
	fmt.Printf("Pairs: min=%d, max=%d\n", min2, max2)

	fmt.Println("\nNaive: 2(n-1) comparisons")
	fmt.Println("Tournament/Pairs: 3⌊n/2⌋ comparisons")
}
```

---

## Example 8: Skyline Problem (D&C)

```go
package main

import (
	"fmt"
	"sort"
)

// LeetCode 218: The Skyline Problem
// D&C: merge two skylines like merge sort

type Point struct{ x, h int }

func getSkyline(buildings [][]int) [][]int {
	if len(buildings) == 0 { return nil }
	if len(buildings) == 1 {
		b := buildings[0]
		return [][]int{{b[0], b[2]}, {b[1], 0}}
	}

	mid := len(buildings) / 2
	left := getSkyline(buildings[:mid])
	right := getSkyline(buildings[mid:])
	return mergeSkylines(left, right)
}

func mergeSkylines(a, b [][]int) [][]int {
	var result [][]int
	i, j := 0, 0
	h1, h2 := 0, 0

	for i < len(a) && j < len(b) {
		var x, maxH int
		if a[i][0] < b[j][0] {
			x = a[i][0]; h1 = a[i][1]; i++
		} else if a[i][0] > b[j][0] {
			x = b[j][0]; h2 = b[j][1]; j++
		} else {
			x = a[i][0]; h1 = a[i][1]; h2 = b[j][1]; i++; j++
		}
		maxH = max(h1, h2)
		if len(result) == 0 || result[len(result)-1][1] != maxH {
			result = append(result, []int{x, maxH})
		}
	}
	for ; i < len(a); i++ {
		if len(result) == 0 || result[len(result)-1][1] != a[i][1] {
			result = append(result, a[i])
		}
	}
	for ; j < len(b); j++ {
		if len(result) == 0 || result[len(result)-1][1] != b[j][1] {
			result = append(result, b[j])
		}
	}
	return result
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	buildings := [][]int{{2, 9, 10}, {3, 7, 15}, {5, 12, 12}, {15, 20, 10}, {19, 24, 8}}
	sort.Slice(buildings, func(i, j int) bool { return buildings[i][0] < buildings[j][0] })

	result := getSkyline(buildings)
	fmt.Println("Buildings:", buildings)
	fmt.Println("Skyline:", result)

	fmt.Println("\nD&C merge like merge sort: O(n log n)")
}
```

---

## Example 9: Tiling Problem (D&C)

```go
package main

import "fmt"

// Fill a 2^n × 2^n board (with one square removed) using L-shaped trominoes
// Classic D&C: split into 4 quadrants, place tromino at center, recurse

var tileID int

func tile(board [][]int, size, topR, topC, missR, missC int) {
	if size == 1 { return }
	tileID++
	id := tileID
	half := size / 2

	// Determine which quadrant has the missing square
	for _, q := range [][2]int{{0, 0}, {0, 1}, {1, 0}, {1, 1}} {
		qr, qc := topR+q[0]*half, topC+q[1]*half
		hasMissing := missR >= qr && missR < qr+half && missC >= qc && missC < qc+half

		if !hasMissing {
			// Fill the corner closest to center with current tromino
			fillR := qr + half - 1
			if q[0] == 1 { fillR = qr }
			fillC := qc + half - 1
			if q[1] == 1 { fillC = qc }
			board[fillR][fillC] = id
		}
	}

	// Recurse into 4 quadrants
	for _, q := range [][2]int{{0, 0}, {0, 1}, {1, 0}, {1, 1}} {
		qr, qc := topR+q[0]*half, topC+q[1]*half
		hasMissing := missR >= qr && missR < qr+half && missC >= qc && missC < qc+half

		newMissR, newMissC := missR, missC
		if !hasMissing {
			newMissR = qr + half - 1
			if q[0] == 1 { newMissR = qr }
			newMissC = qc + half - 1
			if q[1] == 1 { newMissC = qc }
		}
		tile(board, half, qr, qc, newMissR, newMissC)
	}
}

func main() {
	n := 4 // 4×4 board
	board := make([][]int, n)
	for i := range board { board[i] = make([]int, n) }
	board[0][0] = -1 // missing square

	tileID = 0
	tile(board, n, 0, 0, 0, 0)

	fmt.Println("Tromino tiling of 4×4 board (missing=[0,0]):")
	for _, row := range board {
		for _, v := range row {
			if v == -1 { fmt.Print("  X") } else { fmt.Printf("%3d", v) }
		}
		fmt.Println()
	}

	fmt.Printf("\nUsed %d trominoes to fill %d squares\n", tileID, n*n-1)
}
```

---

## Example 10: D&C Pattern Recognition

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Divide & Conquer Pattern Recognition ===\n")

	patterns := []struct {
		pattern   string
		divide    string
		combine   string
		examples  string
	}{
		{
			"Split in half",
			"Two equal subproblems",
			"Merge/combine results",
			"Merge sort, count inversions, skyline",
		},
		{
			"Eliminate half",
			"One subproblem of size n/2",
			"Direct answer from subproblem",
			"Binary search, peak finding, rotated array",
		},
		{
			"Partition by pivot",
			"Two unequal parts",
			"Already in place",
			"Quick sort, quick select, k-th element",
		},
		{
			"Four-way split",
			"2D subdivision",
			"Merge quadrant results",
			"Closest pair, tiling, quadtree",
		},
		{
			"Reduce problem size",
			"Transform then recurse",
			"Transform back",
			"Karatsuba, Strassen, FFT",
		},
	}

	for _, p := range patterns {
		fmt.Printf("Pattern: %s\n", p.pattern)
		fmt.Printf("  Divide:   %s\n", p.divide)
		fmt.Printf("  Combine:  %s\n", p.combine)
		fmt.Printf("  Examples: %s\n\n", p.examples)
	}

	fmt.Println("--- When to Use D&C ---")
	fmt.Println("✅ Problem has optimal substructure")
	fmt.Println("✅ Can combine sub-solutions efficiently")
	fmt.Println("✅ Subproblems are independent (no overlap → use DP instead)")
	fmt.Println("✅ Reducing n to n/2 achieves O(log n) depth")
}
```

---

## Key Takeaways

1. **Combine step** determines the overall complexity — minimize it
2. **Master Theorem**: T(n) = aT(n/b) + O(n^d) gives the complexity class
3. **Count inversions**: classic merge-sort modification — count during merge
4. **Skyline/Max subarray**: merge two sub-solutions at the boundary
5. **Eliminate half**: binary search variants — O(log n) because only one branch
6. **Tournament method**: 1.5n comparisons for min+max simultaneously
7. **D&C vs DP**: D&C when subproblems are independent; DP when they overlap

> **Next up:** Merge Sort Based Problems →
