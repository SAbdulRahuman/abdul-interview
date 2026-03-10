# Phase 22: Segment Tree & Fenwick Tree — Fenwick Tree (Binary Indexed Tree)

## Overview

A **Fenwick Tree (BIT)** supports **prefix sum queries** and **point updates** in O(log n) with minimal code and memory.

**Key Idea**: Each index `i` stores the sum of elements in a range determined by the lowest set bit of `i`.

| Operation | BIT | Segment Tree |
|-----------|-----|-------------|
| Point update | O(log n) | O(log n) |
| Prefix query | O(log n) | O(log n) |
| Range query | O(log n) | O(log n) |
| Space | O(n) | O(4n) |
| Code complexity | Very simple | Moderate |

**lowbit(x) = x & (-x)** — key operation for traversal.

---

## Example 1: Basic Fenwick Tree (Point Update + Prefix Sum)

```go
package main

import "fmt"

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT {
	return &BIT{tree: make([]int, n+1), n: n}
}

func (b *BIT) Update(i, delta int) {
	for i++; i <= b.n; i += i & (-i) {
		b.tree[i] += delta
	}
}

func (b *BIT) PrefixSum(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) {
		s += b.tree[i]
	}
	return s
}

func (b *BIT) RangeSum(l, r int) int {
	if l == 0 { return b.PrefixSum(r) }
	return b.PrefixSum(r) - b.PrefixSum(l-1)
}

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	bit := NewBIT(len(arr))
	for i, v := range arr {
		bit.Update(i, v)
	}

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Prefix sum [0..2] = %d\n", bit.PrefixSum(2))
	fmt.Printf("Range sum [1..4] = %d\n", bit.RangeSum(1, 4))

	bit.Update(2, 5) // arr[2] += 5
	fmt.Printf("\nAfter arr[2]+=5:\n")
	fmt.Printf("Prefix sum [0..2] = %d\n", bit.PrefixSum(2))
}
```

**Textual Figure:**

```
Array: [1, 3, 5, 7, 9, 11]   (0-indexed: indices 0..5)
BIT uses 1-indexed tree[1..6]

lowbit(x) = x & (-x)  — determines range each index covers:

  Index i  │ Binary │ lowbit │ Range covered     │ tree[i]
  ─────────┼────────┼────────┼──────────────────┼────────
  1        │ 001    │ 1      │ arr[0]            │ 1
  2        │ 010    │ 2      │ arr[0..1]         │ 4
  3        │ 011    │ 1      │ arr[2]            │ 5
  4        │ 100    │ 4      │ arr[0..3]         │ 16
  5        │ 101    │ 1      │ arr[4]            │ 9
  6        │ 110    │ 2      │ arr[4..5]         │ 20

BIT structure (responsible ranges):
  tree[4]=16 covers [0..3]  ──┬── tree[6]=20 covers [4..5]
              │                    │
  tree[2]=4 [0..1]  tree[3]=5 [2]   tree[5]=9 [4]
      │
  tree[1]=1 [0]

PrefixSum(2): i=3 → tree[3]=5, i=3-1=2 → tree[2]=4, i=2-2=0 stop
              sum = 5 + 4 = 9  (1+3+5=9) ✓

RangeSum(1,4): PrefixSum(4) - PrefixSum(0)
               = (tree[5]+tree[4]) - (tree[1])
               = (9+16) - 1 = 24  (3+5+7+9=24) ✓

Update(2, +5): i=3 → tree[3]+=5, i=3+1=4 → tree[4]+=5, i=4+4=8 > 6 stop
  tree[3]: 5→10, tree[4]: 16→21
```

---

## Example 2: Build BIT in O(n) Time

```go
package main

import "fmt"

type BIT struct {
	tree []int
	n    int
}

func BuildBIT(arr []int) *BIT {
	n := len(arr)
	b := &BIT{tree: make([]int, n+1), n: n}
	copy(b.tree[1:], arr)

	// O(n) construction
	for i := 1; i <= n; i++ {
		parent := i + (i & (-i))
		if parent <= n {
			b.tree[parent] += b.tree[i]
		}
	}
	return b
}

func (b *BIT) PrefixSum(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) {
		s += b.tree[i]
	}
	return s
}

func (b *BIT) Update(i, delta int) {
	for i++; i <= b.n; i += i & (-i) {
		b.tree[i] += delta
	}
}

func main() {
	arr := []int{3, 1, 4, 1, 5, 9, 2, 6}
	bit := BuildBIT(arr)

	fmt.Printf("Array: %v\n", arr)
	for i := 0; i < len(arr); i++ {
		fmt.Printf("  Prefix sum [0..%d] = %d\n", i, bit.PrefixSum(i))
	}

	fmt.Println("\nO(n) construction vs O(n log n) with individual updates")
}
```

**Textual Figure:**

```
Array: [3, 1, 4, 1, 5, 9, 2, 6]   (n=8)

O(n) Build: copy arr to tree[1..8], then propagate upward:

Step 1: tree[1..8] = [3, 1, 4, 1, 5, 9, 2, 6]

Step 2: for i=1..8, parent = i + lowbit(i)
  i=1: parent=2 → tree[2] += tree[1]  → tree[2] = 1+3 = 4
  i=2: parent=4 → tree[4] += tree[2]  → tree[4] = 1+4 = 5
  i=3: parent=4 → tree[4] += tree[3]  → tree[4] = 5+4 = 9
  i=4: parent=8 → tree[8] += tree[4]  → tree[8] = 6+9 = 15
  i=5: parent=6 → tree[6] += tree[5]  → tree[6] = 9+5 = 14
  i=6: parent=8 → tree[8] += tree[6]  → tree[8] = 15+14= 29
  i=7: parent=8 → tree[8] += tree[7]  → tree[8] = 29+2 = 31
  i=8: parent=16> 8, skip

Final tree[1..8] = [3, 4, 4, 9, 5, 14, 2, 31]

Verification - prefix sums:
  PrefixSum(0) = tree[1]                    = 3  ✓
  PrefixSum(1) = tree[2]                    = 4  ✓ (3+1)
  PrefixSum(3) = tree[4]                    = 9  ✓ (3+1+4+1)
  PrefixSum(7) = tree[8]                    = 31 ✓ (3+1+4+1+5+9+2+6)
```

---

## Example 3: Count Inversions with BIT

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

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+2), n: n} }

func (b *BIT) Update(i int) {
	for i++; i <= b.n+1; i += i & (-i) { b.tree[i]++ }
}

func (b *BIT) PrefixSum(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s += b.tree[i] }
	return s
}

func countInversions(arr []int) int {
	// Coordinate compress
	sorted := make([]int, len(arr))
	copy(sorted, arr)
	sort.Ints(sorted)
	sorted = unique(sorted)
	rank := map[int]int{}
	for i, v := range sorted { rank[v] = i }

	bit := NewBIT(len(sorted))
	inversions := 0

	for i := len(arr) - 1; i >= 0; i-- {
		r := rank[arr[i]]
		inversions += bit.PrefixSum(r - 1) // elements smaller than arr[i] to its right
		bit.Update(r)
	}
	return inversions
}

func unique(a []int) []int {
	if len(a) == 0 { return a }
	result := []int{a[0]}
	for i := 1; i < len(a); i++ {
		if a[i] != a[i-1] { result = append(result, a[i]) }
	}
	return result
}

func main() {
	tests := [][]int{
		{2, 4, 1, 3, 5},
		{5, 4, 3, 2, 1},
		{1, 2, 3, 4, 5},
	}

	for _, arr := range tests {
		fmt.Printf("  %v → inversions = %d\n", arr, countInversions(arr))
	}
}
```

**Textual Figure:**

```
Count inversions: arr = [2, 4, 1, 3, 5]
Coordinate compressed ranks: 2→0, 4→2, 1→0_wait..
Sorted unique: [1,2,3,4,5] → rank: 1→0, 2→1, 3→2, 4→3, 5→4

Process RIGHT to LEFT, count elements smaller already inserted:

  i=4: arr[4]=5, rank=4.  PrefixSum(3)=0.  inversions+=0.  Insert rank 4.
       BIT: [0,0,0,0,1]

  i=3: arr[3]=3, rank=2.  PrefixSum(1)=0.  inversions+=0.  Insert rank 2.
       BIT: [0,0,1,0,1]

  i=2: arr[2]=1, rank=0.  PrefixSum(-1)=0. inversions+=0.  Insert rank 0.
       BIT: [1,0,1,0,1]

  i=1: arr[1]=4, rank=3.  PrefixSum(2)=2.  inversions+=2.  Insert rank 3.
       BIT: [1,0,1,1,1]    (elements 1,3 are to right of 4 and smaller)

  i=0: arr[0]=2, rank=1.  PrefixSum(0)=1.  inversions+=1.  Insert rank 1.
       BIT: [1,1,1,1,1]    (element 1 is to right of 2 and smaller)

  Total inversions = 0+0+0+2+1 = 3
  Inversion pairs: (2,1), (4,1), (4,3)
```

---

## Example 4: Range Sum Mutable (LeetCode 307)

```go
package main

import "fmt"

type NumArray struct {
	bit  []int
	nums []int
	n    int
}

func Constructor(nums []int) NumArray {
	n := len(nums)
	na := NumArray{bit: make([]int, n+1), nums: make([]int, n), n: n}
	copy(na.nums, nums)

	// Build O(n)
	copy(na.bit[1:], nums)
	for i := 1; i <= n; i++ {
		p := i + (i & (-i))
		if p <= n { na.bit[p] += na.bit[i] }
	}
	return na
}

func (na *NumArray) Update(index int, val int) {
	delta := val - na.nums[index]
	na.nums[index] = val
	for i := index + 1; i <= na.n; i += i & (-i) {
		na.bit[i] += delta
	}
}

func (na *NumArray) prefixSum(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s += na.bit[i] }
	return s
}

func (na *NumArray) SumRange(left, right int) int {
	if left == 0 { return na.prefixSum(right) }
	return na.prefixSum(right) - na.prefixSum(left-1)
}

func main() {
	na := Constructor([]int{1, 3, 5})
	fmt.Printf("SumRange(0,2) = %d\n", na.SumRange(0, 2))

	na.Update(1, 2) // nums = [1, 2, 5]
	fmt.Printf("After Update(1,2): SumRange(0,2) = %d\n", na.SumRange(0, 2))
	fmt.Printf("SumRange(1,2) = %d\n", na.SumRange(1, 2))
}
```

**Textual Figure:**

```
LeetCode 307: Range Sum Query - Mutable
nums = [1, 3, 5]   (n=3)

BIT after build (O(n)):
  tree[1]=1, tree[2]=4(=1+3), tree[3]=5

  SumRange(0,2) = PrefixSum(2)
    i=3: tree[3]=5, i=3-1=2
    i=2: tree[2]=4, i=2-2=0 stop
    sum = 5+4 = 9  ✓  (1+3+5)

  Update(1, 2):  delta = 2 - nums[1] = 2-3 = -1
    nums[1] = 2
    i=2: tree[2] += -1 → tree[2]=3
    i=4: > n=3, stop

  After update: nums = [1, 2, 5]
  tree[1]=1, tree[2]=3(=1+2), tree[3]=5

  SumRange(0,2) = PrefixSum(2) = 5+3 = 8  (1+2+5) ✓
  SumRange(1,2) = PrefixSum(2) - PrefixSum(0) = 8-1 = 7  (2+5) ✓
```

---

## Example 5: BIT for Count Smaller After Self (LeetCode 315)

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
	res := []int{a[0]}
	for i := 1; i < len(a); i++ { if a[i] != a[i-1] { res = append(res, a[i]) } }
	return res
}

func main() {
	tests := [][]int{
		{5, 2, 6, 1},
		{-1},
		{-1, -1},
	}

	for _, nums := range tests {
		fmt.Printf("  %v → %v\n", nums, countSmaller(nums))
	}
}
```

**Textual Figure:**

```
LeetCode 315: Count Smaller After Self
nums = [5, 2, 6, 1]
Ranks: 1→0, 2→1, 5→2, 6→3

Process RIGHT to LEFT, query how many already inserted < current:

  i=3: nums[3]=1, rank=0. Query(rank-1)=Query(-1)=0. Insert rank 0.
       result[3] = 0
       BIT: [1, 0, 0, 0]

  i=2: nums[2]=6, rank=3. Query(2)=1. Insert rank 3.
       result[2] = 1    (element 1 is smaller and to the right)
       BIT: [1, 0, 0, 1]

  i=1: nums[1]=2, rank=1. Query(0)=1. Insert rank 1.
       result[1] = 1    (element 1 is smaller and to the right)
       BIT: [1, 1, 0, 1]

  i=0: nums[0]=5, rank=2. Query(1)=2. Insert rank 2.
       result[0] = 2    (elements 2,1 are smaller and to the right)
       BIT: [1, 1, 1, 1]

  Final result: [2, 1, 1, 0]
```

---

## Example 6: 2D Fenwick Tree

```go
package main

import "fmt"

type BIT2D struct {
	tree   [][]int
	n, m   int
}

func NewBIT2D(n, m int) *BIT2D {
	tree := make([][]int, n+1)
	for i := range tree { tree[i] = make([]int, m+1) }
	return &BIT2D{tree: tree, n: n, m: m}
}

func (b *BIT2D) Update(r, c, delta int) {
	for i := r + 1; i <= b.n; i += i & (-i) {
		for j := c + 1; j <= b.m; j += j & (-j) {
			b.tree[i][j] += delta
		}
	}
}

func (b *BIT2D) PrefixSum(r, c int) int {
	s := 0
	for i := r + 1; i > 0; i -= i & (-i) {
		for j := c + 1; j > 0; j -= j & (-j) {
			s += b.tree[i][j]
		}
	}
	return s
}

func (b *BIT2D) RangeSum(r1, c1, r2, c2 int) int {
	s := b.PrefixSum(r2, c2)
	if r1 > 0 { s -= b.PrefixSum(r1-1, c2) }
	if c1 > 0 { s -= b.PrefixSum(r2, c1-1) }
	if r1 > 0 && c1 > 0 { s += b.PrefixSum(r1-1, c1-1) }
	return s
}

func main() {
	matrix := [][]int{
		{3, 0, 1, 4, 2},
		{5, 6, 3, 2, 1},
		{1, 2, 0, 1, 5},
		{4, 1, 0, 1, 7},
		{1, 0, 3, 0, 5},
	}

	n, m := len(matrix), len(matrix[0])
	bit := NewBIT2D(n, m)
	for i := 0; i < n; i++ {
		for j := 0; j < m; j++ {
			bit.Update(i, j, matrix[i][j])
		}
	}

	fmt.Printf("Sum [2,1] to [4,3] = %d\n", bit.RangeSum(2, 1, 4, 3))
	fmt.Printf("Sum [0,0] to [2,2] = %d\n", bit.RangeSum(0, 0, 2, 2))
}
```

**Textual Figure:**

```
2D Fenwick Tree — 5×5 matrix:

  Matrix:   3  0  1  4  2
            5  6  3  2  1
            1  2  0  1  5
            4  1  0  1  7
            1  0  3  0  5

2D BIT: tree[i][j] stores partial sums based on
        lowbit(i) × lowbit(j) rectangle.

RangeSum(r1,c1,r2,c2) uses inclusion-exclusion:
  Sum = P(r2,c2) - P(r1-1,c2) - P(r2,c1-1) + P(r1-1,c1-1)

  ┌─────────────────────┐
  │ P(r1-1,c1-1) │ +     │
  │      +       │       │  ← P(r1-1,c2)
  ├──────────────┼──────┤
  │              │ Query │
  │  P(r2,c1-1)  │ Area  │  ← P(r2,c2)
  └──────────────┴──────┘

Query Sum [2,1] to [4,3]:
  Submatrix:  2  0  1
              1  0  1
              0  3  0   → sum = 8

Query Sum [0,0] to [2,2]:
  Submatrix:  3  0  1
              5  6  3
              1  2  0   → sum = 21
```

---

## Example 7: BIT for Range Update + Point Query

```go
package main

import "fmt"

// Use difference array concept with BIT
// range_add(l, r, val) → update(l, val), update(r+1, -val)
// point_query(i) → prefix_sum(i)

type RUPQ struct {
	tree []int
	n    int
}

func NewRUPQ(n int) *RUPQ {
	return &RUPQ{tree: make([]int, n+2), n: n}
}

func (b *RUPQ) update(i, delta int) {
	for i++; i <= b.n+1; i += i & (-i) { b.tree[i] += delta }
}

func (b *RUPQ) RangeAdd(l, r, val int) {
	b.update(l, val)
	if r+1 <= b.n { b.update(r+1, -val) }
}

func (b *RUPQ) PointQuery(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s += b.tree[i] }
	return s
}

func main() {
	n := 8
	rupq := NewRUPQ(n)

	rupq.RangeAdd(2, 5, 10) // add 10 to positions [2,5]
	rupq.RangeAdd(3, 7, 5)  // add 5 to positions [3,7]

	fmt.Println("After range_add(2,5,10) and range_add(3,7,5):")
	for i := 0; i < n; i++ {
		fmt.Printf("  arr[%d] = %d\n", i, rupq.PointQuery(i))
	}
}
```

**Textual Figure:**

```
BIT for Range Update + Point Query (difference array in BIT)
n=8, initially all zeros

Key insight: treat BIT as a difference array
  range_add(l, r, val)  →  bit.update(l, +val), bit.update(r+1, -val)
  point_query(i)        →  bit.prefix_sum(i)

Operation 1: RangeAdd(2, 5, 10)  → add 10 to [2..5]
  BIT diff:  [0, 0, +10, 0, 0, 0, -10, 0]
  Values:     0  0  10  10  10  10   0   0

Operation 2: RangeAdd(3, 7, 5)   → add 5 to [3..7]
  BIT diff:  [0, 0, +10, +5, 0, 0, -10, 0]  (index 8 gets -5 but >n)
  Values:     0  0  10  15  15  15   5   5

  Verification via prefix sums:
    point(0) = 0       point(4) = 10+5 = 15
    point(1) = 0       point(5) = 10+5 = 15
    point(2) = 10      point(6) = 10+5-10 = 5
    point(3) = 10+5=15 point(7) = 10+5-10 = 5
```

---

## Example 8: BIT for Range Update + Range Query

```go
package main

import "fmt"

// Uses two BITs: B1 and B2
// prefix_sum(x) = B1[x]*x - B2[x]

type RURQ struct {
	b1, b2 []int64
	n      int
}

func NewRURQ(n int) *RURQ {
	return &RURQ{b1: make([]int64, n+2), b2: make([]int64, n+2), n: n}
}

func (r *RURQ) addBIT(bit []int64, i int, delta int64) {
	for i++; i <= r.n+1; i += i & (-i) { bit[i] += delta }
}

func (r *RURQ) sumBIT(bit []int64, i int) int64 {
	s := int64(0)
	for i++; i > 0; i -= i & (-i) { s += bit[i] }
	return s
}

func (r *RURQ) RangeAdd(l, ri, val int) {
	v := int64(val)
	r.addBIT(r.b1, l, v)
	r.addBIT(r.b1, ri+1, -v)
	r.addBIT(r.b2, l, v*int64(l))
	r.addBIT(r.b2, ri+1, -v*int64(ri+1))
}

func (r *RURQ) prefixSum(i int) int64 {
	return r.sumBIT(r.b1, i)*int64(i+1) - r.sumBIT(r.b2, i)
}

func (r *RURQ) RangeSum(l, ri int) int64 {
	s := r.prefixSum(ri)
	if l > 0 { s -= r.prefixSum(l - 1) }
	return s
}

func main() {
	n := 8
	rurq := NewRURQ(n)

	rurq.RangeAdd(1, 5, 3) // add 3 to [1,5]
	rurq.RangeAdd(3, 7, 2) // add 2 to [3,7]

	fmt.Println("After range_add(1,5,3) and range_add(3,7,2):")
	for i := 0; i < n; i++ {
		fmt.Printf("  arr[%d] = %d\n", i, rurq.RangeSum(i, i))
	}
	fmt.Printf("\nRange sum [2,6] = %d\n", rurq.RangeSum(2, 6))
}
```

**Textual Figure:**

```
BIT for Range Update + Range Query (two BITs: B1 and B2)
n=8, initially all zeros

Formula: prefix_sum(x) = B1[x] * (x+1) - B2[x]

Range_add(l, r, val):
  B1: add(l, val), add(r+1, -val)
  B2: add(l, val*l), add(r+1, -val*(r+1))

Op1: RangeAdd(1, 5, 3)   → add 3 to [1..5]
  B1: update(1,+3), update(6,-3)
  B2: update(1,+3), update(6,-18)
  Values: [0, 3, 3, 3, 3, 3, 0, 0]

Op2: RangeAdd(3, 7, 2)   → add 2 to [3..7]
  B1: update(3,+2), update(8,-2)
  B2: update(3,+6), update(8,-16)
  Values: [0, 3, 3, 5, 5, 5, 2, 2]

  arr[0]=0  arr[1]=3  arr[2]=3  arr[3]=5
  arr[4]=5  arr[5]=5  arr[6]=2  arr[7]=2

  RangeSum(2,6) = prefix(6) - prefix(1)
    prefix(6) = 0+3+3+5+5+5+2 = 23
    prefix(1) = 0+3 = 3
    Result = 23 - 3 = 20  (3+5+5+5+2 = 20) ✓
```

---

## Example 9: K-th Smallest Element with BIT

```go
package main

import "fmt"

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+1), n: n} }

func (b *BIT) Update(i, delta int) {
	for i++; i <= b.n; i += i & (-i) { b.tree[i] += delta }
}

// Find smallest i such that prefix_sum(i) >= k
func (b *BIT) KthSmallest(k int) int {
	pos := 0
	bitMask := 1
	for bitMask <= b.n { bitMask <<= 1 }
	bitMask >>= 1

	for bitMask > 0 {
		next := pos + bitMask
		if next <= b.n && b.tree[next] < k {
			k -= b.tree[next]
			pos = next
		}
		bitMask >>= 1
	}
	return pos // 0-indexed
}

func main() {
	maxVal := 10
	bit := NewBIT(maxVal + 1)

	// Insert elements
	elements := []int{5, 2, 8, 1, 3, 7}
	for _, e := range elements {
		bit.Update(e, 1)
	}

	fmt.Printf("Elements: %v\n", elements)
	for k := 1; k <= len(elements); k++ {
		fmt.Printf("  %d-th smallest = %d\n", k, bit.KthSmallest(k))
	}

	// Remove element 3
	bit.Update(3, -1)
	fmt.Println("\nAfter removing 3:")
	for k := 1; k <= len(elements)-1; k++ {
		fmt.Printf("  %d-th smallest = %d\n", k, bit.KthSmallest(k))
	}
}
```

**Textual Figure:**

```
K-th Smallest via BIT with binary lifting
Elements: [5, 2, 8, 1, 3, 7]  → maxVal=10

BIT stores frequency counts:
  Value:  0  1  2  3  4  5  6  7  8  9  10
  Freq:   0  1  1  1  0  1  0  1  1  0   0
  tree[]: BIT of the above frequencies

Binary lifting to find k-th smallest:
  Start pos=0, walk MSB to LSB of tree size

  1st smallest (k=1):
    bitMask=8: pos+8=8, tree[8]=6 ≥ 1 → don't jump
    bitMask=4: pos+4=4, tree[4]=3 ≥ 1 → don't jump
    bitMask=2: pos+2=2, tree[2]=2 ≥ 1 → don't jump
    bitMask=1: pos+1=1, tree[1]=1 ≥ 1 → don't jump
    Result: pos=0 → but 0 has count 0, so actual = 1 ✓

  3rd smallest (k=3):
    Walk until prefix count ≥ 3 → value 3 ✓

  Sorted order: 1, 2, 3, 5, 7, 8

  After removing 3 (Update(3, -1)):
    Sorted: 1, 2, 5, 7, 8
    3rd smallest = 5 ✓
```

---

## Example 10: BIT vs Segment Tree Comparison

```go
package main

import (
	"fmt"
	"math"
	"time"
)

// BIT implementation
type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+1), n: n} }
func (b *BIT) Update(i, v int) { for i++; i <= b.n; i += i & (-i) { b.tree[i] += v } }
func (b *BIT) Query(i int) int { s := 0; for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s }

// Segment Tree implementation
type SegTree struct {
	tree []int
	n    int
}

func NewSeg(n int) *SegTree { return &SegTree{tree: make([]int, 4*n), n: n} }
func (s *SegTree) update(node, lo, hi, i, v int) {
	if lo == hi { s.tree[node] += v; return }
	mid := (lo + hi) / 2
	if i <= mid { s.update(2*node, lo, mid, i, v) } else { s.update(2*node+1, mid+1, hi, i, v) }
	s.tree[node] = s.tree[2*node] + s.tree[2*node+1]
}
func (s *SegTree) query(node, lo, hi, l, r int) int {
	if r < lo || hi < l { return 0 }
	if l <= lo && hi <= r { return s.tree[node] }
	mid := (lo + hi) / 2
	return s.query(2*node, lo, mid, l, r) + s.query(2*node+1, mid+1, hi, l, r)
}

func main() {
	n := 100000
	ops := 100000

	// Benchmark BIT
	bit := NewBIT(n)
	start := time.Now()
	for i := 0; i < ops; i++ {
		bit.Update(i%n, i)
		_ = bit.Query(i % n)
	}
	bitTime := time.Since(start)

	// Benchmark Segment Tree
	seg := NewSeg(n)
	start = time.Now()
	for i := 0; i < ops; i++ {
		seg.update(1, 0, n-1, i%n, i)
		_ = seg.query(1, 0, n-1, 0, i%n)
	}
	segTime := time.Since(start)

	fmt.Printf("n=%d, ops=%d\n", n, ops)
	fmt.Printf("  BIT:          %v\n", bitTime)
	fmt.Printf("  Segment Tree: %v\n", segTime)

	fmt.Println("\n=== When to Use Which ===")
	fmt.Println("  BIT: point update + prefix/range sum (simpler, faster constant)")
	fmt.Println("  Seg: range update, non-invertible ops (min, max, gcd)")
	fmt.Println("  BIT: 2-3x less memory than segment tree")
	fmt.Println("  BIT: can be extended to 2D easily")

	_ = math.MaxInt64
}
```

**Textual Figure:**

```
BIT vs Segment Tree — Performance Comparison:

  ┌────────────────┬─────────────────┬─────────────────┐
  │                │ Fenwick (BIT)     │ Segment Tree      │
  ├────────────────┼─────────────────┼─────────────────┤
  │ Memory         │ 1× array (n+1)   │ 4× array (4n)    │
  │ Code lines     │ ~15 lines        │ ~40 lines        │
  │ Point update   │ O(log n)         │ O(log n)         │
  │ Range query    │ O(log n)         │ O(log n)         │
  │ Range update   │ ✓ (with tricks)  │ ✓ (native lazy)  │
  │ Min/Max query  │ ✘                │ ✓                │
  │ Constant factor│ Faster           │ Slower           │
  └────────────────┴─────────────────┴─────────────────┘

  BIT update path:   i, i+lowbit(i), i+lowbit(i)+lowbit(...), ...
  BIT query path:    i, i-lowbit(i), i-lowbit(i)-lowbit(...), ...

  BIT update(5):  5 ─→ 6 ─→ 8   (3 hops for n=8)
  Seg update(5):  leaf ─→ parent ─→ ... root  (log n hops + recursion overhead)

  Decision: Use BIT for sum/xor; use SegTree for min/max/gcd
```

---

## Key Takeaways

| Feature | BIT | Segment Tree |
|---------|-----|-------------|
| Point update | ✅ O(log n) | ✅ O(log n) |
| Range update | ✅ With tricks | ✅ Native with lazy |
| Prefix query | ✅ O(log n) | ✅ O(log n) |
| Range query | ✅ prefix(r)-prefix(l-1) | ✅ Native |
| Non-invertible (min,max) | ❌ | ✅ |
| Space | O(n) | O(4n) |
| Code complexity | Very simple | Moderate |
| K-th smallest | ✅ Binary lifting | ✅ Walk down |

1. **BIT is preferred** when only sum/xor queries are needed (invertible operations)
2. **lowbit = x & (-x)** is the fundamental operation
3. **O(n) build** by propagating to parent: `tree[i + lowbit(i)] += tree[i]`
4. **Range update + point query**: difference BIT
5. **Range update + range query**: two BITs (B1, B2)

> **Next up:** Point Update Range Query →
