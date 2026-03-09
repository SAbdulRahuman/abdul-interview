# Phase 16: Sorting Algorithms — Stability of Sorting Algorithms

## Overview

A sorting algorithm is **stable** if it preserves the relative order of elements with equal keys. Stability matters when sorting by multiple fields or when the original order has meaning.

| Stable Sorts | Unstable Sorts |
|--------------|----------------|
| Merge Sort | Quick Sort |
| Insertion Sort | Heap Sort |
| Bubble Sort | Selection Sort |
| Counting Sort | |
| Radix Sort | |
| Tim Sort | |

---

## Example 1: What Stability Means

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

func main() {
	students := []Student{
		{"Alice", 90},
		{"Bob", 85},
		{"Charlie", 90},
		{"Dave", 85},
	}

	// sort.SliceStable preserves original order for equal elements
	sort.SliceStable(students, func(i, j int) bool {
		return students[i].Grade < students[j].Grade
	})

	fmt.Println("Stable sort by grade:")
	for _, s := range students {
		fmt.Printf("  %s: %d\n", s.Name, s.Grade)
	}
	// Bob(85) before Dave(85) — original order preserved
	// Alice(90) before Charlie(90) — original order preserved
}
```

---

## Example 2: Unstable Sort Breaks Relative Order

```go
package main

import (
	"fmt"
	"sort"
)

type Item struct {
	Key   int
	Label string
}

func main() {
	items := []Item{
		{3, "A"}, {1, "B"}, {3, "C"}, {2, "D"}, {1, "E"},
	}

	// sort.Slice is NOT guaranteed stable
	sort.Slice(items, func(i, j int) bool {
		return items[i].Key < items[j].Key
	})

	fmt.Println("sort.Slice (unstable):")
	for _, it := range items {
		fmt.Printf("  Key=%d Label=%s\n", it.Key, it.Label)
	}
	// Items with Key=1 or Key=3 may swap relative order

	items2 := []Item{
		{3, "A"}, {1, "B"}, {3, "C"}, {2, "D"}, {1, "E"},
	}

	sort.SliceStable(items2, func(i, j int) bool {
		return items2[i].Key < items2[j].Key
	})

	fmt.Println("\nsort.SliceStable (stable):")
	for _, it := range items2 {
		fmt.Printf("  Key=%d Label=%s\n", it.Key, it.Label)
	}
	// Key=1: B before E (original order preserved)
	// Key=3: A before C (original order preserved)
}
```

---

## Example 3: Multi-Key Sorting with Stability

```go
package main

import (
	"fmt"
	"sort"
)

type Employee struct {
	Name string
	Dept string
	Age  int
}

func main() {
	employees := []Employee{
		{"Alice", "Engineering", 30},
		{"Bob", "Sales", 25},
		{"Charlie", "Engineering", 25},
		{"Dave", "Sales", 30},
		{"Eve", "Engineering", 28},
	}

	// Step 1: Sort by name (secondary key)
	sort.SliceStable(employees, func(i, j int) bool {
		return employees[i].Name < employees[j].Name
	})

	// Step 2: Sort by department (primary key) — stability preserves name order
	sort.SliceStable(employees, func(i, j int) bool {
		return employees[i].Dept < employees[j].Dept
	})

	fmt.Println("Sorted by Dept, then Name:")
	for _, e := range employees {
		fmt.Printf("  %s | %s | %d\n", e.Dept, e.Name, e.Age)
	}
	// Within each dept, names are alphabetical
}
```

---

## Example 4: Stable Insertion Sort Implementation

```go
package main

import "fmt"

type Pair struct {
	Key int
	Val string
}

func stableInsertionSort(arr []Pair) {
	for i := 1; i < len(arr); i++ {
		key := arr[i]
		j := i - 1
		// Use < (not <=) to maintain stability
		for j >= 0 && arr[j].Key > key.Key {
			arr[j+1] = arr[j]
			j--
		}
		arr[j+1] = key
	}
}

func main() {
	pairs := []Pair{
		{3, "first"}, {1, "second"}, {3, "third"}, {2, "fourth"},
	}
	stableInsertionSort(pairs)
	for _, p := range pairs {
		fmt.Printf("Key=%d Val=%s\n", p.Key, p.Val)
	}
	// Key=3: "first" before "third" — stable!
}
```

---

## Example 5: Why Selection Sort is Unstable

```go
package main

import "fmt"

type Card struct {
	Rank int
	Suit string
}

func selectionSort(cards []Card) {
	n := len(cards)
	for i := 0; i < n-1; i++ {
		minIdx := i
		for j := i + 1; j < n; j++ {
			if cards[j].Rank < cards[minIdx].Rank {
				minIdx = j
			}
		}
		// This SWAP can move equal elements past each other
		cards[i], cards[minIdx] = cards[minIdx], cards[i]
	}
}

func main() {
	cards := []Card{
		{5, "♥"}, {3, "♠"}, {5, "♣"}, {3, "♦"},
	}

	fmt.Println("Before:", cards)
	selectionSort(cards)
	fmt.Println("After: ", cards)
	// 5♥ and 5♣ may swap relative order — UNSTABLE
	// The swap in selection sort can jump over equal elements
}
```

---

## Example 6: Making Unstable Sort Stable (Index Trick)

```go
package main

import (
	"fmt"
	"sort"
)

type IndexedItem struct {
	Value    int
	OrigIdx  int
}

func makeStable(arr []int) []int {
	// Attach original index
	indexed := make([]IndexedItem, len(arr))
	for i, v := range arr {
		indexed[i] = IndexedItem{v, i}
	}

	// Sort with tie-breaking by original index
	sort.Slice(indexed, func(i, j int) bool {
		if indexed[i].Value != indexed[j].Value {
			return indexed[i].Value < indexed[j].Value
		}
		return indexed[i].OrigIdx < indexed[j].OrigIdx // stability
	})

	result := make([]int, len(arr))
	for i, item := range indexed {
		result[i] = item.Value
	}
	return result
}

func main() {
	arr := []int{3, 1, 3, 2, 1}
	fmt.Println("Stabilized sort:", makeStable(arr))
}
```

---

## Example 7: Stable Merge Sort

```go
package main

import "fmt"

type Record struct {
	Key int
	ID  string
}

func mergeSort(arr []Record) []Record {
	if len(arr) <= 1 { return arr }
	mid := len(arr) / 2
	left := mergeSort(append([]Record{}, arr[:mid]...))
	right := mergeSort(append([]Record{}, arr[mid:]...))
	return merge(left, right)
}

func merge(left, right []Record) []Record {
	result := make([]Record, 0, len(left)+len(right))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		// Use <= for stability: equal elements from left come first
		if left[i].Key <= right[j].Key {
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
	records := []Record{
		{3, "A"}, {1, "B"}, {3, "C"}, {2, "D"}, {1, "E"},
	}
	sorted := mergeSort(records)
	for _, r := range sorted {
		fmt.Printf("Key=%d ID=%s\n", r.Key, r.ID)
	}
	// Key=1: B before E — stable
	// Key=3: A before C — stable
}
```

---

## Example 8: Stable Counting Sort

```go
package main

import "fmt"

type Entry struct {
	Score int
	Name  string
}

func stableCountingSortEntries(arr []Entry) []Entry {
	if len(arr) == 0 { return arr }

	maxScore := arr[0].Score
	for _, e := range arr {
		if e.Score > maxScore { maxScore = e.Score }
	}

	count := make([]int, maxScore+1)
	for _, e := range arr {
		count[e.Score]++
	}

	// Prefix sum
	for i := 1; i <= maxScore; i++ {
		count[i] += count[i-1]
	}

	// Place in reverse order for stability
	output := make([]Entry, len(arr))
	for i := len(arr) - 1; i >= 0; i-- {
		count[arr[i].Score]--
		output[count[arr[i].Score]] = arr[i]
	}
	return output
}

func main() {
	entries := []Entry{
		{90, "Alice"}, {85, "Bob"}, {90, "Charlie"}, {85, "Dave"},
	}
	sorted := stableCountingSortEntries(entries)
	for _, e := range sorted {
		fmt.Printf("Score=%d Name=%s\n", e.Score, e.Name)
	}
	// Bob before Dave (both 85) — stable
	// Alice before Charlie (both 90) — stable
}
```

---

## Example 9: Verifying Sort Stability

```go
package main

import (
	"fmt"
	"sort"
)

type Tagged struct {
	Val int
	Tag int // original position
}

func isStable(original, sorted []Tagged) bool {
	for i := 0; i < len(sorted)-1; i++ {
		if sorted[i].Val == sorted[i+1].Val {
			if sorted[i].Tag > sorted[i+1].Tag {
				return false
			}
		}
	}
	return true
}

func main() {
	data := []Tagged{
		{3, 0}, {1, 1}, {3, 2}, {2, 3}, {1, 4},
	}

	// Test sort.SliceStable
	stable := make([]Tagged, len(data))
	copy(stable, data)
	sort.SliceStable(stable, func(i, j int) bool {
		return stable[i].Val < stable[j].Val
	})
	fmt.Println("SliceStable is stable:", isStable(data, stable)) // true

	// Test sort.Slice
	unstable := make([]Tagged, len(data))
	copy(unstable, data)
	sort.Slice(unstable, func(i, j int) bool {
		return unstable[i].Val < unstable[j].Val
	})
	fmt.Println("Slice is stable:", isStable(data, unstable)) // may be false
}
```

---

## Example 10: Stability Summary and Decision Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Sorting Stability Guide ===")
	fmt.Println()

	fmt.Println("Stable Algorithms:")
	fmt.Println("  Bubble Sort     O(n²) — stable by design (adjacent swaps)")
	fmt.Println("  Insertion Sort  O(n²) — stable with > instead of >=")
	fmt.Println("  Merge Sort      O(n log n) — stable with <= in merge")
	fmt.Println("  Counting Sort   O(n+k) — stable with reverse placement")
	fmt.Println("  Radix Sort      O(d(n+k)) — stable (uses counting sort)")
	fmt.Println("  Tim Sort        O(n log n) — stable (Go's sort.SliceStable)")
	fmt.Println()

	fmt.Println("Unstable Algorithms:")
	fmt.Println("  Selection Sort  O(n²) — swaps can jump over equals")
	fmt.Println("  Quick Sort      O(n log n) — partition may reorder equals")
	fmt.Println("  Heap Sort       O(n log n) — heap operations break order")
	fmt.Println()

	fmt.Println("When stability matters:")
	fmt.Println("  • Multi-key sorting (sort by secondary, then primary)")
	fmt.Println("  • Database queries with ORDER BY multiple columns")
	fmt.Println("  • Already partially sorted data with meaning")
	fmt.Println("  • UI sorting where user expects consistent ordering")
	fmt.Println()

	fmt.Println("Go standard library:")
	fmt.Println("  sort.Slice()       — NOT stable (uses introsort)")
	fmt.Println("  sort.SliceStable() — stable (uses merge sort variant)")
}
```

---

## Key Takeaways

1. Stable sort preserves relative order of equal elements
2. Essential for multi-field sorting — sort by secondary key first, then primary
3. Merge sort is the go-to O(n log n) stable sort
4. Any unstable sort can be made stable by appending original index as tie-breaker
5. Go: use `sort.SliceStable()` when stability is needed

> **Next up:** In-Place Sorting →
