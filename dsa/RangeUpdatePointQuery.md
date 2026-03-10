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

**Textual Figure:**

```
Difference Array (Offline RUPQ)
n = 8, all zeros initially

RangeAdd(1, 4, +5):  diff[1]+=5, diff[5]-=5
  diff: [0, 5, 0, 0, 0,-5, 0, 0, 0]

RangeAdd(2, 6, +3):  diff[2]+=3, diff[7]-=3
  diff: [0, 5, 3, 0, 0,-5, 0,-3, 0]

RangeAdd(0, 7, +1):  diff[0]+=1
  diff: [1, 5, 3, 0, 0,-5, 0,-3, 0]

Build (prefix sum of diff):
  idx:   0  1  2  3  4  5  6  7
  arr:  [1, 6, 9, 9, 9, 4, 4, 1]
         │  │  │  │  │  │  │  │
         1 +5 +3 +0 +0 -5 +0 -3

  Verify idx 3: adds that cover 3 = +5(1..4) +3(2..6) +1(all) = 9 ✓
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

**Textual Figure:**

```
BIT-Based RUPQ (Online Difference BIT)
n=8, BIT stores difference array

After RangeAdd(1, 4, +5):
  BIT.add(1, +5), BIT.add(5, -5)
  PointQuery(i) = BIT.prefixSum(i)
  arr: [0, 5, 5, 5, 5, 0, 0, 0]

After RangeAdd(2, 6, +3):
  BIT.add(2, +3), BIT.add(7, -3)
  arr: [0, 5, 8, 8, 8, 3, 3, 0]

After RangeAdd(0, 7, +1):
  BIT.add(0, +1)
  arr: [1, 6, 9, 9, 9, 4, 4, 1]

BIT prefix sum at index i gives arr[i]:
  PointQuery(3):
    i=4 → bit[4]  (covers 1..4)
    sum = 9  ✓

  Key: RUPQ = PURQ on the implicit difference array
  RangeAdd(l,r,v) → add(l,+v), add(r+1,-v)
  PointQuery(i)   → prefixSum(0..i)
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

**Textual Figure:**

```
Segment Tree RUPQ (Lazy only, no merge needed)
n=6, all zeros initially

RangeAdd(0, 3, +10):
  lazy accumulated on covering nodes

       ┌─────────────┐
       │ [0,5] lazy=0  │
       └────┬────┬────┘
  ┌────┴─────┐  ┌──┴──────┐
  │[0,2] l=10│  │[3,5] l=0  │
  └──────────┘  └──┬───────┘
               ┌──┴─────┐ ┌──────┐
               │[3,4] l=0│ │[5] l=0│
               └┬────────┘ └──────┘
         [3]=10, [4,5]=0

RangeAdd(2, 5, +5):
  After both ops:
  PointQuery(i) = sum of lazy along root-to-leaf path

  arr: [10, 10, 15, 15, 5, 5]
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

**Textual Figure:**

```
Range Set (Paint) + Point Query
n=8, initialize all to 0

Paint [1,4] = 3:
  ┌─────────────────────────────┐
  │ 0 │ 3 │ 3 │ 3 │ 3 │ 0 │ 0 │ 0 │
  └───┴───┴───┴───┴───┴───┴───┴───┘

Paint [3,6] = 5 (overwrites [3,4]):
  ┌─────────────────────────────┐
  │ 0 │ 3 │ 3 │ 5 │ 5 │ 5 │ 5 │ 0 │
  └───┴───┴───┴───┴───┴───┴───┴───┘

Lazy tree: "last write wins" push-down
  When querying i=4: traverse root → leaf
  At each node, push lazy to children before descending
  → leaf gets the most recent paint = 5  ✓

  Key: hasOp[] flag determines if lazy should be pushed
  paint ops compose as: new paint replaces old
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

**Textual Figure:**

```
Corporate Flight Bookings (Difference Array)
bookings = [[1,2,10], [2,3,20], [2,5,25]], n=5

Convert to 0-indexed and apply to diff[]:

  [1,2,10]: diff[0]+=10, diff[2]-=10
    diff: [10, 0,-10, 0, 0, 0]

  [2,3,20]: diff[1]+=20, diff[3]-=20
    diff: [10, 20,-10,-20, 0, 0]

  [2,5,25]: diff[1]+=25
    diff: [10, 45,-10,-20, 0, 0]

Prefix sum to get result:
  idx:    0    1    2    3    4
  result: 10   55   45   25   25
           │    │    │    │    │
          10  +45  -10  -20   +0

  Flight 1: 10 seats
  Flight 2: 10+20+25 = 55 seats
  Flight 3: 55-10 = 45 seats
  Flight 4: 45-20 = 25 seats
  Flight 5: 25 seats
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

**Textual Figure:**

```
Car Pooling (Difference Array on Timeline)
trips = [[2,1,5], [3,3,7]], capacity = 4

Timeline diff array (passengers on/off at locations):
  Trip [2,1,5]: diff[1]+=2, diff[5]-=2
  Trip [3,3,7]: diff[3]+=3, diff[7]-=3

  loc:  0  1  2  3  4  5  6  7
  diff: 0  2  0  3  0 -2  0 -3

  Prefix sum (current passengers):
  loc:  0  1  2  3  4  5  6  7
  pass: 0  2  2  5  5  3  3  0
                  ^
                  5 > 4 = capacity exceeded!

  Result: false ✓

  With capacity = 5: max passengers = 5 ≤ 5 → true
  With trips not overlapping: passengers never exceed cap
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

**Textual Figure:**

```
Range Add + Max Point Query (BIT on diff + linear scan)
arr = [1, 5, 3, 8, 2, 7]

After RangeAdd(1, 4, +10):
  BIT diff: add(1,+10), add(5,-10)

  PointQuery(i) = arr[i] + BIT.prefixSum(i)
  idx:  0    1    2    3    4    5
  arr: [1,  15,  13,  18,  12,   7]
        │    │    │    │    │    │
       +0  +10  +10  +10  +10   +0

FindMax: scan all to find maximum
  arr[3] = 18  ← maximum

  Note: no O(log n) way to find max with just BIT
  For O(log n) max queries, need segment tree with lazy
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

**Textual Figure:**

```
2D Range Update Point Query (2D Difference BIT)
5×5 grid, initially all zeros

RangeAdd([1,1]-[3,3], +5):
  2D inclusion-exclusion on BIT:
  add(1,1,+5), add(1,4,-5), add(4,1,-5), add(4,4,+5)

  Grid after:
    0  0  0  0  0
    0  5  5  5  0
    0  5  5  5  0
    0  5  5  5  0
    0  0  0  0  0

RangeAdd([2,2]-[4,4], +3):
  add(2,2,+3), add(2,5,-3), add(5,2,-3), add(5,5,+3)

  Grid after:
    0  0  0  0  0
    0  5  5  5  0
    0  5  8  8  3
    0  5  8  8  3
    0  0  3  3  3

  PointQuery(2,2) = BIT.prefixSum2D(2,2) = 8  ✓
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

**Textual Figure:**

```
Range XOR + Point Query (XOR is self-inverse)
n=8, all zeros initially

RangeXOR(1, 5, 7):  BIT.xor(1, 7), BIT.xor(6, 7)
  idx:  0  1  2  3  4  5  6  7
  arr: [0, 7, 7, 7, 7, 7, 0, 0]
              7 = 0b00000111

RangeXOR(3, 7, 3):  BIT.xor(3, 3), BIT.xor(8, 3)
  idx:  0  1  2  3  4  5  6  7
  arr: [0, 7, 7, 4, 4, 4, 3, 3]

  idx 3: 7 XOR 3 = 0b111 XOR 0b011 = 0b100 = 4
  idx 6: 0 XOR 3 = 3

  Key: XOR is its own inverse (a XOR a = 0)
  So BIT trick works naturally:
    RangeXOR(l,r,v) → xor(l,v), xor(r+1,v)
    PointQuery(i) → prefix XOR from 0 to i
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

**Textual Figure:**

```
RUPQ Patterns — Core Duality:

  PURQ (Point Update, Range Query)
    update(i, v)     → change one element
    query(l, r)      → aggregate over range

  RUPQ (Range Update, Point Query) = PURQ on DIFF array
    rangeAdd(l, r, v) → diff[l]+=v, diff[r+1]-=v
    pointQuery(i)     → prefixSum(0..i)

  ┌────────────────┬─────────┬─────────┬───────────────┐
  │ Method         │ Update  │ Query   │ When to use?   │
  ├────────────────┼─────────┼─────────┼───────────────┤
  │ Diff array     │ O(1)    │ O(n)    │ Offline batch  │
  │ BIT + diff     │ O(lg n) │ O(lg n) │ Online         │
  │ Lazy seg tree  │ O(lg n) │ O(lg n) │ Flexible       │
  │ Range set      │ O(lg n) │ O(lg n) │ Paint/assign   │
  │ 2D BIT diff    │ O(lg²)  │ O(lg²)  │ 2D rectangle   │
  │ XOR BIT        │ O(lg n) │ O(lg n) │ Toggle bits    │
  └────────────────┴─────────┴─────────┴───────────────┘
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
