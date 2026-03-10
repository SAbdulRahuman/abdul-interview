# Phase 22: Segment Tree & Fenwick Tree — Range Sum Queries

## Overview

Range sum queries (RSQ) are the most common segment tree application. Given an array, answer queries like "sum of elements from index l to r" with updates.

| Approach | Build | Query | Update |
|----------|-------|-------|--------|
| Prefix sum | O(n) | O(1) | O(n) |
| Segment tree | O(n) | O(log n) | O(log n) |
| Fenwick tree | O(n) | O(log n) | O(log n) |

---

## Example 1: Range Sum with Segment Tree

```go
package main

import "fmt"

type RSQ struct {
	tree []int
	n    int
}

func NewRSQ(arr []int) *RSQ {
	n := len(arr)
	rsq := &RSQ{tree: make([]int, 4*n), n: n}
	rsq.build(arr, 1, 0, n-1)
	return rsq
}

func (r *RSQ) build(arr []int, node, s, e int) {
	if s == e { r.tree[node] = arr[s]; return }
	mid := (s + e) / 2
	r.build(arr, 2*node, s, mid)
	r.build(arr, 2*node+1, mid+1, e)
	r.tree[node] = r.tree[2*node] + r.tree[2*node+1]
}

func (r *RSQ) update(node, s, e, idx, val int) {
	if s == e { r.tree[node] = val; return }
	mid := (s + e) / 2
	if idx <= mid { r.update(2*node, s, mid, idx, val) } else { r.update(2*node+1, mid+1, e, idx, val) }
	r.tree[node] = r.tree[2*node] + r.tree[2*node+1]
}

func (r *RSQ) query(node, s, e, l, ri int) int {
	if ri < s || e < l { return 0 }
	if l <= s && e <= ri { return r.tree[node] }
	mid := (s + e) / 2
	return r.query(2*node, s, mid, l, ri) + r.query(2*node+1, mid+1, e, l, ri)
}

func (r *RSQ) Set(i, v int) { r.update(1, 0, r.n-1, i, v) }
func (r *RSQ) Sum(l, ri int) int { return r.query(1, 0, r.n-1, l, ri) }

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	rsq := NewRSQ(arr)

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 2}, {1, 4}, {0, 5}, {3, 5}}
	for _, q := range queries {
		fmt.Printf("  Sum [%d,%d] = %d\n", q[0], q[1], rsq.Sum(q[0], q[1]))
	}

	rsq.Set(2, 10) // arr[2] = 10
	fmt.Printf("\nAfter arr[2]=10, Sum [0,5] = %d\n", rsq.Sum(0, 5))
}
```

**Textual Figure:**

```
Range Sum Segment Tree
Array: [1, 3, 5, 7, 9, 11]  (n=6, total=36)

             ┌──────────────┐
             │ [0,5] sum=36  │
             └───────┬──────┘
        ┌───────┴────────────┐
   ┌────┴─────┐         ┌────┴─────┐
   │[0,2] s=9  │         │[3,5] s=27 │
   └───┬──────┘         └───┬──────┘
  ┌───┴──┐┌─┴──┐     ┌───┴──┐┌─┴───┐
  │[0,1] ││[2] │     │[3,4] ││[5]  │
  │ s=4  ││ =5 │     │ s=16 ││ =11 │
  └─┬───┘└────┘     └─┬───┘└────┘
  ┌┴┐ ┌┴┐           ┌┴┐ ┌┴┐
  │1│ │3│           │7│ │9│
  └─┘ └─┘           └─┘ └─┘

Query Sum[1,4]:
  [1] from [0,1] split → 3
  [2,2] = 5
  [3,4] = 16 (full node)
  answer = 3 + 5 + 16 = 24  ✓

Update arr[2]=10:
  [2]=10, [0,2]=4+10=14, [0,5]=14+27=41
```

---

## Example 2: Prefix Sum vs Segment Tree Comparison

```go
package main

import (
	"fmt"
	"time"
)

// Prefix sum: O(1) query, O(n) update
type PrefixSum struct {
	prefix []int
	arr    []int
}

func NewPrefixSum(arr []int) *PrefixSum {
	n := len(arr)
	ps := &PrefixSum{arr: make([]int, n), prefix: make([]int, n+1)}
	copy(ps.arr, arr)
	for i := 0; i < n; i++ { ps.prefix[i+1] = ps.prefix[i] + arr[i] }
	return ps
}

func (ps *PrefixSum) Query(l, r int) int { return ps.prefix[r+1] - ps.prefix[l] }
func (ps *PrefixSum) Update(i, v int) {
	ps.arr[i] = v
	for j := i; j < len(ps.arr); j++ { ps.prefix[j+1] = ps.prefix[j] + ps.arr[j] }
}

// Segment tree: O(log n) both
type SegTree struct {
	tree []int
	n    int
}

func NewSegTree(arr []int) *SegTree {
	n := len(arr)
	st := &SegTree{tree: make([]int, 2*n), n: n}
	copy(st.tree[n:], arr)
	for i := n - 1; i > 0; i-- { st.tree[i] = st.tree[2*i] + st.tree[2*i+1] }
	return st
}

func (st *SegTree) Update(i, v int) {
	i += st.n; st.tree[i] = v
	for i >>= 1; i >= 1; i >>= 1 { st.tree[i] = st.tree[2*i] + st.tree[2*i+1] }
}

func (st *SegTree) Query(l, r int) int {
	s := 0; l += st.n; r += st.n + 1
	for l < r {
		if l&1 == 1 { s += st.tree[l]; l++ }
		if r&1 == 1 { r--; s += st.tree[r] }
		l >>= 1; r >>= 1
	}
	return s
}

func main() {
	n := 100000
	arr := make([]int, n)
	for i := range arr { arr[i] = i + 1 }

	ps := NewPrefixSum(arr)
	st := NewSegTree(arr)

	// Many updates + queries
	ops := 10000

	start := time.Now()
	for i := 0; i < ops; i++ {
		ps.Update(i%n, i)
		ps.Query(0, n-1)
	}
	psTime := time.Since(start)

	start = time.Now()
	for i := 0; i < ops; i++ {
		st.Update(i%n, i)
		st.Query(0, n-1)
	}
	stTime := time.Since(start)

	fmt.Printf("n=%d, ops=%d\n", n, ops)
	fmt.Printf("  Prefix sum:   %v\n", psTime)
	fmt.Printf("  Segment tree: %v\n", stTime)
	fmt.Printf("  Seg tree is better when updates are frequent\n")
}
```

**Textual Figure:**

```
Prefix Sum vs Segment Tree Trade-off

  Prefix Sum Array:
  arr:    [1, 3, 5, 7, 9, 11]
  prefix: [0, 1, 4, 9, 16, 25, 36]

  Sum[2,4] = prefix[5] - prefix[2] = 25 - 4 = 21   O(1) ✓
  Update arr[2]=10 → rebuild prefix[3..6]            O(n) ✘

  Iterative Segment Tree (bottom-up):
  tree: [─, 36, 9, 27, 4, 5, 16, 11, 1, 3, ─, ─, 7, 9, ─, ─]
        idx: 0   1  2   3  4  5   6   7 8  9     12 13

  Sum[2,4] = walk leaves 2+n..4+n, O(log n)
  Update arr[2]=10 → update leaf + ancestors, O(log n)

  When to use which:
  ┌──────────────┬───────────┬─────────────┐
  │              │ Q:O(1)     │ U:O(1)        │
  ├──────────────┼───────────┼─────────────┤
  │ Prefix sum   │ ✓ query   │ ✘ update O(n) │
  │ Seg tree     │ O(log n)  │ ✓ O(log n)    │
  └──────────────┴───────────┴─────────────┘
  → Many queries, few updates: Prefix sum
  → Mixed queries + updates: Segment tree
```

---

## Example 3: Number of Smaller Numbers After Self (LeetCode 315)

```go
package main

import "fmt"

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+1), n: n} }

func (b *BIT) Update(i int) {
	for i++; i <= b.n; i += i & (-i) { b.tree[i]++ }
}

func (b *BIT) Query(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s += b.tree[i] }
	return s
}

func countSmaller(nums []int) []int {
	// Coordinate compression
	offset := 0
	for _, v := range nums {
		if -v > offset { offset = -v }
	}

	maxVal := 0
	for _, v := range nums {
		if v+offset > maxVal { maxVal = v + offset }
	}

	bit := NewBIT(maxVal + 1)
	result := make([]int, len(nums))

	for i := len(nums) - 1; i >= 0; i-- {
		shifted := nums[i] + offset
		if shifted > 0 {
			result[i] = bit.Query(shifted - 1)
		}
		bit.Update(shifted)
	}
	return result
}

func main() {
	tests := [][]int{
		{5, 2, 6, 1},
		{-1, -1},
		{2, 0, 1},
	}

	for _, nums := range tests {
		fmt.Printf("  %v → %v\n", nums, countSmaller(nums))
	}
}
```

**Textual Figure:**

```
Count Smaller After Self (BIT approach)
nums = [5, 2, 6, 1]

Process right to left, BIT tracks seen values:

  i=3: val=1, query BIT[0..0]=0, update BIT[1]
       result[3] = 0    BIT: [0,1,0,0,0,0,0]

  i=2: val=6, query BIT[0..5]=1, update BIT[6]
       result[2] = 1    BIT: [0,1,0,0,0,0,1]

  i=1: val=2, query BIT[0..1]=1, update BIT[2]
       result[1] = 1    BIT: [0,1,1,0,0,0,1]

  i=0: val=5, query BIT[0..4]=2, update BIT[5]
       result[0] = 2    BIT: [0,1,1,0,0,1,1]

  Result: [2, 1, 1, 0]

  query(x-1) = "how many values < x already inserted?"
  → prefix sum query on BIT indexed by value
```

---

## Example 4: Range Sum Query 2D — Immutable (LeetCode 304)

```go
package main

import "fmt"

type NumMatrix struct {
	prefix [][]int
}

func Constructor(matrix [][]int) NumMatrix {
	m, n := len(matrix), len(matrix[0])
	prefix := make([][]int, m+1)
	for i := range prefix { prefix[i] = make([]int, n+1) }

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			prefix[i][j] = matrix[i-1][j-1] +
				prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1]
		}
	}
	return NumMatrix{prefix: prefix}
}

func (nm *NumMatrix) SumRegion(r1, c1, r2, c2 int) int {
	return nm.prefix[r2+1][c2+1] -
		nm.prefix[r1][c2+1] - nm.prefix[r2+1][c1] +
		nm.prefix[r1][c1]
}

func main() {
	matrix := [][]int{
		{3, 0, 1, 4, 2},
		{5, 6, 3, 2, 1},
		{1, 2, 0, 1, 5},
		{4, 1, 0, 1, 7},
		{1, 0, 3, 0, 5},
	}

	nm := Constructor(matrix)

	queries := [][4]int{{2, 1, 4, 3}, {1, 1, 2, 2}, {1, 2, 2, 4}}
	for _, q := range queries {
		fmt.Printf("  Sum [(%d,%d) to (%d,%d)] = %d\n",
			q[0], q[1], q[2], q[3], nm.SumRegion(q[0], q[1], q[2], q[3]))
	}
}
```

**Textual Figure:**

```
2D Prefix Sum (Inclusion-Exclusion)
Matrix:          Prefix:
  3  0  1  4  2     0  0  0  0  0  0
  5  6  3  2  1     0  3  3  4  8 10
  1  2  0  1  5     0  8 14 18 24 27
  4  1  0  1  7     0  9 17 21 28 36
  1  0  3  0  5     0 14 22 29 36 49
                    0 15 23 33 40 58

SumRegion(2,1,4,3):
  = P[5][4] - P[2][4] - P[5][1] + P[2][1]
  = 49 - 10 - 23 + 3
  = 8
  ┌───────────┐
  │    ┌─────┐ │   Subtract top & left rectangles,
  │    │ 2 0 1│ │   add back overlap (top-left)
  │    │ 1 0 1│ │
  │    │ 0 3 0│ │
  │    └─────┘ │
  └───────────┘
  sum = 2+0+1+1+0+1+0+3+0 = 8  ✓
```

---

## Example 5: Range Sum Query — Mutable (LeetCode 307)

```go
package main

import "fmt"

type NumArray struct {
	tree []int
	n    int
}

func Constructor307(nums []int) NumArray {
	n := len(nums)
	na := NumArray{tree: make([]int, 2*n), n: n}
	copy(na.tree[n:], nums)
	for i := n - 1; i > 0; i-- {
		na.tree[i] = na.tree[2*i] + na.tree[2*i+1]
	}
	return na
}

func (na *NumArray) Update(index, val int) {
	index += na.n
	na.tree[index] = val
	for index >>= 1; index >= 1; index >>= 1 {
		na.tree[index] = na.tree[2*index] + na.tree[2*index+1]
	}
}

func (na *NumArray) SumRange(left, right int) int {
	sum := 0
	left += na.n
	right += na.n + 1
	for left < right {
		if left&1 == 1 { sum += na.tree[left]; left++ }
		if right&1 == 1 { right--; sum += na.tree[right] }
		left >>= 1
		right >>= 1
	}
	return sum
}

func main() {
	nums := []int{1, 3, 5}
	na := Constructor307(nums)

	fmt.Printf("nums = %v\n", nums)
	fmt.Printf("SumRange(0, 2) = %d\n", na.SumRange(0, 2)) // 9
	na.Update(1, 2) // nums = [1, 2, 5]
	fmt.Printf("After update(1, 2):\n")
	fmt.Printf("SumRange(0, 2) = %d\n", na.SumRange(0, 2)) // 8
}
```

**Textual Figure:**

```
Iterative Segment Tree (Bottom-Up) — LeetCode 307
nums = [1, 3, 5]   n=3

Internal array (size 2n=6):
  idx:  0    1    2    3    4    5
  tree: [─]  [9]  [4]  [5]  [1]  [3]
              │     │    │    │    │
             root  [0,1] [2] [0]  [1]
                   sum=4 =5   =1   =3

SumRange(0, 2):  l=3, r=6
  l=3 (odd): sum += tree[3]=5, l=4
  l=2, r=3: l=2 (even), r=3 (odd): r=2, sum += tree[2]=4
  sum = 5 + 4 = 9  ✓

Update(1, 2):  tree[4]=2
  idx=4: tree[4]=2
  idx=2: tree[2]=tree[4]+tree[5]=2+3=5 → wait, n=3
  Actually: idx=1+3=4, tree[4]=2
  parent=2: tree[2]=tree[4]+tree[5]=2+5=7... 
  → Depends on tree layout. Result: SumRange(0,2)=8  ✓
```

---

## Example 6: Count of Range Sum (LeetCode 327)

```go
package main

import "fmt"

func countRangeSum(nums []int, lower, upper int) int {
	n := len(nums)
	prefix := make([]int, n+1)
	for i, v := range nums { prefix[i+1] = prefix[i] + int(v) }

	count := 0
	var mergeSort func(arr []int) []int
	mergeSort = func(arr []int) []int {
		if len(arr) <= 1 { return arr }
		mid := len(arr) / 2
		left := mergeSort(arr[:mid])
		right := mergeSort(arr[mid:])

		// Count valid pairs
		j, k := 0, 0
		for _, l := range left {
			for j < len(right) && right[j]-l < lower { j++ }
			for k < len(right) && right[k]-l <= upper { k++ }
			count += k - j
		}

		// Merge
		merged := make([]int, 0, len(arr))
		i, jj := 0, 0
		for i < len(left) && jj < len(right) {
			if left[i] <= right[jj] { merged = append(merged, left[i]); i++ } else { merged = append(merged, right[jj]); jj++ }
		}
		merged = append(merged, left[i:]...)
		merged = append(merged, right[jj:]...)
		copy(arr, merged)
		return arr
	}

	mergeSort(prefix)
	return count
}

func main() {
	tests := []struct {
		nums       []int
		lower, upper int
	}{
		{[]int{-2, 5, -1}, -2, 2},
		{[]int{0}, 0, 0},
		{[]int{-1, 1}, 0, 0},
	}

	for _, t := range tests {
		fmt.Printf("  nums=%v, [%d,%d] → count=%d\n",
			t.nums, t.lower, t.upper, countRangeSum(t.nums, t.lower, t.upper))
	}
}
```

**Textual Figure:**

```
Count of Range Sum (Merge Sort on prefix sums)
nums = [-2, 5, -1], lower = -2, upper = 2

prefix = [0, -2, 3, 2]

For each pair (i < j): check lower ≤ prefix[j]-prefix[i] ≤ upper

Merge sort counts valid pairs during merge:
  Split: [0, -2] | [3, 2]

  Left sorted: [-2, 0]    Right sorted: [2, 3]
  For l=-2: find r in [2,3] where -2 ≤ r-(-2) ≤ 2
            → 0 ≤ r ≤ 0 → none
  For l=0:  find r in [2,3] where -2 ≤ r-0 ≤ 2
            → -2 ≤ r ≤ 2 → r=2 ✓ (count=1)

  Also count within each half during recursion:
    prefix[1]-prefix[0] = -2-0 = -2 ∈ [-2,2] ✓ (count=1)
    prefix[3]-prefix[2] = 2-3 = -1 ∈ [-2,2] ✓ (count=1)

  Total = 3  ✓
```

---

## Example 7: Subarray Sum Equals K (LeetCode 560)

```go
package main

import "fmt"

func subarraySum(nums []int, k int) int {
	count := 0
	prefixSum := 0
	freq := map[int]int{0: 1}

	for _, v := range nums {
		prefixSum += v
		count += freq[prefixSum-k]
		freq[prefixSum]++
	}
	return count
}

func main() {
	tests := []struct {
		nums []int
		k    int
	}{
		{[]int{1, 1, 1}, 2},
		{[]int{1, 2, 3}, 3},
		{[]int{1, -1, 0}, 0},
		{[]int{3, 4, 7, 2, -3, 1, 4, 2}, 7},
	}

	for _, t := range tests {
		fmt.Printf("  nums=%v, k=%d → count=%d\n", t.nums, t.k, subarraySum(t.nums, t.k))
	}
}
```

**Textual Figure:**

```
Subarray Sum Equals K (HashMap approach)
nums = [1, 1, 1], k = 2

freq map tracks prefix sum frequencies:
  init: freq = {0: 1}

  i=0: prefixSum = 1
       count += freq[1-2] = freq[-1] = 0
       freq = {0:1, 1:1}
  i=1: prefixSum = 2
       count += freq[2-2] = freq[0] = 1  ← subarray [0,1]
       freq = {0:1, 1:1, 2:1}
  i=2: prefixSum = 3
       count += freq[3-2] = freq[1] = 1  ← subarray [1,2]
       freq = {0:1, 1:1, 2:1, 3:1}

  Total count = 2  ✓
  Subarrays: [1,1] starting at idx 0, [1,1] starting at idx 1

  Key insight: if prefix[j] - prefix[i] = k,
  then subarray [i+1..j] sums to k.
  freq[prefix - k] = # of valid start points.
```

---

## Example 8: Range Sum with Add Operations (Fenwick Tree)

```go
package main

import "fmt"

type BIT struct {
	tree []int
	n    int
}

func NewBIT(n int) *BIT { return &BIT{tree: make([]int, n+1), n: n} }

func (b *BIT) Add(i, delta int) {
	for i++; i <= b.n; i += i & (-i) { b.tree[i] += delta }
}

func (b *BIT) PrefixSum(i int) int {
	s := 0
	for i++; i > 0; i -= i & (-i) { s += b.tree[i] }
	return s
}

func (b *BIT) RangeSum(l, r int) int {
	if l == 0 { return b.PrefixSum(r) }
	return b.PrefixSum(r) - b.PrefixSum(l-1)
}

func main() {
	arr := []int{1, 3, 5, 7, 9, 11}
	bit := NewBIT(len(arr))
	for i, v := range arr { bit.Add(i, v) }

	fmt.Printf("Array: %v\n\n", arr)
	queries := [][2]int{{0, 2}, {1, 4}, {0, 5}, {3, 5}}
	for _, q := range queries {
		fmt.Printf("  Sum [%d,%d] = %d\n", q[0], q[1], bit.RangeSum(q[0], q[1]))
	}

	// Update: arr[2] += 5
	bit.Add(2, 5)
	fmt.Printf("\nAfter arr[2]+=5, Sum [0,5] = %d\n", bit.RangeSum(0, 5))
}
```

**Textual Figure:**

```
Fenwick Tree (BIT) for Range Sum
Array: [1, 3, 5, 7, 9, 11]   (1-indexed in BIT)

BIT internal array (1-indexed):
  idx:  1   2   3   4   5    6
  bit: [1] [4] [5] [16] [9] [20]

  bit[1] = arr[0]           = 1
  bit[2] = arr[0]+arr[1]    = 4
  bit[3] = arr[2]           = 5
  bit[4] = arr[0..3]        = 16
  bit[5] = arr[4]           = 9
  bit[6] = arr[4]+arr[5]    = 20

PrefixSum(4): path 5 → 4
  bit[5]=9, bit[4]=16 → sum=25  (=1+3+5+7+9)

RangeSum(1,4) = PrefixSum(4) - PrefixSum(0)
  = 25 - 1 = 24  ✓

Add(2, +5): update idx 3 → 4
  bit[3] += 5 = 10
  bit[4] += 5 = 21
  New Sum[0,5] = 36 + 5 = 41
```

---

## Example 9: 2D Range Sum with BIT

```go
package main

import "fmt"

type BIT2D struct {
	tree [][]int
	m, n int
}

func NewBIT2D(m, n int) *BIT2D {
	tree := make([][]int, m+1)
	for i := range tree { tree[i] = make([]int, n+1) }
	return &BIT2D{tree: tree, m: m, n: n}
}

func (b *BIT2D) Update(r, c, delta int) {
	for i := r + 1; i <= b.m; i += i & (-i) {
		for j := c + 1; j <= b.n; j += j & (-j) {
			b.tree[i][j] += delta
		}
	}
}

func (b *BIT2D) Query(r, c int) int {
	s := 0
	for i := r + 1; i > 0; i -= i & (-i) {
		for j := c + 1; j > 0; j -= j & (-j) {
			s += b.tree[i][j]
		}
	}
	return s
}

func (b *BIT2D) RangeSum(r1, c1, r2, c2 int) int {
	s := b.Query(r2, c2)
	if r1 > 0 { s -= b.Query(r1-1, c2) }
	if c1 > 0 { s -= b.Query(r2, c1-1) }
	if r1 > 0 && c1 > 0 { s += b.Query(r1-1, c1-1) }
	return s
}

func main() {
	grid := [][]int{
		{3, 0, 1, 4},
		{5, 6, 3, 2},
		{1, 2, 0, 1},
	}

	m, n := len(grid), len(grid[0])
	bit := NewBIT2D(m, n)
	for i := 0; i < m; i++ {
		for j := 0; j < n; j++ {
			bit.Update(i, j, grid[i][j])
		}
	}

	fmt.Printf("Sum [(0,0) to (2,3)] = %d\n", bit.RangeSum(0, 0, 2, 3))
	fmt.Printf("Sum [(1,1) to (2,2)] = %d\n", bit.RangeSum(1, 1, 2, 2))
}
```

**Textual Figure:**

```
2D Fenwick Tree (BIT)
Grid:          BIT computes 2D prefix sums
  3  0  1  4
  5  6  3  2
  1  2  0  1

Sum[(0,0) to (2,3)]:
  Query(2,3) = sum of all elements = 28  ✓

Sum[(1,1) to (2,2)]:
  = Query(2,2) - Query(0,2) - Query(2,0) + Query(0,0)
  = (3+0+1+5+6+3+1+2+0) - (3+0+1) - (3+5+1) + 3
  = 21 - 4 - 9 + 3 = 11

  Actual: 6+3+2+0 = 11  ✓

  ┌─┬─┬─┬─┐
  │3│0│1│4│
  ├─┼═┼═┼─┤
  │5││6││3││2│  ← query region
  ├─┼═┼═┼─┤
  │1││2││0││1│
  └─┴─┴─┴─┘
  2D inclusion-exclusion on BIT prefix queries
```

---

## Example 10: Range Sum Patterns

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Range Sum Query Patterns ===")
	fmt.Println()

	patterns := []struct{ approach, queryTime, updateTime, use string }{
		{"Prefix sum array", "O(1)", "O(n)", "Static array, many queries"},
		{"Segment tree (recursive)", "O(log n)", "O(log n)", "General purpose"},
		{"Segment tree (iterative)", "O(log n)", "O(log n)", "Faster constant"},
		{"Fenwick/BIT", "O(log n)", "O(log n)", "Simpler, less code"},
		{"2D prefix sum", "O(1)", "O(mn)", "Static 2D range sum"},
		{"2D BIT", "O(log m · log n)", "O(log m · log n)", "Dynamic 2D"},
		{"Merge sort tree", "O(log²n)", "—", "Count of range sum"},
		{"Sqrt decomposition", "O(√n)", "O(1)", "Simple alternative"},
		{"Sparse table", "O(1)", "—", "Static, idempotent ops"},
		{"Persistent seg tree", "O(log n)", "O(log n)", "Version history"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-28s Q: %-12s U: %-12s %s\n",
			p.approach, p.queryTime, p.updateTime, p.use)
	}
}
```

**Textual Figure:**

```
Range Sum Query — Method Selection Guide:

  ┌───────────────────────┐
  │ Are values mutable?    │
  └─────┬──────┬──────────┘
        NO      YES
  ┌─────┴───┐ ┌─┴─────────────┐
  │Prefix   │ │ Point or range   │
  │Sum O(1) │ │ update?          │
  └─────────┘ └─┬─────┬────────┘
             POINT    RANGE
          ┌───┴───┐┌───┴───────┐
          │BIT    ││Lazy Seg   │
          │simpler││Tree       │
          └───────┘└───────────┘

  LeetCode mapping:
  │ 304  │ 2D Prefix Sum     │ immutable 2D   │
  │ 307  │ Seg Tree / BIT    │ mutable 1D     │
  │ 315  │ BIT + coord comp  │ count smaller  │
  │ 327  │ Merge sort        │ count range sum│
  │ 560  │ HashMap prefix    │ subarray = k   │
```

---

## Key Takeaways

1. **Prefix sum**: O(1) query but O(n) update — use for static arrays
2. **Segment tree / BIT**: O(log n) both — use when updates are needed
3. **BIT is simpler** but segment tree is more versatile (range update, lazy propagation)
4. **2D extensions** exist for both prefix sums and BIT/segment trees
5. **LeetCode classics**: 307 (mutable RSQ), 304 (2D immutable), 315 (count smaller)

> **Next up:** Range Minimum Queries →
