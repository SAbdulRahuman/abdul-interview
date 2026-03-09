# Phase 23: Design Data Structures — Lazy Propagation Patterns

## Overview

Beyond basic segment tree lazy propagation, there are **design patterns** for handling complex operations, composing multiple lazy types, and choosing the right approach.

| Pattern | Description |
|---------|------------|
| **Tag composition** | Combine two pending operations into one |
| **Beats** | Segment tree beats for complex conditional updates |
| **Chtholly Tree** | Interval assign with set-based approach |
| **Sqrt decomposition** | Block-level lazy for simpler implementation |

---

## Example 1: Lazy Tag Composition Rules

```go
package main

import "fmt"

// Demonstration of how lazy tags compose
// Operations: add(v), set(v), multiply(v)

type LazyTag struct {
	hasSet bool
	setVal int64
	addVal int64
	mulVal int64
}

func identity() LazyTag {
	return LazyTag{mulVal: 1}
}

func compose(existing, new_ LazyTag) LazyTag {
	result := existing

	if new_.hasSet {
		// SET overrides everything
		result.hasSet = true
		result.setVal = new_.setVal
		result.addVal = new_.addVal // add applied after set
		result.mulVal = new_.mulVal // mul applied after set
		return result
	}

	// MULTIPLY: applies to both set value and pending add
	if new_.mulVal != 1 {
		if result.hasSet {
			result.setVal *= new_.mulVal
		}
		result.addVal *= new_.mulVal
		result.mulVal *= new_.mulVal
	}

	// ADD: just accumulates
	result.addVal += new_.addVal

	return result
}

func main() {
	fmt.Println("=== Lazy Tag Composition Rules ===\n")

	// Scenario: start with value 5
	// Op1: add(3) → 8
	// Op2: mul(2) → 16
	// Op3: add(1) → 17
	// Op4: set(10) → 10

	tag := identity()
	fmt.Printf("Start: %+v\n", tag)

	tag = compose(tag, LazyTag{addVal: 3, mulVal: 1})
	fmt.Printf("After add(3): %+v\n", tag)

	tag = compose(tag, LazyTag{mulVal: 2})
	fmt.Printf("After mul(2): %+v\n", tag)

	tag = compose(tag, LazyTag{addVal: 1, mulVal: 1})
	fmt.Printf("After add(1): %+v\n", tag)

	tag = compose(tag, LazyTag{hasSet: true, setVal: 10, mulVal: 1})
	fmt.Printf("After set(10): %+v\n", tag)

	// Apply tag to value 5: 
	// Since hasSet, result = setVal * mulVal + addVal = 10 * 1 + 0 = 10
	fmt.Println("\nComposition order matters!")
	fmt.Println("SET resets; MUL scales existing; ADD accumulates")
}
```

---

## Example 2: Segment Tree Beats (Range Chmin)

```go
package main

import (
	"fmt"
	"math"
)

// Ji Driver Segment Tree (Segment Tree Beats)
// Supports: range chmin(x), range sum query

type BeatsNode struct {
	sum      int64
	mx       int  // maximum
	cnt      int  // count of maximum
	secondMx int  // strict second maximum
}

type BeatsTree struct {
	tree []BeatsNode
	n    int
}

func NewBeats(arr []int) *BeatsTree {
	n := len(arr)
	bt := &BeatsTree{tree: make([]BeatsNode, 4*n), n: n}
	bt.build(arr, 1, 0, n-1)
	return bt
}

func (bt *BeatsTree) merge(l, r BeatsNode) BeatsNode {
	node := BeatsNode{sum: l.sum + r.sum}
	if l.mx == r.mx {
		node.mx = l.mx
		node.cnt = l.cnt + r.cnt
		node.secondMx = max(l.secondMx, r.secondMx)
	} else if l.mx > r.mx {
		node.mx = l.mx
		node.cnt = l.cnt
		node.secondMx = max(l.secondMx, r.mx)
	} else {
		node.mx = r.mx
		node.cnt = r.cnt
		node.secondMx = max(l.mx, r.secondMx)
	}
	return node
}

func (bt *BeatsTree) build(arr []int, node, s, e int) {
	if s == e {
		bt.tree[node] = BeatsNode{sum: int64(arr[s]), mx: arr[s], cnt: 1, secondMx: math.MinInt32}
		return
	}
	mid := (s + e) / 2
	bt.build(arr, 2*node, s, mid)
	bt.build(arr, 2*node+1, mid+1, e)
	bt.tree[node] = bt.merge(bt.tree[2*node], bt.tree[2*node+1])
}

func (bt *BeatsTree) pushDown(node int) {
	for _, c := range []int{2 * node, 2*node + 1} {
		if bt.tree[node].mx < bt.tree[c].mx {
			bt.tree[c].sum -= int64(bt.tree[c].mx-bt.tree[node].mx) * int64(bt.tree[c].cnt)
			bt.tree[c].mx = bt.tree[node].mx
		}
	}
}

func (bt *BeatsTree) chmin(node, s, e, l, r, val int) {
	if r < s || e < l || bt.tree[node].mx <= val { return }
	if l <= s && e <= r && bt.tree[node].secondMx < val {
		// Apply: only the maximums are affected
		bt.tree[node].sum -= int64(bt.tree[node].mx-val) * int64(bt.tree[node].cnt)
		bt.tree[node].mx = val
		return
	}
	bt.pushDown(node)
	mid := (s + e) / 2
	bt.chmin(2*node, s, mid, l, r, val)
	bt.chmin(2*node+1, mid+1, e, l, r, val)
	bt.tree[node] = bt.merge(bt.tree[2*node], bt.tree[2*node+1])
}

func (bt *BeatsTree) querySum(node, s, e, l, r int) int64 {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return bt.tree[node].sum }
	bt.pushDown(node)
	mid := (s + e) / 2
	return bt.querySum(2*node, s, mid, l, r) + bt.querySum(2*node+1, mid+1, e, l, r)
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	arr := []int{5, 3, 7, 2, 6, 4, 8, 1}
	bt := NewBeats(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Sum [0,7] = %d\n", bt.querySum(1, 0, bt.n-1, 0, 7))

	bt.chmin(1, 0, bt.n-1, 0, 7, 5) // clamp all to min(val, 5)
	fmt.Printf("\nAfter chmin(5) on [0,7]:\n")
	fmt.Printf("Sum [0,7] = %d\n", bt.querySum(1, 0, bt.n-1, 0, 7))
	// arr becomes [5,3,5,2,5,4,5,1] → sum=30

	fmt.Println("\nSegment Tree Beats: O(n log²n) amortized for chmin+sum")
}
```

---

## Example 3: Chtholly Tree (Interval Assign)

```go
package main

import (
	"fmt"
	"sort"
)

// ODT (Old Driver Tree) / Chtholly Tree
// Efficient for "assign value to range" + queries
// Works well when assigns are frequent (reduces interval count)

type Interval struct {
	l, r int
	val  int64
}

type ChthollyTree struct {
	intervals []Interval // sorted by l
}

func NewChtholly(n int, initVal int64) *ChthollyTree {
	return &ChthollyTree{intervals: []Interval{{0, n - 1, initVal}}}
}

func (ct *ChthollyTree) findIdx(pos int) int {
	// Find interval containing pos
	lo, hi := 0, len(ct.intervals)-1
	for lo <= hi {
		mid := (lo + hi) / 2
		if ct.intervals[mid].r < pos { lo = mid + 1 } else if ct.intervals[mid].l > pos { hi = mid - 1 } else { return mid }
	}
	return -1
}

func (ct *ChthollyTree) Assign(l, r int, val int64) {
	// Remove all intervals overlapping [l,r], add new one
	newIntervals := []Interval{}
	for _, iv := range ct.intervals {
		if iv.r < l || iv.l > r {
			newIntervals = append(newIntervals, iv)
		} else {
			if iv.l < l { newIntervals = append(newIntervals, Interval{iv.l, l - 1, iv.val}) }
			if iv.r > r { newIntervals = append(newIntervals, Interval{r + 1, iv.r, iv.val}) }
		}
	}
	newIntervals = append(newIntervals, Interval{l, r, val})
	sort.Slice(newIntervals, func(i, j int) bool { return newIntervals[i].l < newIntervals[j].l })
	ct.intervals = newIntervals
}

func (ct *ChthollyTree) Sum(l, r int) int64 {
	sum := int64(0)
	for _, iv := range ct.intervals {
		if iv.l > r { break }
		if iv.r < l { continue }
		lo := max(iv.l, l)
		hi := min(iv.r, r)
		sum += int64(hi-lo+1) * iv.val
	}
	return sum
}

func max(a, b int) int { if a > b { return a }; return b }
func min(a, b int) int { if a < b { return a }; return b }

func main() {
	ct := NewChtholly(10, 0)

	ct.Assign(2, 5, 3)
	ct.Assign(4, 8, 7)

	fmt.Println("After assign [2,5]=3 then [4,8]=7:")
	fmt.Printf("Sum [0,9] = %d\n", ct.Sum(0, 9))
	fmt.Printf("Sum [2,5] = %d\n", ct.Sum(2, 5))
	fmt.Printf("Intervals: %v\n", ct.intervals)

	fmt.Println("\nChtholly tree: amortized efficient when assigns dominate")
}
```

---

## Example 4: Sqrt Decomposition with Block-Level Lazy

```go
package main

import (
	"fmt"
	"math"
)

type SqrtLazy struct {
	arr    []int64
	blocks []int64  // block sums
	lazy   []int64  // pending add per block
	bSize  int
	n      int
}

func NewSqrtLazy(arr []int) *SqrtLazy {
	n := len(arr)
	bSize := int(math.Sqrt(float64(n))) + 1
	nBlocks := (n + bSize - 1) / bSize
	sl := &SqrtLazy{
		arr: make([]int64, n), blocks: make([]int64, nBlocks),
		lazy: make([]int64, nBlocks), bSize: bSize, n: n,
	}
	for i, v := range arr {
		sl.arr[i] = int64(v)
		sl.blocks[i/bSize] += int64(v)
	}
	return sl
}

func (sl *SqrtLazy) RangeAdd(l, r int, val int64) {
	bl, br := l/sl.bSize, r/sl.bSize
	if bl == br {
		for i := l; i <= r; i++ {
			sl.arr[i] += val
			sl.blocks[bl] += val
		}
		return
	}
	// Partial left block
	for i := l; i < (bl+1)*sl.bSize; i++ {
		sl.arr[i] += val
		sl.blocks[bl] += val
	}
	// Full middle blocks
	for b := bl + 1; b < br; b++ {
		sl.lazy[b] += val
		sl.blocks[b] += val * int64(sl.bSize)
	}
	// Partial right block
	end := min64(int64((br+1)*sl.bSize), int64(sl.n))
	for i := br * sl.bSize; int64(i) < end && i <= r; i++ {
		sl.arr[i] += val
		sl.blocks[br] += val
	}
}

func (sl *SqrtLazy) RangeSum(l, r int) int64 {
	sum := int64(0)
	bl, br := l/sl.bSize, r/sl.bSize
	if bl == br {
		for i := l; i <= r; i++ { sum += sl.arr[i] + sl.lazy[bl] }
		return sum
	}
	for i := l; i < (bl+1)*sl.bSize; i++ { sum += sl.arr[i] + sl.lazy[bl] }
	for b := bl + 1; b < br; b++ { sum += sl.blocks[b] }
	end := min64(int64((br+1)*sl.bSize), int64(sl.n))
	for i := br * sl.bSize; int64(i) < end && i <= r; i++ { sum += sl.arr[i] + sl.lazy[br] }
	return sum
}

func min64(a, b int64) int64 { if a < b { return a }; return b }

func main() {
	arr := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	sl := NewSqrtLazy(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Sum [0,9] = %d\n", sl.RangeSum(0, 9))

	sl.RangeAdd(2, 7, 5)
	fmt.Printf("\nAfter adding 5 to [2,7]:\n")
	fmt.Printf("Sum [0,9] = %d\n", sl.RangeSum(0, 9))

	fmt.Println("\nSqrt decomposition: simpler than segment tree lazy")
}
```

---

## Example 5: Range Add + Range Max Count

```go
package main

import (
	"fmt"
	"math"
)

// Track: how many elements equal the maximum in a range
// Support: range add, range max+count query

type MaxCntSeg struct {
	tree [][2]int64 // [max, count]
	lazy []int64
	n    int
}

func NewMaxCntSeg(arr []int) *MaxCntSeg {
	n := len(arr)
	ms := &MaxCntSeg{tree: make([][2]int64, 4*n), lazy: make([]int64, 4*n), n: n}
	ms.build(arr, 1, 0, n-1)
	return ms
}

func mergeMaxCnt(a, b [2]int64) [2]int64 {
	if a[0] > b[0] { return a }
	if b[0] > a[0] { return b }
	return [2]int64{a[0], a[1] + b[1]}
}

func (ms *MaxCntSeg) build(arr []int, node, s, e int) {
	if s == e { ms.tree[node] = [2]int64{int64(arr[s]), 1}; return }
	mid := (s + e) / 2
	ms.build(arr, 2*node, s, mid)
	ms.build(arr, 2*node+1, mid+1, e)
	ms.tree[node] = mergeMaxCnt(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MaxCntSeg) push(node int) {
	if ms.lazy[node] != 0 {
		for _, c := range []int{2 * node, 2*node + 1} {
			ms.tree[c][0] += ms.lazy[node]
			ms.lazy[c] += ms.lazy[node]
		}
		ms.lazy[node] = 0
	}
}

func (ms *MaxCntSeg) update(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		ms.tree[node][0] += val
		ms.lazy[node] += val
		return
	}
	ms.push(node)
	mid := (s + e) / 2
	ms.update(2*node, s, mid, l, r, val)
	ms.update(2*node+1, mid+1, e, l, r, val)
	ms.tree[node] = mergeMaxCnt(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MaxCntSeg) query(node, s, e, l, r int) [2]int64 {
	if r < s || e < l { return [2]int64{math.MinInt64, 0} }
	if l <= s && e <= r { return ms.tree[node] }
	ms.push(node)
	mid := (s + e) / 2
	return mergeMaxCnt(ms.query(2*node, s, mid, l, r), ms.query(2*node+1, mid+1, e, l, r))
}

func main() {
	arr := []int{3, 1, 4, 1, 5, 9, 2, 6}
	ms := NewMaxCntSeg(arr)

	result := ms.query(1, 0, ms.n-1, 0, 7)
	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Max [0,7] = %d (count=%d)\n", result[0], result[1])

	ms.update(1, 0, ms.n-1, 0, 3, 6) // add 6 to [0,3]
	result = ms.query(1, 0, ms.n-1, 0, 7)
	fmt.Printf("\nAfter add 6 to [0,3]:\n")
	fmt.Printf("Max [0,7] = %d (count=%d)\n", result[0], result[1])
}
```

---

## Example 6: Range Update with History (Persistent Lazy)

```go
package main

import "fmt"

// Persistent segment tree with lazy — each update creates new version

type PNode struct {
	left, right *PNode
	sum         int64
	lazy        int64
}

func build(arr []int, lo, hi int) *PNode {
	node := &PNode{}
	if lo == hi { node.sum = int64(arr[lo]); return node }
	mid := (lo + hi) / 2
	node.left = build(arr, lo, mid)
	node.right = build(arr, mid+1, hi)
	node.sum = node.left.sum + node.right.sum
	return node
}

func pushDown(node *PNode, lo, hi int) (*PNode, *PNode) {
	mid := (lo + hi) / 2
	left := &PNode{left: node.left.left, right: node.left.right, sum: node.left.sum, lazy: node.left.lazy}
	right := &PNode{left: node.right.left, right: node.right.right, sum: node.right.sum, lazy: node.right.lazy}
	if node.lazy != 0 {
		left.sum += node.lazy * int64(mid-lo+1)
		left.lazy += node.lazy
		right.sum += node.lazy * int64(hi-mid)
		right.lazy += node.lazy
	}
	return left, right
}

func update(node *PNode, lo, hi, l, r int, val int64) *PNode {
	if r < lo || hi < l { return node }
	newNode := &PNode{sum: node.sum, lazy: node.lazy, left: node.left, right: node.right}
	if l <= lo && hi <= r {
		newNode.sum += val * int64(hi-lo+1)
		newNode.lazy += val
		return newNode
	}
	mid := (lo + hi) / 2
	left, right := pushDown(newNode, lo, hi)
	newNode.lazy = 0
	newNode.left = update(left, lo, mid, l, r, val)
	newNode.right = update(right, mid+1, hi, l, r, val)
	newNode.sum = newNode.left.sum + newNode.right.sum
	return newNode
}

func query(node *PNode, lo, hi, l, r int) int64 {
	if node == nil || r < lo || hi < l { return 0 }
	if l <= lo && hi <= r { return node.sum }
	mid := (lo + hi) / 2
	left, right := pushDown(node, lo, hi)
	return query(left, lo, mid, l, r) + query(right, mid+1, hi, l, r)
}

func main() {
	arr := []int{1, 2, 3, 4, 5}
	n := len(arr)
	versions := []*PNode{build(arr, 0, n-1)}

	// Version 1: add 10 to [1,3]
	versions = append(versions, update(versions[0], 0, n-1, 1, 3, 10))
	// Version 2: add 5 to [0,4]
	versions = append(versions, update(versions[1], 0, n-1, 0, 4, 5))

	for i, v := range versions {
		fmt.Printf("Version %d: sum [0,4] = %d\n", i, query(v, 0, n-1, 0, 4))
	}

	// Can still query old versions!
	fmt.Printf("\nVersion 0, sum [1,3] = %d\n", query(versions[0], 0, n-1, 1, 3))
	fmt.Printf("Version 2, sum [1,3] = %d\n", query(versions[2], 0, n-1, 1, 3))
}
```

---

## Example 7: Range Color Flip with Interval Tracking

```go
package main

import "fmt"

// Track number of distinct color segments after range flips
// Array of 0s and 1s, flip range [l,r], count segments

type FlipSeg struct {
	tree [][4]int // [count_segments, left_val, right_val, size]
	lazy []bool
	n    int
}

func NewFlipSeg(arr []int) *FlipSeg {
	n := len(arr)
	fs := &FlipSeg{tree: make([][4]int, 4*n), lazy: make([]bool, 4*n), n: n}
	fs.build(arr, 1, 0, n-1)
	return fs
}

func (fs *FlipSeg) makeLeaf(val int) [4]int {
	return [4]int{1, val, val, 1}
}

func (fs *FlipSeg) merge(l, r [4]int) [4]int {
	if l[3] == 0 { return r }
	if r[3] == 0 { return l }
	segs := l[0] + r[0]
	if l[2] == r[1] { segs-- } // merge adjacent same-color segments
	return [4]int{segs, l[1], r[2], l[3] + r[3]}
}

func (fs *FlipSeg) flipNode(node int) {
	fs.tree[node][1] ^= 1
	fs.tree[node][2] ^= 1
	fs.lazy[node] = !fs.lazy[node]
}

func (fs *FlipSeg) build(arr []int, node, s, e int) {
	if s == e { fs.tree[node] = fs.makeLeaf(arr[s]); return }
	mid := (s + e) / 2
	fs.build(arr, 2*node, s, mid)
	fs.build(arr, 2*node+1, mid+1, e)
	fs.tree[node] = fs.merge(fs.tree[2*node], fs.tree[2*node+1])
}

func (fs *FlipSeg) push(node int) {
	if fs.lazy[node] {
		fs.flipNode(2 * node)
		fs.flipNode(2*node + 1)
		fs.lazy[node] = false
	}
}

func (fs *FlipSeg) flip(node, s, e, l, r int) {
	if r < s || e < l { return }
	if l <= s && e <= r { fs.flipNode(node); return }
	fs.push(node)
	mid := (s + e) / 2
	fs.flip(2*node, s, mid, l, r)
	fs.flip(2*node+1, mid+1, e, l, r)
	fs.tree[node] = fs.merge(fs.tree[2*node], fs.tree[2*node+1])
}

func (fs *FlipSeg) countSegments() int { return fs.tree[1][0] }

func main() {
	arr := []int{0, 0, 1, 1, 0, 1, 1, 0}
	fs := NewFlipSeg(arr)

	fmt.Printf("Array: %v → segments=%d\n", arr, fs.countSegments())

	fs.flip(1, 0, fs.n-1, 2, 5)
	fmt.Printf("After flip [2,5]: segments=%d\n", fs.countSegments())

	fs.flip(1, 0, fs.n-1, 0, 7)
	fmt.Printf("After flip [0,7]: segments=%d\n", fs.countSegments())
}
```

---

## Example 8: Range GCD with Point Update

```go
package main

import "fmt"

// GCD is not easily compatible with lazy range updates
// But point update + range GCD query works perfectly

type GCDTree struct {
	tree []int
	n    int
}

func gcd(a, b int) int {
	if a < 0 { a = -a }
	if b < 0 { b = -b }
	for b != 0 { a, b = b, a%b }
	return a
}

func NewGCDTree(arr []int) *GCDTree {
	n := len(arr)
	gt := &GCDTree{tree: make([]int, 4*n), n: n}
	gt.build(arr, 1, 0, n-1)
	return gt
}

func (gt *GCDTree) build(arr []int, node, s, e int) {
	if s == e { gt.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	gt.build(arr, 2*node, s, mid)
	gt.build(arr, 2*node+1, mid+1, e)
	gt.tree[node] = gcd(gt.tree[2*node], gt.tree[2*node+1])
}

func (gt *GCDTree) update(node, s, e, idx, val int) {
	if s == e { gt.tree[node] = val; return }
	mid := (s + e) / 2
	if idx <= mid { gt.update(2*node, s, mid, idx, val) } else { gt.update(2*node+1, mid+1, e, idx, val) }
	gt.tree[node] = gcd(gt.tree[2*node], gt.tree[2*node+1])
}

func (gt *GCDTree) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return gt.tree[node] }
	mid := (s + e) / 2
	return gcd(gt.query(2*node, s, mid, l, r), gt.query(2*node+1, mid+1, e, l, r))
}

func main() {
	arr := []int{12, 18, 24, 36, 48, 60}
	gt := NewGCDTree(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 5}, {0, 2}, {3, 5}, {1, 4}}
	for _, q := range queries {
		fmt.Printf("  GCD [%d,%d] = %d\n", q[0], q[1], gt.query(1, 0, gt.n-1, q[0], q[1]))
	}

	gt.update(1, 0, gt.n-1, 2, 15)
	fmt.Printf("\nAfter arr[2]=15: GCD [0,5] = %d\n", gt.query(1, 0, gt.n-1, 0, 5))

	fmt.Println("\nNote: Range add + range GCD is hard!")
	fmt.Println("Use: diff array + GCD tree for range add + range GCD")
}
```

---

## Example 9: Lazy with Custom Monoid (Matrix Product)

```go
package main

import "fmt"

type Matrix [2][2]int64

const MOD = 1_000_000_007

func matMul(a, b Matrix) Matrix {
	var c Matrix
	for i := 0; i < 2; i++ {
		for j := 0; j < 2; j++ {
			for k := 0; k < 2; k++ {
				c[i][j] = (c[i][j] + a[i][k]*b[k][j]) % MOD
			}
		}
	}
	return c
}

var identity = Matrix{{1, 0}, {0, 1}}

type MatSeg struct {
	tree []Matrix
	n    int
}

func NewMatSeg(mats []Matrix) *MatSeg {
	n := len(mats)
	ms := &MatSeg{tree: make([]Matrix, 4*n), n: n}
	ms.build(mats, 1, 0, n-1)
	return ms
}

func (ms *MatSeg) build(mats []Matrix, node, s, e int) {
	if s == e { ms.tree[node] = mats[s]; return }
	mid := (s + e) / 2
	ms.build(mats, 2*node, s, mid)
	ms.build(mats, 2*node+1, mid+1, e)
	ms.tree[node] = matMul(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MatSeg) update(node, s, e, idx int, mat Matrix) {
	if s == e { ms.tree[node] = mat; return }
	mid := (s + e) / 2
	if idx <= mid { ms.update(2*node, s, mid, idx, mat) } else { ms.update(2*node+1, mid+1, e, idx, mat) }
	ms.tree[node] = matMul(ms.tree[2*node], ms.tree[2*node+1])
}

func (ms *MatSeg) query(node, s, e, l, r int) Matrix {
	if r < s || e < l { return identity }
	if l <= s && e <= r { return ms.tree[node] }
	mid := (s + e) / 2
	return matMul(ms.query(2*node, s, mid, l, r), ms.query(2*node+1, mid+1, e, l, r))
}

func main() {
	// Fibonacci matrices
	fib := Matrix{{1, 1}, {1, 0}}
	mats := []Matrix{fib, fib, fib, fib, fib}
	ms := NewMatSeg(mats)

	// Product of 5 fib matrices = fib(5)
	result := ms.query(1, 0, ms.n-1, 0, 4)
	fmt.Printf("Product of 5 Fibonacci matrices:\n")
	fmt.Printf("  [%d %d]\n  [%d %d]\n", result[0][0], result[0][1], result[1][0], result[1][1])
	fmt.Printf("  F(6)=%d, F(5)=%d\n", result[0][0], result[0][1])

	fmt.Println("\nSegment tree on non-commutative monoids (matrix product)")
	fmt.Println("Useful for range linear recurrence queries")
}
```

---

## Example 10: Lazy Propagation Design Checklist

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Lazy Propagation Design Checklist ===")
	fmt.Println()

	fmt.Println("1. DEFINE YOUR MONOID")
	fmt.Println("   - What does each node store? (sum, max, count, matrix, ...)")
	fmt.Println("   - merge(left, right) = ?\n")

	fmt.Println("2. DEFINE YOUR LAZY TAG")
	fmt.Println("   - What pending operation? (add, set, multiply, flip, ...)")
	fmt.Println("   - What is the identity tag? (0 for add, 1 for mul, ...)\n")

	fmt.Println("3. DEFINE TAG APPLICATION")
	fmt.Println("   - apply(node, tag, range_size) → updated node")
	fmt.Println("   - How does the tag modify the monoid value?\n")

	fmt.Println("4. DEFINE TAG COMPOSITION")
	fmt.Println("   - compose(old_tag, new_tag) → combined_tag")
	fmt.Println("   - Must be associative!")
	fmt.Println("   - SET overrides; ADD accumulates; MUL scales\n")

	fmt.Println("5. VERIFY PROPERTIES")
	fmt.Println("   - merge is associative")
	fmt.Println("   - apply distributes over merge")
	fmt.Println("   - compose is associative")
	fmt.Println("   - identity tag doesn't change node\n")

	compatible := []struct{ update, query, works string }{
		{"add", "sum", "✅ tree += val*(r-l+1)"},
		{"add", "min/max", "✅ tree += val"},
		{"set", "sum", "✅ tree = val*(r-l+1)"},
		{"set", "min/max", "✅ tree = val"},
		{"mul+add", "sum", "✅ composition: (m2*m1, m2*a1+a2)"},
		{"flip", "count 1s", "✅ tree = (r-l+1) - tree"},
		{"add", "gcd", "❌ use diff array trick"},
		{"set", "gcd", "✅ tree = val"},
	}

	fmt.Println("Compatible Update+Query pairs:")
	for _, c := range compatible {
		fmt.Printf("  %-10s + %-10s %s\n", c.update, c.query, c.works)
	}
}
```

---

## Key Takeaways

1. **Tag composition is the hard part**: ensure `compose(old, new)` is correct and associative
2. **SET overrides everything**: clears pending ADD/MUL
3. **Segment Tree Beats**: handles conditional updates (chmin/chmax) with auxiliary info
4. **Chtholly Tree**: simple for interval assign when assigns are frequent
5. **Sqrt decomposition**: easier to implement lazy at block level
6. **Not all combinations work**: ADD + GCD needs diff array trick
7. **Persistent lazy**: creates new nodes, needs careful pushdown

> **Next up:** Composite Data Structure Design →
