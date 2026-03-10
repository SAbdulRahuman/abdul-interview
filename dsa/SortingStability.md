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

**Textual Figure:**

```
Stability: equal keys keep original relative order.

  Input:  [(Alice,90), (Bob,85), (Charlie,90), (Dave,85)]
            idx 0         idx 1       idx 2         idx 3

  Sort by Grade (stable):
  ┌─────────┬───────┬──────────────────────────────┐
  │ Grade   │ Name  │ Reason                       │
  ├─────────┼───────┼──────────────────────────────┤
  │   85    │ Bob   │ Was idx 1 (before Dave idx 3)│
  │   85    │ Dave  │ Was idx 3 (after Bob idx 1)  │
  │   90    │ Alice │ Was idx 0 (before Charlie 2) │
  │   90    │Charlie│ Was idx 2 (after Alice idx 0)│
  └─────────┴───────┴──────────────────────────────┘

  ✓ Equal grades → original order preserved.
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

**Textual Figure:**

```
Unstable vs Stable: items = [(3,A),(1,B),(3,C),(2,D),(1,E)]

  Unstable (sort.Slice) — may reorder equals:
  ┌───────┬───────┬─────────────────────┐
  │ Key=1 │ E, B  │ ✗ E before B (was  │
  │       │       │   B before E)       │
  │ Key=2 │ D     │                     │
  │ Key=3 │ C, A  │ ✗ C before A (was  │
  │       │       │   A before C)       │
  └───────┴───────┴─────────────────────┘

  Stable (sort.SliceStable) — preserves order:
  ┌───────┬───────┬─────────────────────┐
  │ Key=1 │ B, E  │ ✓ B before E        │
  │ Key=2 │ D     │                     │
  │ Key=3 │ A, C  │ ✓ A before C        │
  └───────┴───────┴─────────────────────┘
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

**Textual Figure:**

```
Multi-Key Sort: Dept (primary), Name (secondary)

  Step 1 — Stable sort by Name (secondary key):
  ┌─────────────┬───────────┬─────┐
  │ Alice       │ Eng       │  30 │
  │ Bob         │ Sales     │  25 │
  │ Charlie     │ Eng       │  25 │
  │ Dave        │ Sales     │  30 │
  │ Eve         │ Eng       │  28 │
  └─────────────┴───────────┴─────┘

  Step 2 — Stable sort by Dept (primary key):
  ┌─────────────┬───────────┬─────┐
  │ Alice       │ Eng       │  30 │  ← names in
  │ Charlie     │ Eng       │  25 │    alphabetical
  │ Eve         │ Eng       │  28 │    order within
  │ Bob         │ Sales     │  25 │    each dept
  │ Dave        │ Sales     │  30 │    (stable!)
  └─────────────┴───────────┴─────┘

  Stability preserves Step 1's name order within same dept.
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

**Textual Figure:**

```
Stable Insertion Sort: [(3,"first"),(1,"second"),(3,"third"),(2,"fourth")]

  i=1: key=(1,"second")
  ┌─────────┬──────────┬──────────┬──────────┐
  │(1,"sec")│(3,"fir") │(3,"thi") │(2,"fou") │
  └─────────┴──────────┴──────────┴──────────┘
           ↑ inserted before (3,"first") since 3 > 1

  i=2: key=(3,"third") — arr[1].Key=3, NOT > 3 → stop
  ┌─────────┬──────────┬──────────┬──────────┐
  │(1,"sec")│(3,"fir") │(3,"thi") │(2,"fou") │
  └─────────┴──────────┴──────────┴──────────┘
           (3,"first") stays before (3,"third") ✓ STABLE

  i=3: key=(2,"fourth")
  ┌─────────┬──────────┬──────────┬──────────┐
  │(1,"sec")│(2,"fou") │(3,"fir") │(3,"thi") │
  └─────────┴──────────┴──────────┴──────────┘

  Key: using > (not >=) in comparison → stable.
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

**Textual Figure:**

```
Why Selection Sort is Unstable:
  cards = [(5,♥), (3,♠), (5,♣), (3,♦)]

  i=0: find min → (3,♠) at idx 1
  Swap idx 0 ↔ idx 1:
  ┌───────┬───────┬───────┬───────┐
  │(3,♠)  │(5,♥)  │(5,♣)  │(3,♦)  │
  └───────┴───────┴───────┴───────┘
        ↑ swapped with (5,♥)

  i=1: find min → (3,♦) at idx 3
  Swap idx 1 ↔ idx 3:
  ┌───────┬───────┬───────┬───────┐
  │(3,♠)  │(3,♦)  │(5,♣)  │(5,♥)  │
  └───────┴───────┴───────┴───────┘
        ↑ swapped with (5,♥) which jumped to end!

  Original: 5♥ before 5♣
  After:    5♣ before 5♥  ← ORDER REVERSED! ✗ UNSTABLE

  The long-range swap skips over equal elements.
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

**Textual Figure:**

```
Making Unstable Sort Stable (Index Trick):
  arr = [3, 1, 3, 2, 1]

  Step 1 — Attach original index:
  ┌───────┬───────┬───────┬───────┬───────┐
  │(3, 0)│(1, 1)│(3, 2)│(2, 3)│(1, 4)│
  └───────┴───────┴───────┴───────┴───────┘

  Step 2 — Sort with tie-breaking by OrigIdx:
  Compare: if values equal, smaller index wins.
  ┌───────┬───────┬───────┬───────┬───────┐
  │(1, 1)│(1, 4)│(2, 3)│(3, 0)│(3, 2)│
  └───────┴───────┴───────┴───────┴───────┘
    idx1<idx4 ✓   idx0<idx2 ✓  → stable!

  Result: [1, 1, 2, 3, 3]
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

**Textual Figure:**

```
Stable Merge Sort: [(3,A),(1,B),(3,C),(2,D),(1,E)]

  Split:  [(3,A),(1,B),(3,C)]  |  [(2,D),(1,E)]
  Split:  [(3,A),(1,B)] [(3,C)]   [(2,D)] [(1,E)]
  Split:  [(3,A)] [(1,B)]

  Merge [(3,A)] + [(1,B)]:  1<3 → [(1,B),(3,A)]
  Merge [(1,B),(3,A)] + [(3,C)]:
    (1,B) first, then (3,A) vs (3,C): A.Key<=C.Key → A first
    → [(1,B),(3,A),(3,C)]   ← A before C (stable!)

  Merge [(2,D)] + [(1,E)]:  1<2 → [(1,E),(2,D)]

  Final merge:
    (1,B) vs (1,E): B.Key<=E.Key → B first
    → [(1,B),(1,E),(2,D),(3,A),(3,C)]

  Key: use <= (not <) when left.Key == right.Key
  → left element goes first → stability preserved.
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

**Textual Figure:**

```
Stable Counting Sort:
  entries = [(90,Alice),(85,Bob),(90,Charlie),(85,Dave)]

  Step 1 — Count:
  count[85]=2, count[90]=2

  Step 2 — Prefix sum:
  count[85]=2, count[90]=4

  Step 3 — Place in REVERSE order (right-to-left):
  ┌────┬─────────────┬─────────────┬─────────────────┐
  │ i  │ Entry       │ count[key] │ Place at        │
  ├────┼─────────────┼─────────────┼─────────────────┤
  │ 3  │ (85,Dave)   │ 2→1       │ output[1]       │
  │ 2  │ (90,Charlie)│ 4→3       │ output[3]       │
  │ 1  │ (85,Bob)    │ 1→0       │ output[0]       │
  │ 0  │ (90,Alice)  │ 3→2       │ output[2]       │
  └────┴─────────────┴─────────────┴─────────────────┘

  Result: [Bob(85), Dave(85), Alice(90), Charlie(90)]
  Reverse traversal + prefix sum = stable placement.
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

**Textual Figure:**

```
Verifying Stability:
  data = [(3,tag0),(1,tag1),(3,tag2),(2,tag3),(1,tag4)]

  After stable sort:
  ┌─────┬─────┬────────────────────────────┐
  │ Val │ Tag │ Check                      │
  ├─────┼─────┼────────────────────────────┤
  │  1  │  1  │                            │
  │  1  │  4  │ tag1 < tag4 ✓ stable      │
  │  2  │  3  │                            │
  │  3  │  0  │                            │
  │  3  │  2  │ tag0 < tag2 ✓ stable      │
  └─────┴─────┴────────────────────────────┘

  Verification rule:
  For consecutive equal values: tag[i] < tag[i+1] → stable.
  If any tag[i] > tag[i+1] for equal values → NOT stable.
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

**Textual Figure:**

```
Sorting Stability Decision Guide:

  ┌─────────────────────────────────────────────┐
  │ Stable                │ Unstable              │
  ├───────────────────────┼─────────────────────┤
  │ Bubble Sort    O(n²)  │ Selection Sort O(n²) │
  │ Insertion Sort O(n²)  │ Quick Sort  O(nlogn) │
  │ Merge Sort   O(nlogn) │ Heap Sort   O(nlogn) │
  │ Counting Sort  O(n+k) │                     │
  │ Radix Sort  O(d(n+k)) │                     │
  │ Tim Sort     O(nlogn) │                     │
  └───────────────────────┴─────────────────────┘

  Need stability?  ───Yes───→  sort.SliceStable()
         │
         No → sort.Slice() (faster, less memory)

  Multi-key: sort by least significant key first,
  then more significant keys → stability cascades.
```

---

## Key Takeaways

1. Stable sort preserves relative order of equal elements
2. Essential for multi-field sorting — sort by secondary key first, then primary
3. Merge sort is the go-to O(n log n) stable sort
4. Any unstable sort can be made stable by appending original index as tie-breaker
5. Go: use `sort.SliceStable()` when stability is needed

> **Next up:** In-Place Sorting →
