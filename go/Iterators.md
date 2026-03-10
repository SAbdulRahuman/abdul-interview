# Chapter 32 — Iterators & Range-Over-Func (Go 1.23+)

Go 1.23 introduced **range-over-func** — the ability to use `for range` over custom iterator functions. This eliminates the need for channels or collecting slices just to iterate. Iterators are defined using the `iter` package types `iter.Seq[V]` and `iter.Seq2[K, V]`.

```
┌──────────────────────────────────────────────────────────────┐
│             Range-Over-Func — Iterator Pattern               │
│                                                              │
│  Before Go 1.23:                                             │
│  • Collect all results into []T, then range over it          │
│  • Use channels (goroutine overhead, leak risk)              │
│  • Use callback func(T) bool pattern manually                │
│                                                              │
│  Go 1.23+:                                                   │
│  • iter.Seq[V]   = func(yield func(V) bool)                 │
│  • iter.Seq2[K,V]= func(yield func(K, V) bool)              │
│  • Use directly with for-range!                              │
│                                                              │
│  for v := range myIterator {                                 │
│      // v comes from yield(v) calls                          │
│  }                                                           │
│                                                              │
│  for k, v := range myIterator2 {                             │
│      // k, v come from yield(k, v) calls                     │
│  }                                                           │
└──────────────────────────────────────────────────────────────┘
```

## Iterator Function Signatures

**Tutorial: Understanding iter.Seq and iter.Seq2**

An iterator in Go 1.23+ is a function that accepts a `yield` callback. The iterator calls `yield` for each value. If `yield` returns `false`, the consumer broke out of the loop and the iterator should stop. The `iter` package defines two types:

```
┌────────────────────────────────────────────────────────────┐
│  iter.Seq[V]   = func(yield func(V) bool)                 │
│  → Single-value iterator (like iterating a set)            │
│  → for v := range seq { ... }                              │
│                                                            │
│  iter.Seq2[K,V] = func(yield func(K, V) bool)             │
│  → Two-value iterator (like iterating a map: key + value)  │
│  → for k, v := range seq2 { ... }                         │
│                                                            │
│  The yield function:                                       │
│  • yield(v) → returns true: consumer wants more            │
│  • yield(v) → returns false: consumer called break          │
│  • Iterator MUST stop when yield returns false              │
│                                                            │
│  ┌───────────────────────────────────────────────┐         │
│  │  Iterator calls:  yield(1) → true  (continue) │         │
│  │                   yield(2) → true  (continue) │         │
│  │                   yield(3) → false (break!)   │         │
│  │                   ← stop iterating             │         │
│  └───────────────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"iter"
)

// iter.Seq[V] — single value iterator
func Fibonacci(max int) iter.Seq[int] {
	return func(yield func(int) bool) {
		a, b := 0, 1
		for a <= max {
			if !yield(a) {
				return // consumer broke out
			}
			a, b = b, a+b
		}
	}
}

// iter.Seq2[K,V] — two-value iterator
func Enumerate[T any](s []T) iter.Seq2[int, T] {
	return func(yield func(int, T) bool) {
		for i, v := range s {
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	// Single-value: Fibonacci numbers up to 100
	fmt.Println("Fibonacci ≤ 100:")
	for n := range Fibonacci(100) {
		fmt.Printf("  %d", n)
	}
	fmt.Println()

	// Two-value: enumerate a slice
	fruits := []string{"apple", "banana", "cherry"}
	fmt.Println("\nEnumerated:")
	for i, fruit := range Enumerate(fruits) {
		fmt.Printf("  %d: %s\n", i, fruit)
	}
}
```

---

## Basic Iterator Patterns

### Filtering Iterator

**Tutorial: Lazy Filtering Without Allocating a New Slice**

A filter iterator wraps another iterator and only yields values that match a predicate. This is lazy — it doesn't allocate a new slice. Values are produced one at a time as the consumer ranges over them.

```
┌────────────────────────────────────────────────┐
│  Filter(seq, predicate)                        │
│                                                │
│  source: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]     │
│  predicate: n % 2 == 0 (even numbers)          │
│                                                │
│  yield flow:                                   │
│  source yields 1  → predicate false → skip     │
│  source yields 2  → predicate true  → yield(2) │
│  source yields 3  → predicate false → skip     │
│  source yields 4  → predicate true  → yield(4) │
│  ...                                           │
│  result: [2, 4, 6, 8, 10] (lazily produced)    │
└────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"iter"
)

// Filter returns an iterator that yields only values matching the predicate
func Filter[V any](seq iter.Seq[V], predicate func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range seq {
			if predicate(v) {
				if !yield(v) {
					return
				}
			}
		}
	}
}

// Range generates integers from start to end (exclusive)
func Range(start, end int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := start; i < end; i++ {
			if !yield(i) {
				return
			}
		}
	}
}

func main() {
	// Filter even numbers from 1..20
	evens := Filter(Range(1, 21), func(n int) bool {
		return n%2 == 0
	})

	fmt.Println("Even numbers 1-20:")
	for n := range evens {
		fmt.Printf("  %d", n)
	}
	fmt.Println()
}
```

### Mapping Iterator

**Tutorial: Transforming Values Lazily**

A map iterator applies a transformation function to each element from a source iterator, yielding the transformed values.

```go
package main

import (
	"fmt"
	"iter"
	"strings"
)

// Map transforms each value from an iterator
func Map[In, Out any](seq iter.Seq[In], fn func(In) Out) iter.Seq[Out] {
	return func(yield func(Out) bool) {
		for v := range seq {
			if !yield(fn(v)) {
				return
			}
		}
	}
}

// SliceIter creates an iterator from a slice
func SliceIter[T any](s []T) iter.Seq[T] {
	return func(yield func(T) bool) {
		for _, v := range s {
			if !yield(v) {
				return
			}
		}
	}
}

func main() {
	words := []string{"hello", "world", "go", "iterators"}

	// Map to uppercase
	upper := Map(SliceIter(words), strings.ToUpper)

	for w := range upper {
		fmt.Println(w)
	}
	// HELLO
	// WORLD
	// GO
	// ITERATORS
}
```

### Take / Limit Iterator

**Tutorial: Taking Only N Elements from an Iterator**

`Take` creates an iterator that yields at most N values from the source, then stops. This is useful for paginating through large or infinite sequences.

```go
package main

import (
	"fmt"
	"iter"
)

// Take yields at most n values from the source iterator
func Take[V any](seq iter.Seq[V], n int) iter.Seq[V] {
	return func(yield func(V) bool) {
		count := 0
		for v := range seq {
			if count >= n {
				return
			}
			if !yield(v) {
				return
			}
			count++
		}
	}
}

// Naturals generates 1, 2, 3, ... (infinite)
func Naturals() iter.Seq[int] {
	return func(yield func(int) bool) {
		n := 1
		for {
			if !yield(n) {
				return
			}
			n++
		}
	}
}

func main() {
	// Take first 5 natural numbers from an infinite iterator
	fmt.Println("First 5 naturals:")
	for n := range Take(Naturals(), 5) {
		fmt.Printf("  %d", n)
	}
	fmt.Println()
	// 1 2 3 4 5
}
```

---

## Chaining Iterators (Pipeline)

**Tutorial: Composing Multiple Iterator Operations**

Iterators compose naturally — chain `Filter`, `Map`, and `Take` to build processing pipelines. No intermediate slices are allocated; values flow one at a time through the chain.

```
┌──────────────────────────────────────────────────────┐
│  Pipeline:                                           │
│  Range(1, 100)                                       │
│      │   yields: 1, 2, 3, 4, 5, ...                 │
│      ▼                                               │
│  Filter(n % 3 == 0)                                  │
│      │   yields: 3, 6, 9, 12, 15, ...               │
│      ▼                                               │
│  Map(n * n)                                          │
│      │   yields: 9, 36, 81, 144, 225, ...            │
│      ▼                                               │
│  Take(5)                                             │
│      │   yields: 9, 36, 81, 144, 225                 │
│      ▼                                               │
│  for v := range ...                                  │
│                                                      │
│  No intermediate slices allocated!                   │
│  Each value flows through the chain one at a time.   │
└──────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"iter"
)

func Filter[V any](seq iter.Seq[V], pred func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range seq {
			if pred(v) {
				if !yield(v) {
					return
				}
			}
		}
	}
}

func Map[In, Out any](seq iter.Seq[In], fn func(In) Out) iter.Seq[Out] {
	return func(yield func(Out) bool) {
		for v := range seq {
			if !yield(fn(v)) {
				return
			}
		}
	}
}

func Take[V any](seq iter.Seq[V], n int) iter.Seq[V] {
	return func(yield func(V) bool) {
		count := 0
		for v := range seq {
			if count >= n {
				return
			}
			if !yield(v) {
				return
			}
			count++
		}
	}
}

func Range(start, end int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := start; i < end; i++ {
			if !yield(i) {
				return
			}
		}
	}
}

func main() {
	// Pipeline: multiples of 3 in 1..100, squared, first 5
	pipeline := Take(
		Map(
			Filter(Range(1, 100), func(n int) bool { return n%3 == 0 }),
			func(n int) int { return n * n },
		),
		5,
	)

	fmt.Println("First 5 squares of multiples of 3:")
	for v := range pipeline {
		fmt.Printf("  %d", v)
	}
	fmt.Println()
	// 9 36 81 144 225
}
```

---

## Seq2 — Two-Value Iterators

**Tutorial: Iterating with Key-Value Pairs**

`iter.Seq2[K, V]` is for iterators that produce pairs — like map entries, indexed elements, or key-value results. The standard library uses `Seq2` extensively in the `maps` and `slices` packages.

```go
package main

import (
	"fmt"
	"iter"
)

// Zip combines two slices into key-value pairs
func Zip[A, B any](as []A, bs []B) iter.Seq2[A, B] {
	return func(yield func(A, B) bool) {
		minLen := len(as)
		if len(bs) < minLen {
			minLen = len(bs)
		}
		for i := 0; i < minLen; i++ {
			if !yield(as[i], bs[i]) {
				return
			}
		}
	}
}

// MapEntries iterates over a map in a consistent way
func MapEntries[K comparable, V any](m map[K]V) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for k, v := range m {
			if !yield(k, v) {
				return
			}
		}
	}
}

func main() {
	names := []string{"Alice", "Bob", "Charlie"}
	ages := []int{30, 25, 35}

	fmt.Println("Zipped:")
	for name, age := range Zip(names, ages) {
		fmt.Printf("  %s is %d\n", name, age)
	}

	scores := map[string]int{"math": 95, "english": 88, "science": 92}
	fmt.Println("\nMap entries:")
	for subject, score := range MapEntries(scores) {
		fmt.Printf("  %s: %d\n", subject, score)
	}
}
```

---

## Standard Library Iterator Support

**Tutorial: Built-in Iterators in slices and maps Packages**

Go 1.23+ added iterator-returning functions to the `slices` and `maps` packages. These work directly with `for range`.

```
┌────────────────────────────────────────────────────────────┐
│  slices package (Go 1.23+):                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ slices.All(s)        → iter.Seq2[int, E]  (index,v) │  │
│  │ slices.Values(s)     → iter.Seq[E]        (values)  │  │
│  │ slices.Backward(s)   → iter.Seq2[int, E]  (reverse) │  │
│  │ slices.Chunk(s, n)   → iter.Seq([]E)      (chunks)  │  │
│  │ slices.Sorted(seq)   → []E      (collect + sort)    │  │
│  │ slices.Collect(seq)  → []E      (collect to slice)  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  maps package (Go 1.23+):                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ maps.All(m)          → iter.Seq2[K, V]  (all pairs) │  │
│  │ maps.Keys(m)         → iter.Seq[K]      (keys only) │  │
│  │ maps.Values(m)       → iter.Seq[V]      (vals only) │  │
│  │ maps.Collect(seq2)   → map[K]V  (collect to map)    │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"maps"
	"slices"
)

func main() {
	nums := []int{10, 20, 30, 40, 50}

	// slices.All — iterate with index
	fmt.Println("slices.All:")
	for i, v := range slices.All(nums) {
		fmt.Printf("  [%d] = %d\n", i, v)
	}

	// slices.Values — iterate values only
	fmt.Print("slices.Values: ")
	for v := range slices.Values(nums) {
		fmt.Printf("%d ", v)
	}
	fmt.Println()

	// slices.Backward — reverse iteration
	fmt.Print("slices.Backward: ")
	for _, v := range slices.Backward(nums) {
		fmt.Printf("%d ", v)
	}
	fmt.Println()

	// slices.Chunk — split into chunks of size n
	fmt.Println("slices.Chunk(3):")
	for chunk := range slices.Chunk(nums, 3) {
		fmt.Printf("  %v\n", chunk)
	}

	// maps.Keys — iterate map keys
	m := map[string]int{"a": 1, "b": 2, "c": 3}
	fmt.Print("maps.Keys: ")
	for k := range maps.Keys(m) {
		fmt.Printf("%s ", k)
	}
	fmt.Println()

	// slices.Collect — materialize an iterator into a slice
	fibs := slices.Collect(Take(Fibonacci(), 8))
	fmt.Println("Collected Fibonacci:", fibs)
}

// Helper iterators for the example
func Take[V any](seq func(func(V) bool), n int) func(func(V) bool) {
	return func(yield func(V) bool) {
		count := 0
		seq(func(v V) bool {
			if count >= n {
				return false
			}
			count++
			return yield(v)
		})
	}
}

func Fibonacci() func(func(int) bool) {
	return func(yield func(int) bool) {
		a, b := 0, 1
		for {
			if !yield(a) {
				return
			}
			a, b = b, a+b
		}
	}
}
```

---

## Custom Data Structure Iterators

**Tutorial: Adding Range Support to Your Own Types**

Any type can support `for range` by providing methods that return `iter.Seq` or `iter.Seq2`. This is the idiomatic way to make custom collections iterable.

```
┌────────────────────────────────────────────────────────────┐
│  type TreeNode[T cmp.Ordered] struct {                     │
│      Value       T                                         │
│      Left, Right *TreeNode[T]                              │
│  }                                                         │
│                                                            │
│  // Returns iter.Seq[T] — in-order traversal               │
│  func (t *TreeNode[T]) InOrder() iter.Seq[T]               │
│                                                            │
│  Usage:                                                    │
│  for v := range tree.InOrder() {                           │
│      fmt.Println(v)  // sorted order!                      │
│  }                                                         │
│                                                            │
│       4                                                    │
│      / \          InOrder yields: 1, 2, 3, 4, 5, 6, 7     │
│     2   6                                                  │
│    / \ / \                                                 │
│   1  3 5  7                                                │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"cmp"
	"fmt"
	"iter"
)

type TreeNode[T cmp.Ordered] struct {
	Value       T
	Left, Right *TreeNode[T]
}

func (t *TreeNode[T]) Insert(val T) *TreeNode[T] {
	if t == nil {
		return &TreeNode[T]{Value: val}
	}
	if val < t.Value {
		t.Left = t.Left.Insert(val)
	} else if val > t.Value {
		t.Right = t.Right.Insert(val)
	}
	return t
}

// InOrder returns an iterator that yields values in sorted order
func (t *TreeNode[T]) InOrder() iter.Seq[T] {
	return func(yield func(T) bool) {
		t.inOrder(yield)
	}
}

func (t *TreeNode[T]) inOrder(yield func(T) bool) bool {
	if t == nil {
		return true
	}
	// Left → Value → Right
	if !t.Left.inOrder(yield) {
		return false
	}
	if !yield(t.Value) {
		return false
	}
	return t.Right.inOrder(yield)
}

// PreOrder — root first
func (t *TreeNode[T]) PreOrder() iter.Seq[T] {
	return func(yield func(T) bool) {
		t.preOrder(yield)
	}
}

func (t *TreeNode[T]) preOrder(yield func(T) bool) bool {
	if t == nil {
		return true
	}
	if !yield(t.Value) {
		return false
	}
	if !t.Left.preOrder(yield) {
		return false
	}
	return t.Right.preOrder(yield)
}

func main() {
	var root *TreeNode[int]
	for _, v := range []int{4, 2, 6, 1, 3, 5, 7} {
		root = root.Insert(v)
	}

	fmt.Print("InOrder:  ")
	for v := range root.InOrder() {
		fmt.Printf("%d ", v)
	}
	fmt.Println() // 1 2 3 4 5 6 7

	fmt.Print("PreOrder: ")
	for v := range root.PreOrder() {
		fmt.Printf("%d ", v)
	}
	fmt.Println() // 4 2 1 3 6 5 7

	// Break early — iterator handles it correctly
	fmt.Print("First 3:  ")
	count := 0
	for v := range root.InOrder() {
		fmt.Printf("%d ", v)
		count++
		if count == 3 {
			break
		}
	}
	fmt.Println() // 1 2 3
}
```

---

## Pull-Based Iterators

**Tutorial: Converting Push Iterators to Pull with iter.Pull**

`iter.Pull` converts a push-based iterator (func that calls yield) into a pull-based one — you explicitly call `next()` to get each value. This is useful when you need manual control over iteration, like merging two sorted sequences or implementing coroutine-style logic.

```
┌────────────────────────────────────────────────────────────┐
│  Push Iterator (normal):                                   │
│  for v := range seq { ... }   ← runtime calls yield       │
│                                                            │
│  Pull Iterator (via iter.Pull):                            │
│  next, stop := iter.Pull(seq)                              │
│  v, ok := next()        ← YOU call next                    │
│  v, ok = next()         ← YOU call next again              │
│  stop()                 ← YOU decide when to stop          │
│                                                            │
│  ┌──────────────────────────────────────────────┐          │
│  │  next, stop := iter.Pull(Fibonacci(100))     │          │
│  │  defer stop()                                │          │
│  │                                              │          │
│  │  v1, ok := next()  → (0, true)               │          │
│  │  v2, ok := next()  → (1, true)               │          │
│  │  v3, ok := next()  → (1, true)               │          │
│  │  v4, ok := next()  → (2, true)               │          │
│  │  ...                                         │          │
│  │  vN, ok := next()  → (0, false) exhausted    │          │
│  └──────────────────────────────────────────────┘          │
│                                                            │
│  ⚠️ Always defer stop() to avoid goroutine leak            │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"fmt"
	"iter"
)

func Fibonacci(max int) iter.Seq[int] {
	return func(yield func(int) bool) {
		a, b := 0, 1
		for a <= max {
			if !yield(a) {
				return
			}
			a, b = b, a+b
		}
	}
}

// MergeSorted merges two sorted iterators into one sorted iterator
// Uses iter.Pull to control both sequences manually
func MergeSorted(a, b iter.Seq[int]) iter.Seq[int] {
	return func(yield func(int) bool) {
		nextA, stopA := iter.Pull(a)
		defer stopA()
		nextB, stopB := iter.Pull(b)
		defer stopB()

		valA, okA := nextA()
		valB, okB := nextB()

		for okA && okB {
			if valA <= valB {
				if !yield(valA) {
					return
				}
				valA, okA = nextA()
			} else {
				if !yield(valB) {
					return
				}
				valB, okB = nextB()
			}
		}

		// Drain remaining
		for okA {
			if !yield(valA) {
				return
			}
			valA, okA = nextA()
		}
		for okB {
			if !yield(valB) {
				return
			}
			valB, okB = nextB()
		}
	}
}

func SliceIter(s []int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for _, v := range s {
			if !yield(v) {
				return
			}
		}
	}
}

func main() {
	// Pull-based: manually control iteration
	next, stop := iter.Pull(Fibonacci(50))
	defer stop()

	fmt.Println("Pull-based Fibonacci:")
	for {
		v, ok := next()
		if !ok {
			break
		}
		fmt.Printf("  %d", v)
	}
	fmt.Println()

	// MergeSorted: combining two sorted sequences
	a := SliceIter([]int{1, 3, 5, 7, 9})
	b := SliceIter([]int{2, 4, 6, 8, 10})

	fmt.Print("\nMerged: ")
	for v := range MergeSorted(a, b) {
		fmt.Printf("%d ", v)
	}
	fmt.Println()
	// 1 2 3 4 5 6 7 8 9 10
}
```

---

## Interview Questions

1. **What is range-over-func in Go 1.23?**
   - Go 1.23 allows `for range` to iterate over functions matching `func(yield func(V) bool)` or `func(yield func(K, V) bool)`. The function calls `yield` for each value. When the consumer `break`s, `yield` returns `false` and the iterator should stop.

2. **What are `iter.Seq` and `iter.Seq2`?**
   - `iter.Seq[V]` is `func(yield func(V) bool)` — single-value iterator. `iter.Seq2[K, V]` is `func(yield func(K, V) bool)` — two-value iterator. They're type aliases in the `iter` package that standardize iterator signatures.

3. **What happens if an iterator ignores yield returning false?**
   - The runtime panics. When the consumer breaks out of a `for range` loop, `yield` returns `false`. The iterator function must check this and return. Ignoring it violates the contract and causes a runtime error.

4. **What is `iter.Pull` and when is it useful?**
   - `iter.Pull(seq)` converts a push-based iterator into a pull-based one, returning `(next func() (V, bool), stop func())`. Useful when you need manual control — e.g., merging two sorted sequences, interleaving, or coroutine-style logic. Always `defer stop()` to avoid goroutine leaks.

5. **How do iterators compare to channels for iteration?**
   - Iterators are cheaper (no goroutine, no channel overhead), can't leak as easily, and compose better. Channels require a goroutine and can leak if the consumer doesn't drain them. Iterators are synchronous; channels enable async/concurrent patterns.

6. **What standard library functions return iterators?**
   - `slices.All`, `slices.Values`, `slices.Backward`, `slices.Chunk`, `maps.All`, `maps.Keys`, `maps.Values`. `slices.Collect` and `maps.Collect` go the other direction — materializing iterators into concrete types.

---

## Interview Problems & Solutions

### Problem 1 — Implement a Paginated Database Iterator

**Problem:** You have a function that fetches records from a database in pages of N records. Implement an iterator that transparently fetches pages and yields individual records, so the consumer can simply `for record := range AllRecords(pageSize)`.

```go
package main

import (
	"fmt"
	"iter"
)

type Record struct {
	ID   int
	Name string
}

// Simulated database — returns a page of records starting from offset
func fetchPage(offset, limit int) []Record {
	// Simulate 25 total records
	totalRecords := 25
	var page []Record
	for i := offset; i < offset+limit && i < totalRecords; i++ {
		page = append(page, Record{ID: i + 1, Name: fmt.Sprintf("Record-%d", i+1)})
	}
	return page
}

// AllRecords returns an iterator that lazily fetches pages
func AllRecords(pageSize int) iter.Seq[Record] {
	return func(yield func(Record) bool) {
		offset := 0
		for {
			page := fetchPage(offset, pageSize)
			if len(page) == 0 {
				return // no more records
			}
			for _, record := range page {
				if !yield(record) {
					return // consumer stopped
				}
			}
			if len(page) < pageSize {
				return // last page (partial)
			}
			offset += pageSize
		}
	}
}

func main() {
	fmt.Println("All records (page size 10):")
	count := 0
	for r := range AllRecords(10) {
		fmt.Printf("  %d: %s\n", r.ID, r.Name)
		count++
	}
	fmt.Printf("Total: %d records\n\n", count)

	// Early break — only read 5 records
	fmt.Println("First 5 records:")
	n := 0
	for r := range AllRecords(10) {
		fmt.Printf("  %d: %s\n", r.ID, r.Name)
		n++
		if n == 5 {
			break // fetchPage called only once (page size 10, took 5)
		}
	}
}
```

**Key Points:**
- Pages are fetched lazily — only when the consumer needs more records
- Breaking early avoids fetching unnecessary pages
- Consumer code is clean — no pagination logic exposed

---

### Problem 2 — Reduce / Fold Over an Iterator

**Problem:** Implement `Reduce` that takes an iterator and a combining function, producing a single result. Use it to compute sum, product, and find the maximum of a sequence without collecting to a slice.

```go
package main

import (
	"fmt"
	"iter"
)

// Reduce folds an iterator into a single value
func Reduce[V any](seq iter.Seq[V], initial V, fn func(acc, val V) V) V {
	acc := initial
	for v := range seq {
		acc = fn(acc, v)
	}
	return acc
}

// Range iterator
func Range(start, end int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := start; i < end; i++ {
			if !yield(i) {
				return
			}
		}
	}
}

// Filter iterator
func Filter[V any](seq iter.Seq[V], pred func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range seq {
			if pred(v) {
				if !yield(v) {
					return
				}
			}
		}
	}
}

func main() {
	nums := Range(1, 11) // 1..10

	// Sum
	sum := Reduce(Range(1, 11), 0, func(acc, v int) int { return acc + v })
	fmt.Println("Sum 1..10:", sum) // 55

	// Product
	product := Reduce(Range(1, 11), 1, func(acc, v int) int { return acc * v })
	fmt.Println("Product 1..10:", product) // 3628800

	// Max
	max := Reduce(nums, 0, func(acc, v int) int {
		if v > acc {
			return v
		}
		return acc
	})
	fmt.Println("Max 1..10:", max) // 10

	// Sum of even numbers only (pipeline)
	evenSum := Reduce(
		Filter(Range(1, 101), func(n int) bool { return n%2 == 0 }),
		0,
		func(acc, v int) int { return acc + v },
	)
	fmt.Println("Sum of evens 1..100:", evenSum) // 2550
}
```

**Key Points:**
- `Reduce` consumes an iterator, producing a single value
- Combines with `Filter`, `Map`, `Take` for full functional pipelines
- No intermediate allocations — everything streams through
