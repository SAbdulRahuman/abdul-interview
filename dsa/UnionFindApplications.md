# Phase 13: Union Find — Applications

## Overview

Union-Find (Disjoint Set) has wide-ranging applications in graph algorithms, network analysis, and equivalence class problems.

| Application | What Union-Find Does | Why It's Best |
|-------------|---------------------|---------------|
| **Kruskal's MST** | Detect if edge creates cycle | O(E log E) total |
| **Cycle detection** | Check if endpoints share root | O(α(n)) per edge |
| **Network connectivity** | Track connected components | Online updates |
| **Percolation** | Check top-to-bottom path | Dynamic grid |
| **Equivalence classes** | Group equal elements | O(α(n)) merge |
| **Image segmentation** | Merge adjacent similar pixels | O(pixels × α(n)) |
| **Least common ancestor** | Offline LCA queries | Tarjan's algorithm |

---

## Example 1: Kruskal's Minimum Spanning Tree

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func kruskal(n int, edges []Edge) (int, []Edge) {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	mst := []Edge{}
	totalWeight := 0
	for _, e := range edges {
		rx, ry := find(e.u), find(e.v)
		if rx == ry { continue }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		if rank[rx] == rank[ry] { rank[rx]++ }
		mst = append(mst, e)
		totalWeight += e.w
		if len(mst) == n-1 { break }
	}
	return totalWeight, mst
}

func main() {
	edges := []Edge{{0,1,4},{0,2,3},{1,2,1},{1,3,2},{2,3,5},{3,4,7},{2,4,6}}
	w, mst := kruskal(5, edges)
	fmt.Printf("MST weight: %d\n", w)
	for _, e := range mst {
		fmt.Printf("  %d─%d (w=%d)\n", e.u, e.v, e.w)
	}
}
```

**Textual Figure:**
```
  Graph with 5 vertices, 7 edges:
       4         7
    0─────1    3─────4
    │╲    │    │    ╱
   3│  ╲  │2   │5  ╱6
    │   1╲│    │ ╱
    2─────1    2

  Sorted edges by weight:
  ┌──────┬────┬───────────────────────────────────────────────┐
  │ Edge │ W  │ Action                                        │
  ├──────┼────┼───────────────────────────────────────────────┤
  │ 1─2  │  1 │ find(1)=1 ≠ find(2)=2 → Union → MST edge ✓ │
  │ 1─3  │  2 │ find(1)=1 ≠ find(3)=3 → Union → MST edge ✓ │
  │ 0─2  │  3 │ find(0)=0 ≠ find(2)=1 → Union → MST edge ✓ │
  │ 0─1  │  4 │ find(0)=1 == find(1)=1 → SKIP (cycle)       │
  │ 2─3  │  5 │ find(2)=1 == find(3)=1 → SKIP (cycle)       │
  │ 2─4  │  6 │ find(2)=1 ≠ find(4)=4 → Union → MST edge ✓ │
  │ 3─4  │  7 │ 4 MST edges found → DONE                     │
  └──────┴────┴───────────────────────────────────────────────┘

  MST (weight = 1+2+3+6 = 12):
      0───3───2───1───1───6───4
          │       │
          0       3
    0─────2   1───3

  Union-Find forest at completion:
        ┌───┐
        │ 1 │  (root, rank=2)
        └─┬─┘
      ┌───┼───┐
    ┌─┴─┐│ ┌─┴─┐
    │ 2 ││ │ 3 │
    └─┬─┘│ └───┘
      │  │
    ┌─┴┐┌┴─┐
    │ 0││ 4│
    └──┘└──┘
```

---

## Example 2: Network Connectivity Check

```go
package main

import "fmt"

type Network struct {
	parent []int
	rank   []int
	count  int
}

func NewNetwork(n int) *Network {
	p := make([]int, n)
	r := make([]int, n)
	for i := range p { p[i] = i }
	return &Network{p, r, n}
}

func (net *Network) Find(x int) int {
	if net.parent[x] != x { net.parent[x] = net.Find(net.parent[x]) }
	return net.parent[x]
}

func (net *Network) Connect(x, y int) bool {
	rx, ry := net.Find(x), net.Find(y)
	if rx == ry { return false }
	if net.rank[rx] < net.rank[ry] { rx, ry = ry, rx }
	net.parent[ry] = rx
	if net.rank[rx] == net.rank[ry] { net.rank[rx]++ }
	net.count--
	return true
}

func (net *Network) IsConnected(x, y int) bool { return net.Find(x) == net.Find(y) }

func main() {
	net := NewNetwork(6)
	connections := [][2]int{{0,1},{1,2},{3,4},{4,5}}
	for _, c := range connections {
		net.Connect(c[0], c[1])
	}
	fmt.Printf("Components: %d\n", net.count) // 2
	fmt.Println("0↔2:", net.IsConnected(0, 2)) // true
	fmt.Println("0↔3:", net.IsConnected(0, 3)) // false

	net.Connect(2, 3) // Bridge the two networks
	fmt.Printf("After bridge: components=%d\n", net.count) // 1
	fmt.Println("0↔5:", net.IsConnected(0, 5)) // true
}
```

**Textual Figure:**
```
  6 servers, 4 initial connections:

  After Connect(0,1), Connect(1,2), Connect(3,4), Connect(4,5):

    Network A:          Network B:
      ┌───┐               ┌───┐
      │ 0 │ (root,r=1)    │ 3 │ (root,r=1)
      └─┬─┘               └─┬─┘
        │                    │
      ┌─┴─┐               ┌─┴─┐
      │ 1 │               │ 4 │
      └─┬─┘               └─┬─┘
        │                    │
      ┌─┴─┐               ┌─┴─┐
      │ 2 │               │ 5 │
      └───┘               └───┘

    IsConnected(0,2) → Find(0)=0, Find(2)=0 → true ✓
    IsConnected(0,3) → Find(0)=0, Find(3)=3 → false ✓

  After Connect(2,3): bridges the two networks
    Find(2)=0, Find(3)=3 → Union → parent[3]=0

        ┌───┐
        │ 0 │ (root, rank=2)
        └─┬─┘
      ┌───┴───┐
    ┌─┴─┐   ┌─┴─┐
    │ 1 │   │ 3 │
    └─┬─┘   └─┬─┘
      │       │
    ┌─┴─┐   ┌─┴─┐
    │ 2 │   │ 4 │
    └───┘   └─┬─┘
                │
              ┌─┴─┐
              │ 5 │
              └───┘

    Components: 1    IsConnected(0,5) → true ✓
```

---

## Example 3: Cycle Detection in Undirected Graph

```go
package main

import "fmt"

func hasCycle(n int, edges [][2]int) (bool, [2]int) {
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { return true, e }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		if rank[rx] == rank[ry] { rank[rx]++ }
	}
	return false, [2]int{}
}

func main() {
	edges := [][2]int{{0,1},{1,2},{2,3},{3,1}}
	cycle, edge := hasCycle(4, edges)
	fmt.Printf("Cycle: %v, edge: %v\n", cycle, edge) // true, [3 1]
}
```

**Textual Figure:**
```
  Graph: 4 vertices, edges {0,1},{1,2},{2,3},{3,1}

    0 ─── 1
          │╲
          │  3
          │╱
          2

  Union-Find trace:
  ┌────────┬────────────┬────────────────────────────────┐
  │ Edge   │ find(u,v)  │ Action                         │
  ├────────┼────────────┼────────────────────────────────┤
  │ {0,1}  │ 0 ≠ 1     │ Union(0,1): parent[1]=0        │
  │ {1,2}  │ 0 ≠ 2     │ Union(0,2): parent[2]=0        │
  │ {2,3}  │ 0 ≠ 3     │ Union(0,3): parent[3]=0        │
  │ {3,1}  │ 0 == 0    │ SAME ROOT → CYCLE DETECTED!    │
  └────────┴────────────┴────────────────────────────────┘

  Forest at detection point:
      ┌───┐
      │ 0 │ (root, rank=1)
      └─┬─┘
    ┌───┼───┐
  ┌─┴─┐│ ┌─┴─┐
  │ 1 ││ │ 2 │     Edge {3,1}: both 3 and 1 → root 0
  └───┘│ └───┘     → Adding this edge creates a cycle!
     ┌─┴─┐
     │ 3 │
     └───┘

  Result: cycle=true, offending edge=[3, 1]
```

---

## Example 4: Percolation System

```go
package main

import "fmt"

type Percolation struct {
	parent []int
	rank   []int
	open   []bool
	n      int
	top    int // virtual top node
	bottom int // virtual bottom node
}

func NewPercolation(n int) *Percolation {
	size := n*n + 2
	p := make([]int, size)
	for i := range p { p[i] = i }
	return &Percolation{
		parent: p, rank: make([]int, size),
		open: make([]bool, n*n), n: n,
		top: n * n, bottom: n*n + 1,
	}
}

func (perc *Percolation) find(x int) int {
	if perc.parent[x] != x { perc.parent[x] = perc.find(perc.parent[x]) }
	return perc.parent[x]
}

func (perc *Percolation) union(x, y int) {
	rx, ry := perc.find(x), perc.find(y)
	if rx == ry { return }
	if perc.rank[rx] < perc.rank[ry] { rx, ry = ry, rx }
	perc.parent[ry] = rx
	if perc.rank[rx] == perc.rank[ry] { perc.rank[rx]++ }
}

func (perc *Percolation) Open(row, col int) {
	id := row*perc.n + col
	perc.open[id] = true

	if row == 0 { perc.union(id, perc.top) }
	if row == perc.n-1 { perc.union(id, perc.bottom) }

	dirs := [4][2]int{{-1,0},{1,0},{0,-1},{0,1}}
	for _, d := range dirs {
		nr, nc := row+d[0], col+d[1]
		if nr >= 0 && nr < perc.n && nc >= 0 && nc < perc.n && perc.open[nr*perc.n+nc] {
			perc.union(id, nr*perc.n+nc)
		}
	}
}

func (perc *Percolation) Percolates() bool {
	return perc.find(perc.top) == perc.find(perc.bottom)
}

func main() {
	p := NewPercolation(3)
	opens := [][2]int{{0,1},{1,1},{2,1}}
	for _, o := range opens {
		p.Open(o[0], o[1])
		fmt.Printf("Open(%d,%d) → percolates: %v\n", o[0], o[1], p.Percolates())
	}
}
```

**Textual Figure:**
```
  3×3 Percolation grid with virtual top (T) and bottom (B) nodes:

  Cell IDs:  0  1  2          Virtual nodes: T=9, B=10
             3  4  5
             6  7  8

  Step 1: Open(0,1) → open cell 1, union with T (row 0)
    ┌───┬───┬───┐
    │ ░ │ ■ │ ░ │   ■ = open, ░ = blocked
    ├───┼───┼───┤   parent[1] = T
    │ ░ │ ░ │ ░ │   Percolates? Find(T) ≠ Find(B) → false
    ├───┼───┼───┤
    │ ░ │ ░ │ ░ │
    └───┴───┴───┘

  Step 2: Open(1,1) → open cell 4, union with cell 1 (neighbor above)
    ┌───┬───┬───┐
    │ ░ │ ■ │ ░ │   Union(4, 1) → both under T
    ├───┼───┼───┤   Percolates? Find(T) ≠ Find(B) → false
    │ ░ │ ■ │ ░ │
    ├───┼───┼───┤
    │ ░ │ ░ │ ░ │
    └───┴───┴───┘

  Step 3: Open(2,1) → open cell 7, union with cell 4 AND union with B (row 2)
    ┌───┬───┬───┐
    │ ░ │ ■ │ ░ │   Union(7, 4) → chain to T
    ├───┼───┼───┤   Union(7, B) → B joins T's tree
    │ ░ │ ■ │ ░ │
    ├───┼───┼───┤   Percolates? Find(T) == Find(B) → true ✓
    │ ░ │ ■ │ ░ │
    └───┴───┴───┘

  Union-Find forest:
       ┌───┐
       │ T │ (virtual top, root)
       └─┬─┘
         │
       ┌─┴─┐
       │ 1 │ (0,1)
       └─┬─┘
         │
       ┌─┴─┐
       │ 4 │ (1,1)
       └─┬─┘
         │
       ┌─┴─┐
       │ 7 │ (2,1)
       └─┬─┘
         │
       ┌─┴─┐
       │ B │ (virtual bottom)
       └───┘

  T and B share same root → system percolates!
```

---

## Example 5: Friend Circles / Social Network

```go
package main

import "fmt"

func findCircleNum(isConnected [][]int) int {
	n := len(isConnected)
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			if isConnected[i][j] == 1 {
				rx, ry := find(i), find(j)
				if rx != ry {
					if rank[rx] < rank[ry] { rx, ry = ry, rx }
					parent[ry] = rx
					if rank[rx] == rank[ry] { rank[rx]++ }
				}
			}
		}
	}

	provinces := 0
	for i := 0; i < n; i++ {
		if find(i) == i { provinces++ }
	}
	return provinces
}

func main() {
	adj := [][]int{
		{1, 1, 0, 0},
		{1, 1, 1, 0},
		{0, 1, 1, 0},
		{0, 0, 0, 1},
	}
	fmt.Println("Friend circles:", findCircleNum(adj)) // 2
}
```

**Textual Figure:**
```
  Adjacency matrix (4 people):
       0  1  2  3
    0 [1, 1, 0, 0]    0 ↔ 1 (friends)
    1 [1, 1, 1, 0]    1 ↔ 2 (friends)
    2 [0, 1, 1, 0]
    3 [0, 0, 0, 1]    3 is alone

  Social graph:
    ┌───┐     ┌───┐
    │ 0 │─────│ 1 │       ┌───┐
    └───┘     └─┬─┘       │ 3 │ (isolated)
                │         └───┘
              ┌─┴─┐
              │ 2 │
              └───┘

  Union-Find trace:
  ┌─────────┬────────────────────────────┬──────────────┐
  │ Pair    │ Action                     │ parent[]     │
  ├─────────┼────────────────────────────┼──────────────┤
  │ (0, 1)  │ Union(0,1): parent[1]=0   │ [0,0,2,3]   │
  │ (1, 2)  │ Union(0,2): parent[2]=0   │ [0,0,0,3]   │
  └─────────┴────────────────────────────┴──────────────┘

  Final forest:
    Circle 1:         Circle 2:
      ┌───┐             ┌───┐
      │ 0 │ (root)      │ 3 │ (root)
      └─┬─┘             └───┘
     ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐
   │ 1 │ │ 2 │
   └───┘ └───┘

  Roots: {0, 3} → 2 friend circles
```

---

## Example 6: Equivalence Classes (Equation Satisfaction)

```go
package main

import "fmt"

func equationsPossible(equations []string) bool {
	parent := [26]int{}
	for i := range parent { parent[i] = i }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	// Phase 1: process equalities
	for _, eq := range equations {
		if eq[1] == '=' {
			parent[find(int(eq[0]-'a'))] = find(int(eq[3]-'a'))
		}
	}

	// Phase 2: check inequalities
	for _, eq := range equations {
		if eq[1] == '!' {
			if find(int(eq[0]-'a')) == find(int(eq[3]-'a')) { return false }
		}
	}
	return true
}

func main() {
	fmt.Println(equationsPossible([]string{"a==b", "b==c", "c!=a"})) // false
	fmt.Println(equationsPossible([]string{"a==b", "c==d", "a!=d"})) // true
}
```

**Textual Figure:**
```
  Test 1: ["a==b", "b==c", "c!=a"]

  Phase 1 — Process equalities (==):
    a==b → Union(a, b):  parent[a] = b
    b==c → Union(b, c):  parent[b] = c

    Equivalence class:
      ┌───┐
      │ c │ (root)
      └─┬─┘
        │
      ┌─┴─┐
      │ b │
      └─┬─┘
        │
      ┌─┴─┐
      │ a │
      └───┘
    All in same class: {a, b, c}

  Phase 2 — Check inequalities (!=):
    c!=a → Find(c)=c, Find(a)=c → SAME ROOT!
    Contradiction: a must equal c, but equation says a ≠ c
    Result: false ✗

  Test 2: ["a==b", "c==d", "a!=d"]

  Phase 1:
    a==b → {a, b}    c==d → {c, d}

    ┌───┐   ┌───┐
    │ b │   │ d │
    └─┬─┘   └─┬─┘
    ┌─┴─┐   ┌─┴─┐
    │ a │   │ c │
    └───┘   └───┘

  Phase 2:
    a!=d → Find(a)=b, Find(d)=d → different roots
    No contradiction → Result: true ✓
```

---

## Example 7: Largest Component After Removing Node

```go
package main

import "fmt"

func largestComponentWithout(n int, edges [][2]int, remove int) int {
	parent := make([]int, n)
	rank := make([]int, n)
	size := make([]int, n)
	for i := range parent { parent[i] = i; size[i] = 1 }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		rx, ry := find(x), find(y)
		if rx == ry { return }
		if rank[rx] < rank[ry] { rx, ry = ry, rx }
		parent[ry] = rx
		size[rx] += size[ry]
		if rank[rx] == rank[ry] { rank[rx]++ }
	}

	for _, e := range edges {
		if e[0] == remove || e[1] == remove { continue }
		union(e[0], e[1])
	}

	maxSize := 0
	for i := 0; i < n; i++ {
		if i != remove && find(i) == i && size[i] > maxSize {
			maxSize = size[i]
		}
	}
	return maxSize
}

func main() {
	edges := [][2]int{{0,1},{1,2},{2,3},{3,4},{1,3}}
	fmt.Println("Remove node 1, largest:", largestComponentWithout(5, edges, 1))
	fmt.Println("Remove node 2, largest:", largestComponentWithout(5, edges, 2))
}
```

**Textual Figure:**
```
  Original graph (5 nodes):
    0 ─── 1 ─── 2
          │╲    │
          │  ╲  │
          │   ╲ │
          3 ─── 4
          (actually: 0-1, 1-2, 2-3, 3-4, 1-3)

    0 ── 1 ── 2
         │    │
         3 ── 4    (with edge 1-3)

  Remove node 1:
    Skip edges: {0,1}, {1,2}, {1,3}
    Process: {2,3}, {3,4}
    ┌───┐     ┌───┐
    │ 0 │     │ 2 │ (root)     Node 1 removed
    └───┘     └─┬─┘
                │
              ┌─┴─┐
              │ 3 │
              └─┬─┘
                │
              ┌─┴─┐
              │ 4 │
              └───┘
    Largest component: {2,3,4} → size = 3

  Remove node 2:
    Skip edges: {1,2}, {2,3}
    Process: {0,1}, {3,4}, {1,3}
      ┌───┐
      │ 0 │ (root)    Node 2 removed
      └─┬─┘
     ┌──┴──┐
   ┌─┴─┐ ┌─┴─┐
   │ 1 │ │ 3 │
   └───┘ └─┬─┘
            │
          ┌─┴─┐
          │ 4 │
          └───┘
    Largest component: {0,1,3,4} → size = 4
```

---

## Example 8: Regions Cut By Slashes (LeetCode 959)

```go
package main

import "fmt"

func regionsBySlashes(grid []string) int {
	n := len(grid)
	// Each cell divided into 4 triangles: 0=top, 1=right, 2=bottom, 3=left
	size := n * n * 4
	parent := make([]int, size)
	for i := range parent { parent[i] = i }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		rx, ry := find(x), find(y)
		if rx != ry { parent[rx] = ry }
	}

	idx := func(r, c, t int) int { return (r*n+c)*4 + t }

	for r := 0; r < n; r++ {
		for c := 0; c < n; c++ {
			ch := grid[r][c]
			// Internal merges based on character
			if ch == '/' {
				union(idx(r,c,0), idx(r,c,3)) // top+left
				union(idx(r,c,1), idx(r,c,2)) // right+bottom
			} else if ch == '\\' {
				union(idx(r,c,0), idx(r,c,1)) // top+right
				union(idx(r,c,2), idx(r,c,3)) // bottom+left
			} else {
				union(idx(r,c,0), idx(r,c,1))
				union(idx(r,c,1), idx(r,c,2))
				union(idx(r,c,2), idx(r,c,3))
			}
			// Merge with neighbors
			if r+1 < n { union(idx(r,c,2), idx(r+1,c,0)) }
			if c+1 < n { union(idx(r,c,1), idx(r,c+1,3)) }
		}
	}

	regions := 0
	for i := 0; i < size; i++ {
		if find(i) == i { regions++ }
	}
	return regions
}

func main() {
	grid := []string{" /", "/ "}
	fmt.Println("Regions:", regionsBySlashes(grid)) // 2
}
```

**Textual Figure:**
```
  Grid: [" /", "/ "]  (2×2)

  Each cell split into 4 triangles:
    ┌───────┐
    │╲  0  ╱│     0 = top triangle
    │ ╲   ╱ │     1 = right triangle
    │3 ╲ ╱ 1│     2 = bottom triangle
    │   X   │     3 = left triangle
    │3 ╱ ╲ 1│
    │ ╱   ╲ │
    │╱  2  ╲│
    └───────┘

  Cell (0,0) = ' ':  merge all 4 → one region
  Cell (0,1) = '/':  merge {top,left} and {right,bottom}
  Cell (1,0) = '/':  merge {top,left} and {right,bottom}
  Cell (1,1) = ' ':  merge all 4 → one region

  After internal merges + neighbor merges:
    ┌─────┬─────┐
    │     │   ╱ │
    │ all │  ╱  │     Cell(0,0) all merged
    │     │ ╱   │     Cell(0,1) split by /
    ├─────┼─────┤
    │   ╱ │     │
    │  ╱  │ all │     Cell(1,0) split by /
    │ ╱   │     │     Cell(1,1) all merged
    └─────┴─────┘

  Neighbor unions connect across boundaries:
    Region A: top-left area (connected through boundaries)
    Region B: bottom-right area

  Result: 2 regions
```

---

## Example 9: Smallest String With Swaps (LeetCode 1202)

```go
package main

import (
	"fmt"
	"sort"
)

func smallestStringWithSwaps(s string, pairs [][]int) string {
	n := len(s)
	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, p := range pairs {
		rx, ry := find(p[0]), find(p[1])
		if rx != ry { parent[rx] = ry }
	}

	// Group indices by component
	groups := map[int][]int{}
	for i := 0; i < n; i++ {
		root := find(i)
		groups[root] = append(groups[root], i)
	}

	result := []byte(s)
	for _, indices := range groups {
		chars := make([]byte, len(indices))
		for i, idx := range indices { chars[i] = s[idx] }
		sort.Slice(chars, func(i, j int) bool { return chars[i] < chars[j] })
		sort.Ints(indices)
		for i, idx := range indices { result[idx] = chars[i] }
	}
	return string(result)
}

func main() {
	fmt.Println(smallestStringWithSwaps("dcab", [][]int{{0,3},{1,2}}))       // "bacd"
	fmt.Println(smallestStringWithSwaps("dcab", [][]int{{0,3},{1,2},{0,2}})) // "abcd"
}
```

**Textual Figure:**
```
  Test 1: s="dcab", pairs=[[0,3],[1,2]]

  Union-Find on index pairs:
    Union(0,3) → {0, 3}    Union(1,2) → {1, 2}

    ┌───┐ ┌───┐
    │ 3 │ │ 2 │   Two components
    └─┬─┘ └─┬─┘
    ┌─┴─┐ ┌─┴─┐
    │ 0 │ │ 1 │
    └───┘ └───┘

  Groups:
    Component {0,3}: chars = s[0],s[3] = 'd','b' → sorted: 'b','d'
    Component {1,2}: chars = s[1],s[2] = 'c','a' → sorted: 'a','c'

    Index:  0  1  2  3
    Before: d  c  a  b
    After:  b  a  c  d  → "bacd"

  Test 2: pairs=[[0,3],[1,2],[0,2]]  → all indices connected!

    Union(0,3), Union(1,2), Union(0,2):
        ┌───┐
        │ 2 │ (root)
        └─┬─┘
      ┌──┬┴──┐
    ┌─┴┐┌┴─┐┌┴─┐
    │ 3││ 1││ 0│
    └──┘└──┘└──┘

  One component {0,1,2,3}: chars 'd','c','a','b' → sorted 'a','b','c','d'
  Result: "abcd"
```

---

## Example 10: Union-Find Applications Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Union-Find Applications Summary ===")
	fmt.Println()

	apps := []struct{ name, problem, whyUF string }{
		{"Kruskal's MST", "Minimum cost to connect all nodes", "Cycle detection per edge O(α(n))"},
		{"Network Connectivity", "Are two nodes connected?", "Dynamic edge insertion + query"},
		{"Cycle Detection", "Does adding edge create cycle?", "Same-root check in O(α(n))"},
		{"Percolation", "Does liquid flow top to bottom?", "Virtual nodes + dynamic open"},
		{"Friend Circles", "Count social groups", "Merge friends, count roots"},
		{"Equation Satisfaction", "Can all == and != hold?", "Equivalence classes of variables"},
		{"Image Segmentation", "Group similar adjacent pixels", "Merge neighboring pixels"},
		{"Redundant Connection", "Find cycle-causing edge", "First edge with same root"},
		{"Accounts Merge", "Merge accounts sharing email", "Emails as union keys"},
		{"Dynamic Connectivity", "Online connect/query", "No disconnect support"},
	}

	fmt.Printf("%-25s %-40s %s\n", "Application", "Problem", "Why Union-Find")
	fmt.Println("─────────────────────────────────────────────────────────────────────────────────────────────────────")
	for _, a := range apps {
		fmt.Printf("%-25s %-40s %s\n", a.name, a.problem, a.whyUF)
	}
}
```

**Textual Figure:**
```
  Union-Find Application Decision Tree:
  ═══════════════════════════════════════

  Is the problem about grouping/connectivity?
  │
  ├─ YES → Are elements merged dynamically?
  │        │
  │        ├─ YES → Do you need to split/disconnect?
  │        │        │
  │        │        ├─ YES → Union-Find CANNOT help
  │        │        │        (use other structure)
  │        │        │
  │        │        └─ NO → ✓ USE UNION-FIND
  │        │                 │
  │        │                 ├─ Cycle detection?
  │        │                 │   → Check if same root before union
  │        │                 │
  │        │                 ├─ Count components?
  │        │                 │   → Track count, decrement on union
  │        │                 │
  │        │                 ├─ MST (Kruskal)?
  │        │                 │   → Sort edges + union if different roots
  │        │                 │
  │        │                 └─ Equivalence classes?
  │        │                     → Union equals, check not-equals
  │        │
  │        └─ NO (static) → DFS/BFS may be simpler
  │
  └─ NO → Union-Find not applicable

  Complexity:  m operations on n elements → O(m · α(n)) ≈ O(m)
```

---

## Key Takeaways

1. Union-Find excels at dynamic connectivity: merge sets, query membership
2. Cannot split sets — only merge (offline deletion needs offline algorithms)
3. Kruskal's MST: sort edges + union-find for cycle detection
4. Percolation: virtual top/bottom nodes simplify connectivity check
5. Equivalence classes: union on equals, query on not-equals

> **Next up:** Union by Size →
