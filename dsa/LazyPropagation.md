# Phase 22: Segment Tree & Fenwick Tree — Lazy Propagation

## Overview

**Lazy Propagation** defers updates to segment tree nodes, applying them only when needed. This enables **range updates** in O(log n) instead of O(n).

**Key Idea**: Instead of updating all leaves, store pending updates at internal nodes and "push down" when we need to query or update children.

| Operation | Without Lazy | With Lazy |
|-----------|-------------|-----------|
| Point update | O(log n) | O(log n) |
| Range update | O(n log n) | O(log n) |
| Range query | O(log n) | O(log n) |

---

## Example 1: Range Add + Range Sum Query

```go
package main

import "fmt"

type LazySegTree struct {
	tree, lazy []int64
	n          int
}

func NewLazySegTree(arr []int) *LazySegTree {
	n := len(arr)
	st := &LazySegTree{tree: make([]int64, 4*n), lazy: make([]int64, 4*n), n: n}
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *LazySegTree) build(arr []int, node, s, e int) {
	if s == e { st.tree[node] = int64(arr[s]); return }
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *LazySegTree) push(node, s, e int) {
	if st.lazy[node] != 0 {
		mid := (s + e) / 2
		st.apply(2*node, s, mid, st.lazy[node])
		st.apply(2*node+1, mid+1, e, st.lazy[node])
		st.lazy[node] = 0
	}
}

func (st *LazySegTree) apply(node, s, e int, val int64) {
	st.tree[node] += val * int64(e-s+1)
	st.lazy[node] += val
}

func (st *LazySegTree) update(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		st.apply(node, s, e, val)
		return
	}
	st.push(node, s, e)
	mid := (s + e) / 2
	st.update(2*node, s, mid, l, r, val)
	st.update(2*node+1, mid+1, e, l, r, val)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *LazySegTree) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	st.push(node, s, e)
	mid := (s + e) / 2
	return st.query(2*node, s, mid, l, r) + st.query(2*node+1, mid+1, e, l, r)
}

func (st *LazySegTree) RangeAdd(l, r int, val int64) { st.update(1, 0, st.n-1, l, r, val) }
func (st *LazySegTree) RangeSum(l, r int) int64       { return st.query(1, 0, st.n-1, l, r) }

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	st := NewLazySegTree(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Sum [1,3] = %d\n", st.RangeSum(1, 3))

	st.RangeAdd(1, 5, 10)
	fmt.Printf("\nAfter adding 10 to [1,5]:\n")
	fmt.Printf("Sum [1,3] = %d\n", st.RangeSum(1, 3))
	fmt.Printf("Sum [0,5] = %d\n", st.RangeSum(0, 5))
}
```

**Textual Figure:**

```
Array: [1, 3, 5, 7, 9, 11]   (n=6)

Lazy Segment Tree (Range Add + Range Sum):

After build:
              ┌──────────────────────┐
              │ [0,5] sum=36 lazy=0  │
              └──────────┬───────────┘
          ┌────────┴───────┐
  ┌───────┴───────┐ ┌───────┴───────┐
  │ [0,2] sum=9 L=0 │ │ [3,5] sum=27 L=0│
  └────────────────┘ └────────────────┘

Query Sum [1,3] = 15  (3+5+7)

RangeAdd(1, 5, +10):  add 10 to positions [1..5]
  Node [0,5]: partially covered, push down
  Node [0,2]: partially covered
    Node [0,0]: not covered, skip
    Node [1,2]: fully covered! → sum += 10×2=20, lazy=10
  Node [3,5]: fully covered! → sum += 10×3=30, lazy=10

After RangeAdd:
              ┌───────────────────────┐
              │ [0,5] sum=86  lazy=0  │
              └──────────┬────────────┘
          ┌────────┴───────┐
  ┌───────┴────────┐ ┌───────┴────────┐
  │[0,2] sum=29 L=0 │ │[3,5] sum=57 L=10│
  └─────────────────┘ └─────────────────┘
       │                    │
  [0,0]=1  [1,2]=28 L=10    [3,5] lazy=10 (deferred)

  Sum [1,3] = 45   (13+15+17)
  Sum [0,5] = 86   (1+13+15+17+19+21)
```

---

## Example 2: Range Set + Range Sum Query

```go
package main

import (
	"fmt"
	"math"
)

const NONE = math.MinInt64

type SetSegTree struct {
	tree, lazy []int64
	n          int
}

func NewSetSegTree(arr []int) *SetSegTree {
	n := len(arr)
	st := &SetSegTree{tree: make([]int64, 4*n), lazy: make([]int64, 4*n), n: n}
	for i := range st.lazy { st.lazy[i] = NONE }
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *SetSegTree) build(arr []int, node, s, e int) {
	st.lazy[node] = NONE
	if s == e { st.tree[node] = int64(arr[s]); return }
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *SetSegTree) push(node, s, e int) {
	if st.lazy[node] != NONE {
		mid := (s + e) / 2
		st.applyNode(2*node, s, mid, st.lazy[node])
		st.applyNode(2*node+1, mid+1, e, st.lazy[node])
		st.lazy[node] = NONE
	}
}

func (st *SetSegTree) applyNode(node, s, e int, val int64) {
	st.tree[node] = val * int64(e-s+1)
	st.lazy[node] = val
}

func (st *SetSegTree) update(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		st.applyNode(node, s, e, val)
		return
	}
	st.push(node, s, e)
	mid := (s + e) / 2
	st.update(2*node, s, mid, l, r, val)
	st.update(2*node+1, mid+1, e, l, r, val)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *SetSegTree) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	st.push(node, s, e)
	mid := (s + e) / 2
	return st.query(2*node, s, mid, l, r) + st.query(2*node+1, mid+1, e, l, r)
}

func (st *SetSegTree) RangeSet(l, r int, val int64) { st.update(1, 0, st.n-1, l, r, val) }
func (st *SetSegTree) RangeSum(l, r int) int64       { return st.query(1, 0, st.n-1, l, r) }

func main() {
	arr := []int{1, 2, 3, 4, 5}
	st := NewSetSegTree(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Sum [0,4] = %d\n", st.RangeSum(0, 4))

	st.RangeSet(1, 3, 10) // set arr[1..3] = 10
	fmt.Printf("\nAfter setting [1,3] to 10:\n")
	fmt.Printf("Sum [0,4] = %d → 1+10+10+10+5=%d\n", st.RangeSum(0, 4), 36)
}
```

**Textual Figure:**

```
Array: [1, 2, 3, 4, 5]   (n=5)

Lazy Segment Tree (Range Set + Range Sum):

After build:  sum[0,4] = 15

              ┌───────────────────────┐
              │ [0,4] sum=15 lazy=NONE│
              └──────────┬────────────┘
          ┌────────┴───────┐
  ┌───────┴───────┐ ┌───────┴───────┐
  │ [0,2] sum=6     │ │ [3,4] sum=9     │
  └────────────────┘ └───────────────┘

RangeSet(1, 3, 10):  set arr[1..3] = 10
  Node [0,2]: partial overlap→push→recurse
    Node [1,2]: fully covered → sum=10×2=20, lazy=10
  Node [3,4]: partial overlap→push→recurse
    Node [3,3]: fully covered → sum=10×1=10, lazy=10
    Node [4,4]: not covered

After RangeSet:
  arr conceptually = [1, 10, 10, 10, 5]
              ┌───────────────────────┐
              │ [0,4] sum=36 lazy=NONE│
              └──────────┬────────────┘
          ┌────────┴───────┐
  ┌───────┴───────┐  ┌──────┴───────┐
  │[0,2] sum=21    │  │[3,4] sum=15    │
  └────────────────┘  └───────────────┘
    │                     │
  [0,0]=1 [1,2] L=10    [3,3] L=10  [4,4]=5
             sum=20         sum=10

SUM [0,4] = 1+10+10+10+5 = 36 ✓
```

---

## Example 3: Combined Add + Set Lazy (Two Layer Lazy)

```go
package main

import (
	"fmt"
	"math"
)

const NO_SET = math.MinInt64

type DualLazy struct {
	tree    []int64
	lazyAdd []int64
	lazySet []int64
	n       int
}

func NewDualLazy(arr []int) *DualLazy {
	n := len(arr)
	dl := &DualLazy{
		tree: make([]int64, 4*n), lazyAdd: make([]int64, 4*n),
		lazySet: make([]int64, 4*n), n: n,
	}
	for i := range dl.lazySet { dl.lazySet[i] = NO_SET }
	dl.build(arr, 1, 0, n-1)
	return dl
}

func (dl *DualLazy) build(arr []int, node, s, e int) {
	dl.lazySet[node] = NO_SET
	if s == e { dl.tree[node] = int64(arr[s]); return }
	mid := (s + e) / 2
	dl.build(arr, 2*node, s, mid)
	dl.build(arr, 2*node+1, mid+1, e)
	dl.tree[node] = dl.tree[2*node] + dl.tree[2*node+1]
}

func (dl *DualLazy) applySet(node, s, e int, val int64) {
	dl.tree[node] = val * int64(e-s+1)
	dl.lazySet[node] = val
	dl.lazyAdd[node] = 0
}

func (dl *DualLazy) applyAdd(node, s, e int, val int64) {
	dl.tree[node] += val * int64(e-s+1)
	dl.lazyAdd[node] += val
}

func (dl *DualLazy) push(node, s, e int) {
	if dl.lazySet[node] != NO_SET {
		mid := (s + e) / 2
		dl.applySet(2*node, s, mid, dl.lazySet[node])
		dl.applySet(2*node+1, mid+1, e, dl.lazySet[node])
		dl.lazySet[node] = NO_SET
	}
	if dl.lazyAdd[node] != 0 {
		mid := (s + e) / 2
		dl.applyAdd(2*node, s, mid, dl.lazyAdd[node])
		dl.applyAdd(2*node+1, mid+1, e, dl.lazyAdd[node])
		dl.lazyAdd[node] = 0
	}
}

func (dl *DualLazy) update(node, s, e, l, r int, val int64, isSet bool) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		if isSet { dl.applySet(node, s, e, val) } else { dl.applyAdd(node, s, e, val) }
		return
	}
	dl.push(node, s, e)
	mid := (s + e) / 2
	dl.update(2*node, s, mid, l, r, val, isSet)
	dl.update(2*node+1, mid+1, e, l, r, val, isSet)
	dl.tree[node] = dl.tree[2*node] + dl.tree[2*node+1]
}

func (dl *DualLazy) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return dl.tree[node] }
	dl.push(node, s, e)
	mid := (s + e) / 2
	return dl.query(2*node, s, mid, l, r) + dl.query(2*node+1, mid+1, e, l, r)
}

func main() {
	arr := []int{1, 2, 3, 4, 5}
	dl := NewDualLazy(arr)

	fmt.Printf("Initial sum [0,4] = %d\n", dl.query(1, 0, dl.n-1, 0, 4))

	dl.update(1, 0, dl.n-1, 1, 3, 10, true) // SET [1,3] = 10
	fmt.Printf("After SET [1,3]=10: sum = %d\n", dl.query(1, 0, dl.n-1, 0, 4))

	dl.update(1, 0, dl.n-1, 0, 2, 5, false) // ADD 5 to [0,2]
	fmt.Printf("After ADD 5 to [0,2]: sum = %d\n", dl.query(1, 0, dl.n-1, 0, 4))
}
```

**Textual Figure:**

```
Combined SET + ADD Lazy Propagation
Array: [1, 2, 3, 4, 5]   (n=5)

Priority: SET clears ADD; ADD accumulates after SET.

Initial:  sum[0,4] = 15

Step 1: SET [1,3] = 10
  Node [1,3]: lazySet=10, lazyAdd=0
  sum[1..3] = 10×3 = 30
  arr conceptually: [1, 10, 10, 10, 5] → sum = 36

Step 2: ADD 5 to [0,2]
  Node [0,2]: need to push SET from parent first
  After push: nodes get SET applied
  Then ADD 5 to [0,2]:
    [0,0]: 1 + 5 = 6
    [1,2]: SET=10, then ADD 5 → each = 15
  arr conceptually: [6, 15, 15, 10, 5] → sum = 51

Push-down rules:
  ┌────────────────────────────────────────┐
  │ 1. Push SET first (overrides children)  │
  │ 2. Push ADD second (accumulates)        │
  │ 3. SET clears any pending ADD           │
  │ 4. New ADD after SET: add to lazyAdd    │
  └────────────────────────────────────────┘
```

---

## Example 4: Range Add + Range Min (Lazy on Min)

```go
package main

import (
	"fmt"
	"math"
)

type LazyMin struct {
	tree, lazy []int64
	n          int
}

func NewLazyMin(arr []int) *LazyMin {
	n := len(arr)
	lm := &LazyMin{tree: make([]int64, 4*n), lazy: make([]int64, 4*n), n: n}
	lm.build(arr, 1, 0, n-1)
	return lm
}

func (lm *LazyMin) build(arr []int, node, s, e int) {
	if s == e { lm.tree[node] = int64(arr[s]); return }
	mid := (s + e) / 2
	lm.build(arr, 2*node, s, mid)
	lm.build(arr, 2*node+1, mid+1, e)
	lm.tree[node] = min64(lm.tree[2*node], lm.tree[2*node+1])
}

func (lm *LazyMin) push(node int) {
	if lm.lazy[node] != 0 {
		for _, c := range []int{2 * node, 2*node + 1} {
			lm.tree[c] += lm.lazy[node]
			lm.lazy[c] += lm.lazy[node]
		}
		lm.lazy[node] = 0
	}
}

func (lm *LazyMin) update(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		lm.tree[node] += val
		lm.lazy[node] += val
		return
	}
	lm.push(node)
	mid := (s + e) / 2
	lm.update(2*node, s, mid, l, r, val)
	lm.update(2*node+1, mid+1, e, l, r, val)
	lm.tree[node] = min64(lm.tree[2*node], lm.tree[2*node+1])
}

func (lm *LazyMin) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return math.MaxInt64 }
	if l <= s && e <= r { return lm.tree[node] }
	lm.push(node)
	mid := (s + e) / 2
	return min64(lm.query(2*node, s, mid, l, r), lm.query(2*node+1, mid+1, e, l, r))
}

func min64(a, b int64) int64 { if a < b { return a }; return b }

func main() {
	arr := []int{5, 3, 7, 2, 8, 1}
	lm := NewLazyMin(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Min [0,5] = %d\n", lm.query(1, 0, lm.n-1, 0, 5))

	lm.update(1, 0, lm.n-1, 0, 3, -10) // subtract 10 from [0..3]
	fmt.Printf("\nAfter subtracting 10 from [0,3]:\n")
	fmt.Printf("Min [0,5] = %d\n", lm.query(1, 0, lm.n-1, 0, 5))
	fmt.Printf("Min [0,3] = %d\n", lm.query(1, 0, lm.n-1, 0, 3))
}
```

**Textual Figure:**

```
Array: [5, 3, 7, 2, 8, 1]   (n=6)

Lazy Segment Tree (Range Add + Range Min):

After build:
              ┌───────────────────────┐
              │ [0,5] min=1  lazy=0  │
              └──────────┬────────────┘
          ┌────────┴────────┐
  ┌───────┴───────┐  ┌──────┴───────┐
  │[0,2] min=3 L=0 │  │[3,5] min=1 L=0 │
  └────────────────┘  └───────────────┘

RangeAdd(0, 3, -10):  subtract 10 from [0..3]
  Apply -10 to covered nodes:
  Node [0,2]: fully covered → min = 3+(-10) = -7, lazy = -10
  Node [3,3]: fully covered → min = 2+(-10) = -8, lazy = -10
  Root updates: min([0,5]) = min(-7, -8, 8, 1) = -8

After update:
  arr conceptually: [-5, -7, -3, -8, 8, 1]
              ┌────────────────────────┐
              │ [0,5] min=-8  lazy=0  │
              └──────────┬─────────────┘
  [0,2] min=-7 L=-10      [3,5] min=-8 L=0
                       [3,3]=-8  [4,5] min=1

  Min [0,5] = -8    Min [0,3] = -8
```

---

## Example 5: Range Add + Range Max Query

```go
package main

import (
	"fmt"
	"math"
)

type LazyMax struct {
	tree, lazy []int64
	n          int
}

func NewLazyMax(arr []int) *LazyMax {
	n := len(arr)
	lm := &LazyMax{tree: make([]int64, 4*n), lazy: make([]int64, 4*n), n: n}
	lm.build(arr, 1, 0, n-1)
	return lm
}

func (lm *LazyMax) build(arr []int, node, s, e int) {
	if s == e { lm.tree[node] = int64(arr[s]); return }
	mid := (s + e) / 2
	lm.build(arr, 2*node, s, mid)
	lm.build(arr, 2*node+1, mid+1, e)
	lm.tree[node] = max64(lm.tree[2*node], lm.tree[2*node+1])
}

func (lm *LazyMax) push(node int) {
	if lm.lazy[node] != 0 {
		for _, c := range []int{2 * node, 2*node + 1} {
			lm.tree[c] += lm.lazy[node]
			lm.lazy[c] += lm.lazy[node]
		}
		lm.lazy[node] = 0
	}
}

func (lm *LazyMax) update(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		lm.tree[node] += val
		lm.lazy[node] += val
		return
	}
	lm.push(node)
	mid := (s + e) / 2
	lm.update(2*node, s, mid, l, r, val)
	lm.update(2*node+1, mid+1, e, l, r, val)
	lm.tree[node] = max64(lm.tree[2*node], lm.tree[2*node+1])
}

func (lm *LazyMax) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return math.MinInt64 }
	if l <= s && e <= r { return lm.tree[node] }
	lm.push(node)
	mid := (s + e) / 2
	return max64(lm.query(2*node, s, mid, l, r), lm.query(2*node+1, mid+1, e, l, r))
}

func max64(a, b int64) int64 { if a > b { return a }; return b }

func main() {
	arr := []int{5, 3, 7, 2, 8, 1}
	lm := NewLazyMax(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Max [0,5] = %d\n", lm.query(1, 0, lm.n-1, 0, 5))

	lm.update(1, 0, lm.n-1, 2, 5, 10)
	fmt.Printf("\nAfter adding 10 to [2,5]:\n")
	fmt.Printf("Max [0,5] = %d\n", lm.query(1, 0, lm.n-1, 0, 5))
	fmt.Printf("Max [0,1] = %d\n", lm.query(1, 0, lm.n-1, 0, 1))
}
```

**Textual Figure:**

```
Array: [5, 3, 7, 2, 8, 1]   (n=6)

Lazy Segment Tree (Range Add + Range Max):

After build:  max[0,5] = 8
              ┌──────────────────────┐
              │ [0,5] max=8  lazy=0 │
              └──────────┬───────────┘
          ┌────────┴───────┐
  ┌───────┴───────┐ ┌──────┴───────┐
  │[0,2] max=7 L=0│ │[3,5] max=8 L=0│
  └───────────────┘ └──────────────┘

RangeAdd(2, 5, +10):  add 10 to [2..5]
  Node [0,2]: partial → push, recurse
    [2,2]: fully covered → max = 7+10 = 17, lazy = 10
  Node [3,5]: fully covered → max = 8+10 = 18, lazy = 10

After update:
  arr conceptually: [5, 3, 17, 12, 18, 11]
              ┌───────────────────────┐
              │ [0,5] max=18  lazy=0 │
              └──────────┬────────────┘
  [0,2] max=17 L=0       [3,5] max=18 L=10
  │                       (lazy deferred)
  [0,1] max=5  [2,2]=17

  Max [0,5] = 18    Max [0,1] = 5
```

---

## Example 6: Range Multiply + Range Sum (Multiplicative Lazy)

```go
package main

import "fmt"

const MOD = 1_000_000_007

type MulSegTree struct {
	tree    []int64
	lazyMul []int64
	lazyAdd []int64
	n       int
}

func NewMulSegTree(arr []int) *MulSegTree {
	n := len(arr)
	st := &MulSegTree{
		tree: make([]int64, 4*n), lazyMul: make([]int64, 4*n),
		lazyAdd: make([]int64, 4*n), n: n,
	}
	for i := range st.lazyMul { st.lazyMul[i] = 1 }
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *MulSegTree) build(arr []int, node, s, e int) {
	st.lazyMul[node] = 1
	if s == e { st.tree[node] = int64(arr[s]) % MOD; return }
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = (st.tree[2*node] + st.tree[2*node+1]) % MOD
}

func (st *MulSegTree) apply(node, s, e int, mul, add int64) {
	st.tree[node] = (st.tree[node]*mul%MOD + add*int64(e-s+1)%MOD) % MOD
	st.lazyMul[node] = st.lazyMul[node] * mul % MOD
	st.lazyAdd[node] = (st.lazyAdd[node]*mul%MOD + add) % MOD
}

func (st *MulSegTree) push(node, s, e int) {
	if st.lazyMul[node] != 1 || st.lazyAdd[node] != 0 {
		mid := (s + e) / 2
		st.apply(2*node, s, mid, st.lazyMul[node], st.lazyAdd[node])
		st.apply(2*node+1, mid+1, e, st.lazyMul[node], st.lazyAdd[node])
		st.lazyMul[node] = 1
		st.lazyAdd[node] = 0
	}
}

func (st *MulSegTree) updateMul(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r { st.apply(node, s, e, val, 0); return }
	st.push(node, s, e)
	mid := (s + e) / 2
	st.updateMul(2*node, s, mid, l, r, val)
	st.updateMul(2*node+1, mid+1, e, l, r, val)
	st.tree[node] = (st.tree[2*node] + st.tree[2*node+1]) % MOD
}

func (st *MulSegTree) updateAdd(node, s, e, l, r int, val int64) {
	if r < s || e < l { return }
	if l <= s && e <= r { st.apply(node, s, e, 1, val); return }
	st.push(node, s, e)
	mid := (s + e) / 2
	st.updateAdd(2*node, s, mid, l, r, val)
	st.updateAdd(2*node+1, mid+1, e, l, r, val)
	st.tree[node] = (st.tree[2*node] + st.tree[2*node+1]) % MOD
}

func (st *MulSegTree) query(node, s, e, l, r int) int64 {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	st.push(node, s, e)
	mid := (s + e) / 2
	return (st.query(2*node, s, mid, l, r) + st.query(2*node+1, mid+1, e, l, r)) % MOD
}

func main() {
	arr := []int{1, 2, 3, 4, 5}
	st := NewMulSegTree(arr)

	fmt.Printf("Sum [0,4] = %d\n", st.query(1, 0, st.n-1, 0, 4))

	st.updateMul(1, 0, st.n-1, 1, 3, 3) // multiply [1,3] by 3
	fmt.Printf("After *3 on [1,3]: sum = %d\n", st.query(1, 0, st.n-1, 0, 4))

	st.updateAdd(1, 0, st.n-1, 0, 4, 1) // add 1 to all
	fmt.Printf("After +1 on [0,4]: sum = %d\n", st.query(1, 0, st.n-1, 0, 4))
}
```

**Textual Figure:**

```
Range Multiply + Range Sum with composite lazy (mul, add)
Array: [1, 2, 3, 4, 5]   sum = 15

Lazy tag = (mul, add): apply(val) = val * mul + add
Compose: (m1,a1) then (m2,a2) = (m1*m2, a1*m2 + a2)

Step 1: Multiply [1,3] by 3
  lazy = (mul=3, add=0)
  arr[1..3]: [2,3,4] → [6,9,12]
  sum = 1 + 6+9+12 + 5 = 33

  Before:  [1, 2, 3, 4, 5]  sum=15
  After:   [1, 6, 9, 12, 5] sum=33

Step 2: Add 1 to [0,4]
  lazy = (mul=1, add=1)
  Each element += 1
  sum = 33 + 5 = 38

  After:   [2, 7, 10, 13, 6] sum=38

Composite lazy at node [1,3]:
  Originally (mul=3, add=0), then (mul=1, add=1)
  Composed: (3*1, 0*1+1) = (3, 1)
  Meaning: val → val*3 + 1
```

---

## Example 7: Flip Bits in Range (XOR Lazy)

```go
package main

import "fmt"

type FlipSegTree struct {
	tree []int  // count of 1s
	lazy []bool // pending flip
	n    int
}

func NewFlipSegTree(arr []int) *FlipSegTree {
	n := len(arr)
	st := &FlipSegTree{tree: make([]int, 4*n), lazy: make([]bool, 4*n), n: n}
	st.build(arr, 1, 0, n-1)
	return st
}

func (st *FlipSegTree) build(arr []int, node, s, e int) {
	if s == e { st.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	st.build(arr, 2*node, s, mid)
	st.build(arr, 2*node+1, mid+1, e)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *FlipSegTree) push(node, s, e int) {
	if st.lazy[node] {
		mid := (s + e) / 2
		st.flipNode(2*node, s, mid)
		st.flipNode(2*node+1, mid+1, e)
		st.lazy[node] = false
	}
}

func (st *FlipSegTree) flipNode(node, s, e int) {
	st.tree[node] = (e - s + 1) - st.tree[node]
	st.lazy[node] = !st.lazy[node]
}

func (st *FlipSegTree) flip(node, s, e, l, r int) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		st.flipNode(node, s, e)
		return
	}
	st.push(node, s, e)
	mid := (s + e) / 2
	st.flip(2*node, s, mid, l, r)
	st.flip(2*node+1, mid+1, e, l, r)
	st.tree[node] = st.tree[2*node] + st.tree[2*node+1]
}

func (st *FlipSegTree) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return st.tree[node] }
	st.push(node, s, e)
	mid := (s + e) / 2
	return st.query(2*node, s, mid, l, r) + st.query(2*node+1, mid+1, e, l, r)
}

func main() {
	arr := []int{1, 0, 1, 1, 0, 0, 1, 0}
	st := NewFlipSegTree(arr)

	fmt.Printf("Array: %v\n", arr)
	fmt.Printf("Count 1s [0,7] = %d\n", st.query(1, 0, st.n-1, 0, 7))

	st.flip(1, 0, st.n-1, 2, 5)
	fmt.Printf("\nAfter flipping [2,5]:\n")
	fmt.Printf("Count 1s [0,7] = %d\n", st.query(1, 0, st.n-1, 0, 7))
	fmt.Printf("Count 1s [2,5] = %d\n", st.query(1, 0, st.n-1, 2, 5))
}
```

**Textual Figure:**

```
Bit Flip Lazy Propagation
Array: [1, 0, 1, 1, 0, 0, 1, 0]   (n=8)

tree[node] = count of 1s in range
flip(node): tree[node] = (range_size) - tree[node], lazy = !lazy

After build:
  Count 1s [0,7] = 4

          ┌───────────────────────┐
          │  [0,7] cnt=4 lazy=F   │
          └──────────┬────────────┘
      ┌───────┴───────┐  ┌──────┴───────┐
      │[0,3] cnt=3 L=F│  │[4,7] cnt=1 L=F│
      └───────────────┘  └──────────────┘

Flip [2,5]:
  arr: [1,0,1,1,0,0,1,0] → [1,0,0,0,1,1,1,0]
  [2,3]: cnt was 2 → flip → cnt = 2-2 = 0, lazy=T
  [4,5]: cnt was 0 → flip → cnt = 2-0 = 2, lazy=T

  After flip:
  Count 1s [0,7] = 1+0+0+0+1+1+1+0 = 4
  Count 1s [2,5] = 0+0+1+1 = 2

  Lazy toggles: flip ⊕ flip = no-op (self-canceling)
```

---

## Example 8: Range Set + Range Max (Painting Problem)

```go
package main

import (
	"fmt"
	"math"
)

const UNSET = math.MinInt64

type PaintMax struct {
	tree, lazy []int
	n          int
}

func NewPaintMax(n int) *PaintMax {
	pm := &PaintMax{tree: make([]int, 4*n), lazy: make([]int, 4*n), n: n}
	for i := range pm.lazy { pm.lazy[i] = UNSET }
	return pm
}

func (pm *PaintMax) push(node, s, e int) {
	if pm.lazy[node] != UNSET {
		mid := (s + e) / 2
		pm.applyNode(2*node, s, mid, pm.lazy[node])
		pm.applyNode(2*node+1, mid+1, e, pm.lazy[node])
		pm.lazy[node] = UNSET
	}
}

func (pm *PaintMax) applyNode(node, s, e, val int) {
	pm.tree[node] = val
	pm.lazy[node] = val
}

func (pm *PaintMax) update(node, s, e, l, r, val int) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		pm.applyNode(node, s, e, val)
		return
	}
	pm.push(node, s, e)
	mid := (s + e) / 2
	pm.update(2*node, s, mid, l, r, val)
	pm.update(2*node+1, mid+1, e, l, r, val)
	pm.tree[node] = max(pm.tree[2*node], pm.tree[2*node+1])
}

func (pm *PaintMax) query(node, s, e, l, r int) int {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return pm.tree[node] }
	pm.push(node, s, e)
	mid := (s + e) / 2
	return max(pm.query(2*node, s, mid, l, r), pm.query(2*node+1, mid+1, e, l, r))
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	n := 10
	pm := NewPaintMax(n)

	// Paint different ranges with heights
	pm.update(1, 0, n-1, 0, 4, 5)
	pm.update(1, 0, n-1, 3, 7, 8)
	pm.update(1, 0, n-1, 6, 9, 3)

	fmt.Printf("After painting [0,4]=5, [3,7]=8, [6,9]=3:\n")
	for i := 0; i < n; i++ {
		fmt.Printf("  pos %d = %d\n", i, pm.query(1, 0, n-1, i, i))
	}
	fmt.Printf("Max [0,9] = %d\n", pm.query(1, 0, n-1, 0, 9))
}
```

**Textual Figure:**

```
Range Set + Range Max (Painting Problem)
n=10, initially all zeros

Op 1: Paint [0,4] = 5
  pos:  5  5  5  5  5  0  0  0  0  0

Op 2: Paint [3,7] = 8  (overwrites [3,4])
  pos:  5  5  5  8  8  8  8  8  0  0

Op 3: Paint [6,9] = 3  (overwrites [6,7])
  pos:  5  5  5  8  8  8  3  3  3  3

  Position: 0  1  2  3  4  5  6  7  8  9
  Value:    5  5  5  8  8  8  3  3  3  3

  Segment tree stores max with lazy SET:
           ┌───────────────────┐
           │ [0,9] max=8       │
           └───────┬───────────┘
      ┌─────┴─────┐      ┌────┴─────┐
      │[0,4] max=8│      │[5,9] max=8│
      └───────────┘      └───────────┘

  Max [0,9] = 8
  Last-write-wins: each SET overrides previous lazy
```

---

## Example 9: Count Distinct Colors in Range (Lazy + Bitmask)

```go
package main

import (
	"fmt"
	"math/bits"
)

// Each element has a color (1-based, up to 30 colors)
// Query: count distinct colors in range [l,r]
// Update: paint range [l,r] with color c

type ColorSeg struct {
	tree []uint32 // bitmask of colors
	lazy []int    // -1 or color to paint
	n    int
}

func NewColorSeg(colors []int) *ColorSeg {
	n := len(colors)
	cs := &ColorSeg{tree: make([]uint32, 4*n), lazy: make([]int, 4*n), n: n}
	for i := range cs.lazy { cs.lazy[i] = -1 }
	cs.build(colors, 1, 0, n-1)
	return cs
}

func (cs *ColorSeg) build(colors []int, node, s, e int) {
	cs.lazy[node] = -1
	if s == e {
		cs.tree[node] = 1 << uint(colors[s])
		return
	}
	mid := (s + e) / 2
	cs.build(colors, 2*node, s, mid)
	cs.build(colors, 2*node+1, mid+1, e)
	cs.tree[node] = cs.tree[2*node] | cs.tree[2*node+1]
}

func (cs *ColorSeg) push(node int) {
	if cs.lazy[node] != -1 {
		c := cs.lazy[node]
		for _, child := range []int{2 * node, 2*node + 1} {
			cs.tree[child] = 1 << uint(c)
			cs.lazy[child] = c
		}
		cs.lazy[node] = -1
	}
}

func (cs *ColorSeg) paint(node, s, e, l, r, c int) {
	if r < s || e < l { return }
	if l <= s && e <= r {
		cs.tree[node] = 1 << uint(c)
		cs.lazy[node] = c
		return
	}
	cs.push(node)
	mid := (s + e) / 2
	cs.paint(2*node, s, mid, l, r, c)
	cs.paint(2*node+1, mid+1, e, l, r, c)
	cs.tree[node] = cs.tree[2*node] | cs.tree[2*node+1]
}

func (cs *ColorSeg) query(node, s, e, l, r int) uint32 {
	if r < s || e < l { return 0 }
	if l <= s && e <= r { return cs.tree[node] }
	cs.push(node)
	mid := (s + e) / 2
	return cs.query(2*node, s, mid, l, r) | cs.query(2*node+1, mid+1, e, l, r)
}

func (cs *ColorSeg) CountDistinct(l, r int) int {
	mask := cs.query(1, 0, cs.n-1, l, r)
	return bits.OnesCount32(mask)
}

func main() {
	colors := []int{1, 2, 1, 3, 2, 1, 4, 3}
	cs := NewColorSeg(colors)

	fmt.Printf("Colors: %v\n", colors)
	fmt.Printf("Distinct [0,7] = %d\n", cs.CountDistinct(0, 7))
	fmt.Printf("Distinct [0,2] = %d\n", cs.CountDistinct(0, 2))

	cs.paint(1, 0, cs.n-1, 2, 5, 5)
	fmt.Printf("\nAfter painting [2,5] with color 5:\n")
	fmt.Printf("Distinct [0,7] = %d\n", cs.CountDistinct(0, 7))
}
```

**Textual Figure:**

```
Count Distinct Colors via Bitmask Segment Tree
Colors: [1, 2, 1, 3, 2, 1, 4, 3]   (n=8)

Each node stores OR-bitmask of colors.
Leaf for color c: bitmask = 1 << c
Merge: left | right

After build:
  [0,7] mask = 0b11110 = colors {1,2,3,4}  → 4 distinct
  [0,3] mask = 0b01110 = colors {1,2,3}    → 3 distinct
  [4,7] mask = 0b11110 = colors {1,2,3,4}  → 4 distinct

  ┌───────────────────────┐
  │ [0,7] mask=11110 (4)    │
  └───────┬───────────────┘
     ┌────┴────┐      ┌────┴────┐
     │[0,3]=1110│      │[4,7]=11110│
     └─────────┘      └──────────┘

Paint [2,5] with color 5:
  [2,5] becomes all color 5 → mask = 1<<5 = 100000
  lazy[2,5] = 5

After paint:
  Colors: [1, 2, 5, 5, 5, 5, 4, 3]
  [0,7] mask = 0b110110 = {1,2,3,4,5} → 5 distinct

  CountDistinct = popcount(mask) = bits.OnesCount32(mask)
```

---

## Example 10: Lazy Propagation Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Lazy Propagation Patterns ===")
	fmt.Println()

	patterns := []struct{ update, query, lazyType, compose string }{
		{"Range add", "Range sum", "add", "lazy += new"},
		{"Range add", "Range min", "add", "lazy += new"},
		{"Range add", "Range max", "add", "lazy += new"},
		{"Range set", "Range sum", "set", "lazy = new"},
		{"Range set", "Range min", "set", "lazy = new"},
		{"Range set", "Range max", "set", "lazy = new"},
		{"Range mul+add", "Range sum", "mul,add", "(mul,add)×(m,a)=(mul*m, add*m+a)"},
		{"Range flip", "Count 1s", "bool", "lazy = !lazy"},
		{"Range paint", "Distinct colors", "int+bitmask", "lazy = new color"},
		{"Range mul", "Range product", "mul", "lazy *= new"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-15s + %-15s | lazy: %-10s | compose: %s\n",
			p.update, p.query, p.lazyType, p.compose)
	}

	fmt.Println()
	fmt.Println("Key Rules:")
	fmt.Println("  1. Always push BEFORE accessing children")
	fmt.Println("  2. Lazy compose: new op applied AFTER existing lazy")
	fmt.Println("  3. Identity element: must not affect result (0 for add, 1 for mul)")
	fmt.Println("  4. Apply to node: update tree[node] using lazy and range size")
}
```

**Textual Figure:**

```
Lazy Propagation Pattern Decision Tree:

  What is your update operation?
  │
  ├── Range ADD
  │    ├── Query SUM? → lazy += val; tree += val * size
  │    ├── Query MIN? → lazy += val; tree += val
  │    └── Query MAX? → lazy += val; tree += val
  │
  ├── Range SET
  │    ├── Query SUM? → lazy = val; tree = val * size
  │    ├── Query MIN? → lazy = val; tree = val
  │    └── Query MAX? → lazy = val; tree = val
  │
  ├── Range MUL + ADD
  │    └── Query SUM? → lazy=(mul,add); compose: (m2*m1, m2*a1+a2)
  │
  └── Range FLIP
       └── Count 1s? → lazy^=true; tree = size - tree

  Composition rule:  compose(old_tag, new_tag) = combined_tag
  ┌──────────┬───────────────────────────────────┐
  │ Type     │ Composition                       │
  ├──────────┼───────────────────────────────────┤
  │ ADD      │ lazy += new_val                   │
  │ SET      │ lazy = new_val (overrides)        │
  │ MUL+ADD  │ (m1*m2, a1*m2 + a2)              │
  │ FLIP     │ lazy = !lazy (toggle)             │
  └──────────┴───────────────────────────────────┘
```

---

## Key Takeaways

1. **Lazy = deferred work**: store at internal nodes, push to children on demand
2. **Push before descend**: always push lazy before going to children
3. **Composition**: applying two lazy values must yield a single combined lazy
4. **Range add + sum**: most common — `tree += val*(r-l+1)`, `lazy += val`
5. **Range set**: needs sentinel value (e.g., MinInt64) to distinguish "no pending"
6. **Mixed ops**: set resets add; add after set composes; careful ordering needed
7. **Bit flips**: lazy is boolean toggle — XOR composition
8. **Color bitmask**: O(log n) query for distinct elements via OR merge

> **Next up:** Fenwick Tree (Binary Indexed Tree) →
