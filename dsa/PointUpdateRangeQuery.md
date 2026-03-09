# Phase 22: Segment Tree & Fenwick Tree — Point Update Range Query (PURQ)

## Overview

**PURQ (Point Update, Range Query)** is the most fundamental segment tree / BIT pattern:
- **Update**: Change a single element `arr[i] = val` or `arr[i] += delta`
- **Query**: Aggregate over range `[l, r]` — sum, min, max, GCD, etc.

| Data Structure | Update | Query | Space |
|---------------|--------|-------|-------|
| BIT | O(log n) | O(log n) | O(n) |
| Segment Tree | O(log n) | O(log n) | O(4n) |
| Sqrt Decomposition | O(1) | O(√n) | O(n) |

---

## Example 1: PURQ Sum with BIT

```go
package main

import "fmt"

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+1), n: n} }

func (b *BIT) PointUpdate(i, delta int) {
	for i++; i <= b.n; i += i & (-i) { b.tree[i] += delta }
}

func (b *BIT) PrefixSum(i int) int {
	s := 0; for i++; i > 0; i -= i & (-i) { s += b.tree[i] }; return s
}

func (b *BIT) RangeSum(l, r int) int {
	if l == 0 { return b.PrefixSum(r) }
	return b.PrefixSum(r) - b.PrefixSum(l-1)
}

func main() {
	arr := []int{1, 7, 3, 0, 5, 8, 3, 2, 6, 2}
	n := len(arr)
	bit := NewBIT(n)
	for i, v := range arr { bit.PointUpdate(i, v) }

	fmt.Printf("Array: %v\n\n", arr)

	queries := [][2]int{{0, 4}, {2, 7}, {0, 9}, {5, 8}}
	for _, q := range queries {
		fmt.Printf("  Sum [%d,%d] = %d\n", q[0], q[1], bit.RangeSum(q[0], q[1]))
	}

	bit.PointUpdate(3, 10) // arr[3] += 10
	fmt.Printf("\nAfter arr[3]+=10: Sum [0,4] = %d\n", bit.RangeSum(0, 4))
}
```

---

## Example 2: PURQ Min with Segment Tree

```go
package main

import (
	"fmt"
	"math"
)

type MinSeg struct {
	tree []int
	n    int
}

func NewMinSeg(arr []int) *MinSeg {
	n := len(arr)
	ms := &MinSeg{tree: make([]int, 4*n), n: n}
	ms.build(arr, 1, 0, n-1)
	return ms
}

func (ms *MinSeg) build(arr []int, node, s, e int) {
	if s == e { ms.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	ms.build(arr, 2*node, s, mid)
	ms.build(arr, 2*node+1, mid+1, e)
	ms.tree[node] = min(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MinSeg) Update(i, val int) { ms.update(1, 0, ms.n-1, i, val) }

func (ms *MinSeg) update(node, s, e, i, val int) {
	if s == e { ms.tree[node] = val; return }
	mid := (s + e) / 2
	if i <= mid { ms.update(2*node, s, mid, i, val) } else { ms.update(2*node+1, mid+1, e, i, val) }
	ms.tree[node] = min(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MinSeg) Query(l, r int) int { return ms.query(1, 0, ms.n-1, l, r) }

func (ms *MinSeg) query(node, s, e, l, r int) int {
	if r < s || e < l { return math.MaxInt64 }
	if l <= s && e <= r { return ms.tree[node] }
	mid := (s + e) / 2
	return min(ms.query(2*node, s, mid, l, r), ms.query(2*node+1, mid+1, e, l, r))
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	arr := []int{5, 2, 4, 7, 1, 3}
	ms := NewMinSeg(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Min [0,5] = %d\n", ms.Query(0, 5))
	fmt.Printf("Min [2,4] = %d\n", ms.Query(2, 4))

	ms.Update(4, 10) // arr[4] = 10
	fmt.Printf("\nAfter arr[4]=10: Min [0,5] = %d\n", ms.Query(0, 5))
}
```

---

## Example 3: PURQ Max with Segment Tree

```go
package main

import (
	"fmt"
	"math"
)

type MaxSeg struct {
	tree []int
	n    int
}

func NewMaxSeg(arr []int) *MaxSeg {
	n := len(arr)
	ms := &MaxSeg{tree: make([]int, 4*n), n: n}
	ms.build(arr, 1, 0, n-1)
	return ms
}

func (ms *MaxSeg) build(arr []int, node, s, e int) {
	if s == e { ms.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	ms.build(arr, 2*node, s, mid)
	ms.build(arr, 2*node+1, mid+1, e)
	ms.tree[node] = max(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MaxSeg) Update(i, val int) { ms.update(1, 0, ms.n-1, i, val) }

func (ms *MaxSeg) update(node, s, e, i, val int) {
	if s == e { ms.tree[node] = val; return }
	mid := (s + e) / 2
	if i <= mid { ms.update(2*node, s, mid, i, val) } else { ms.update(2*node+1, mid+1, e, i, val) }
	ms.tree[node] = max(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MaxSeg) Query(l, r int) int { return ms.query(1, 0, ms.n-1, l, r) }

func (ms *MaxSeg) query(node, s, e, l, r int) int {
	if r < s || e < l { return math.MinInt64 }
	if l <= s && e <= r { return ms.tree[node] }
	mid := (s + e) / 2
	return max(ms.query(2*node, s, mid, l, r), ms.query(2*node+1, mid+1, e, l, r))
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	arr := []int{5, 2, 4, 7, 1, 3}
	ms := NewMaxSeg(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Max [0,5] = %d\n", ms.Query(0, 5))
	fmt.Printf("Max [1,3] = %d\n", ms.Query(1, 3))

	ms.Update(1, 20) // arr[1] = 20
	fmt.Printf("\nAfter arr[1]=20: Max [0,5] = %d\n", ms.Query(0, 5))
}
```

---

## Example 4: PURQ GCD

```go
package main

import "fmt"

type GCDSeg struct {
	tree []int
	n    int
}

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

func NewGCDSeg(arr []int) *GCDSeg {
	n := len(arr)
	gs := &GCDSeg{tree: make([]int, 4*n), n: n}
	gs.build(arr, 1, 0, n-1)
	return gs
}

func (gs *GCDSeg) build(arr []int, node, s, e int) {
	if s == e { gs.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	gs.build(arr, 2*node, s, mid)
	gs.build(arr, 2*node+1, mid+1, e)
	gs.tree[node] = gcd(gs.tree[2*node], gs.tree[2*node+1])
}

func (gs *GCDSeg) Update(i, val int) { gs.update(1, 0, gs.n-1, i, val) }

func (gs *GCDSeg) update(node, s, e, i, val int) {
	if s == e { gs.tree[node] = val; return }
	mid := (s + e) / 2
	if i <= mid { gs.update(2*node, s, mid, i, val) } else { gs.update(2*node+1, mid+1, e, i, val) }
	gs.tree[node] = gcd(gs.tree[2*node], gs.tree[2*node+1])
}

func (gs *GCDSeg) Query(l, r int) int { return gs.query(1, 0, gs.n-1, l, r) }

func (gs *GCDSeg) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return gs.tree[node] }
	mid := (s + e) / 2
	return gcd(gs.query(2*node, s, mid, l, r), gs.query(2*node+1, mid+1, e, l, r))
}

func main() {
	arr := []int{12, 18, 24, 36, 48}
	gs := NewGCDSeg(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("GCD [0,4] = %d\n", gs.Query(0, 4))
	fmt.Printf("GCD [1,3] = %d\n", gs.Query(1, 3))
	fmt.Printf("GCD [2,4] = %d\n", gs.Query(2, 4))

	gs.Update(2, 15)
	fmt.Printf("\nAfter arr[2]=15: GCD [0,4] = %d\n", gs.Query(0, 4))
}
```

---

## Example 5: PURQ XOR

```go
package main

import "fmt"

type XORSeg struct {
	tree []int
	n    int
}

func NewXORSeg(arr []int) *XORSeg {
	n := len(arr)
	xs := &XORSeg{tree: make([]int, 4*n), n: n}
	xs.build(arr, 1, 0, n-1)
	return xs
}

func (xs *XORSeg) build(arr []int, node, s, e int) {
	if s == e { xs.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	xs.build(arr, 2*node, s, mid)
	xs.build(arr, 2*node+1, mid+1, e)
	xs.tree[node] = xs.tree[2*node] ^ xs.tree[2*node+1]
}

func (xs *XORSeg) Update(i, val int) { xs.update(1, 0, xs.n-1, i, val) }

func (xs *XORSeg) update(node, s, e, i, val int) {
	if s == e { xs.tree[node] = val; return }
	mid := (s + e) / 2
	if i <= mid { xs.update(2*node, s, mid, i, val) } else { xs.update(2*node+1, mid+1, e, i, val) }
	xs.tree[node] = xs.tree[2*node] ^ xs.tree[2*node+1]
}

func (xs *XORSeg) Query(l, r int) int { return xs.query(1, 0, xs.n-1, l, r) }

func (xs *XORSeg) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 } // XOR identity
	if l <= s && e <= r { return xs.tree[node] }
	mid := (s + e) / 2
	return xs.query(2*node, s, mid, l, r) ^ xs.query(2*node+1, mid+1, e, l, r)
}

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	xs := NewXORSeg(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("XOR [0,5] = %d\n", xs.Query(0, 5))
	fmt.Printf("XOR [1,4] = %d\n", xs.Query(1, 4))
	fmt.Printf("XOR [2,2] = %d\n", xs.Query(2, 2))

	xs.Update(3, 0)
	fmt.Printf("\nAfter arr[3]=0: XOR [0,5] = %d\n", xs.Query(0, 5))
}
```

---

## Example 6: PURQ Count of Elements in Range (Merge Sort Tree)

```go
package main

import (
	"fmt"
	"sort"
)

// Count elements in [l,r] that are ≤ k using merge sort tree
type MergeSortTree struct {
	tree [][]int
	n    int
}

func NewMergeSortTree(arr []int) *MergeSortTree {
	n := len(arr)
	mst := &MergeSortTree{tree: make([][]int, 4*n), n: n}
	mst.build(arr, 1, 0, n-1)
	return mst
}

func (mst *MergeSortTree) build(arr []int, node, s, e int) {
	if s == e {
		mst.tree[node] = []int{arr[s]}
		return
	}
	mid := (s + e) / 2
	mst.build(arr, 2*node, s, mid)
	mst.build(arr, 2*node+1, mid+1, e)
	mst.tree[node] = mergeSorted(mst.tree[2*node], mst.tree[2*node+1])
}

func mergeSorted(a, b []int) []int {
	result := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		if a[i] <= b[j] { result = append(result, a[i]); i++ } else { result = append(result, b[j]); j++ }
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result
}

func (mst *MergeSortTree) countLessEqual(node, s, e, l, r, k int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r {
		return sort.SearchInts(mst.tree[node], k+1)
	}
	mid := (s + e) / 2
	return mst.countLessEqual(2*node, s, mid, l, r, k) +
		mst.countLessEqual(2*node+1, mid+1, e, l, r, k)
}

func (mst *MergeSortTree) CountInRange(l, r, lo, hi int) int {
	leHi := mst.countLessEqual(1, 0, mst.n-1, l, r, hi)
	leLo := mst.countLessEqual(1, 0, mst.n-1, l, r, lo-1)
	return leHi - leLo
}

func main() {
	arr := []int{3, 1, 4, 1, 5, 9, 2, 6}
	mst := NewMergeSortTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Count [0,7] with value in [2,5] = %d\n", mst.CountInRange(0, 7, 2, 5))
	fmt.Printf("Count [0,3] with value in [1,3] = %d\n", mst.CountInRange(0, 3, 1, 3))
	fmt.Printf("Count [4,7] with value in [1,9] = %d\n", mst.CountInRange(4, 7, 1, 9))
}
```

---

## Example 7: PURQ Product Modulo

```go
package main

import "fmt"

const MOD = 1_000_000_007

type ProdSeg struct {
	tree []int64
	n    int
}

func NewProdSeg(arr []int) *ProdSeg {
	n := len(arr)
	ps := &ProdSeg{tree: make([]int64, 4*n), n: n}
	ps.build(arr, 1, 0, n-1)
	return ps
}

func (ps *ProdSeg) build(arr []int, node, s, e int) {
	if s == e { ps.tree[node] = int64(arr[s]) % MOD; return }
	mid := (s + e) / 2
	ps.build(arr, 2*node, s, mid)
	ps.build(arr, 2*node+1, mid+1, e)
	ps.tree[node] = ps.tree[2*node] * ps.tree[2*node+1] % MOD
}

func (ps *ProdSeg) Update(i, val int) { ps.update(1, 0, ps.n-1, i, val) }

func (ps *ProdSeg) update(node, s, e, i, val int) {
	if s == e { ps.tree[node] = int64(val) % MOD; return }
	mid := (s + e) / 2
	if i <= mid { ps.update(2*node, s, mid, i, val) } else { ps.update(2*node+1, mid+1, e, i, val) }
	ps.tree[node] = ps.tree[2*node] * ps.tree[2*node+1] % MOD
}

func (ps *ProdSeg) Query(l, r int) int64 { return ps.query(1, 0, ps.n-1, l, r) }

func (ps *ProdSeg) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return 1 } // multiplicative identity
	if l <= s && e <= r { return ps.tree[node] }
	mid := (s + e) / 2
	return ps.query(2*node, s, mid, l, r) * ps.query(2*node+1, mid+1, e, l, r) % MOD
}

func main() {
	arr := []int{2, 3, 5, 7, 11}
	ps := NewProdSeg(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Product [0,4] = %d (mod %d)\n", ps.Query(0, 4), MOD)
	fmt.Printf("Product [1,3] = %d\n", ps.Query(1, 3))

	ps.Update(2, 1) // arr[2] = 1
	fmt.Printf("\nAfter arr[2]=1: Product [0,4] = %d\n", ps.Query(0, 4))
}
```

---

## Example 8: Iterative Bottom-Up Segment Tree (PURQ Sum)

```go
package main

import "fmt"

type IterSeg struct {
	tree []int
	n    int
}

func NewIterSeg(arr []int) *IterSeg {
	n := len(arr)
	tree := make([]int, 2*n)
	copy(tree[n:], arr)
	for i := n - 1; i > 0; i-- {
		tree[i] = tree[2*i] + tree[2*i+1]
	}
	return &IterSeg{tree: tree, n: n}
}

func (s *IterSeg) Update(i, val int) {
	i += s.n
	s.tree[i] = val
	for i >>= 1; i > 0; i >>= 1 {
		s.tree[i] = s.tree[2*i] + s.tree[2*i+1]
	}
}

func (s *IterSeg) Query(l, r int) int {
	res := 0
	for l, r = l+s.n, r+s.n+1; l < r; l, r = l>>1, r>>1 {
		if l&1 == 1 { res += s.tree[l]; l++ }
		if r&1 == 1 { r--; res += s.tree[r] }
	}
	return res
}

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	seg := NewIterSeg(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Sum [0,5] = %d\n", seg.Query(0, 5))
	fmt.Printf("Sum [1,4] = %d\n", seg.Query(1, 4))

	seg.Update(2, 10) // arr[2] = 10
	fmt.Printf("\nAfter arr[2]=10:\n")
	fmt.Printf("Sum [0,5] = %d\n", seg.Query(0, 5))

	fmt.Println("\nIterative: no recursion, faster constant factor")
}
```

---

## Example 9: Dynamic Segment Tree (PURQ on Large Range)

```go
package main

import "fmt"

type Node struct {
	left, right *Node
	sum         int64
}

type DynSeg struct {
	root *Node
	lo, hi int
}

func NewDynSeg(lo, hi int) *DynSeg {
	return &DynSeg{root: &Node{}, lo: lo, hi: hi}
}

func (ds *DynSeg) update(node *Node, lo, hi, pos int, val int64) {
	if lo == hi { node.sum += val; return }
	mid := lo + (hi-lo)/2
	if pos <= mid {
		if node.left == nil { node.left = &Node{} }
		ds.update(node.left, lo, mid, pos, val)
	} else {
		if node.right == nil { node.right = &Node{} }
		ds.update(node.right, mid+1, hi, pos, val)
	}
	node.sum = getSum(node.left) + getSum(node.right)
}

func getSum(n *Node) int64 {
	if n == nil { return 0 }
	return n.sum
}

func (ds *DynSeg) query(node *Node, lo, hi, l, r int) int64 {
	if node == nil || r < lo || hi < l { return 0 }
	if l <= lo && hi <= r { return node.sum }
	mid := lo + (hi-lo)/2
	return ds.query(node.left, lo, mid, l, r) + ds.query(node.right, mid+1, hi, l, r)
}

func (ds *DynSeg) Update(pos int, val int64) { ds.update(ds.root, ds.lo, ds.hi, pos, val) }
func (ds *DynSeg) Query(l, r int) int64       { return ds.query(ds.root, ds.lo, ds.hi, l, r) }

func main() {
	// Supports range [0, 10^9] without allocating 10^9 nodes
	ds := NewDynSeg(0, 1_000_000_000)

	ds.Update(100, 5)
	ds.Update(500_000_000, 3)
	ds.Update(999_999_999, 7)

	fmt.Printf("Sum [0, 10^9] = %d\n", ds.Query(0, 1_000_000_000))
	fmt.Printf("Sum [0, 500000000] = %d\n", ds.Query(0, 500_000_000))
	fmt.Printf("Sum [100, 100] = %d\n", ds.Query(100, 100))

	fmt.Println("\nDynamic segment tree: only allocates nodes as needed")
	fmt.Println("Perfect for sparse data over huge ranges")
}
```

---

## Example 10: PURQ Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Point Update Range Query Patterns ===")
	fmt.Println()

	patterns := []struct{ aggregate, identity, merge, structures string }{
		{"Sum", "0", "a + b", "BIT, Seg Tree"},
		{"Min", "MaxInt", "min(a, b)", "Seg Tree only"},
		{"Max", "MinInt", "max(a, b)", "Seg Tree only"},
		{"GCD", "0", "gcd(a, b)", "Seg Tree only"},
		{"XOR", "0", "a ^ b", "BIT, Seg Tree"},
		{"Product", "1", "a * b", "Seg Tree"},
		{"OR", "0", "a | b", "Seg Tree only"},
		{"AND", "AllOnes", "a & b", "Seg Tree only"},
		{"Count ≤ k", "[]", "merge sort", "Merge Sort Tree"},
		{"K-th smallest", "count", "binary lift", "BIT, Seg Tree"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-14s | identity: %-8s | merge: %-14s | %s\n",
			p.aggregate, p.identity, p.merge, p.structures)
	}

	fmt.Println()
	fmt.Println("BIT works for INVERTIBLE operations: sum, xor")
	fmt.Println("Segment tree works for ALL associative operations")
	fmt.Println("Iterative seg tree: faster constant, harder to extend")
	fmt.Println("Dynamic seg tree: for huge ranges with sparse updates")
}
```

---

## Key Takeaways

1. **BIT for invertible ops** (sum, xor): simpler, faster, less memory
2. **Segment tree for all associative ops**: min, max, gcd, and, or
3. **Identity element**: the value that doesn't affect the result (0 for sum, MaxInt for min)
4. **Iterative segment tree**: faster constant factor, no recursion overhead
5. **Dynamic segment tree**: pointer-based, allocates only needed nodes — O(n) space for sparse data
6. **Merge sort tree**: answer k-th smallest or count queries, O(n log²n) per query

> **Next up:** Range Update Point Query →
