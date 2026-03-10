# Phase 22: Segment Tree & Fenwick Tree — Coordinate Compression

## Overview

**Coordinate Compression** maps large or sparse value ranges to a compact range `[0, k)` where `k` is the number of distinct values. This enables using array-based data structures (BIT, segment tree) on values that could be up to $10^9$.

**Key Idea**: Only relative order matters, not actual values.

| Before | After Compression |
|--------|------------------|
| Values: {100, 5000, 3, 999999} | → {1, 3, 0, 2} |
| Range: [0, 10^9] | → [0, 3] |

---

## Example 1: Basic Coordinate Compression

```go
package main

import (
	"fmt"
	"sort"
)

func compress(arr []int) ([]int, []int) {
	// Get sorted unique values
	sorted := make([]int, len(arr))
	copy(sorted, arr)
	sort.Ints(sorted)
	unique := []int{sorted[0]}
	for i := 1; i < len(sorted); i++ {
		if sorted[i] != sorted[i-1] { unique = append(unique, sorted[i]) }
	}

	// Map each value to its rank
	rank := map[int]int{}
	for i, v := range unique { rank[v] = i }

	compressed := make([]int, len(arr))
	for i, v := range arr { compressed[i] = rank[v] }
	return compressed, unique
}

func main() {
	arr := []int{100, 5000, 3, 999999, 3, 100}
	compressed, mapping := compress(arr)

	fmt.Printf("Original:   %v\n", arr)
	fmt.Printf("Compressed: %v\n", compressed)
	fmt.Printf("Mapping:    %v\n", mapping)
	fmt.Println("\nOnly", len(mapping), "distinct values instead of range [0, 999999]")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Basic Coordinate Compression                   │
├─────────────────────────────────────────────────┤
│  Original: [100, 5000, 3, 999999, 3, 100]       │
│                                                 │
│  Step 1: Sort unique values                     │
│    [3, 100, 5000, 999999]                       │
│                                                 │
│  Step 2: Assign ranks                           │
│    3 → 0,  100 → 1,  5000 → 2,  999999 → 3    │
│                                                 │
│  Step 3: Replace originals                      │
│    [100, 5000, 3, 999999, 3, 100]               │
│     ↓     ↓    ↓    ↓      ↓   ↓                  │
│    [ 1,    2,  0,    3,    0,  1]               │
│                                                 │
│  Range: [0, 999999] → [0, 3]  (4 distinct)     │
└─────────────────────────────────────────────────┘
```

---

## Example 2: Count Inversions with Compression + BIT

```go
package main

import (
	"fmt"
	"sort"
)

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+2), n: n + 1} }
func (b *BIT) Update(i int) { for i++; i <= b.n; i += i & (-i) { b.tree[i]++ } }
func (b *BIT) Query(i int) int { s := 0; for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s }

func countInversions(arr []int) int {
	// Compress
	sorted := make([]int, len(arr))
	copy(sorted, arr)
	sort.Ints(sorted)
	sorted = unique(sorted)
	rank := map[int]int{}
	for i, v := range sorted { rank[v] = i }

	bit := NewBIT(len(sorted))
	count := 0

	for i := len(arr) - 1; i >= 0; i-- {
		r := rank[arr[i]]
		count += bit.Query(r - 1)
		bit.Update(r)
	}
	return count
}

func unique(a []int) []int {
	if len(a) == 0 { return a }
	r := []int{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { r = append(r, a[i]) } }
	return r
}

func main() {
	tests := [][]int{
		{7, 5, 6, 4},
		{1000000, 1, 500000, 2},
		{5, 4, 3, 2, 1},
	}

	for _, arr := range tests {
		fmt.Printf("  %v → inversions = %d\n", arr, countInversions(arr))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Count Inversions with BIT + Compression        │
├─────────────────────────────────────────────────┤
│  arr = [7, 5, 6, 4]                             │
│  Compressed ranks: {4→0, 5→1, 6→2, 7→3}      │
│  → [3, 1, 2, 0]                                 │
│                                                 │
│  Process right-to-left, query BIT:              │
│    i=3: val=0, query(0-1)=0, add 0. inv +=0    │
│    i=2: val=2, query(2-1)=1, add 2. inv +=1    │
│    i=1: val=1, query(1-1)=1, add 1. inv +=1    │
│    i=0: val=3, query(3-1)=3, add 3. inv +=3    │
│                                                 │
│  Total inversions: 0+1+1+3 = 5                  │
│  BIT enables O(n log n) counting                │
└─────────────────────────────────────────────────┘
```

---

## Example 3: Count Smaller Numbers After Self (LeetCode 315)

```go
package main

import (
	"fmt"
	"sort"
)

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+2), n: n + 1} }
func (b *BIT) Update(i int) { for i++; i <= b.n; i += i & (-i) { b.tree[i]++ } }
func (b *BIT) Query(i int) int { s := 0; for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s }

func countSmaller(nums []int) []int {
	// Coordinate compression
	sorted := make([]int, len(nums))
	copy(sorted, nums)
	sort.Ints(sorted)
	sorted = unique(sorted)
	rank := map[int]int{}
	for i, v := range sorted { rank[v] = i }

	n := len(nums)
	result := make([]int, n)
	bit := NewBIT(len(sorted))

	for i := n - 1; i >= 0; i-- {
		r := rank[nums[i]]
		result[i] = bit.Query(r - 1)
		bit.Update(r)
	}
	return result
}

func unique(a []int) []int {
	if len(a) == 0 { return a }
	r := []int{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { r = append(r, a[i]) } }
	return r
}

func main() {
	tests := [][]int{
		{5, 2, 6, 1},
		{-1},
		{-1, -1},
		{26, 78, 27, 100, 33, 67, 90, 23, 66, 5, 38, 7, 35, 23, 52, 22, 83, 51, 98, 69},
	}

	for _, nums := range tests {
		fmt.Printf("  %v\n  → %v\n\n", nums, countSmaller(nums))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Count Smaller After Self                       │
├─────────────────────────────────────────────────┤
│  nums = [5, 2, 6, 1]                            │
│  Ranks: {1→0, 2→1, 5→2, 6→3}                 │
│                                                 │
│  Process right-to-left:                         │
│    i=3: add rank 0 (val=1)                      │
│          BIT: [1,0,0,0]  query(-1)=0           │
│    i=2: add rank 3 (val=6)                      │
│          BIT: [1,0,0,1]  query(2)=1            │
│    i=1: add rank 1 (val=2)                      │
│          BIT: [1,1,0,1]  query(0)=1            │
│    i=0: add rank 2 (val=5)                      │
│          BIT: [1,1,1,1]  query(1)=2            │
│                                                 │
│  Result: [2, 1, 1, 0]                           │
└─────────────────────────────────────────────────┘
```

---

## Example 4: Range Sum with Coordinate Compression

```go
package main

import (
	"fmt"
	"sort"
)

type BIT struct {
	tree []int64
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int64, n+2), n: n + 1} }
func (b *BIT) Update(i int, v int64) { for i++; i <= b.n; i += i & (-i) { b.tree[i] += v } }
func (b *BIT) Query(i int) int64 { s := int64(0); for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s }
func (b *BIT) RangeQuery(l, r int) int64 {
	if l == 0 { return b.Query(r) }
	return b.Query(r) - b.Query(l-1)
}

func main() {
	// Sparse points on a huge range
	type Event struct {
		pos   int
		value int64
	}
	events := []Event{
		{100, 5}, {1000000, 3}, {50, 7}, {999999999, 10}, {100, 2},
	}

	// Collect all positions and compress
	positions := make([]int, len(events))
	for i, e := range events { positions[i] = e.pos }
	sort.Ints(positions)
	positions = uniqueInts(positions)
	rank := map[int]int{}
	for i, v := range positions { rank[v] = i }

	bit := NewBIT(len(positions))
	for _, e := range events {
		bit.Update(rank[e.pos], e.value)
	}

	fmt.Printf("Positions: %v\n", positions)
	fmt.Printf("Sum of all: %d\n", bit.Query(len(positions)-1))
	fmt.Printf("Sum at pos 100: %d\n", bit.RangeQuery(rank[100], rank[100]))
}

func uniqueInts(a []int) []int {
	if len(a) == 0 { return a }
	r := []int{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { r = append(r, a[i]) } }
	return r
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Range Sum on Sparse Coordinates                │
├─────────────────────────────────────────────────┤
│  Events at positions: 50, 100, 10⁶, 10⁹-1       │
│  Range [0, 10⁹] is huge but only 4 positions!   │
│                                                 │
│  Compress:                                      │
│  pos:   50  100  10⁶   999999999               │
│  rank:   0    1    2        3                    │
│                                                 │
│  BIT on ranks [0..3]:                           │
│  ┌───┬───┬───┬───┐                                │
│  │ 7 │ 7 │ 3 │10 │  (values at each rank)      │
│  └───┴───┴───┴───┘                                │
│  Sum all = 27,  Sum at pos 100 = 7              │
│                                                 │
│  BIT size = 4 instead of 10⁹!                   │
└─────────────────────────────────────────────────┘
```

---

## Example 5: Count of Range Sum (LeetCode 327)

```go
package main

import (
	"fmt"
	"sort"
)

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+2), n: n + 1} }
func (b *BIT) Update(i int) { for i++; i <= b.n; i += i & (-i) { b.tree[i]++ } }
func (b *BIT) Query(i int) int { s := 0; for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s }
func (b *BIT) RangeCount(l, r int) int {
	if l > r { return 0 }
	if l == 0 { return b.Query(r) }
	return b.Query(r) - b.Query(l-1)
}

func countRangeSum(nums []int, lower, upper int) int {
	n := len(nums)
	prefix := make([]int64, n+1)
	for i := 0; i < n; i++ {
		prefix[i+1] = prefix[i] + int64(nums[i])
	}

	// Collect all prefix sums and their shifted versions for compression
	vals := make([]int64, 0, 3*(n+1))
	for _, p := range prefix {
		vals = append(vals, p, p-int64(lower), p-int64(upper))
	}
	sort.Slice(vals, func(i, j int) bool { return vals[i] < vals[j] })
	vals = uniqueInt64(vals)
	rank := map[int64]int{}
	for i, v := range vals { rank[v] = i }

	bit := NewBIT(len(vals))
	count := 0

	for i := 0; i <= n; i++ {
		// Count prefix[j] where lower <= prefix[i] - prefix[j] <= upper
		// → prefix[i]-upper <= prefix[j] <= prefix[i]-lower
		lo := rank[prefix[i]-int64(upper)]
		hi := rank[prefix[i]-int64(lower)]
		count += bit.RangeCount(lo, hi)
		bit.Update(rank[prefix[i]])
	}
	return count
}

func uniqueInt64(a []int64) []int64 {
	if len(a) == 0 { return a }
	r := []int64{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { r = append(r, a[i]) } }
	return r
}

func main() {
	tests := []struct {
		nums        []int
		lower, upper int
	}{
		{[]int{-2, 5, -1}, -2, 2},
		{[]int{0}, 0, 0},
	}

	for _, t := range tests {
		fmt.Printf("  nums=%v, [%d,%d] → %d\n",
			t.nums, t.lower, t.upper, countRangeSum(t.nums, t.lower, t.upper))
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Count of Range Sum                             │
├─────────────────────────────────────────────────┤
│  nums=[-2,5,-1], lower=-2, upper=2              │
│                                                 │
│  Prefix sums: [0, -2, 3, 2]                     │
│                                                 │
│  For each i, count j<i where:                   │
│    lower ≤ prefix[i]-prefix[j] ≤ upper           │
│    → prefix[i]-upper ≤ prefix[j] ≤ prefix[i]-lower│
│                                                 │
│  i=0: prefix=0, range [-2,2] → 0 matches       │
│  i=1: prefix=-2, range [-4,0]                   │
│    prefix[0]=0 is in [-4,0] ✓                   │
│  i=2: prefix=3, range [1,5]                     │
│    prefix[1]=-2 ✘, prefix[0]=0 ✘               │
│  i=3: prefix=2, range [0,4]                     │
│    prefix[0]=0 ✓, prefix[2]=3 ✓                │
│  Total: 3                                       │
└─────────────────────────────────────────────────┘
```

---

## Example 6: Rectangle Area with Sweep Line + Compression

```go
package main

import (
	"fmt"
	"sort"
)

// Compute total area covered by rectangles using sweep line + coordinate compression

func computeArea(rects [][4]int) int {
	if len(rects) == 0 { return 0 }

	// Collect all x-coordinates
	xCoords := []int{}
	type Event struct {
		y, x1, x2, typ int // typ: +1 enter, -1 exit
	}
	events := []Event{}

	for _, r := range rects {
		x1, y1, x2, y2 := r[0], r[1], r[2], r[3]
		xCoords = append(xCoords, x1, x2)
		events = append(events, Event{y1, x1, x2, 1}, Event{y2, x1, x2, -1})
	}

	sort.Slice(events, func(i, j int) bool { return events[i].y < events[j].y })
	sort.Ints(xCoords)
	xCoords = uniqueInts(xCoords)
	xRank := map[int]int{}
	for i, v := range xCoords { xRank[v] = i }

	n := len(xCoords)
	count := make([]int, n) // count of active rectangles covering each x-segment

	totalArea := 0
	prevY := events[0].y

	for _, e := range events {
		if e.y != prevY {
			// Calculate active width
			activeWidth := 0
			for i := 0; i < n-1; i++ {
				if count[i] > 0 {
					activeWidth += xCoords[i+1] - xCoords[i]
				}
			}
			totalArea += activeWidth * (e.y - prevY)
			prevY = e.y
		}
		// Update segment counts
		l, r := xRank[e.x1], xRank[e.x2]
		for i := l; i < r; i++ { count[i] += e.typ }
	}
	return totalArea
}

func uniqueInts(a []int) []int {
	if len(a) == 0 { return a }
	r := []int{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { r = append(r, a[i]) } }
	return r
}

func main() {
	rects := [][4]int{
		{0, 0, 4, 4},
		{2, 2, 6, 6},
	}
	fmt.Printf("Rectangles: %v\n", rects)
	fmt.Printf("Total area: %d\n", computeArea(rects))
	// Two 4×4 squares overlapping in 2×2 region → 28
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Rectangle Area: Sweep Line + Compression       │
├─────────────────────────────────────────────────┤
│  R1=(0,0,4,4)  R2=(2,2,6,6)                     │
│                                                 │
│  6 │    ┌────────┐                               │
│  4 │────┼────┐   │  overlap region            │
│    │    │████│   │  (2,2)-(4,4) = 4 area      │
│  2 │    └────┼───┘                               │
│    │         │                                   │
│  0 ┴─────────┘                                   │
│    0  2  4  6                                    │
│                                                 │
│  Sweep line (bottom to top), track active x     │
│  Compress x-coords: {0,2,4,6} → ranks {0,1,2,3}│
│  Area = 16 + 16 - 4 = 28                        │
└─────────────────────────────────────────────────┘
```

---

## Example 7: Offline Queries with Compression

```go
package main

import (
	"fmt"
	"sort"
)

// Process queries offline: for each query [l,r], count distinct values

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+2), n: n + 1} }
func (b *BIT) Update(i, v int) { for i++; i <= b.n; i += i & (-i) { b.tree[i] += v } }
func (b *BIT) Query(i int) int { s := 0; for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s }

func countDistinct(arr []int, queries [][2]int) []int {
	n := len(arr)
	nq := len(queries)

	// Sort queries by right endpoint
	type Query struct{ l, r, idx int }
	qs := make([]Query, nq)
	for i, q := range queries { qs[i] = Query{q[0], q[1], i} }
	sort.Slice(qs, func(i, j int) bool { return qs[i].r < qs[j].r })

	bit := NewBIT(n)
	lastSeen := map[int]int{} // value → last index
	results := make([]int, nq)
	qi := 0

	for i := 0; i < n && qi < nq; i++ {
		if prev, ok := lastSeen[arr[i]]; ok {
			bit.Update(prev, -1)
		}
		bit.Update(i, 1)
		lastSeen[arr[i]] = i

		for qi < nq && qs[qi].r == i {
			q := qs[qi]
			if q.l == 0 {
				results[q.idx] = bit.Query(q.r)
			} else {
				results[q.idx] = bit.Query(q.r) - bit.Query(q.l-1)
			}
			qi++
		}
	}
	return results
}

func main() {
	arr := []int{1, 2, 1, 3, 2, 1, 4}
	queries := [][2]int{{0, 6}, {0, 2}, {3, 6}, {1, 4}}

	results := countDistinct(arr, queries)
	fmt.Printf("Array: %v\n\n", arr)
	for i, q := range queries {
		fmt.Printf("  Distinct [%d,%d] = %d\n", q[0], q[1], results[i])
	}
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Offline Queries: Count Distinct in Range       │
├─────────────────────────────────────────────────┤
│  arr = [1, 2, 1, 3, 2, 1, 4]                    │
│  idx:   0  1  2  3  4  5  6                     │
│                                                 │
│  Sort queries by right endpoint:                │
│    [0,2], [1,4], [3,6], [0,6]                   │
│                                                 │
│  Process with BIT, track lastSeen:              │
│    At r=2: seen {1:2, 2:1}                      │
│      [0,2] → distinct = 2                       │
│    At r=4: seen {1:2, 2:4, 3:3}                 │
│      [1,4] → distinct = 3                       │
│    At r=6: seen {1:5, 2:4, 3:3, 4:6}            │
│      [3,6] → distinct = 4                       │
│      [0,6] → distinct = 4                       │
└─────────────────────────────────────────────────┘
```

---

## Example 8: Compress for Segment Tree Range Operations

```go
package main

import (
	"fmt"
	"sort"
)

// Given intervals, find the maximum number of overlapping intervals at any point
// Uses coordinate compression + difference array on compressed coordinates

func maxOverlap(intervals [][2]int) int {
	// Collect all endpoints
	points := []int{}
	for _, iv := range intervals {
		points = append(points, iv[0], iv[1])
	}
	sort.Ints(points)
	points = uniqueInts(points)
	rank := map[int]int{}
	for i, v := range points { rank[v] = i }

	n := len(points)
	diff := make([]int, n+1)

	for _, iv := range intervals {
		l, r := rank[iv[0]], rank[iv[1]]
		diff[l]++
		diff[r]-- // half-open: [l, r)
	}

	maxOv, curr := 0, 0
	for i := 0; i < n; i++ {
		curr += diff[i]
		if curr > maxOv { maxOv = curr }
	}
	return maxOv
}

func uniqueInts(a []int) []int {
	if len(a) == 0 { return a }
	r := []int{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { r = append(r, a[i]) } }
	return r
}

func main() {
	intervals := [][2]int{
		{1, 5}, {2, 7}, {4, 6}, {8, 10}, {3, 9},
	}

	fmt.Printf("Intervals: %v\n", intervals)
	fmt.Printf("Max overlap: %d\n", maxOverlap(intervals))
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Max Overlapping Intervals + Compression        │
├─────────────────────────────────────────────────┤
│  Intervals: [1,5] [2,7] [4,6] [8,10] [3,9]      │
│                                                 │
│  Timeline:                                      │
│  1  2  3  4  5  6  7  8  9  10                  │
│  ────────── [1,5]                            │
│     ───────────── [2,7]                       │
│           ────── [4,6]                         │
│        ─────────────── [3,9]                  │
│                       ───── [8,10]             │
│                                                 │
│  Diff array on compressed coords:               │
│  At point 4: [1,5]✓ [2,7]✓ [4,6]✓ [3,9]✓ = 4  │
│  Max overlap = 4                                 │
└─────────────────────────────────────────────────┘
```

---

## Example 9: Sort + Binary Search Compression (In-Place)

```go
package main

import (
	"fmt"
	"sort"
)

// Alternative compression: sort array, use binary search instead of map

func compressBinarySearch(arr []int) (compressed []int, decompressMap []int) {
	sorted := make([]int, len(arr))
	copy(sorted, arr)
	sort.Ints(sorted)

	// Remove duplicates
	decompressMap = []int{sorted[0]}
	for i := 1; i < len(sorted); i++ {
		if sorted[i] != sorted[i-1] {
			decompressMap = append(decompressMap, sorted[i])
		}
	}

	compressed = make([]int, len(arr))
	for i, v := range arr {
		compressed[i] = sort.SearchInts(decompressMap, v)
	}
	return
}

func main() {
	arr := []int{1000000000, 1, 500000000, 2, 1}

	compressed, decompressMap := compressBinarySearch(arr)
	fmt.Printf("Original:     %v\n", arr)
	fmt.Printf("Compressed:   %v\n", compressed)
	fmt.Printf("Decompress:   %v\n", decompressMap)

	// Decompress back
	for i, c := range compressed {
		fmt.Printf("  compressed[%d]=%d → original=%d\n", i, c, decompressMap[c])
	}

	fmt.Println("\nBinary search compression avoids hash map overhead")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Binary Search Compression                     │
├─────────────────────────────────────────────────┤
│  arr = [10⁹, 1, 5×10⁸, 2, 1]                   │
│                                                 │
│  Sorted unique: [1, 2, 500000000, 1000000000]  │
│                                                 │
│  SearchInts(sorted, val) → index:              │
│    10⁹ → bisect → idx 3                        │
│    1   → bisect → idx 0                        │
│    5e8 → bisect → idx 2                        │
│    2   → bisect → idx 1                        │
│    1   → bisect → idx 0                        │
│                                                 │
│  Result: [3, 0, 2, 1, 0]                        │
│                                                 │
│  vs HashMap: avoids alloc, cache-friendly       │
│  Trade-off: O(log k) per lookup vs O(1) avg     │
└─────────────────────────────────────────────────┘
```

---

## Example 10: Coordinate Compression Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Coordinate Compression Patterns ===")
	fmt.Println()

	patterns := []struct{ pattern, when, how string }{
		{"Sort + rank map", "General", "Sort unique, build map[val]→rank"},
		{"Sort + binary search", "Performance", "Sort unique, use sort.SearchInts"},
		{"BIT + compression", "Inversions, count smaller", "Compress values, BIT on rank space"},
		{"Sweep line + compression", "Rectangle union area", "Compress x-coords, sweep y"},
		{"Offline queries", "Count distinct in range", "Sort queries by endpoint, BIT"},
		{"Range sum on sparse data", "Huge range, few points", "Map positions to [0,k)"},
		{"Segment tree + compression", "Range ops on large values", "Compress value domain"},
		{"Multi-dimensional", "2D problems", "Compress each dimension separately"},
		{"Prefix sum on compressed", "Range queries offline", "Diff array on compressed coords"},
		{"Event-based", "Interval scheduling", "Compress time points"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-30s\n    When: %s\n    How:  %s\n\n", p.pattern, p.when, p.how)
	}

	fmt.Println("Steps:")
	fmt.Println("  1. Collect all relevant values")
	fmt.Println("  2. Sort and deduplicate")
	fmt.Println("  3. Map original values to ranks [0, k)")
	fmt.Println("  4. Use ranks with BIT/segment tree/array")
	fmt.Println("  5. Decompress results if needed")
}
```

**Textual Figure:**
```
┌─────────────────────────────────────────────────┐
│  Coordinate Compression Workflow                │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌───────────────┐                               │
│  │ Collect values │  all relevant positions     │
│  └───────┬───────┘                               │
│          ↓                                      │
│  ┌───────────────┐                               │
│  │ Sort + dedup  │  O(k log k)                 │
│  └───────┬───────┘                               │
│          ↓                                      │
│  ┌───────────────┐                               │
│  │ Map val→rank  │  hash map or binary search  │
│  └───────┬───────┘                               │
│          ↓                                      │
│  ┌───────────────┐                               │
│  │ Use BIT/SegT  │  on ranks [0, k)            │
│  └───────────────┘                               │
└─────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Compress when values are large but count is small** — map $10^9$ range to $10^5$ ranks
2. **Two approaches**: hash map (simpler) or binary search (faster, no allocation)
3. **Essential for BIT/segment tree** — they need contiguous index range
4. **Common applications**: count inversions, count smaller, range sum on sparse data
5. **Sweep line problems** almost always need coordinate compression
6. **Offline query optimization**: sort queries, process in order, use BIT

> **Phase 22 Complete! Next up: Phase 23 — Design Data Structures →**
