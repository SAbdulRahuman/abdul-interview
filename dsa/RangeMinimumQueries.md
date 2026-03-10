# Phase 22: Segment Tree & Fenwick Tree — Range Minimum Queries

## Overview

**Range Minimum Query (RMQ)** finds the minimum element in a subarray [l, r]. Multiple approaches exist with different trade-offs.

| Method | Build | Query | Update | Space |
|--------|-------|-------|--------|-------|
| Naive | O(1) | O(n) | O(1) | O(n) |
| Sparse Table | O(n log n) | O(1) | ❌ | O(n log n) |
| Segment Tree | O(n) | O(log n) | O(log n) | O(4n) |
| Sqrt Decomposition | O(n) | O(√n) | O(1) | O(n+√n) |

---

## Example 1: Sparse Table (O(1) Query, No Updates)

```go
package main

import (
	"fmt"
	"math/bits"
)

type SparseTable struct {
	table [][]int
	arr   []int
}

func NewSparseTable(arr []int) *SparseTable {
	n := len(arr)
	k := bits.Len(uint(n))
	table := make([][]int, k)
	table[0] = make([]int, n)
	copy(table[0], arr)

	for j := 1; j < k; j++ {
		table[j] = make([]int, n-(1<<j)+1)
		for i := 0; i+(1<<j) <= n; i++ {
			table[j][i] = min(table[j-1][i], table[j-1][i+(1<<(j-1))])
		}
	}
	return &SparseTable{table: table, arr: arr}
}

func (st *SparseTable) Query(l, r int) int {
	length := r - l + 1
	k := bits.Len(uint(length)) - 1
	return min(st.table[k][l], st.table[k][r-(1<<k)+1])
}

func min(a, b int) int { if a < b { return a }; return b }

func main() {
	arr := []int{1, 3, 2, 7, 9, 11, 3, 5, 6, 2}
	st := NewSparseTable(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 3}, {1, 5}, {4, 9}, {0, 9}, {2, 7}}
	for _, q := range queries {
		fmt.Printf("  Min [%d,%d] = %d\n", q[0], q[1], st.Query(q[0], q[1]))
	}

	fmt.Println("\nSparse Table: O(n log n) space, O(1) query, NO updates")
}
```

**Textual Figure:**

```
Sparse Table for RMQ
Array: [1, 3, 2, 7, 9, 11, 3, 5, 6, 2]   (indices 0..9)

table[j][i] = min of 2^j elements starting at i

  j=0 (len=1): [1, 3, 2, 7, 9, 11, 3, 5, 6, 2]
  j=1 (len=2): [1, 2, 2, 7, 9,  3, 3, 5, 2, -]
  j=2 (len=4): [1, 2, 2, 3, 3,  3, 2, -, -, -]
  j=3 (len=8): [1, 2, 2, -, -, -, -, -, -, -]

Query Min[1,5]:  len=5, k=floor(log2(5))=2
  table[2][1] = min(3,2,7,9) = 2
  table[2][5-4+1] = table[2][2] = min(2,7,9,11) = 2
  answer = min(2, 2) = 2  ✓

  Overlapping ranges are OK since min is idempotent:
  ┌───────────────┐
  │ [1..4]  min=2 │
  └───┬───────────┘
      │   ┌───────────────┐
      │   │ [2..5]  min=2 │
      └───┴───────────────┘
      overlap doesn't affect min
```

---

## Example 2: Segment Tree for RMQ (With Updates)

```go
package main

import (
	"fmt"
	"math"
)

type RMQ struct {
	tree []int
	n    int
}

func NewRMQ(arr []int) *RMQ {
	n := len(arr)
	rmq := &RMQ{tree: make([]int, 4*n), n: n}
	rmq.build(arr, 1, 0, n-1)
	return rmq
}

func (r *RMQ) build(arr []int, node, s, e int) {
	if s == e { r.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	r.build(arr, 2*node, s, mid)
	r.build(arr, 2*node+1, mid+1, e)
	r.tree[node] = min(r.tree[2*node], r.tree[2*node+1])
}

func (r *RMQ) update(node, s, e, idx, val int) {
	if s == e { r.tree[node] = val; return }
	mid := (s + e) / 2
	if idx <= mid { r.update(2*node, s, mid, idx, val) } else { r.update(2*node+1, mid+1, e, idx, val) }
	r.tree[node] = min(r.tree[2*node], r.tree[2*node+1])
}

func (r *RMQ) query(node, s, e, l, ri int) int {
	if ri < s || e < l { return math.MaxInt64 }
	if l <= s && e <= ri { return r.tree[node] }
	mid := (s + e) / 2
	return min(r.query(2*node, s, mid, l, ri), r.query(2*node+1, mid+1, e, l, ri))
}

func min(a, b int) int { if a < b { return a }; return b }

func (r *RMQ) Update(i, v int) { r.update(1, 0, r.n-1, i, v) }
func (r *RMQ) Query(l, ri int) int { return r.query(1, 0, r.n-1, l, ri) }

func main() {
	arr := []int{5, 2, 4, 7, 1, 3}
	rmq := NewRMQ(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 5}, {0, 2}, {2, 4}, {1, 3}}
	for _, q := range queries {
		fmt.Printf("  Min [%d,%d] = %d\n", q[0], q[1], rmq.Query(q[0], q[1]))
	}

	rmq.Update(4, 10) // arr[4] = 10
	fmt.Printf("\nAfter arr[4]=10, Min [0,5] = %d\n", rmq.Query(0, 5))
}
```

**Textual Figure:**

```
Segment Tree RMQ  — Array: [5, 2, 4, 7, 1, 3]

              ┌─────────────┐
              │ [0,5] min=1  │
              └──────┬───────┘
         ┌───────────┴───────────┐
    ┌────┴─────┐           ┌────┴─────┐
    │[0,2] m=2 │           │[3,5] m=1 │
    └────┬─────┘           └────┬─────┘
    ┌────┴────┐           ┌────┴────┐
 ┌──┴──┐ ┌──┴──┐       ┌──┴──┐ ┌──┴──┐
 │[0,1]│ │[2]=4│       │[3,4]│ │[5]=3│
 │ m=2 │ └─────┘       │ m=1 │ └─────┘
 └──┬──┘               └──┬──┘
 ┌──┴──┐┌──┴──┐       ┌──┴──┐┌──┴──┐
 │[0]=5││[1]=2│       │[3]=7││[4]=1│
 └─────┘└─────┘       └─────┘└─────┘

Query Min[2,4]: go right at [0,2] → 4, left at [3,5]:
  [3,4].min=1, combine: min(4,1) = 1  ✓

Update arr[4]=10:
  [4]=10 → [3,4]=min(7,10)=7 → [3,5]=min(7,3)=3
  → [0,5]=min(2,3)=2
```

---

## Example 3: RMQ with Index (Return Position of Minimum)

```go
package main

import (
	"fmt"
	"math"
)

type RMQIndex struct {
	tree []int // stores indices
	arr  []int
	n    int
}

func NewRMQIndex(arr []int) *RMQIndex {
	n := len(arr)
	rmq := &RMQIndex{tree: make([]int, 4*n), arr: arr, n: n}
	rmq.build(1, 0, n-1)
	return rmq
}

func (r *RMQIndex) better(i, j int) int {
	if i == -1 { return j }
	if j == -1 { return i }
	if r.arr[i] <= r.arr[j] { return i }
	return j
}

func (r *RMQIndex) build(node, s, e int) {
	if s == e { r.tree[node] = s; return }
	mid := (s + e) / 2
	r.build(2*node, s, mid)
	r.build(2*node+1, mid+1, e)
	r.tree[node] = r.better(r.tree[2*node], r.tree[2*node+1])
}

func (r *RMQIndex) query(node, s, e, l, ri int) int {
	if ri < s || e < l { return -1 }
	if l <= s && e <= ri { return r.tree[node] }
	mid := (s + e) / 2
	return r.better(r.query(2*node, s, mid, l, ri), r.query(2*node+1, mid+1, e, l, ri))
}

func (r *RMQIndex) QueryIdx(l, ri int) int { return r.query(1, 0, r.n-1, l, ri) }

func main() {
	arr := []int{5, 2, 4, 7, 1, 3}
	rmq := NewRMQIndex(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 5}, {0, 2}, {2, 4}, {3, 5}}
	for _, q := range queries {
		idx := rmq.QueryIdx(q[0], q[1])
		fmt.Printf("  Min [%d,%d] = arr[%d] = %d\n", q[0], q[1], idx, arr[idx])
	}
}

func max(a, b int) int { if a > b { return a }; return b }
func abs(a int) int { if a < 0 { return -a }; return a }
func unused() { _ = math.MaxInt64 }
```

**Textual Figure:**

```
RMQ with Index — returns position of minimum, not just value
Array: [5, 2, 4, 7, 1, 3]  (indices 0..5)

Tree stores INDICES (not values):
              ┌───────────┐
              │ [0,5] → 4  │  (arr[4]=1 is min)
              └─────┬──────┘
         ┌──────────┴──────────┐
    ┌────┴─────┐         ┌────┴─────┐
    │[0,2] → 1 │         │[3,5] → 4 │
    └────┬─────┘         └────┬─────┘
    ┌────┴────┐          ┌────┴────┐
  ┌─┴──┐ ┌──┴─┐       ┌─┴──┐ ┌──┴─┐
  │→ 1 │ │→ 2 │       │→ 4 │ │→ 5 │
  └─┬──┘ └────┘       └─┬──┘ └────┘
  ┌─┴─┐┌──┴─┐        ┌─┴─┐┌──┴─┐
  │→ 0││→ 1 │        │→ 3││→ 4 │
  └───┘└────┘        └───┘└────┘

  better(i, j) = i if arr[i] ≤ arr[j]

  Query [2,4]: returns index 4 (arr[4]=1)
  Query [0,2]: returns index 1 (arr[1]=2)
```

---

## Example 4: Sqrt Decomposition for RMQ

```go
package main

import (
	"fmt"
	"math"
)

type SqrtRMQ struct {
	arr    []int
	blocks []int
	bSize  int
	n      int
}

func NewSqrtRMQ(arr []int) *SqrtRMQ {
	n := len(arr)
	bSize := int(math.Sqrt(float64(n))) + 1
	nBlocks := (n + bSize - 1) / bSize
	blocks := make([]int, nBlocks)
	for i := range blocks { blocks[i] = math.MaxInt64 }

	data := make([]int, n)
	copy(data, arr)

	for i, v := range arr {
		bi := i / bSize
		if v < blocks[bi] { blocks[bi] = v }
	}
	return &SqrtRMQ{arr: data, blocks: blocks, bSize: bSize, n: n}
}

func (s *SqrtRMQ) Query(l, r int) int {
	result := math.MaxInt64
	for l <= r {
		if l%s.bSize == 0 && l+s.bSize-1 <= r {
			// Full block
			if s.blocks[l/s.bSize] < result { result = s.blocks[l/s.bSize] }
			l += s.bSize
		} else {
			if s.arr[l] < result { result = s.arr[l] }
			l++
		}
	}
	return result
}

func (s *SqrtRMQ) Update(idx, val int) {
	s.arr[idx] = val
	bi := idx / s.bSize
	s.blocks[bi] = math.MaxInt64
	start := bi * s.bSize
	end := start + s.bSize
	if end > s.n { end = s.n }
	for i := start; i < end; i++ {
		if s.arr[i] < s.blocks[bi] { s.blocks[bi] = s.arr[i] }
	}
}

func main() {
	arr := []int{5, 2, 4, 7, 1, 3, 8, 6, 9, 0}
	rmq := NewSqrtRMQ(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 9}, {2, 7}, {0, 4}, {5, 9}}
	for _, q := range queries {
		fmt.Printf("  Min [%d,%d] = %d\n", q[0], q[1], rmq.Query(q[0], q[1]))
	}

	fmt.Println("\nSqrt decomposition: O(√n) query, O(√n) update")
}
```

**Textual Figure:**

```
Sqrt Decomposition RMQ
Array: [5, 2, 4, 7, 1, 3, 8, 6, 9, 0]  (n=10, bSize=4)

  Block 0       Block 1       Block 2
  ┌─┬─┬─┬─┐   ┌─┬─┬─┬─┐   ┌─┬─┐
  │5│2│4│7│   │1│3│8│6│   │9│0│
  └─┴─┴─┴─┘   └─┴─┴─┴─┘   └─┴─┘
   min = 2      min = 1      min = 0

Query Min[2,7]:
  2,3 → partial block 0: scan arr[2]=4, arr[3]=7 → min=4
  4,5,6,7 → full block 1: blocks[1]=1
  answer = min(4, 1) = 1  ✓

Query Min[0,9]:
  full block 0 (min=2), full block 1 (min=1),
  full block 2 (min=0) → answer = 0  ✓

Update arr[4]=10:
  arr[4]=10 → rescan block 1: min(10,3,8,6)=3
  blocks[1] = 3
```

---

## Example 5: Sliding Window Minimum (Deque-Based)

```go
package main

import "fmt"

// LeetCode 239: Sliding Window Maximum (adapted for min)
func slidingWindowMin(nums []int, k int) []int {
	n := len(nums)
	result := make([]int, 0, n-k+1)
	deque := []int{} // stores indices

	for i := 0; i < n; i++ {
		// Remove elements outside window
		for len(deque) > 0 && deque[0] < i-k+1 {
			deque = deque[1:]
		}
		// Maintain increasing order
		for len(deque) > 0 && nums[deque[len(deque)-1]] >= nums[i] {
			deque = deque[:len(deque)-1]
		}
		deque = append(deque, i)

		if i >= k-1 {
			result = append(result, nums[deque[0]])
		}
	}
	return result
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{1, 3, -1, -3, 5, 3, 6, 7}, 3},
		{[]int{2, 1, 4, 5, 3, 4, 1, 2}, 4},
	}

	for _, t := range tests {
		fmt.Printf("  nums=%v, k=%d\n", t.nums, t.k)
		fmt.Printf("  Window mins: %v\n\n", slidingWindowMin(t.nums, t.k))
	}
}
```

**Textual Figure:**

```
Sliding Window Minimum (Monotonic Deque), k=3
Array: [1, 3, -1, -3, 5, 3, 6, 7]

  i=0: deque=[0]       → (waiting, i < k-1)
       1
  i=1: deque=[0,1]     → (waiting)
       1  3
  i=2: deque=[2]       → window[0,2] min=arr[2]=-1
       pop 0,1 (≥ -1)
  i=3: deque=[3]       → window[1,3] min=arr[3]=-3
       pop 2 (≥ -3)
  i=4: deque=[3,4]     → window[2,4] min=arr[3]=-3
       3 is still in window
  i=5: deque=[3,5]     → window[3,5] min=arr[3]=-3
       pop 4 (≥ 3)
  i=6: deque=[5,6]     → window[4,6] min=arr[5]=3
       evict 3 (out of window)
  i=7: deque=[5,6,7]   → window[5,7] min=arr[5]=3

  Result: [-1, -3, -3, -3, 3, 3]

  Deque invariant: indices in deque → values INCREASING
  Front = index of minimum in current window
```

---

## Example 6: Largest Rectangle in Histogram (LeetCode 84)

```go
package main

import "fmt"

func largestRectangleArea(heights []int) int {
	stack := []int{}
	maxArea := 0
	heights = append(heights, 0) // sentinel

	for i, h := range heights {
		for len(stack) > 0 && heights[stack[len(stack)-1]] > h {
			height := heights[stack[len(stack)-1]]
			stack = stack[:len(stack)-1]
			width := i
			if len(stack) > 0 { width = i - stack[len(stack)-1] - 1 }
			area := height * width
			if area > maxArea { maxArea = area }
		}
		stack = append(stack, i)
	}
	return maxArea
}

func main() {
	tests := [][]int{
		{2, 1, 5, 6, 2, 3},
		{2, 4},
		{1, 1, 1, 1, 1},
	}

	for _, h := range tests {
		fmt.Printf("  heights=%v → area=%d\n", h, largestRectangleArea(h))
	}
}
```

**Textual Figure:**

```
Largest Rectangle in Histogram (monotone stack)
heights = [2, 1, 5, 6, 2, 3]

  6│     ██
  5│   ████
  4│   ████
  3│   ████  ██
  2│██ ████████
  1│████████████
   └─────────────
    0  1  2  3  4  5

Stack-based processing (+ sentinel h=0 at end):
  i=0: push 0        stack=[0]
  i=1: pop 0 (h=2, w=1, area=2), push 1   stack=[1]
  i=2: push 2        stack=[1,2]
  i=3: push 3        stack=[1,2,3]
  i=4: pop 3 (h=6, w=1, area=6)
       pop 2 (h=5, w=2, area=10)  ← max!
       push 4        stack=[1,4]
  i=5: push 5        stack=[1,4,5]
  i=6: pop 5 (h=3, w=1, area=3)
       pop 4 (h=2, w=4, area=8)
       pop 1 (h=1, w=6, area=6)

  Answer = 10 (height=5, width=2, columns 2-3)
```

---

## Example 7: RMQ for LCA (Euler Tour Reduction)

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
)

// LCA via Euler tour + sparse table RMQ
type LCA struct {
	euler    []int
	depth    []int
	first    []int
	sparse   [][]int
}

func NewLCA(adj [][]int, root int) *LCA {
	n := len(adj)
	lca := &LCA{first: make([]int, n)}
	for i := range lca.first { lca.first[i] = -1 }

	// Euler tour
	var dfs func(u, d, parent int)
	dfs = func(u, d, parent int) {
		lca.first[u] = len(lca.euler)
		lca.euler = append(lca.euler, u)
		lca.depth = append(lca.depth, d)
		for _, v := range adj[u] {
			if v != parent {
				dfs(v, d+1, u)
				lca.euler = append(lca.euler, u)
				lca.depth = append(lca.depth, d)
			}
		}
	}
	dfs(root, 0, -1)

	// Build sparse table on depth array
	m := len(lca.depth)
	k := bits.Len(uint(m))
	lca.sparse = make([][]int, k)
	lca.sparse[0] = make([]int, m)
	for i := range lca.sparse[0] { lca.sparse[0][i] = i }

	for j := 1; j < k; j++ {
		lca.sparse[j] = make([]int, m-(1<<j)+1)
		for i := 0; i+(1<<j) <= m; i++ {
			l := lca.sparse[j-1][i]
			r := lca.sparse[j-1][i+(1<<(j-1))]
			if lca.depth[l] <= lca.depth[r] {
				lca.sparse[j][i] = l
			} else {
				lca.sparse[j][i] = r
			}
		}
	}
	return lca
}

func (lca *LCA) Query(u, v int) int {
	l, r := lca.first[u], lca.first[v]
	if l > r { l, r = r, l }
	length := r - l + 1
	k := bits.Len(uint(length)) - 1
	li := lca.sparse[k][l]
	ri := lca.sparse[k][r-(1<<k)+1]
	if lca.depth[li] <= lca.depth[ri] {
		return lca.euler[li]
	}
	return lca.euler[ri]
}

func main() {
	//       0
	//      / \
	//     1   2
	//    / \   \
	//   3   4   5
	adj := [][]int{
		{1, 2}, {0, 3, 4}, {0, 5}, {1}, {1}, {2},
	}
	lca := NewLCA(adj, 0)

	queries := [][2]int{{3, 4}, {3, 5}, {4, 5}, {1, 2}}
	for _, q := range queries {
		fmt.Printf("  LCA(%d, %d) = %d\n", q[0], q[1], lca.Query(q[0], q[1]))
	}
	_ = math.MaxInt64
}
```

**Textual Figure:**

```
LCA via Euler Tour + RMQ
Tree:         0
            /   \
           1     2
          / \     \
         3   4     5

Euler tour (DFS): 0 1 3 1 4 1 0 2 5 2 0
Depths:           0 1 2 1 2 1 0 1 2 1 0
first[]:  node 0→0  1→1  2→7  3→2  4→4  5→8

LCA(3,4): first[3]=2, first[4]=4
  Euler[2..4] = [3, 1, 4]
  Depth[2..4] = [2, 1, 2]
  Min depth at index 3 → euler[3] = 1
  LCA(3,4) = 1  ✓

LCA(3,5): first[3]=2, first[5]=8
  Euler[2..8] = [3, 1, 4, 1, 0, 2, 5]
  Min depth at index 6 → euler[6] = 0
  LCA(3,5) = 0  ✓

  Sparse table on depth[] → O(1) LCA queries
```

---

## Example 8: Range Min with Lazy (Range Update, Range Query)

```go
package main

import (
	"fmt"
	"math"
)

type LazyRMQ struct {
	tree, lazy []int
	n          int
}

func NewLazyRMQ(arr []int) *LazyRMQ {
	n := len(arr)
	rmq := &LazyRMQ{tree: make([]int, 4*n), lazy: make([]int, 4*n), n: n}
	rmq.build(arr, 1, 0, n-1)
	return rmq
}

func (r *LazyRMQ) build(arr []int, node, s, e int) {
	if s == e { r.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	r.build(arr, 2*node, s, mid)
	r.build(arr, 2*node+1, mid+1, e)
	r.tree[node] = min(r.tree[2*node], r.tree[2*node+1])
}

func (r *LazyRMQ) push(node int) {
	if r.lazy[node] != 0 {
		for _, child := range []int{2 * node, 2*node + 1} {
			r.tree[child] += r.lazy[node]
			r.lazy[child] += r.lazy[node]
		}
		r.lazy[node] = 0
	}
}

func (r *LazyRMQ) rangeAdd(node, s, e, l, ri, val int) {
	if ri < s || e < l { return }
	if l <= s && e <= ri {
		r.tree[node] += val
		r.lazy[node] += val
		return
	}
	r.push(node)
	mid := (s + e) / 2
	r.rangeAdd(2*node, s, mid, l, ri, val)
	r.rangeAdd(2*node+1, mid+1, e, l, ri, val)
	r.tree[node] = min(r.tree[2*node], r.tree[2*node+1])
}

func (r *LazyRMQ) query(node, s, e, l, ri int) int {
	if ri < s || e < l { return math.MaxInt64 }
	if l <= s && e <= ri { return r.tree[node] }
	r.push(node)
	mid := (s + e) / 2
	return min(r.query(2*node, s, mid, l, ri), r.query(2*node+1, mid+1, e, l, ri))
}

func min(a, b int) int { if a < b { return a }; return b }

func (r *LazyRMQ) RangeAdd(l, ri, val int) { r.rangeAdd(1, 0, r.n-1, l, ri, val) }
func (r *LazyRMQ) Query(l, ri int) int      { return r.query(1, 0, r.n-1, l, ri) }

func main() {
	arr := []int{5, 3, 7, 2, 8, 1}
	rmq := NewLazyRMQ(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Min [0,5] = %d\n", rmq.Query(0, 5))

	rmq.RangeAdd(0, 3, 10) // add 10 to arr[0..3]
	fmt.Printf("\nAfter adding 10 to [0,3]:\n")
	fmt.Printf("Min [0,5] = %d\n", rmq.Query(0, 5))
	fmt.Printf("Min [0,3] = %d\n", rmq.Query(0, 3))
	fmt.Printf("Min [4,5] = %d\n", rmq.Query(4, 5))
}
```

**Textual Figure:**

```
Lazy Segment Tree for Range Add + Range Min
Array: [5, 3, 7, 2, 8, 1]

Initial tree:
           ┌─────────────┐
           │ [0,5] min=1  │
           └──────┬──────┘
      ┌─────┴────┐   ┌────┴────┐
      │[0,2] m=3│   │[3,5] m=1│
      └──┬──────┘   └──┬──────┘
   ┌──┴──┐┌──┴─┐ ┌──┴──┐┌──┴─┐
   │[0,1]││[2] │ │[3,4]││[5] │
   │ m=3 ││ =7 │ │ m=2 ││ =1 │
   └─┬──┘└────┘ └─┬──┘└────┘
   5  3           2  8

RangeAdd(0, 3, +10):  lazy pushed to children
  [0,2].min += 10 = 13, lazy=10
  [3,3] ⊂ [3,5] split → [3,3].min=12, [4,5] untouched
  [0,5].min = min(13, 1) = 1

Minimum queries after +10 on [0,3]:
  Min[0,5] = 1   (arr[5] still 1)
  Min[0,3] = 12  (arr[3] was 2, now 12)
  Min[4,5] = 1   (unaffected)
```

---

## Example 9: Second Minimum in Range

```go
package main

import (
	"fmt"
	"math"
)

type TwoMin struct {
	tree [][2]int // [min1, min2]
	n    int
}

func NewTwoMin(arr []int) *TwoMin {
	n := len(arr)
	tm := &TwoMin{tree: make([][2]int, 4*n), n: n}
	tm.build(arr, 1, 0, n-1)
	return tm
}

func merge(a, b [2]int) [2]int {
	vals := []int{a[0], a[1], b[0], b[1]}
	m1, m2 := math.MaxInt64, math.MaxInt64
	for _, v := range vals {
		if v < m1 { m2 = m1; m1 = v } else if v < m2 { m2 = v }
	}
	return [2]int{m1, m2}
}

func (tm *TwoMin) build(arr []int, node, s, e int) {
	if s == e {
		tm.tree[node] = [2]int{arr[s], math.MaxInt64}
		return
	}
	mid := (s + e) / 2
	tm.build(arr, 2*node, s, mid)
	tm.build(arr, 2*node+1, mid+1, e)
	tm.tree[node] = merge(tm.tree[2*node], tm.tree[2*node+1])
}

func (tm *TwoMin) query(node, s, e, l, r int) [2]int {
	if r < s || e < l { return [2]int{math.MaxInt64, math.MaxInt64} }
	if l <= s && e <= r { return tm.tree[node] }
	mid := (s + e) / 2
	return merge(tm.query(2*node, s, mid, l, r), tm.query(2*node+1, mid+1, e, l, r))
}

func (tm *TwoMin) Query(l, r int) (int, int) {
	result := tm.query(1, 0, tm.n-1, l, r)
	return result[0], result[1]
}

func main() {
	arr := []int{3, 1, 5, 2, 8, 4, 7, 6}
	tm := NewTwoMin(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 7}, {0, 3}, {4, 7}, {2, 5}}
	for _, q := range queries {
		m1, m2 := tm.Query(q[0], q[1])
		fmt.Printf("  Range [%d,%d]: min1=%d, min2=%d\n", q[0], q[1], m1, m2)
	}
}
```

**Textual Figure:**

```
Second Minimum Query
Array: [3, 1, 5, 2, 8, 4, 7, 6]  (n=8)

Each node stores (min1, min2):

              ┌───────────────┐
              │ [0,7] (1, 2)    │
              └──────┬────────┘
        ┌──────┴──────┐  ┌────┬────────┐
   ┌───┴──────┐    ┌──┴───────┐
   │[0,3] (1,2) │    │[4,7] (4,6)  │
   └───┬───────┘    └───┬───────┘

Merge of (a1,a2) and (b1,b2):
  sorted = sort_unique([a1,a2,b1,b2])
  return (sorted[0], sorted[1])

  [0,3]: merge(1,3) and (2,5) → sort [1,2,3,5] → (1,2)
  [4,7]: merge(4,8) and (6,7) → sort [4,6,7,8] → (4,6)
  [0,7]: merge(1,2) and (4,6) → sort [1,2,4,6] → (1,2)

  Query [2,5]: (2, 4)   min1=2, min2=4
```

---

## Example 10: RMQ Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== RMQ Patterns ===")
	fmt.Println()

	patterns := []struct{ method, build, query, update string }{
		{"Sparse Table", "O(n log n)", "O(1)", "❌ (static)"},
		{"Segment Tree", "O(n)", "O(log n)", "O(log n)"},
		{"Sqrt Decomposition", "O(n)", "O(√n)", "O(√n)"},
		{"Deque (sliding window)", "—", "O(1) amortized", "O(1)"},
		{"Cartesian Tree", "O(n)", "via LCA O(1)", "❌"},
		{"Lazy Seg Tree", "O(n)", "O(log n)", "O(log n) range"},
		{"Persistent", "O(n)", "O(log n)", "O(log n) + version"},
		{"RMQ → LCA", "Euler tour", "O(1) via sparse", "❌"},
		{"Second min", "O(n)", "O(log n)", "O(log n)"},
		{"2D RMQ", "O(nm log n log m)", "O(1)", "❌"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-22s Build: %-14s Q: %-16s U: %s\n",
			p.method, p.build, p.query, p.update)
	}
}
```

**Textual Figure:**

```
RMQ Patterns Decision Guide:

  ┌─────────────────────┐
  │  Need updates?       │
  └──────┬─────┬────────┘
         │ NO   │ YES
  ┌──────┴──┐  ┌─┴───────────────┐
  │Sparse     │  │ Range updates?    │
  │Table      │  └─┬──────┬─────────┘
  │O(1) query │    │ NO   │ YES
  └──────────┘  ┌─┴───┐┌─┴───────┐
                  │Seg  ││Lazy Seg  │
                  │Tree ││Tree      │
                  └─────┘└─────────┘

  Special cases:
  ┌──────────────────┬─────────────────┐
  │ Sliding window    │ Use deque O(1)  │
  ├──────────────────┼─────────────────┤
  │ LCA problem       │ RMQ on Euler    │
  ├──────────────────┼─────────────────┤
  │ 2nd minimum       │ Store [m1, m2]  │
  ├──────────────────┼─────────────────┤
  │ Histogram area    │ Monotone stack  │
  └──────────────────┴─────────────────┘
```

---

## Key Takeaways

1. **Sparse Table**: O(1) query for static arrays — best when no updates needed
2. **Segment tree**: O(log n) both — best for dynamic arrays
3. **Sliding window min/max**: use deque for O(1) amortized
4. **RMQ ↔ LCA**: reducible to each other via Euler tour
5. **Sqrt decomposition**: simpler to implement, good for competitions

> **Next up:** Lazy Propagation →
