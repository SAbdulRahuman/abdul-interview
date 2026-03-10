# Phase 13: Union Find — Union by Rank

## Overview

**Union by rank** attaches the shorter tree under the root of the taller tree during union operations. The "rank" is an upper bound on the height of the tree. This prevents the tree from degenerating into a linked list.

| Without Union by Rank | With Union by Rank |
|----------------------|-------------------|
| O(n) tree height possible | O(log n) tree height |
| Random attachment | Shorter tree under taller |
| Combined with path compression → O(log n) | Combined with path compression → O(α(n)) |

**Rank rules:**
- Initially, every node has rank 0
- Rank increases only when two trees of equal rank are merged
- Rank is an upper bound on height (not exact height after compression)

---

## Example 1: Basic Union by Rank

```go
package main

import "fmt"

type UnionFind struct {
	parent []int
	rank   []int
}

func NewUF(n int) *UnionFind {
	p := make([]int, n)
	r := make([]int, n)
	for i := range p { p[i] = i }
	return &UnionFind{parent: p, rank: r}
}

func (uf *UnionFind) Find(x int) int {
	if uf.parent[x] != x {
		uf.parent[x] = uf.Find(uf.parent[x])
	}
	return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) bool {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry { return false }

	// Attach smaller rank under larger rank
	if uf.rank[rx] < uf.rank[ry] {
		uf.parent[rx] = ry
	} else if uf.rank[rx] > uf.rank[ry] {
		uf.parent[ry] = rx
	} else {
		uf.parent[ry] = rx
		uf.rank[rx]++
	}
	return true
}

func main() {
	uf := NewUF(8)
	pairs := [][2]int{{0,1},{2,3},{4,5},{6,7},{0,2},{4,6},{0,4}}
	for _, p := range pairs {
		uf.Union(p[0], p[1])
		fmt.Printf("Union(%d,%d): parent=%v, rank=%v\n", p[0], p[1], uf.parent, uf.rank)
	}
}
```

**Textual Figure:**
```
  Union by Rank: 8 elements, pairs {0,1},{2,3},{4,5},{6,7},{0,2},{4,6},{0,4}

  Step 1-4: pair unions (equal ranks → rank++):
    ┌───┐  ┌───┐  ┌───┐  ┌───┐
    │0 r1│  │2 r1│  │4 r1│  │6 r1│
    └─┬─┘  └─┬─┘  └─┬─┘  └─┬─┘
    ┌─┴─┐  ┌─┴─┐  ┌─┴─┐  ┌─┴─┐
    │ 1 │  │ 3 │  │ 5 │  │ 7 │
    └───┘  └───┘  └───┘  └───┘

  Step 5: Union(0,2) → rank[0]=1 == rank[2]=1 → parent[2]=0, rank[0]=2
  Step 6: Union(4,6) → rank[4]=1 == rank[6]=1 → parent[6]=4, rank[4]=2
      ┌───┐              ┌───┐
      │0 r2│              │4 r2│
      └─┬─┘              └─┬─┘
     ┌──┴──┐           ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐       ┌─┴─┐ ┌─┴─┐
   │ 1 │ │ 2 │       │ 5 │ │ 6 │
   └───┘ └─┬─┘       └───┘ └─┬─┘
         ┌─┴─┐             ┌─┴─┐
         │ 3 │             │ 7 │
         └───┘             └───┘

  Step 7: Union(0,4) → rank[0]=2 == rank[4]=2 → parent[4]=0, rank[0]=3
            ┌───┐
            │0 r3│ (root)
            └─┬─┘
        ┌───┼───┐
      ┌─┴─┐┌┴─┐┌┴─┐
      │ 1 ││2 ││4 │
      └───┘└┬─┘└┬─┘
           │  ┌──┴──┐
          ┌┴┐┌┴─┐┌┴─┐
          │3││5 ││6 │
          └─┘└──┘└┬─┘
                  │
                 ┌┴┐
                 │7│
                 └─┘
  Max height = 3 = ⌊log₂(8)⌋  ✓
```

---

## Example 2: Why Rank Keeps Trees Balanced

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Union by Rank: Balance Guarantee ===")
	fmt.Println()

	// Without rank: naive union creates chain
	parent1 := make([]int, 5)
	for i := range parent1 { parent1[i] = i }
	// Naive: always put first under second
	parent1[0] = 1; parent1[1] = 2; parent1[2] = 3; parent1[3] = 4
	fmt.Println("Naive (chain): 0→1→2→3→4")
	fmt.Println("Height =", 4)

	// With rank: balanced merging
	parent2 := make([]int, 8)
	rank2 := make([]int, 8)
	for i := range parent2 { parent2[i] = i }

	union := func(x, y int) {
		if rank2[x] < rank2[y] {
			parent2[x] = y
		} else if rank2[x] > rank2[y] {
			parent2[y] = x
		} else {
			parent2[y] = x
			rank2[x]++
		}
	}

	// Merge pairs: {0,1}, {2,3}, {4,5}, {6,7}
	union(0, 1); union(2, 3); union(4, 5); union(6, 7)
	// Merge groups: {0,2}, {4,6}
	union(0, 2); union(4, 6)
	// Final merge
	union(0, 4)

	fmt.Printf("\nWith rank: parent=%v, rank=%v\n", parent2, rank2)
	fmt.Println("Max height = 3 (log₂(8))")
	fmt.Println()
	fmt.Println("Key property: A tree of rank r has at least 2^r nodes")
	fmt.Println("Therefore max rank ≤ log₂(n)")
}
```

**Textual Figure:**
```
  Naive union (chain):             Union by rank (balanced):

    0→1→2→3→4                        ┌───┐
    Height = 4 = O(n)              │0 r3│
                                   └─┬─┘
    ┌─┐                          ┌──┬┴─┬──┐
    │4│                        ┌─┴┐┌┴┐┌┴┐┌┴─┐
    └┬┘                        │1││2││5││ 4 │
    ┌┴┐                        └─┘└┬┘└─┘└┬─┘
    │3│                           │   ┌─┴─┐
    └┬┘                          ┌┴┐ ┌┴─┐┌┴┐
    ┌┴┐                          │3│ │6 ││7│
    │2│                          └─┘ └──┘└─┘
    └┬┘
    ┌┴┐                        Height = 3 = log₂(8)
    │1│
    └┬┘                        Key property:
    ┌┴┐                          rank r → ≥ 2^r nodes
    │0│                          rank 0: ≥ 1 node
    └─┘                          rank 1: ≥ 2 nodes
                                   rank 2: ≥ 4 nodes
                                   rank 3: ≥ 8 nodes  ✓
```

---

## Example 3: Rank vs Height After Compression

```go
package main

import "fmt"

type UF struct {
	parent []int
	rank   []int
}

func NewUF(n int) *UF {
	p := make([]int, n)
	r := make([]int, n)
	for i := range p { p[i] = i }
	return &UF{parent: p, rank: r}
}

func (uf *UF) Find(x int) int {
	if uf.parent[x] != x { uf.parent[x] = uf.Find(uf.parent[x]) }
	return uf.parent[x]
}

func (uf *UF) Union(x, y int) {
	rx, ry := uf.Find(x), uf.Find(y)
	if rx == ry { return }
	if uf.rank[rx] < uf.rank[ry] { rx, ry = ry, rx }
	uf.parent[ry] = rx
	if uf.rank[rx] == uf.rank[ry] { uf.rank[rx]++ }
}

func (uf *UF) ActualHeight(root int) int {
	maxH := 0
	for i := range uf.parent {
		h := 0
		x := i
		for x != root && uf.parent[x] != x {
			h++; x = uf.parent[x]
		}
		if x == root && h > maxH { maxH = h }
	}
	return maxH
}

func main() {
	uf := NewUF(8)
	uf.Union(0, 1); uf.Union(2, 3); uf.Union(0, 2)
	uf.Union(4, 5); uf.Union(6, 7); uf.Union(4, 6)
	uf.Union(0, 4)

	root := uf.Find(0)
	fmt.Printf("Rank of root %d: %d\n", root, uf.rank[root])

	// After path compression, actual height may be less than rank
	uf.Find(7) // Compress path from 7
	fmt.Printf("Actual height after some compression: %d\n", uf.ActualHeight(root))
	fmt.Println("Rank is upper bound, not exact height after compression")
}
```

**Textual Figure:**
```
  Build tree: Union(0,1), Union(2,3), Union(0,2),
              Union(4,5), Union(6,7), Union(4,6), Union(0,4)

  Before compression:          After Find(7) with compression:
        ┌───┐                        ┌───┐
        │0 r3│                        │0 r3│ ← rank stays 3!
        └─┬─┘                        └─┬─┘
      ┌──┼──┐                    ┌──┼──┬──┐
    ┌─┴┐┌┴┐┌┴─┐                ┌─┴┐┌┴┐┌┴┐┌┴─┐
    │1││2││4 │                │1││2││4││ 7 │ ← 7 now
    └─┘└┬┘└┬─┘                └─┘└┬┘└┬┘└───┘   points
       │ ┌─┴─┐                   │  │         to 0!
      ┌┴┐┌┴─┐┌─┐              ┌┴┐┌┴┐
      │3││5 ││6│              │3││5│
      └─┘└──┘└┬┘              └─┘└─┘
              │
             ┌┴┐                Actual height: 1
             │7│                Rank: 3 (unchanged)
             └─┘                Rank ≥ actual height always
    Actual height: 3
    Rank: 3

  Key insight: rank is an UPPER BOUND on height.
  Path compression reduces height but never updates rank.
  Rank only increases during union of equal-rank trees.
```

---

## Example 4: Connected Components with Rank

```go
package main

import "fmt"

func countComponentsRank(n int, edges [][2]int) int {
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) bool {
		rx, ry := find(x), find(y)
		if rx == ry { return false }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		if rank[rx] == rank[ry] { rank[rx]++ }
		return true
	}

	components := n
	for _, e := range edges {
		if union(e[0], e[1]) { components-- }
	}
	return components
}

func main() {
	edges := [][2]int{{0,1},{1,2},{3,4},{5,6},{6,7},{3,7}}
	fmt.Println("Components:", countComponentsRank(8, edges)) // 2
}
```

**Textual Figure:**
```
  n=8, edges: {0,1},{1,2},{3,4},{5,6},{6,7},{3,7}

  ┌────────┬────────────┬──────────────────────────┬────────────┐
  │ Edge   │ find(u,v)  │ Action                   │ Components │
  ├────────┼────────────┼──────────────────────────┼────────────┤
  │ {0,1}  │ 0 ≠ 1     │ Union, rank++ → r[0]=1   │   8 → 7    │
  │ {1,2}  │ 0 ≠ 2     │ Union, r[0]>r[2]         │   7 → 6    │
  │ {3,4}  │ 3 ≠ 4     │ Union, rank++ → r[3]=1   │   6 → 5    │
  │ {5,6}  │ 5 ≠ 6     │ Union, rank++ → r[5]=1   │   5 → 4    │
  │ {6,7}  │ 5 ≠ 7     │ Union, r[5]>r[7]         │   4 → 3    │
  │ {3,7}  │ 3 ≠ 5     │ Union, rank++ → r[3]=2   │   3 → 2    │
  └────────┴────────────┴──────────────────────────┴────────────┘

  Final forest (2 components):
    Component 1:         Component 2:
      ┌───┐                 ┌───┐
      │ 0 │ (r=1)           │ 3 │ (r=2)
      └─┬─┘                 └─┬─┘
     ┌──┴──┐             ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐         ┌─┴─┐ ┌─┴─┐
   │ 1 │ │ 2 │         │ 4 │ │ 5 │
   └───┘ └───┘         └───┘ └─┬─┘
                              ┌──┴──┐
                            ┌─┴─┐ ┌─┴─┐
                            │ 6 │ │ 7 │
                            └───┘ └───┘

  Result: 2 components
```

---

## Example 5: Redundant Connection with Rank

```go
package main

import "fmt"

func findRedundantConnection(edges [][]int) []int {
	n := len(edges)
	parent := make([]int, n+1)
	rank := make([]int, n+1)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { return e }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		if rank[rx] == rank[ry] { rank[rx]++ }
	}
	return nil
}

func main() {
	edges := [][]int{{1,2},{1,3},{2,3}}
	fmt.Println("Redundant:", findRedundantConnection(edges)) // [2 3]
}
```

**Textual Figure:**
```
  edges = [[1,2], [1,3], [2,3]]

  ┌────────┬────────────┬──────────────────────────────────┐
  │ Edge   │ find(u,v)  │ Action                           │
  ├────────┼────────────┼──────────────────────────────────┤
  │ [1,2]  │ 1 ≠ 2     │ rank eq → parent[2]=1, r[1]++    │
  │ [1,3]  │ 1 ≠ 3     │ r[1]>r[3] → parent[3]=1          │
  │ [2,3]  │ 1 == 1    │ SAME ROOT → CYCLE! Return [2,3]  │
  └────────┴────────────┴──────────────────────────────────┘

  Forest at cycle detection:
      ┌───┐
      │ 1 │ (root, rank=1)
      └─┬─┘
     ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐
   │ 2 │ │ 3 │    Edge [2,3]: find(2)=1, find(3)=1
   └───┘ └───┘    Same root → redundant edge!

  Note: Union by rank kept tree balanced (height=1)
  instead of creating chain 1→2→3
```

---

## Example 6: Rank Property Proof

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Union by Rank: Mathematical Properties ===")
	fmt.Println()

	fmt.Println("CLAIM: A tree with rank r has at least 2^r nodes")
	fmt.Println()
	fmt.Println("PROOF by induction:")
	fmt.Println("  Base: rank 0 → single node (2^0 = 1) ✓")
	fmt.Println("  Step: rank r created only by merging two rank (r-1) trees")
	fmt.Println("        Each has ≥ 2^(r-1) nodes")
	fmt.Println("        Total ≥ 2 × 2^(r-1) = 2^r ✓")
	fmt.Println()
	fmt.Println("COROLLARY: max rank ≤ log₂(n)")
	fmt.Println()

	// Demonstrate
	sizes := []struct{ rank, minNodes int }{
		{0, 1}, {1, 2}, {2, 4}, {3, 8}, {4, 16}, {5, 32},
	}
	fmt.Println("Rank | Min Nodes | Example")
	for _, s := range sizes {
		fmt.Printf("  %d  |    %3d    | 2^%d = %d\n", s.rank, s.minNodes, s.rank, s.minNodes)
	}
}
```

**Textual Figure:**
```
  Rank Property Proof by Induction:
  ═══════════════════════════════════

  Base case: rank 0
    ┌───┐
    │ x │  rank=0  →  1 node  =  2⁰ ✓
    └───┘

  Inductive step: rank r formed by merging two rank (r-1) trees

    rank 1: merge two rank-0 trees
      ┌───┐         ┌───┐           ┌───┐
      │ A │ (r=0) + │ B │ (r=0) →   │ A │ (r=1)   ≥ 2¹ = 2 nodes ✓
      └───┘         └───┘           └─┬─┘
                                    ┌─┴─┐
                                    │ B │
                                    └───┘

    rank 2: merge two rank-1 trees
      ┌───┐           ┌───┐           ┌───┐
      │ A │ (r=1) +   │ C │ (r=1) →  │ A │ (r=2)   ≥ 2² = 4 nodes ✓
      └─┬─┘           └─┬─┘           └─┬─┘
      ┌─┴─┐           ┌─┴─┐          ┌──┴──┐
      │ B │           │ D │        ┌─┴─┐ ┌─┴─┐
      └───┘           └───┘        │ B │ │ C │
                                   └───┘ └─┬─┘
                                         ┌─┴─┐
                                         │ D │
                                         └───┘

  ┌──────┬───────────┬─────────────────────────┐
  │ Rank │ Min Nodes │ Proof                   │
  ├──────┼───────────┼─────────────────────────┤
  │  0   │     1     │ 2⁰ = 1  (base case)     │
  │  1   │     2     │ 2¹ = 2                  │
  │  2   │     4     │ 2² = 4                  │
  │  3   │     8     │ 2³ = 8                  │
  │  4   │    16     │ 2⁴ = 16                 │
  │  5   │    32     │ 2⁵ = 32                 │
  └──────┴───────────┴─────────────────────────┘

  Corollary: max rank ≤ log₂(n)
    n = 1000000 → max rank ≤ 19
```

---

## Example 7: Number of Provinces (LeetCode 547)

```go
package main

import "fmt"

func findCircleNum(isConnected [][]int) int {
	n := len(isConnected)
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		rx, ry := find(x), find(y)
		if rx == ry { return }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		if rank[rx] == rank[ry] { rank[rx]++ }
	}

	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			if isConnected[i][j] == 1 { union(i, j) }
		}
	}

	provinces := 0
	for i := 0; i < n; i++ {
		if find(i) == i { provinces++ }
	}
	return provinces
}

func main() {
	con := [][]int{{1,1,0},{1,1,0},{0,0,1}}
	fmt.Println("Provinces:", findCircleNum(con)) // 2
}
```

**Textual Figure:**
```
  Adjacency matrix (3 cities):
       0  1  2
    0 [1, 1, 0]    0 ↔ 1 (connected)
    1 [1, 1, 0]    1 ↔ 0 (same)
    2 [0, 0, 1]    2 is isolated

  Graph:
    ┌───┐     ┌───┐       ┌───┐
    │ 0 │─────│ 1 │       │ 2 │ (isolated)
    └───┘     └───┘       └───┘

  Union-Find trace:
  ┌──────────┬─────────────────────────────┬──────────────┐
  │ Pair(i,j)│ Action                      │ parent[]     │
  ├──────────┼─────────────────────────────┼──────────────┤
  │ (0, 1)   │ rank eq → parent[1]=0, r++  │ [0, 0, 2]    │
  │ (0, 2)   │ isConnected=0 → skip        │ [0, 0, 2]    │
  │ (1, 2)   │ isConnected=0 → skip        │ [0, 0, 2]    │
  └──────────┴─────────────────────────────┴──────────────┘

  Final forest:
    Province 1:        Province 2:
      ┌───┐              ┌───┐
      │ 0 │ (root,r=1)   │ 2 │ (root,r=0)
      └─┬─┘              └───┘
      ┌─┴─┐
      │ 1 │
      └───┘

  Count roots: {0, 2} → 2 provinces
```

---

## Example 8: Rank Update Trace

```go
package main

import "fmt"

func main() {
	n := 8
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	find := func(x int) int {
		for parent[x] != x { x = parent[x] }
		return x
	}

	union := func(x, y int) {
		rx, ry := find(x), find(y)
		if rx == ry { return }
		if rank[rx] < rank[ry] {
			fmt.Printf("  rank[%d]=%d < rank[%d]=%d → attach %d under %d\n", rx, rank[rx], ry, rank[ry], rx, ry)
			parent[rx] = ry
		} else if rank[rx] > rank[ry] {
			fmt.Printf("  rank[%d]=%d > rank[%d]=%d → attach %d under %d\n", rx, rank[rx], ry, rank[ry], ry, rx)
			parent[ry] = rx
		} else {
			fmt.Printf("  rank[%d]=%d == rank[%d]=%d → attach %d under %d, rank[%d]++\n", rx, rank[rx], ry, rank[ry], ry, rx, rx)
			parent[ry] = rx
			rank[rx]++
		}
	}

	ops := [][2]int{{0,1},{2,3},{0,2},{4,5},{6,7},{4,6},{0,4}}
	for _, op := range ops {
		fmt.Printf("Union(%d, %d):\n", op[0], op[1])
		union(op[0], op[1])
		fmt.Printf("  parent=%v rank=%v\n\n", parent, rank)
	}
}
```

**Textual Figure:**
```
  Rank Update Trace: 8 elements
  Operations: {0,1},{2,3},{0,2},{4,5},{6,7},{4,6},{0,4}

  ┌─────────────┬────────────────────────────────────┬──────────────────────────┐
  │ Operation   │ Rank Decision                      │ parent[]       rank[]    │
  ├─────────────┼────────────────────────────────────┼──────────────────────────┤
  │ Union(0,1)  │ r[0]=0 == r[1]=0 → p[1]=0, r[0]++ │ [0,0,2,3,4,5,6,7] [1,..]│
  │ Union(2,3)  │ r[2]=0 == r[3]=0 → p[3]=2, r[2]++ │ [0,0,2,2,4,5,6,7] [1,0,1]│
  │ Union(0,2)  │ r[0]=1 == r[2]=1 → p[2]=0, r[0]++ │ [0,0,0,2,4,5,6,7] [2,..]│
  │ Union(4,5)  │ r[4]=0 == r[5]=0 → p[5]=4, r[4]++ │ [..,4,4,6,7]    [..,1,..]│
  │ Union(6,7)  │ r[6]=0 == r[7]=0 → p[7]=6, r[6]++ │ [..,4,4,6,6]    [..,1,1] │
  │ Union(4,6)  │ r[4]=1 == r[6]=1 → p[6]=4, r[4]++ │ [..,4,4,4,6]    [..,2,1] │
  │ Union(0,4)  │ r[0]=2 == r[4]=2 → p[4]=0, r[0]++ │ [0,0,0,2,0,4,4,6] [3,..] │
  └─────────────┴────────────────────────────────────┴──────────────────────────┘

  Final tree (rank[0] = 3):
            ┌───┐
            │0 r3│ (root)
            └─┬─┘
        ┌────┬┴────┐
      ┌─┴─┐┌┴──┐┌─┴─┐
      │ 1 ││ 2 ││ 4 │
      └───┘└─┬─┘└─┬─┘
            ┌┴┐┌──┴──┐
            │3││ 5  6│
            └─┘└──┬──┘
                 ┌┴┐
                 │7│
                 └─┘

  Key: rank increments ONLY on equal-rank merges (7 unions, 3 increments)
```

---

## Example 9: Minimum Spanning Forest

```go
package main

import (
	"fmt"
	"sort"
)

func mstForest(n int, edges [][3]int) (int, int) {
	sort.Slice(edges, func(i, j int) bool { return edges[i][2] < edges[j][2] })

	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	totalWeight := 0
	edgesUsed := 0

	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { continue }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		if rank[rx] == rank[ry] { rank[rx]++ }
		totalWeight += e[2]
		edgesUsed++
	}

	// Count trees in forest
	trees := 0
	for i := 0; i < n; i++ {
		if find(i) == i { trees++ }
	}
	return totalWeight, trees
}

func main() {
	edges := [][3]int{{0,1,1},{1,2,2},{4,5,3},{5,6,1}}
	weight, trees := mstForest(7, edges)
	fmt.Printf("MST weight: %d, Trees: %d\n", weight, trees)
}
```

**Textual Figure:**
```
  n=7 nodes, edges: {0,1,w=1}, {1,2,w=2}, {4,5,w=3}, {5,6,w=1}

  Sorted edges by weight:
  ┌──────────┬────┬─────────────────────────────────────┐
  │ Edge     │ W  │ Action                              │
  ├──────────┼────┼─────────────────────────────────────┤
  │ (0, 1)   │  1 │ find(0)≠find(1) → Union → MST ✓    │
  │ (5, 6)   │  1 │ find(5)≠find(6) → Union → MST ✓    │
  │ (1, 2)   │  2 │ find(1)≠find(2) → Union → MST ✓    │
  │ (4, 5)   │  3 │ find(4)≠find(5) → Union → MST ✓    │
  └──────────┴────┴─────────────────────────────────────┘

  MST forest (disconnected graph → forest, not single tree):

    Tree 1:             Tree 2:         Isolated:
      ┌───┐               ┌───┐          ┌───┐
      │ 0 │ (root,r=1)    │ 5 │ (r=1)    │ 3 │
      └─┬─┘               └─┬─┘          └───┘
     ┌──┴──┐             ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐         ┌─┴─┐ ┌─┴─┐
   │ 1 │ │ 2 │         │ 4 │ │ 6 │
   └───┘ └───┘         └───┘ └───┘

  Total weight: 1+1+2+3 = 7
  Number of trees: 3  (nodes 3 has no edges)
```

---

## Example 10: Union by Rank vs Naive Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Union by Rank vs Naive ===")
	fmt.Println()

	rows := []struct{ aspect, naive, ranked string }{
		{"Tree height", "O(n) worst", "O(log n) worst"},
		{"Find time", "O(n)", "O(log n)"},
		{"Extra space", "None", "O(n) rank array"},
		{"With compression", "O(log n) amortized", "O(α(n)) amortized"},
		{"Rank update rule", "N/A", "Increment only when equal ranks merge"},
		{"Rank meaning", "N/A", "Upper bound on subtree height"},
		{"Rank never decreases", "N/A", "True — monotonically increasing"},
		{"Max rank", "N/A", "⌊log₂(n)⌋"},
		{"Implementation", "1 line union", "5 line union"},
		{"Practical speed", "Very slow for chains", "Fast, balanced trees"},
	}

	fmt.Printf("%-22s %-22s %-22s\n", "Aspect", "Naive", "Union by Rank")
	fmt.Println("------------------------------------------------------------------")
	for _, r := range rows {
		fmt.Printf("%-22s %-22s %-22s\n", r.aspect, r.naive, r.ranked)
	}
}
```

**Textual Figure:**
```
  Union by Rank vs Naive — Visual Comparison:
  ════════════════════════════════════════════

  NAIVE UNION (n=8):              UNION BY RANK (n=8):
  Sequential unions → chain      Same unions → balanced tree

    ┌─┐                              ┌───┐
    │7│ ← root                       │0 r3│ ← root
    └┬┘                              └─┬─┘
    ┌┴┐                          ┌────┬┴────┐
    │6│                        ┌─┴─┐┌┴──┐┌─┴─┐
    └┬┘                        │ 1 ││ 2 ││ 4 │
    ┌┴┐                        └───┘└─┬─┘└─┬─┘
    │5│                              │  ┌──┴──┐
    └┬┘                             ┌┴┐┌┴─┐ ┌┴┐
    ┌┴┐                             │3││5 │ │6│
    │4│                             └─┘└──┘ └┬┘
    └┬┘                                     ┌┴┐
    ...                                     │7│
    ┌┴┐                                     └─┘
    │0│
    └─┘
    Height = 7 = O(n)            Height = 3 = O(log n)

  ┌─────────────────────┬─────────────┬──────────────────┐
  │ Aspect              │ Naive       │ Union by Rank    │
  ├─────────────────────┼─────────────┼──────────────────┤
  │ Tree height         │ O(n)        │ O(log n)         │
  │ Find time           │ O(n)        │ O(log n)         │
  │ Extra space         │ None        │ O(n) rank array  │
  │ With compression    │ O(log n)    │ O(α(n)) ≈ O(1)  │
  │ Max rank            │ N/A         │ ⌊log₂(n)⌋       │
  └─────────────────────┴─────────────┴──────────────────┘

  Practical impact: n=10⁶
    Naive:  up to 10⁶ steps per Find
    Ranked: ≤ 20 steps per Find  (log₂(10⁶) ≈ 20)
```

---

## Key Takeaways

1. Union by rank attaches shorter tree under taller → O(log n) height
2. Rank = upper bound on height, not exact (compression doesn't update rank)
3. Rank increases only when two equal-rank trees merge
4. A tree of rank r has at least 2^r nodes → max rank ≤ log₂(n)
5. Combined with path compression → O(α(n)) per operation, effectively O(1)

> **Next up:** Union by Size →
