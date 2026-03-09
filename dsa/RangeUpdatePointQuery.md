# Phase 22: Segment Tree & Fenwick Tree — Range Update Point Query (RUPQ)

## Overview

**RUPQ (Range Update, Point Query)**: Update all elements in a range `[l, r]` and query the value of a single element.

**Key Insight**: Use a **difference array** approach — store differences so that a prefix sum gives the actual value.

| Method | Range Update | Point Query | Space |
|--------|-------------|-------------|-------|
| Naive | O(n) | O(1) | O(n) |
| Difference Array | O(1) | O(n) | O(n) |
| BIT on diff array | O(log n) | O(log n) | O(n) |
| Lazy Segment Tree | O(log n) | O(log n) | O(4n) |

---

## Example 1: Difference Array (Basic, No DS)

```go
package main

import "fmt"

type DiffArray struct {
	diff []int
	n    int
}

func NewDiffArray(n int) *DiffArray {
	return &DiffArray{diff: make([]int, n+1), n: n}
}

func (d *DiffArray) RangeAdd(l, r, val int) {
	d.diff[l] += val
	if r+1 < d.n { d.diff[r+1] -= val }
}

func (d *DiffArray) Build() []int {
	result := make([]int, d.n)
	result[0] = d.diff[0]
	for i := 1; i < d.n; i++ {
		result[i] = result[i-1] + d.diff[i]
	}
	return result
}

func main() {
	n := 8
	da := NewDiffArray(n)

	da.RangeAdd(1, 4, 5) // add 5 to [1,4]
	da.RangeAdd(2, 6, 3) // add 3 to [2,6]
	da.RangeAdd(0, 7, 1) // add 1 to all

	result := da.Build()
	fmt.Println("After range updates:")
	for i, v := range result {
		fmt.Printf("  arr[%d] = %d\n", i, v)
	}

	fmt.Println("\nDifference array: O(1) update, O(n) to reconstruct")
	fmt.Println("Best when all updates happen before any query")
}
```

---

## Example 2: BIT-Based RUPQ (Online)

```go
package main

import "fmt"

type RUPQ struct {
	tree []int
	n    int
}

func NewRUPQ(n int) *RUPQ {
	return &RUPQ{tree: make([]int, n+2), n: n}
}

func (b *RUPQ) add(i, delta int) {
	for i++; i <= b.n+1; i += i & (-i) { b.tree[i] += delta }
}

func (b *RUPQ) RangeAdd(l, r, val int) {
	b.add(l, val)
	if r+1 <= b.n { b.add(r+1, -val) }
}

func (b *RUPQ) PointQuery(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s += b.tree[i] }
	return s
}

func main() {
	n := 8
	rupq := NewRUPQ(n)

	updates := [][3]int{
		{1, 4, 5},  // add 5 to [1,4]
		{2, 6, 3},  // add 3 to [2,6]
		{0, 7, 1},  // add 1 to all
	}

	for _, u := range updates {
		rupq.RangeAdd(u[0], u[1], u[2])
		fmt.Printf("After RangeAdd(%d, %d, %d):\n", u[0], u[1], u[2])
		for i := 0; i < n; i++ {
			fmt.Printf("  arr[%d] = %d\n", i, rupq.PointQuery(i))
		}
		fmt.Println()
	}
}
```

---

## Example 3: Segment Tree RUPQ with Lazy Propagation

```go
package main

import "fmt"

type LazyRUPQ struct {
	lazy []int
	n    int
}

func NewLazyRUPQ(n int) *LazyRUPQ {
	return &LazyRUPQ{lazy: make([]int, 4*n), n: n}
}

func (lr *LazyRUPQ) update(node, s, e, l, r, val int) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		lr.lazy[node] += val
		return
	}
	mid := (s + e) / 2
	lr.update(2*node, s, mid, l, r, val)
	lr.update(2*node+1, mid+1, e, l, r, val)
}

func (lr *LazyRUPQ) query(node, s, e, idx int) int {
	if s == e { return lr.lazy[node] }
	mid := (s + e) / 2
	if idx <= mid {
		return lr.lazy[node] + lr.query(2*node, s, mid, idx)
	}
	return lr.lazy[node] + lr.query(2*node+1, mid+1, e, idx)
}

func (lr *LazyRUPQ) RangeAdd(l, r, val int) { lr.update(1, 0, lr.n-1, l, r, val) }
func (lr *LazyRUPQ) PointQuery(idx int) int  { return lr.query(1, 0, lr.n-1, idx) }

func main() {
	n := 6
	lr := NewLazyRUPQ(n)

	lr.RangeAdd(0, 3, 10)
	lr.RangeAdd(2, 5, 5)

	fmt.Println("After RangeAdd(0,3,10) and RangeAdd(2,5,5):")
	for i := 0; i < n; i++ {
		fmt.Printf("  arr[%d] = %d\n", i, lr.PointQuery(i))
	}
}
```

---

## Example 4: Range Set + Point Query

```go
package main

import "fmt"

// Range paint: set all elements in [l,r] to val
// Point query: what is the current value at index i?
// Uses lazy segment tree with "last write wins"

type PaintRUPQ struct {
	lazy  []int
	hasOp []bool
	n     int
}

func NewPaintRUPQ(n int) *PaintRUPQ {
	return &PaintRUPQ{
		lazy:  make([]int, 4*n),
		hasOp: make([]bool, 4*n),
		n:     n,
	}
}

func (p *PaintRUPQ) push(node int) {
	if p.hasOp[node] {
		for _, c := range []int{2 * node, 2*node + 1} {
			p.lazy[c] = p.lazy[node]
			p.hasOp[c] = true
		}
		p.hasOp[node] = false
	}
}

func (p *PaintRUPQ) update(node, s, e, l, r, val int) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		p.lazy[node] = val
		p.hasOp[node] = true
		return
	}
	p.push(node)
	mid := (s + e) / 2
	p.update(2*node, s, mid, l, r, val)
	p.update(2*node+1, mid+1, e, l, r, val)
}

func (p *PaintRUPQ) query(node, s, e, idx int) int {
	if s == e { return p.lazy[node] }
	p.push(node)
	mid := (s + e) / 2
	if idx <= mid { return p.query(2*node, s, mid, idx) }
	return p.query(2*node+1, mid+1, e, idx)
}

func main() {
	n := 8
	p := NewPaintRUPQ(n)

	p.update(1, 0, n-1, 0, 7, 0) // initialize all to 0
	p.update(1, 0, n-1, 1, 4, 3) // paint [1,4] with color 3
	p.update(1, 0, n-1, 3, 6, 5) // paint [3,6] with color 5

	fmt.Println("After painting [1,4]=3 then [3,6]=5:")
	for i := 0; i < n; i++ {
		fmt.Printf("  arr[%d] = %d\n", i, p.query(1, 0, n-1, i))
	}
}
```

---

## Example 5: Corporate Flight Bookings (LeetCode 1109)

```go
package main

import "fmt"

func corpFlightBookings(bookings [][]int, n int) []int {
	// Difference array approach
	diff := make([]int, n+1)
	for _, b := range bookings {
		first, last, seats := b[0]-1, b[1]-1, b[2]
		diff[first] += seats
		if last+1 < n { diff[last+1] -= seats }
	}

	result := make([]int, n)
	result[0] = diff[0]
	for i := 1; i < n; i++ {
		result[i] = result[i-1] + diff[i]
	}
	return result
}

func main() {
	bookings := [][]int{{1, 2, 10}, {2, 3, 20}, {2, 5, 25}}
	n := 5
	result := corpFlightBookings(bookings, n)
	fmt.Printf("Bookings: %v, n=%d\n", bookings, n)
	fmt.Printf("Result: %v\n", result)
	// Expected: [10, 55, 45, 25, 25]
}
```

---

## Example 6: Car Pooling (LeetCode 1094)

```go
package main

import "fmt"

func carPooling(trips [][]int, capacity int) bool {
	// Difference array on timeline
	maxLoc := 0
	for _, t := range trips {
		if t[2] > maxLoc { maxLoc = t[2] }
	}

	diff := make([]int, maxLoc+1)
	for _, t := range trips {
		passengers, from, to := t[0], t[1], t[2]
		diff[from] += passengers
		if to <= maxLoc { diff[to] -= passengers }
	}

	current := 0
	for _, d := range diff {
		current += d
		if current > capacity { return false }
	}
	return true
}

func main() {
	tests := []struct {
		trips    [][]int
		capacity int
		expected bool
	}{
		{[][]int{{2, 1, 5}, {3, 3, 7}}, 4, false},
		{[][]int{{2, 1, 5}, {3, 3, 7}}, 5, true},
		{[][]int{{2, 1, 5}, {3, 5, 7}}, 3, true},
	}

	for _, t := range tests {
		result := carPooling(t.trips, t.capacity)
		fmt.Printf("  trips=%v, cap=%d → %v (expected %v)\n",
			t.trips, t.capacity, result, t.expected)
	}
}
```

---

## Example 7: Range Increment + Max Point Query

```go
package main

import (
	"fmt"
	"math"
)

// Range add + query single element's value
// Also track the max across all elements

type RUPQMax struct {
	bit  []int
	n    int
	vals []int // actual values for max tracking
}

func NewRUPQMax(arr []int) *RUPQMax {
	n := len(arr)
	r := &RUPQMax{bit: make([]int, n+2), n: n, vals: make([]int, n)}
	copy(r.vals, arr)
	return r
}

func (r *RUPQMax) addBIT(i, delta int) {
	for i++; i <= r.n+1; i += i & (-i) { r.bit[i] += delta }
}

func (r *RUPQMax) queryBIT(i int) int {
	s := 0; for i++; i > 0; i -= i & (-i) { s += r.bit[i] }; return s
}

func (r *RUPQMax) RangeAdd(l, ri, val int) {
	r.addBIT(l, val)
	if ri+1 <= r.n { r.addBIT(ri+1, -val) }
}

func (r *RUPQMax) PointQuery(i int) int {
	return r.vals[i] + r.queryBIT(i)
}

func (r *RUPQMax) FindMax() (int, int) {
	maxVal, maxIdx := math.MinInt64, 0
	for i := 0; i < r.n; i++ {
		v := r.PointQuery(i)
		if v > maxVal { maxVal = v; maxIdx = i }
	}
	return maxIdx, maxVal
}

func main() {
	arr := []int{1, 5, 3, 8, 2, 7}
	rupq := NewRUPQMax(arr)

	fmt.Printf("Initial: %v\n", arr)
	rupq.RangeAdd(1, 4, 10) // add 10 to [1,4]

	fmt.Println("\nAfter adding 10 to [1,4]:")
	for i := 0; i < len(arr); i++ {
		fmt.Printf("  arr[%d] = %d\n", i, rupq.PointQuery(i))
	}
	idx, val := rupq.FindMax()
	fmt.Printf("Max: arr[%d] = %d\n", idx, val)
}
```

---

## Example 8: 2D Range Update Point Query

```go
package main

import "fmt"

type RUPQ2D struct {
	bit  [][]int
	n, m int
}

func NewRUPQ2D(n, m int) *RUPQ2D {
	bit := make([][]int, n+2)
	for i := range bit { bit[i] = make([]int, m+2) }
	return &RUPQ2D{bit: bit, n: n, m: m}
}

func (r *RUPQ2D) add(row, col, val int) {
	for i := row + 1; i <= r.n+1; i += i & (-i) {
		for j := col + 1; j <= r.m+1; j += j & (-j) {
			r.bit[i][j] += val
		}
	}
}

func (r *RUPQ2D) RangeAdd(r1, c1, r2, c2, val int) {
	r.add(r1, c1, val)
	r.add(r1, c2+1, -val)
	r.add(r2+1, c1, -val)
	r.add(r2+1, c2+1, val)
}

func (r *RUPQ2D) PointQuery(row, col int) int {
	s := 0
	for i := row + 1; i > 0; i -= i & (-i) {
		for j := col + 1; j > 0; j -= j & (-j) {
			s += r.bit[i][j]
		}
	}
	return s
}

func main() {
	n, m := 5, 5
	rupq := NewRUPQ2D(n, m)

	rupq.RangeAdd(1, 1, 3, 3, 5) // add 5 to submatrix [1,1]-[3,3]
	rupq.RangeAdd(2, 2, 4, 4, 3) // add 3 to submatrix [2,2]-[4,4]

	fmt.Println("After range updates:")
	for i := 0; i < n; i++ {
		for j := 0; j < m; j++ {
			fmt.Printf("%3d", rupq.PointQuery(i, j))
		}
		fmt.Println()
	}
}
```

---

## Example 9: Range XOR + Point Query

```go
package main

import "fmt"

// XOR is its own inverse, so the same BIT trick works

type XORRUPQ struct {
	tree []int
	n    int
}

func NewXORRUPQ(n int) *XORRUPQ {
	return &XORRUPQ{tree: make([]int, n+2), n: n}
}

func (x *XORRUPQ) xorBIT(i, val int) {
	for i++; i <= x.n+1; i += i & (-i) { x.tree[i] ^= val }
}

func (x *XORRUPQ) RangeXOR(l, r, val int) {
	x.xorBIT(l, val)
	if r+1 <= x.n { x.xorBIT(r+1, val) }
}

func (x *XORRUPQ) PointQuery(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s ^= x.tree[i] }
	return s
}

func main() {
	n := 8
	xr := NewXORRUPQ(n)

	xr.RangeXOR(1, 5, 7) // XOR [1,5] with 7
	xr.RangeXOR(3, 7, 3) // XOR [3,7] with 3

	fmt.Println("After XOR updates:")
	for i := 0; i < n; i++ {
		fmt.Printf("  arr[%d] = %d (binary: %08b)\n", i, xr.PointQuery(i), xr.PointQuery(i))
	}
}
```

---

## Example 10: RUPQ Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Range Update Point Query (RUPQ) Patterns ===")
	fmt.Println()

	patterns := []struct{ method, updateTime, queryTime, note string }{
		{"Difference Array", "O(1)", "O(n) rebuild", "Offline: all updates before queries"},
		{"BIT on Diff Array", "O(log n)", "O(log n)", "Online: interleaved updates & queries"},
		{"Seg Tree + Lazy", "O(log n)", "O(log n)", "More flexible but heavier"},
		{"Range Set (Paint)", "O(log n)", "O(log n)", "Last write wins, lazy seg tree"},
		{"2D BIT Diff", "O(log n log m)", "O(log n log m)", "2D rectangle update"},
		{"Range XOR", "O(log n)", "O(log n)", "XOR is self-inverse"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-20s | Update: %-12s | Query: %-12s | %s\n",
			p.method, p.updateTime, p.queryTime, p.note)
	}

	fmt.Println()
	fmt.Println("Core Insight: RUPQ = PURQ on the DIFFERENCE array")
	fmt.Println("  range_add(l, r, v) → point_update(l, v), point_update(r+1, -v)")
	fmt.Println("  point_query(i)     → prefix_sum(0..i)")
	fmt.Println()
	fmt.Println("LeetCode Applications:")
	fmt.Println("  1109. Corporate Flight Bookings (diff array)")
	fmt.Println("  1094. Car Pooling (diff array)")
	fmt.Println("  370. Range Addition (diff array, premium)")
}
```

---

## Key Takeaways

1. **Difference array**: the fundamental idea — `diff[l] += v, diff[r+1] -= v`
2. **BIT on diff array**: makes both update and query O(log n) online
3. **Seg tree with lazy**: more flexible, supports range set + point query
4. **2D extension**: difference BIT uses 4-corner inclusion-exclusion
5. **XOR**: self-inverse, so same approach works naturally
6. **Offline vs Online**: if all updates before queries, plain diff array suffices

> **Next up:** Coordinate Compression →
