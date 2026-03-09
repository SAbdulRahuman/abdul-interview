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
