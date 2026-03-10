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

**Textual Figure:**

```
Bubble Sort: arr = [64, 34, 25, 12, 22, 11, 90]

  Pass 1 (compare adjacent, bubble largest to end):
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 64 │ 34 │ 25 │ 12 │ 22 │ 11 │ 90 │  Initial
  └────┴────┴────┴────┴────┴────┴────┘
   ^^^  ^^^  64>34 → swap
  [34, 64, 25, 12, 22, 11, 90]
       ^^^  ^^^  64>25 → swap
  [34, 25, 64, 12, 22, 11, 90]
            ^^^  ^^^  64>12 → swap
  [34, 25, 12, 64, 22, 11, 90]
                 ^^^  ^^^  64>22 → swap
  [34, 25, 12, 22, 64, 11, 90]
                      ^^^  ^^^  64>11 → swap
  [34, 25, 12, 22, 11, 64, 90]
                           ^^^  ^^^  64<90 → no swap
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 34 │ 25 │ 12 │ 22 │ 11 │ 64 │ 90 │  90 in place
  └────┴────┴────┴────┴────┴────┴────┘

  Pass 2: [34, 25, 12, 22, 11, │64, 90]
  → [25, 12, 22, 11, 34, │64, 90]     64 in place

  Pass 3: [25, 12, 22, 11, │34, 64, 90]
  → [12, 22, 11, 25, │34, 64, 90]     25 in place

  Pass 4: [12, 22, 11, │25, 34, 64, 90]
  → [12, 11, 22, │25, 34, 64, 90]     22 in place

  Pass 5: [12, 11, │22, 25, 34, 64, 90]
  → [11, 12, │22, 25, 34, 64, 90]     12 in place

  Pass 6: no swaps → DONE (optimized early exit)
  ┌────┬────┬────┬────┬────┬────┬────┐
  │ 11 │ 12 │ 22 │ 25 │ 34 │ 64 │ 90 │  Sorted!
  └────┴────┴────┴────┴────┴────┴────┘
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

**Textual Figure:**

```
Selection Sort: arr = [29, 10, 14, 37, 13]

  Pass 1: find min in [0..4], min=10 at idx=1
  ┌────┬────┬────┬────┬────┐
  │ 29 │ 10 │ 14 │ 37 │ 13 │  swap arr[0]↔arr[1]
  └────┴────┴────┴────┴────┘
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 29 │ 14 │ 37 │ 13 │  10 in final position
  └────┴────┴────┴────┴────┘
   ════

  Pass 2: find min in [1..4], min=13 at idx=4
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 29 │ 14 │ 37 │ 13 │  swap arr[1]↔arr[4]
  └────┴────┴────┴────┴────┘
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 13 │ 14 │ 37 │ 29 │  13 in final position
  └────┴────┴────┴────┴────┘
   ════ ════

  Pass 3: find min in [2..4], min=14 at idx=2 (no swap)
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 13 │ 14 │ 37 │ 29 │  14 already in place
  └────┴────┴────┴────┴────┘
   ════ ════ ════

  Pass 4: find min in [3..4], min=29 at idx=4
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 13 │ 14 │ 37 │ 29 │  swap arr[3]↔arr[4]
  └────┴────┴────┴────┴────┘
  ┌────┬────┬────┬────┬────┐
  │ 10 │ 13 │ 14 │ 29 │ 37 │  Sorted!
  └────┴────┴────┴────┴────┘
   ════ ════ ════ ════ ════
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

**Textual Figure:**

```
Insertion Sort: arr = [12, 11, 13, 5, 6]

  i=1: key=11, compare with 12
  ┌────┬────┬────┬────┬────┐
  │ 12 │[11]│ 13 │  5 │  6 │  11<12 → shift 12 right
  └────┴────┴────┴────┴────┘
  ┌────┬────┬────┬────┬────┐
  │ 11 │ 12 │ 13 │  5 │  6 │  insert 11 at pos 0
  └────┴────┴────┴────┴────┘
   ════ ════

  i=2: key=13, compare with 12
  ┌────┬────┬────┬────┬────┐
  │ 11 │ 12 │[13]│  5 │  6 │  13>12 → no shift needed
  └────┴────┴────┴────┴────┘
   ════ ════ ════

  i=3: key=5, compare with 13,12,11
  ┌────┬────┬────┬────┬────┐
  │ 11 │ 12 │ 13 │[ 5]│  6 │  5<13 → shift 13
  └────┴────┴────┴────┴────┘  5<12 → shift 12
                               5<11 → shift 11
  ┌────┬────┬────┬────┬────┐
  │  5 │ 11 │ 12 │ 13 │  6 │  insert 5 at pos 0
  └────┴────┴────┴────┴────┘
   ════ ════ ════ ════

  i=4: key=6, compare with 13,12,11,5
  ┌────┬────┬────┬────┬────┐
  │  5 │ 11 │ 12 │ 13 │[ 6]│  6<13 → shift, 6<12 → shift
  └────┴────┴────┴────┴────┘  6<11 → shift, 6>5 → stop
  ┌────┬────┬────┬────┬────┐
  │  5 │  6 │ 11 │ 12 │ 13 │  insert 6 at pos 1. Sorted!
  └────┴────┴────┴────┴────┘
   ════ ════ ════ ════ ════
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

**Textual Figure:**

```
Shell Sort: arr = [12, 34, 54, 2, 3]    n=5

  gap = 5/2 = 2:
  Compare elements 2 apart: (12,54), (34,2), (54,3)
  ┌────┬────┬────┬────┬────┐
  │ 12 │ 34 │ 54 │  2 │  3 │  gap=2
  └────┴────┴────┴────┴────┘
    ├─────────┤              12 vs 54 → ok
         ├─────────┤         34 vs 2  → swap
              ├─────────┤    54 vs 3  → swap
  ┌────┬────┬────┬────┬────┐
  │ 12 │  2 │  3 │ 34 │ 54 │  after gap=2 pass
  └────┴────┴────┴────┴────┘
  then: 12 vs 3 → swap
  ┌────┬────┬────┬────┬────┐
  │  3 │  2 │ 12 │ 34 │ 54 │  gap=2 done
  └────┴────┴────┴────┴────┘

  gap = 2/2 = 1 (standard insertion sort):
  ┌────┬────┬────┬────┬────┐
  │  3 │  2 │ 12 │ 34 │ 54 │  gap=1
  └────┴────┴────┴────┴────┘
  i=1: 2<3 → swap
  ┌────┬────┬────┬────┬────┐
  │  2 │  3 │ 12 │ 34 │ 54 │  Sorted!
  └────┴────┴────┴────┴────┘

  Shell sort = insertion sort with decreasing gaps
  Reduces long-range disorder first, then fine-tunes.
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

**Textual Figure:**

```
Decision Tree for Sorting [a, b, c]:

                     a < b ?
                    /       \
                 Yes         No
                /               \
           b < c ?            b < c ?
           /    \              /    \
        Yes      No        Yes      No
        /          \        /          \
    [a,b,c]    a < c ?  [b,a,c]?    a < c ?
               /    \    ...        /    \
            Yes      No          Yes      No
          [a,c,b]  [c,a,b]    [b,c,a]  [c,b,a]

  n=3: 3! = 6 leaves → at least ⌈log₂(6)⌉ = 3 comparisons

  ┌─────┬────────────┬───────────────────────────┐
  │  n  │ min comps  │ ⌈log₂(n!)⌉               │
  ├─────┼────────────┼───────────────────────────┤
  │   4 │         5  │ ≈ 4.6 per element × ~1.2  │
  │   8 │        16  │ ≈ 2.0 per element          │
  │  16 │        45  │ ≈ 2.8 per element          │
  │ 100 │       525  │ ≈ 5.2 per element          │
  │1000 │      8530  │ ≈ 8.5 per element          │
  └─────┴────────────┴───────────────────────────┘

  Lower bound: Ω(n log n) — no comparison sort can beat this.
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

**Textual Figure:**

```
Comparison Counts: arr = [5, 3, 8, 1, 9, 2, 7, 4, 6, 10]  (n=10)

  ┌────────────────┬─────────────┬─────────────────┐
  │ Algorithm      │ Comparisons │ Relative to n  │
  ├────────────────┼─────────────┼─────────────────┤
  │ Bubble Sort    │     45      │ n(n-1)/2 = 45  │
  │ Insertion Sort │    ~22      │ varies by input│
  │ Best possible  │    ~33      │ n log n ≈ 33   │
  └────────────────┴─────────────┴─────────────────┘

  Bubble Sort:    ███████████████████████  45 comps
  Insertion Sort: ███████████         ~22 comps
  Theoretical:    █████████████████   ~33 comps

  Bubble sort always does n(n-1)/2 comparisons.
  Insertion sort is adaptive: fewer for nearly sorted data.
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

**Textual Figure:**

```
Nearly Sorted Array (n=10000, 10 swaps):

  Before: [0, 1, 2, ..., 99, 101, 100, ..., 199, 201, 200, ...]
                           ^^^  ^^^
                        only 10 adjacent pairs swapped

  ┌─────────────────┬───────────────────┬────────────────┐
  │ Algorithm       │ Time (nearly     │ Why?           │
  │                 │ sorted)          │                │
  ├─────────────────┼───────────────────┼────────────────┤
  │ Insertion Sort  │ O(n) ≈ 10020    │ Few inversions │
  │ Go sort.Ints   │ O(n log n)       │ Always nlogn   │
  └─────────────────┴───────────────────┴────────────────┘

  Insertion sort inversions view:
  [0..99, 101, 100, 102..] → only 1 inversion per swap
     inner loop runs ~1 step per inversion
     10 inversions → 10 extra comparisons on top of n
     Total: ≈ n + 10 = O(n)  ← FAST!
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

**Textual Figure:**

```
Custom Comparison Sort (by Age, then Name):

  Input:
  ┌─────────┬─────┐
  │ Name    │ Age │
  ├─────────┼─────┤
  │ Alice   │  30 │
  │ Bob     │  25 │
  │ Charlie │  35 │
  │ Diana   │  25 │
  │ Eve     │  30 │
  └─────────┴─────┘

  Stable sort by Age, then Name:
  ┌─────────┬─────┐
  │ Name    │ Age │  Comparator: if age≠, sort by age
  ├─────────┼─────┤  else sort by name
  │ Bob     │  25 │  ← age 25 tie: Bob < Diana
  │ Diana   │  25 │  ← age 25 tie: Bob < Diana
  │ Alice   │  30 │  ← age 30 tie: Alice < Eve
  │ Eve     │  30 │  ← age 30 tie: Alice < Eve
  │ Charlie │  35 │
  └─────────┴─────┘

  SliceStable ensures tied elements stay in original order
  within the tie-breaking that our comparator defines.
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

**Textual Figure:**

```
Inversion Count: arr = [2, 4, 1, 3, 5]

  Inversions: pairs (i,j) where i<j but arr[i]>arr[j]
    (2,1)  (4,1)  (4,3)  = 3 inversions

  Merge Sort based counting:

        [2, 4, 1, 3, 5]
        /              \
     [2, 4]         [1, 3, 5]
     /    \          /      \
   [2]   [4]      [1]    [3, 5]
     \  /           \     /   \
   merge[2,4]      [3]  [5]
   inv=0             \  /
                   merge[3,5]
                   inv=0
                     \
                  merge[1,3,5]
                  inv=0
              \
         merge [2,4] + [1,3,5]
              1 < 2 → take 1, inv += 2 (both 2,4 > 1)
              2 < 3 → take 2, inv += 0
              3 < 4 → take 3, inv += 1 (4 > 3)
         split inversions = 3
         total = 0 + 0 + 3 = 3 ✔

  Reversed: [5,4,3,2,1] → 10 inversions (max for n=5)
  Formula: max inversions = n(n-1)/2 = 5×4/2 = 10
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

**Textual Figure:**

```
Comparison-Based Sorting Summary:

  ┌────────────┬────────────┬────────────┬────────────┬────────┬────────┐
  │ Algorithm  │ Best       │ Average    │ Worst      │ Space  │ Stable │
  ├────────────┼────────────┼────────────┼────────────┼────────┼────────┤
  │ Bubble     │ O(n)       │ O(n²)      │ O(n²)      │ O(1)   │ Yes    │
  │ Selection  │ O(n²)      │ O(n²)      │ O(n²)      │ O(1)   │ No     │
  │ Insertion  │ O(n)       │ O(n²)      │ O(n²)      │ O(1)   │ Yes    │
  │ Shell      │ O(n log n) │ O(n^1.5)   │ O(n²)      │ O(1)   │ No     │
  │ Merge      │ O(n log n) │ O(n log n) │ O(n log n) │ O(n)   │ Yes    │
  │ Quick      │ O(n log n) │ O(n log n) │ O(n²)      │ O(lgn) │ No     │
  │ Heap       │ O(n log n) │ O(n log n) │ O(n log n) │ O(1)   │ No     │
  └────────────┴────────────┴────────────┴────────────┴────────┴────────┘

  Performance tiers:
  ──────────────────────────────────────
  Tier 1 (O(n²)):  Bubble, Selection, Insertion
                    └→ Simple but slow for large n
  Tier 2 (O(nlogn)): Merge, Quick, Heap
                    └→ Optimal comparison sorts
  Lower bound: Ω(n log n) for all comparison sorts
```

---

## Key Takeaways

1. Comparison-based sorts have a theoretical lower bound of $\Omega(n \log n)$
2. O(n²) sorts (bubble, selection, insertion) are simple but slow for large n
3. Insertion sort is optimal for nearly-sorted data — O(n) best case
4. Stability matters when sort keys have ties and relative order matters
5. Inversion count measures how far from sorted an array is

> **Next up:** Merge Sort →
