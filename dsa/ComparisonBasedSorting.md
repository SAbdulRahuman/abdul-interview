# Phase 16: Sorting Algorithms — Comparison-Based Sorting

## Overview

**Comparison-based sorting** algorithms determine element order solely by comparing pairs of elements. The theoretical lower bound for comparison sorts is $\Omega(n \log n)$.

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |

---

## Example 1: Bubble Sort

```go
package main

import "fmt"

func bubbleSort(arr []int) {
	n := len(arr)
	for i := 0; i < n-1; i++ {
		swapped := false
		for j := 0; j < n-1-i; j++ {
			if arr[j] > arr[j+1] {
				arr[j], arr[j+1] = arr[j+1], arr[j]
				swapped = true
			}
		}
		if !swapped { break } // optimization: already sorted
	}
}

func main() {
	arr := []int{64, 34, 25, 12, 22, 11, 90}
	bubbleSort(arr)
	fmt.Println(arr) // [11 12 22 25 34 64 90]
}
```

---

## Example 2: Selection Sort

```go
package main

import "fmt"

func selectionSort(arr []int) {
	n := len(arr)
	for i := 0; i < n-1; i++ {
		minIdx := i
		for j := i + 1; j < n; j++ {
			if arr[j] < arr[minIdx] { minIdx = j }
		}
		arr[i], arr[minIdx] = arr[minIdx], arr[i]
	}
}

func main() {
	arr := []int{29, 10, 14, 37, 13}
	selectionSort(arr)
	fmt.Println(arr) // [10 13 14 29 37]
}
```

---

## Example 3: Insertion Sort

```go
package main

import "fmt"

func insertionSort(arr []int) {
	for i := 1; i < len(arr); i++ {
		key := arr[i]
		j := i - 1
		for j >= 0 && arr[j] > key {
			arr[j+1] = arr[j]
			j--
		}
		arr[j+1] = key
	}
}

func main() {
	arr := []int{12, 11, 13, 5, 6}
	insertionSort(arr)
	fmt.Println(arr) // [5 6 11 12 13]

	// Best case: already sorted → O(n)
	sorted := []int{1, 2, 3, 4, 5}
	insertionSort(sorted)
	fmt.Println(sorted)
}
```

---

## Example 4: Shell Sort (Improved Insertion Sort)

```go
package main

import "fmt"

func shellSort(arr []int) {
	n := len(arr)
	for gap := n / 2; gap > 0; gap /= 2 {
		for i := gap; i < n; i++ {
			temp := arr[i]
			j := i
			for j >= gap && arr[j-gap] > temp {
				arr[j] = arr[j-gap]
				j -= gap
			}
			arr[j] = temp
		}
	}
}

func main() {
	arr := []int{12, 34, 54, 2, 3}
	shellSort(arr)
	fmt.Println(arr) // [2 3 12 34 54]
}
```

---

## Example 5: Lower Bound Proof — Why O(n log n)?

```go
package main

import (
	"fmt"
	"math"
)

// Decision tree argument: any comparison sort needs at least
// ceil(log2(n!)) comparisons in the worst case
func lowerBound(n int) float64 {
	// log2(n!) = sum of log2(i) for i = 1..n
	// By Stirling's approximation: n log n - n log e
	logFact := 0.0
	for i := 1; i <= n; i++ {
		logFact += math.Log2(float64(i))
	}
	return logFact
}

func main() {
	fmt.Println("Minimum comparisons needed (comparison-based sort):")
	for _, n := range []int{4, 8, 16, 100, 1000} {
		bound := lowerBound(n)
		fmt.Printf("  n=%4d: at least %.0f comparisons (%.1f per element)\n",
			n, math.Ceil(bound), bound/float64(n))
	}

	fmt.Println("\nThis proves no comparison sort can beat O(n log n)")
	fmt.Println("Non-comparison sorts (counting, radix) can achieve O(n)")
}
```

---

## Example 6: Comparison Count of Different Sorts

```go
package main

import "fmt"

func bubbleSortCount(arr []int) int {
	a := make([]int, len(arr))
	copy(a, arr)
	comps := 0
	n := len(a)
	for i := 0; i < n-1; i++ {
		for j := 0; j < n-1-i; j++ {
			comps++
			if a[j] > a[j+1] { a[j], a[j+1] = a[j+1], a[j] }
		}
	}
	return comps
}

func insertionSortCount(arr []int) int {
	a := make([]int, len(arr))
	copy(a, arr)
	comps := 0
	for i := 1; i < len(a); i++ {
		key := a[i]
		j := i - 1
		for j >= 0 {
			comps++
			if a[j] <= key { break }
			a[j+1] = a[j]
			j--
		}
		a[j+1] = key
	}
	return comps
}

func main() {
	arr := []int{5, 3, 8, 1, 9, 2, 7, 4, 6, 10}
	fmt.Printf("Array size: %d\n", len(arr))
	fmt.Printf("Bubble sort comparisons:    %d\n", bubbleSortCount(arr))
	fmt.Printf("Insertion sort comparisons: %d\n", insertionSortCount(arr))
	fmt.Printf("Best possible (n log n):    ~%.0f\n", float64(len(arr))*3.32)
}
```

---

## Example 7: Adaptive Sorting (Nearly Sorted Input)

```go
package main

import (
	"fmt"
	"sort"
	"time"
)

func insertionSort(arr []int) {
	for i := 1; i < len(arr); i++ {
		key := arr[i]
		j := i - 1
		for j >= 0 && arr[j] > key {
			arr[j+1] = arr[j]
			j--
		}
		arr[j+1] = key
	}
}

func main() {
	// Nearly sorted: insertion sort shines
	n := 10000
	nearly := make([]int, n)
	for i := range nearly { nearly[i] = i }
	// Swap a few elements
	for i := 0; i < 10; i++ { nearly[i*100], nearly[i*100+1] = nearly[i*100+1], nearly[i*100] }

	arr1 := make([]int, n)
	copy(arr1, nearly)
	start := time.Now()
	insertionSort(arr1)
	t1 := time.Since(start)

	arr2 := make([]int, n)
	copy(arr2, nearly)
	start = time.Now()
	sort.Ints(arr2)
	t2 := time.Since(start)

	fmt.Printf("Nearly sorted (n=%d):\n", n)
	fmt.Printf("  Insertion sort: %v\n", t1)
	fmt.Printf("  Go's sort:      %v\n", t2)
	fmt.Println("  Insertion sort is O(n) for nearly sorted data!")
}
```

---

## Example 8: Custom Comparison Sort

```go
package main

import (
	"fmt"
	"sort"
)

type Person struct {
	Name string
	Age  int
}

func main() {
	people := []Person{
		{"Alice", 30}, {"Bob", 25}, {"Charlie", 35},
		{"Diana", 25}, {"Eve", 30},
	}

	// Sort by age, then by name (stable sort preserves relative order)
	sort.SliceStable(people, func(i, j int) bool {
		if people[i].Age != people[j].Age {
			return people[i].Age < people[j].Age
		}
		return people[i].Name < people[j].Name
	})

	fmt.Println("Sorted by age, then name:")
	for _, p := range people {
		fmt.Printf("  %s (age %d)\n", p.Name, p.Age)
	}
}
```

---

## Example 9: Inversion Count (Measure Disorder)

```go
package main

import "fmt"

// Count inversions: pairs (i,j) where i < j but arr[i] > arr[j]
func countInversions(arr []int) int {
	temp := make([]int, len(arr))
	return mergeCount(arr, temp, 0, len(arr)-1)
}

func mergeCount(arr, temp []int, left, right int) int {
	if left >= right { return 0 }
	mid := (left + right) / 2
	count := mergeCount(arr, temp, left, mid) +
		mergeCount(arr, temp, mid+1, right)

	i, j, k := left, mid+1, left
	for i <= mid && j <= right {
		if arr[i] <= arr[j] {
			temp[k] = arr[i]; i++
		} else {
			temp[k] = arr[j]; j++
			count += mid - i + 1 // all remaining in left half are inversions
		}
		k++
	}
	for i <= mid { temp[k] = arr[i]; i++; k++ }
	for j <= right { temp[k] = arr[j]; j++; k++ }
	copy(arr[left:right+1], temp[left:right+1])

	return count
}

func main() {
	arr := []int{2, 4, 1, 3, 5}
	fmt.Println("Inversions:", countInversions(arr)) // 3

	arr2 := []int{5, 4, 3, 2, 1}
	fmt.Println("Inversions (reverse):", countInversions(arr2)) // 10
}
```

---

## Example 10: Comparison-Based Sort Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Comparison-Based Sorting ===")
	fmt.Println()

	sorts := []struct{ name, best, avg, worst, space, stable string }{
		{"Bubble", "O(n)", "O(n²)", "O(n²)", "O(1)", "Yes"},
		{"Selection", "O(n²)", "O(n²)", "O(n²)", "O(1)", "No"},
		{"Insertion", "O(n)", "O(n²)", "O(n²)", "O(1)", "Yes"},
		{"Shell", "O(n log n)", "O(n^1.5)", "O(n²)", "O(1)", "No"},
		{"Merge", "O(n log n)", "O(n log n)", "O(n log n)", "O(n)", "Yes"},
		{"Quick", "O(n log n)", "O(n log n)", "O(n²)", "O(log n)", "No"},
		{"Heap", "O(n log n)", "O(n log n)", "O(n log n)", "O(1)", "No"},
	}

	fmt.Printf("%-12s %-12s %-12s %-12s %-8s %s\n",
		"Algorithm", "Best", "Average", "Worst", "Space", "Stable")
	for _, s := range sorts {
		fmt.Printf("%-12s %-12s %-12s %-12s %-8s %s\n",
			s.name, s.best, s.avg, s.worst, s.space, s.stable)
	}

	fmt.Println("\nKey insights:")
	fmt.Println("  • Lower bound: Ω(n log n) for comparison-based sorts")
	fmt.Println("  • Insertion sort best for small/nearly-sorted arrays")
	fmt.Println("  • Go uses Pdqsort (pattern-defeating quicksort)")
	fmt.Println("  • Stable sorts preserve relative order of equal elements")
}
```

---

## Key Takeaways

1. Comparison-based sorts have a theoretical lower bound of $\Omega(n \log n)$
2. O(n²) sorts (bubble, selection, insertion) are simple but slow for large n
3. Insertion sort is optimal for nearly-sorted data — O(n) best case
4. Stability matters when sort keys have ties and relative order matters
5. Inversion count measures how far from sorted an array is

> **Next up:** Merge Sort →
