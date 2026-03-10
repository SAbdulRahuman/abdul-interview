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

**Textual Figure:**

```
Array: [1, 3, 5, 7, 9, 11]   (indices 0..5)

Segment Tree (Range Sum) — Build:

                        ┌─────────────────────┐
                        │  node 1: [0,5] = 36 │
                        └──────────┬──────────┘
               ┌───────────────────┴───────────────────┐
       ┌───────┴───────┐                       ┌───────┴───────┐
       │ node 2:[0,2]=9│                       │node 3:[3,5]=27│
       └───────┬───────┘                       └───────┬───────┘
          ┌────┴────┐                             ┌────┴────┐
    ┌─────┴─────┐ ┌─┴──────────┐           ┌─────┴─────┐ ┌─┴──────────┐
    │n4:[0,1]=4 │ │n5:[2,2]=5  │           │n6:[3,4]=16│ │n7:[5,5]=11 │
    └─────┬─────┘ └────────────┘           └─────┬─────┘ └────────────┘
     ┌────┴────┐                            ┌────┴────┐
  ┌──┴───┐ ┌──┴───┐                     ┌──┴───┐ ┌──┴───┐
  │n8:1  │ │n9:3  │                     │n12:7 │ │n13:9 │
  └──────┘ └──────┘                     └──────┘ └──────┘

Query Sum [1,3]:  visit n9(3) + n5(5) + n12(7) → answer = 15
Query Sum [0,5]:  node 1 fully covers [0,5] → answer = 36

Update arr[3]=10:  path n12→n6→n3→n1
  n12: 7→10, n6: 16→19, n3: 27→30, n1: 36→39

After update, Query Sum [1,3] = 3+5+10 = 18
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

**Textual Figure:**

```
Array: [2, 5, 1, 4, 9, 3]   (indices 0..5)

Segment Tree (Range Min) — Build:

                        ┌─────────────────────┐
                        │  node 1: [0,5] = 1  │
                        └──────────┬──────────┘
               ┌───────────────────┴───────────────────┐
       ┌───────┴───────┐                       ┌───────┴───────┐
       │ node 2:[0,2]=1│                       │node 3:[3,5]=3 │
       └───────┬───────┘                       └───────┬───────┘
          ┌────┴────┐                             ┌────┴────┐
    ┌─────┴─────┐ ┌─┴──────────┐           ┌─────┴─────┐ ┌─┴──────────┐
    │n4:[0,1]=2 │ │n5:[2,2]=1  │           │n6:[3,4]=4 │ │n7:[5,5]=3  │
    └─────┬─────┘ └────────────┘           └─────┬─────┘ └────────────┘
     ┌────┴────┐                            ┌────┴────┐
  ┌──┴───┐ ┌──┴───┐                     ┌──┴───┐ ┌──┴───┐
  │n8:2  │ │n9:5  │                     │n12:4 │ │n13:9 │
  └──────┘ └──────┘                     └──────┘ └──────┘

Query traces:
  Min [0,5] → node 1 covers all → 1
  Min [1,4] → min(n9:5, n5:1, n6:4) → 1
  Min [2,3] → min(n5:1, n12:4) → 1
  Min [0,2] → node 2 covers [0,2] → 1
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

**Textual Figure:**

```
Array: [2, 5, 1, 4, 9, 3]   (indices 0..5)

Segment Tree (Range Max) — Build:

                        ┌─────────────────────┐
                        │  node 1: [0,5] = 9  │
                        └──────────┬──────────┘
               ┌───────────────────┴───────────────────┐
       ┌───────┴───────┐                       ┌───────┴───────┐
       │ node 2:[0,2]=5│                       │node 3:[3,5]=9 │
       └───────┬───────┘                       └───────┬───────┘
          ┌────┴────┐                             ┌────┴────┐
    ┌─────┴─────┐ ┌─┴──────────┐           ┌─────┴─────┐ ┌─┴──────────┐
    │n4:[0,1]=5 │ │n5:[2,2]=1  │           │n6:[3,4]=9 │ │n7:[5,5]=3  │
    └─────┬─────┘ └────────────┘           └─────┬─────┘ └────────────┘
     ┌────┴────┐                            ┌────┴────┐
  ┌──┴───┐ ┌──┴───┐                     ┌──┴───┐ ┌──┴───┐
  │n8:2  │ │n9:5  │                     │n12:4 │ │n13:9 │
  └──────┘ └──────┘                     └──────┘ └──────┘

Query traces:
  Max [0,5] → node 1 fully covers → 9
  Max [0,2] → node 2 fully covers → 5
  Max [3,5] → node 3 fully covers → 9
  Max [1,4] → max(n9:5, n5:1, n6:9) → 9
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

**Textual Figure:**

```
Array: [12, 18, 24, 36, 48, 60]   (indices 0..5)

Segment Tree (Range GCD) — Build:

                        ┌─────────────────────┐
                        │  node 1: [0,5] = 6  │
                        └──────────┬──────────┘
               ┌───────────────────┴───────────────────┐
       ┌───────┴───────┐                       ┌───────┴───────┐
       │ node 2:[0,2]=6│                       │node 3:[3,5]=12│
       └───────┬───────┘                       └───────┬───────┘
          ┌────┴────┐                             ┌────┴────┐
    ┌─────┴──────┐ ┌─┴──────────┐          ┌─────┴──────┐ ┌─┴──────────┐
    │n4:[0,1]=6  │ │n5:[2,2]=24 │          │n6:[3,4]=12 │ │n7:[5,5]=60 │
    └─────┬──────┘ └────────────┘          └─────┬──────┘ └────────────┘
     ┌────┴────┐                            ┌────┴────┐
  ┌──┴───┐ ┌──┴────┐                    ┌──┴────┐ ┌──┴────┐
  │n8:12 │ │n9:18  │                    │n12:36 │ │n13:48 │
  └──────┘ └───────┘                    └───────┘ └───────┘

Query traces:
  GCD [0,5] → node 1 → gcd(6,12) = 6
  GCD [0,2] → node 2 → 6
  GCD [2,4] → gcd(n5:24, n6:12) → 12
  GCD [1,3] → gcd(n9:18, n5:24, n12:36) → gcd(18,24,36) → 6
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

**Textual Figure:**

```
Elements inserted: [3, 1, 4, 1, 5, 9, 2, 6]   (value range 0..9)

Count Segment Tree — frequency of each value:

  Index:  0  1  2  3  4  5  6  7  8  9
  Count:  0  2  1  1  1  1  1  0  0  1

                     ┌───────────────────────┐
                     │  node 1: [0,9] cnt=8  │
                     └───────────┬───────────┘
              ┌──────────────────┴──────────────────┐
       ┌──────┴──────┐                       ┌──────┴──────┐
       │n2:[0,4]=5   │                       │n3:[5,9]=3   │
       └──────┬──────┘                       └──────┬──────┘
         ┌────┴────┐                           ┌────┴────┐
      ┌──┴──┐   ┌──┴──┐                    ┌──┴──┐   ┌──┴──┐
      │[0,2]│   │[3,4]│                    │[5,7]│   │[8,9]│
      │ =3  │   │ =2  │                    │ =2  │   │ =1  │
      └─────┘   └─────┘                    └─────┘   └─────┘

K-th smallest via tree walk:
  1st → left cnt=5 ≥ 1, go left → ... → val 1
  2nd → left cnt=5 ≥ 2, go left → ... → val 1  (count=2)
  3rd → left cnt=5 ≥ 3, go left → ... → val 2
  4th → left cnt=5 ≥ 4, go left → ... → val 3
  5th → left cnt=5 ≥ 5, go left → ... → val 4
  6th → left cnt=5 < 6, go right → k'=1 → val 5

Count in [0,5]: query returns 5+1 = 6
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

**Textual Figure:**

```
Array: [1, 3, 5, 7, 9, 11]   n=6

Iterative Segment Tree (Bottom-Up) — internal array layout:

  Index:   1    2    3    4    5    6    7    8    9   10   11
  Value:  36    9   27    4    5   16   11    1    3    7    9
         root                 leaves start at index n=6

  Leaf positions:  tree[6]=1  tree[7]=3  tree[8]=5  tree[9]=7  tree[10]=9  tree[11]=11

  Build (bottom-up): for i = n-1 downto 1:
    tree[5] = tree[10]+tree[11] = 9+11  = 20  ▒ Wait, let me recalc -
    Actually with n=6: tree[6..11] = [1,3,5,7,9,11]
    tree[5]=tree[10]+tree[11]=9+11=20, tree[4]=tree[8]+tree[9]=5+7=12
    tree[3]=tree[6]+tree[7]=1+3=4,    tree[2]=tree[4]+tree[5]=12+20=32
    tree[1]=tree[2]+tree[3]=32+4=36

  Query [1,3] (l=7, r=10 in tree):
    l=7 odd → add tree[7]=3, l=8 → l=4
    r=10 even → r=5
    l=4 even, r=5 odd → r-- add tree[4]=12? → r=4 → r=2
    l=4 → l=2; l≥r stop.  sum=3+12=15 ✓

  Update arr[3]=10: set tree[9]=10, propagate up:
    tree[4] = tree[8]+tree[9] = 5+10 = 15
    tree[2] = tree[4]+tree[5] = 15+20 = 35
    tree[1] = tree[2]+tree[3] = 35+4 = 39
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

**Textual Figure:**

```
Dynamic Segment Tree — Sparse index range [0, 1,000,000,000)
Only 3 values inserted: idx=100(val=5), idx=1000000(val=3), idx=999999999(val=7)

                    ┌───────────────────┐
                    │ root: [0,1B] =15  │  (sum of all)
                    └───────┬───────────┘
               ┌───────┴──────┐
         ┌─────┴─────────┐      ┌────────────────┐
         │ [0,500M] =8   │      │ [500M+1,1B] =7 │
         └───┬───────────┘      └─────┬──────────┘
            │  ...path to 100        │  ...path to 999999999
            │  (val=5)                │  (val=7)
            │                         │
         ...and path to 1000000 (val=3)

  • Total nodes created: O(3 × log(1B)) ≈ 90 nodes
  • Instead of 4 billion entries in a flat array
  • nil children = no elements in that subtree

  Query [0,1B]:   sum = 15
  Query [0,500]:  sum = 5   (only idx=100 in range)
  Query [500,2M]: sum = 3   (only idx=1000000)
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

**Textual Figure:**

```
Persistent Segment Tree — arr = [1, 2, 3, 4, 5]

Version 0 (original): sum = 15

         ┌──────────────┐
    V0:  │ [0,4] = 15   │
         └───┬────┬────┘
          ┌─┴─┐ ┌─┴─┐
          │ 6  │ │ 9  │    [0,2]=6   [3,4]=9
          └─┬─┘ └─┬─┘
         ┌─┴┐┌┴┐┌─┴┐┌┴┐
         │1+2││3││4+5│  ← leaves
         └──┘└─┘└──┘

Version 1 (arr[2]=10):  path-copy nodes along root→[0,2]→[2,2]

    V1 root ──┐         Shared: [3,4]=9 subtree
         ┌─┴──────────┐
         │ [0,4] = 22   │  (sum=1+2+10+4+5)
         └─┬──────────┘
      new [0,2]=13──┐              old [3,4]=9 (shared)
               new [2,2]=10

Version 2 (arr[4]=20): path-copy from V1 root→[3,4]→[4,4]

    V2 root: [0,4] = 37   (1+2+10+4+20)
         shares [0,2]=13 from V1
         new [3,4]=24, new [4,4]=20

  Version 0 sum[0,4] = 15  (original preserved)
  Version 2 sum[0,4] = 37  (after both updates)
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

**Textual Figure:**

```
Counting inversions with online segment tree — arr = [2, 4, 1, 3, 5]

Process left-to-right; for each element, count how many
already-inserted elements are GREATER (query [v+1, max]):

  Step 1: insert 2        seg: [0]=0 [1]=0 [2]=1 [3]=0 [4]=0 [5]=0
          query [3,5] = 0                                inversions += 0

  Step 2: insert 4        seg: [0]=0 [1]=0 [2]=1 [3]=0 [4]=1 [5]=0
          query [5,5] = 0                                inversions += 0

  Step 3: insert 1        seg: [0]=0 [1]=1 [2]=1 [3]=0 [4]=1 [5]=0
          query [2,5] = 2   (elements 2,4 > 1)           inversions += 2

  Step 4: insert 3        seg: [0]=0 [1]=1 [2]=1 [3]=1 [4]=1 [5]=0
          query [4,5] = 1   (element 4 > 3)              inversions += 1

  Step 5: insert 5        seg: [0]=0 [1]=1 [2]=1 [3]=1 [4]=1 [5]=1
          query [6,5] = 0   (empty range)                inversions += 0

  Total inversions = 0 + 0 + 2 + 1 + 0 = 3
  Pairs: (2,1), (4,1), (4,3)
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

**Textual Figure:**

```
Segment Tree Construction Patterns Overview:

  Generic Template:
  ┌───────────────────────────────────────────────────────┐
  │  Build: leaf = arr[i]; internal = merge(L, R)  │
  │  Query: if covers → return; else merge children │
  │  Update: set leaf; propagate merge upward        │
  └───────────────────────────────────────────────────────┘

  Variant        │ merge(a,b)  │ identity │ Example
  ───────────────┼────────────┼──────────┼──────────────
  Range Sum      │ a + b      │ 0        │ [1,3,5] → 9
  Range Min      │ min(a,b)   │ +∞       │ [2,5,1] → 1
  Range Max      │ max(a,b)   │ -∞       │ [2,5,1] → 5
  Range GCD      │ gcd(a,b)   │ 0        │ [12,18] → 6
  Range XOR      │ a ^ b      │ 0        │ [3,5]   → 6

  Recursive vs Iterative:
  ┌───────────────┬─────────────────┬────────────────┐
  │               │ Recursive         │ Iterative        │
  ├───────────────┼─────────────────┼────────────────┤
  │ Code size     │ Moderate          │ Short            │
  │ Lazy support  │ Natural           │ Complex          │
  │ Speed         │ Slightly slower   │ Faster constant  │
  └───────────────┴─────────────────┴────────────────┘
```

---

## Key Takeaways

1. **Build O(n)**, query/update O(log n) — optimal for frequent range queries
2. **Any associative function** works: sum, min, max, GCD, XOR, product
3. **Iterative** version is faster but recursive is more flexible (lazy propagation)
4. **Dynamic/sparse** trees handle huge index ranges efficiently
5. **Persistent** trees keep version history with O(log n) per update

> **Next up:** Range Sum Queries →
