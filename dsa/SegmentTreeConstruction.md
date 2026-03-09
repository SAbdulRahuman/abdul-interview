# Phase 22: Segment Tree & Fenwick Tree — Segment Tree Construction

## Overview

A **segment tree** is a binary tree that stores interval/segment information. It allows efficient range queries and point updates in O(log n) time.

| Operation | Complexity |
|-----------|-----------|
| Build | O(n) |
| Point update | O(log n) |
| Range query | O(log n) |
| Space | O(4n) |

**Structure**: Array-based tree where node `i` has children `2i` and `2i+1`.

---

## Example 1: Basic Segment Tree (Range Sum)

```go
package main

import "fmt"

type SegTree struct {
	tree []int
	n    int
}

func NewSegTree(arr []int) *SegTree {
	n := len(arr)
	st := &SegTree{tree: make([]int, 4*n), n: n}
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *SegTree) build(arr []int, node, start, end int) {
	if start == end {
		st.tree[node] = arr[start]
		return
	}
	mid := (start + end) / 2
	st.build(arr, 2*node, start, mid)
	st.build(arr, 2*node+1, mid+1, end)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *SegTree) update(node, start, end, idx, val int) {
	if start == end {
		st.tree[node] = val
		return
	}
	mid := (start + end) / 2
	if idx <= mid {
		st.update(2*node, start, mid, idx, val)
	} else {
		st.update(2*node+1, mid+1, end, idx, val)
	}
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *SegTree) query(node, start, end, l, r int) int {
	if r < start || end < l { return 0 }
	if l <= start && end <= r { return st.tree[node] }
	mid := (start + end) / 2
	return st.query(2*node, start, mid, l, r) + st.query(2*node+1, mid+1, end, l, r)
}

func (st *SegTree) Update(idx, val int) { st.update(1, 0, st.n-1, idx, val) }
func (st *SegTree) Query(l, r int) int  { return st.query(1, 0, st.n-1, l, r) }

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	st := NewSegTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Sum [1,3]: %d\n", st.Query(1, 3))   // 3+5+7=15
	fmt.Printf("Sum [0,5]: %d\n", st.Query(0, 5))   // 36

	st.Update(3, 10) // arr[3] = 10
	fmt.Printf("\nAfter arr[3]=10:\n")
	fmt.Printf("Sum [1,3]: %d\n", st.Query(1, 3))   // 3+5+10=18
}
```

---

## Example 2: Range Min Query Segment Tree

```go
package main

import (
	"fmt"
	"math"
)

type MinSegTree struct {
	tree []int
	n    int
}

func NewMinSegTree(arr []int) *MinSegTree {
	n := len(arr)
	st := &MinSegTree{tree: make([]int, 4*n), n: n}
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *MinSegTree) build(arr []int, node, s, e int) {
	if s == e {
		st.tree[node] = arr[s]
		return
	}
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = min(st.tree[2*node], st.tree[2*node+1])
}

func (st *MinSegTree) query(node, s, e, l, r int) int {
	if r < s || e < l { return math.MaxInt64 }
	if l <= s && e <= r { return st.tree[node] }
	mid := (s + e) / 2
	return min(st.query(2*node, s, mid, l, r), st.query(2*node+1, mid+1, e, l, r))
}

func (st *MinSegTree) update(node, s, e, idx, val int) {
	if s == e { st.tree[node] = val; return }
	mid := (s + e) / 2
	if idx <= mid { st.update(2*node, s, mid, idx, val) } else { st.update(2*node+1, mid+1, e, idx, val) }
	st.tree[node] = min(st.tree[2*node], st.tree[2*node+1])
}

func min(a, b int) int { if a < b { return a }; return b }

func (st *MinSegTree) Query(l, r int) int  { return st.query(1, 0, st.n-1, l, r) }
func (st *MinSegTree) Update(idx, val int) { st.update(1, 0, st.n-1, idx, val) }

func main() {
	arr := []int{2, 5, 1, 4, 9, 3}
	st := NewMinSegTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	for _, q := range [][2]int{{0, 5}, {1, 4}, {2, 3}, {0, 2}} {
		fmt.Printf("  Min [%d,%d] = %d\n", q[0], q[1], st.Query(q[0], q[1]))
	}
}
```

---

## Example 3: Range Max Query Segment Tree

```go
package main

import (
	"fmt"
	"math"
)

type MaxSegTree struct {
	tree []int
	n    int
}

func NewMaxSegTree(arr []int) *MaxSegTree {
	n := len(arr)
	st := &MaxSegTree{tree: make([]int, 4*n), n: n}
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *MaxSegTree) build(arr []int, node, s, e int) {
	if s == e { st.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = max(st.tree[2*node], st.tree[2*node+1])
}

func (st *MaxSegTree) query(node, s, e, l, r int) int {
	if r < s || e < l { return math.MinInt64 }
	if l <= s && e <= r { return st.tree[node] }
	mid := (s + e) / 2
	return max(st.query(2*node, s, mid, l, r), st.query(2*node+1, mid+1, e, l, r))
}

func max(a, b int) int { if a > b { return a }; return b }

func (st *MaxSegTree) Query(l, r int) int { return st.query(1, 0, st.n-1, l, r) }

func main() {
	arr := []int{2, 5, 1, 4, 9, 3}
	st := NewMaxSegTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	for _, q := range [][2]int{{0, 5}, {0, 2}, {3, 5}, {1, 4}} {
		fmt.Printf("  Max [%d,%d] = %d\n", q[0], q[1], st.Query(q[0], q[1]))
	}
}
```

---

## Example 4: Range GCD Segment Tree

```go
package main

import "fmt"

type GCDSegTree struct {
	tree []int
	n    int
}

func gcd(a, b int) int {
	for b != 0 { a, b = b, a%b }
	return a
}

func NewGCDSegTree(arr []int) *GCDSegTree {
	n := len(arr)
	st := &GCDSegTree{tree: make([]int, 4*n), n: n}
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *GCDSegTree) build(arr []int, node, s, e int) {
	if s == e { st.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = gcd(st.tree[2*node], st.tree[2*node+1])
}

func (st *GCDSegTree) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	mid := (s + e) / 2
	return gcd(st.query(2*node, s, mid, l, r), st.query(2*node+1, mid+1, e, l, r))
}

func (st *GCDSegTree) update(node, s, e, idx, val int) {
	if s == e { st.tree[node] = val; return }
	mid := (s + e) / 2
	if idx <= mid { st.update(2*node, s, mid, idx, val) } else { st.update(2*node+1, mid+1, e, idx, val) }
	st.tree[node] = gcd(st.tree[2*node], st.tree[2*node+1])
}

func (st *GCDSegTree) Query(l, r int) int  { return st.query(1, 0, st.n-1, l, r) }
func (st *GCDSegTree) Update(idx, val int) { st.update(1, 0, st.n-1, idx, val) }

func main() {
	arr := []int{12, 18, 24, 36, 48, 60}
	st := NewGCDSegTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	for _, q := range [][2]int{{0, 5}, {0, 2}, {2, 4}, {1, 3}} {
		fmt.Printf("  GCD [%d,%d] = %d\n", q[0], q[1], st.Query(q[0], q[1]))
	}
}
```

---

## Example 5: Segment Tree with Node Count (Count Elements in Range)

```go
package main

import "fmt"

type CountSegTree struct {
	tree []int
	n    int
}

func NewCountSegTree(n int) *CountSegTree {
	return &CountSegTree{tree: make([]int, 4*n), n: n}
}

func (st *CountSegTree) update(node, s, e, idx, val int) {
	if s == e { st.tree[node] += val; return }
	mid := (s + e) / 2
	if idx <= mid { st.update(2*node, s, mid, idx, val) } else { st.update(2*node+1, mid+1, e, idx, val) }
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *CountSegTree) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	mid := (s + e) / 2
	return st.query(2*node, s, mid, l, r) + st.query(2*node+1, mid+1, e, l, r)
}

// Find k-th smallest element
func (st *CountSegTree) kth(node, s, e, k int) int {
	if s == e { return s }
	mid := (s + e) / 2
	leftCount := st.tree[2*node]
	if k <= leftCount { return st.kth(2*node, s, mid, k) }
	return st.kth(2*node+1, mid+1, e, k-leftCount)
}

func (st *CountSegTree) Add(idx int)    { st.update(1, 0, st.n-1, idx, 1) }
func (st *CountSegTree) Remove(idx int) { st.update(1, 0, st.n-1, idx, -1) }
func (st *CountSegTree) Count(l, r int) int { return st.query(1, 0, st.n-1, l, r) }
func (st *CountSegTree) Kth(k int) int  { return st.kth(1, 0, st.n-1, k) }

func main() {
	st := NewCountSegTree(10)

	vals := []int{3, 1, 4, 1, 5, 9, 2, 6}
	for _, v := range vals { st.Add(v) }

	fmt.Printf("Elements: %v\n\n", vals)
	fmt.Printf("Count in [0,5]: %d\n", st.Count(0, 5))
	fmt.Printf("Count in [3,9]: %d\n", st.Count(3, 9))

	for k := 1; k <= len(vals); k++ {
		fmt.Printf("  %d-th smallest: %d\n", k, st.Kth(k))
	}
}
```

---

## Example 6: Iterative Segment Tree (Bottom-Up)

```go
package main

import "fmt"

type IterSegTree struct {
	tree []int
	n    int
}

func NewIterSegTree(arr []int) *IterSegTree {
	n := len(arr)
	tree := make([]int, 2*n)
	// Fill leaves
	copy(tree[n:], arr)
	// Build internal nodes
	for i := n - 1; i > 0; i-- {
		tree[i] = tree[2*i] + tree[2*i+1]
	}
	return &IterSegTree{tree: tree, n: n}
}

func (st *IterSegTree) Update(idx, val int) {
	idx += st.n
	st.tree[idx] = val
	for idx >>= 1; idx >= 1; idx >>= 1 {
		st.tree[idx] = st.tree[2*idx] + st.tree[2*idx+1]
	}
}

func (st *IterSegTree) Query(l, r int) int {
	sum := 0
	l += st.n
	r += st.n + 1
	for l < r {
		if l&1 == 1 { sum += st.tree[l]; l++ }
		if r&1 == 1 { r--; sum += st.tree[r] }
		l >>= 1
		r >>= 1
	}
	return sum
}

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	st := NewIterSegTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	fmt.Printf("Sum [1,3]: %d\n", st.Query(1, 3))
	fmt.Printf("Sum [0,5]: %d\n", st.Query(0, 5))

	st.Update(3, 10)
	fmt.Printf("\nAfter arr[3]=10:\n")
	fmt.Printf("Sum [1,3]: %d\n", st.Query(1, 3))

	fmt.Println("\nIterative vs Recursive:")
	fmt.Println("  • Iterative: faster constant factor, less code")
	fmt.Println("  • Recursive: easier lazy propagation")
}
```

---

## Example 7: Dynamic Segment Tree (Sparse)

```go
package main

import "fmt"

type Node struct {
	left, right *Node
	sum         int
}

type DynSegTree struct {
	root *Node
	lo, hi int
}

func NewDynSegTree(lo, hi int) *DynSegTree {
	return &DynSegTree{root: &Node{}, lo: lo, hi: hi}
}

func (st *DynSegTree) update(node *Node, lo, hi, idx, val int) {
	if lo == hi {
		node.sum += val
		return
	}
	mid := (lo + hi) / 2
	if idx <= mid {
		if node.left == nil { node.left = &Node{} }
		st.update(node.left, lo, mid, idx, val)
	} else {
		if node.right == nil { node.right = &Node{} }
		st.update(node.right, mid+1, hi, idx, val)
	}
	leftSum, rightSum := 0, 0
	if node.left != nil { leftSum = node.left.sum }
	if node.right != nil { rightSum = node.right.sum }
	node.sum = leftSum + rightSum
}

func (st *DynSegTree) query(node *Node, lo, hi, l, r int) int {
	if node == nil || r < lo || hi < l { return 0 }
	if l <= lo && hi <= r { return node.sum }
	mid := (lo + hi) / 2
	return st.query(node.left, lo, mid, l, r) + st.query(node.right, mid+1, hi, l, r)
}

func (st *DynSegTree) Update(idx, val int) { st.update(st.root, st.lo, st.hi, idx, val) }
func (st *DynSegTree) Query(l, r int) int  { return st.query(st.root, st.lo, st.hi, l, r) }

func main() {
	// Sparse range: [0, 1_000_000_000)
	st := NewDynSegTree(0, 1_000_000_000)

	st.Update(100, 5)
	st.Update(1_000_000, 3)
	st.Update(999_999_999, 7)

	fmt.Println("Dynamic Segment Tree (sparse indices):")
	fmt.Printf("  Query [0, 1B]:     %d\n", st.Query(0, 1_000_000_000))
	fmt.Printf("  Query [0, 500]:    %d\n", st.Query(0, 500))
	fmt.Printf("  Query [500, 2M]:   %d\n", st.Query(500, 2_000_000))
	fmt.Printf("  Only creates nodes for updated positions\n")
}
```

---

## Example 8: Persistent Segment Tree (Version History)

```go
package main

import "fmt"

type PNode struct {
	left, right *PNode
	sum         int
}

func build(arr []int, lo, hi int) *PNode {
	node := &PNode{}
	if lo == hi {
		node.sum = arr[lo]
		return node
	}
	mid := (lo + hi) / 2
	node.left = build(arr, lo, mid)
	node.right = build(arr, mid+1, hi)
	node.sum = node.left.sum + node.right.sum
	return node
}

func update(prev *PNode, lo, hi, idx, val int) *PNode {
	node := &PNode{}
	if lo == hi {
		node.sum = val
		return node
	}
	mid := (lo + hi) / 2
	if idx <= mid {
		node.left = update(prev.left, lo, mid, idx, val)
		node.right = prev.right // share unchanged subtree
	} else {
		node.left = prev.left
		node.right = update(prev.right, mid+1, hi, idx, val)
	}
	node.sum = node.left.sum + node.right.sum
	return node
}

func query(node *PNode, lo, hi, l, r int) int {
	if r < lo || hi < l { return 0 }
	if l <= lo && hi <= r { return node.sum }
	mid := (lo + hi) / 2
	return query(node.left, lo, mid, l, r) + query(node.right, mid+1, hi, l, r)
}

func main() {
	arr := []int{1, 2, 3, 4, 5}
	n := len(arr)

	versions := make([]*PNode, 0)
	versions = append(versions, build(arr, 0, n-1)) // version 0

	// version 1: arr[2] = 10
	versions = append(versions, update(versions[0], 0, n-1, 2, 10))
	// version 2: arr[4] = 20
	versions = append(versions, update(versions[1], 0, n-1, 4, 20))

	fmt.Println("Persistent Segment Tree:")
	for v := 0; v < len(versions); v++ {
		sum := query(versions[v], 0, n-1, 0, n-1)
		fmt.Printf("  Version %d: sum[0,%d] = %d\n", v, n-1, sum)
	}

	fmt.Printf("\n  Version 0, sum[0,4]: %d (original)\n", query(versions[0], 0, n-1, 0, 4))
	fmt.Printf("  Version 2, sum[0,4]: %d (after changes)\n", query(versions[2], 0, n-1, 0, 4))
}
```

---

## Example 9: Merge Sort Segment Tree (Count Inversions Online)

```go
package main

import "fmt"

type SegTree struct {
	tree []int
	n    int
}

func NewSegTree(n int) *SegTree {
	return &SegTree{tree: make([]int, 4*n), n: n}
}

func (st *SegTree) update(node, s, e, idx int) {
	if s == e { st.tree[node]++; return }
	mid := (s + e) / 2
	if idx <= mid { st.update(2*node, s, mid, idx) } else { st.update(2*node+1, mid+1, e, idx) }
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *SegTree) query(node, s, e, l, r int) int {
	if l > r || r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	mid := (s + e) / 2
	return st.query(2*node, s, mid, l, r) + st.query(2*node+1, mid+1, e, l, r)
}

func countInversions(arr []int) int {
	// Coordinate compression
	maxVal := 0
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	st := NewSegTree(maxVal + 1)
	inversions := 0

	for _, v := range arr {
		// Count elements already inserted that are > v
		inversions += st.query(1, 0, maxVal, v+1, maxVal)
		st.update(1, 0, maxVal, v)
	}
	return inversions
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

## Example 10: Segment Tree Construction Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Segment Tree Construction Patterns ===")
	fmt.Println()

	patterns := []struct{ variant, merge, identity string }{
		{"Range Sum", "a + b", "0"},
		{"Range Min", "min(a, b)", "INT_MAX"},
		{"Range Max", "max(a, b)", "INT_MIN"},
		{"Range GCD", "gcd(a, b)", "0"},
		{"Range XOR", "a ^ b", "0"},
		{"Range Product", "a * b (mod)", "1"},
		{"Count", "a + b", "0"},
		{"Iterative (bottom-up)", "Same merge, faster constant", "—"},
		{"Dynamic (sparse)", "Pointer-based, lazy creation", "Large index range"},
		{"Persistent", "Path copying on update", "Version history"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-25s merge: %-18s identity: %s\n", p.variant, p.merge, p.identity)
	}

	fmt.Println("\n=== Key Properties ===")
	fmt.Println("  • Array size: 4*n (safe upper bound)")
	fmt.Println("  • Node i → children 2i, 2i+1")
	fmt.Println("  • Build: O(n), Query/Update: O(log n)")
	fmt.Println("  • Works for any associative merge function")
}
```

---

## Key Takeaways

1. **Build O(n)**, query/update O(log n) — optimal for frequent range queries
2. **Any associative function** works: sum, min, max, GCD, XOR, product
3. **Iterative** version is faster but recursive is more flexible (lazy propagation)
4. **Dynamic/sparse** trees handle huge index ranges efficiently
5. **Persistent** trees keep version history with O(log n) per update

> **Next up:** Range Sum Queries →
