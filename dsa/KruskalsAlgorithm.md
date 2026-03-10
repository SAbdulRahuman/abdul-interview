# Phase 12: Graphs — Kruskal's Algorithm

## Overview

**Kruskal's** builds an MST by sorting all edges by weight and adding them greedily, skipping those that would create a cycle (detected via Union-Find).

- **Time:** O(E log E) for sorting + O(E α(V)) for union-find ≈ O(E log E)
- **Space:** O(V) for Union-Find
- Best for **sparse graphs** where E is close to V

---

## Example 1: Basic Kruskal's

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func kruskal(n int, edges []Edge) ([]Edge, int) {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) bool {
		px, py := find(x), find(y)
		if px == py { return false }
		if rank[px] < rank[py] { px, py = py, px }
		parent[py] = px
		if rank[px] == rank[py] { rank[px]++ }
		return true
	}

	mst := []Edge{}
	total := 0
	for _, e := range edges {
		if union(e.u, e.v) {
			mst = append(mst, e)
			total += e.w
			if len(mst) == n-1 { break }
		}
	}
	return mst, total
}

func main() {
	edges := []Edge{{0,1,10},{0,2,6},{0,3,5},{1,3,15},{2,3,4}}
	mst, w := kruskal(4, edges)
	for _, e := range mst { fmt.Printf("%d--%d (w=%d)\n", e.u, e.v, e.w) }
	fmt.Println("Total:", w) // 19
}
```

**Textual Figure:**
```
  Graph (undirected, weighted):              MST Result (weight = 19):

       0 ──(10)── 1                              0 ──(10)── 1
       │╲         │                              │
      (6) (5)   (15)                            (5)
       │    ╲     │                               │
       2 ──(4)── 3                              3 ──(4)── 2

  Kruskal's Step-by-Step (edges sorted by weight):
  ┌──────┬────────────┬────────┬─────────┬────────────────────┐
  │ Step │ Edge       │ Action │ Running │ Components         │
  │      │            │        │ Total   │                    │
  ├──────┼────────────┼────────┼─────────┼────────────────────┤
  │  1   │ 2─3 (w=4) │ ADD    │    4    │ {2,3}{0}{1}        │
  │  2   │ 0─3 (w=5) │ ADD    │    9    │ {0,2,3}{1}         │
  │  3   │ 0─2 (w=6) │ SKIP   │    9    │ cycle! 0,2 same set │
  │  4   │ 0─1 (w=10)│ ADD    │   19    │ {0,1,2,3}          │
  │  5   │ 1─3 (w=15)│ SKIP   │   19    │ cycle! (done)      │
  └──────┴────────────┴────────┴─────────┴────────────────────┘

  MST Edges: 2─3(4), 0─3(5), 0─1(10)  →  Total = 19
```

---

## Example 2: Kruskal's with Union-by-Size

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }
type DSU struct {
	parent, size []int
}

func NewDSU(n int) *DSU {
	p, s := make([]int, n), make([]int, n)
	for i := range p { p[i] = i; s[i] = 1 }
	return &DSU{p, s}
}

func (d *DSU) Find(x int) int {
	if d.parent[x] != x { d.parent[x] = d.Find(d.parent[x]) }
	return d.parent[x]
}

func (d *DSU) Union(x, y int) bool {
	px, py := d.Find(x), d.Find(y)
	if px == py { return false }
	if d.size[px] < d.size[py] { px, py = py, px }
	d.parent[py] = px
	d.size[px] += d.size[py]
	return true
}

func main() {
	edges := []Edge{{0,1,1},{1,2,2},{2,3,3},{3,4,4},{0,4,5},{1,3,6}}
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	dsu := NewDSU(5)
	total := 0
	for _, e := range edges {
		if dsu.Union(e.u, e.v) {
			total += e.w
			fmt.Printf("Add: %d--%d (w=%d)\n", e.u, e.v, e.w)
		}
	}
	fmt.Println("MST weight:", total) // 10
}
```

**Textual Figure:**
```
  Graph:  0 ─(1)─ 1 ─(2)─ 2 ─(3)─ 3 ─(4)─ 4
          │                  │             │
          └─────(5)───────────────────┘
                             │
                 1 ───(6)─── 3

  Kruskal's with Union-by-Size:
  ┌──────┬────────────┬────────┬─────────┬───────────────────┐
  │ Step │ Edge       │ Action │ Running │ DSU state         │
  ├──────┼────────────┼────────┼─────────┼───────────────────┤
  │  1   │ 0─1 (w=1) │ ADD    │    1    │ {0,1}{2}{3}{4}    │
  │  2   │ 1─2 (w=2) │ ADD    │    3    │ {0,1,2}{3}{4}     │
  │  3   │ 2─3 (w=3) │ ADD    │    6    │ {0,1,2,3}{4}      │
  │  4   │ 3─4 (w=4) │ ADD    │   10    │ {0,1,2,3,4}       │
  │  5   │ 0─4 (w=5) │ SKIP   │   10    │ cycle (done)      │
  │  6   │ 1─3 (w=6) │ SKIP   │   10    │ cycle             │
  └──────┴────────────┴────────┴─────────┴───────────────────┘

  MST = {0─1(1), 1─2(2), 2─3(3), 3─4(4)} = 10
```

---

## Example 3: Min Cost to Connect All Points (LeetCode 1584)

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func minCostConnectPoints(points [][]int) int {
	n := len(points)
	edges := []Edge{}

	abs := func(x int) int { if x < 0 { return -x }; return x }

	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			w := abs(points[i][0]-points[j][0]) + abs(points[i][1]-points[j][1])
			edges = append(edges, Edge{i, j, w})
		}
	}

	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	total, count := 0, 0
	for _, e := range edges {
		px, py := find(e.u), find(e.v)
		if px == py { continue }
		parent[px] = py
		total += e.w
		count++
		if count == n-1 { break }
	}
	return total
}

func main() {
	points := [][]int{{0,0},{2,2},{3,10},{5,2},{7,0}}
	fmt.Println(minCostConnectPoints(points)) // 20
}
```

**Textual Figure:**
```
  Points (Manhattan distance):

       10│       P2(3,10)
         │        │
         │       d=9
         │        │
        2│ P1(2,2)╎····P3(5,2)        P0─P1: 4   P2─P3: 10
         │  │  d=3   d=4 │            P1─P3: 3   P2─P4: 14
        0│P0 ··········· P4(7,0)      P3─P4: 4   ...
         └───────────────────
          0  1  2  3  4  5  6  7

  Kruskal's on complete graph (all ¹⁰C₂ = 10 edges):
  ┌──────┬─────────────┬────────┬─────────┐
  │ Step │ Edge        │ Action │ Running │
  ├──────┼─────────────┼────────┼─────────┤
  │  1   │ P1─P3 (d=3)│ ADD    │    3    │
  │  2   │ P0─P1 (d=4)│ ADD    │    7    │
  │  3   │ P3─P4 (d=4)│ ADD    │   11    │
  │  4   │ P0─P3 (d=7)│ SKIP   │   11    │
  │  5   │ P1─P2 (d=9)│ ADD    │   20    │  ← done (4 edges)
  └──────┴─────────────┴────────┴─────────┘

  MST total cost = 20
```

---

## Example 4: MST with Maximum Edge Constraint

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func canConnect(n int, edges []Edge, maxWeight int) bool {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	count := 0
	for _, e := range edges {
		if e.w > maxWeight { break }
		px, py := find(e.u), find(e.v)
		if px != py {
			parent[px] = py
			count++
		}
	}
	return count == n-1
}

func main() {
	edges := []Edge{{0,1,1},{1,2,3},{2,3,5},{0,3,7}}
	fmt.Println("Max weight 5:", canConnect(4, edges, 5)) // true
	fmt.Println("Max weight 2:", canConnect(4, edges, 2)) // false
}
```

**Textual Figure:**
```
  Graph:  0 ──(1)── 1 ──(3)── 2 ──(5)── 3
          │                             │
          └──────────(7)────────────────┘

  Query 1: maxWeight = 5
  ┌──────┬────────────┬────────┬──────────────────┐
  │ Step │ Edge       │ w ≤ 5? │ Result           │
  ├──────┼────────────┼────────┼──────────────────┤
  │  1   │ 0─1 (w=1) │  Yes   │ ADD → count=1    │
  │  2   │ 1─2 (w=3) │  Yes   │ ADD → count=2    │
  │  3   │ 2─3 (w=5) │  Yes   │ ADD → count=3=n-1│
  └──────┴────────────┴────────┴──────────────────┘
  → true (count == n-1, all connected)

  Query 2: maxWeight = 2
  Only edge 0─1(w=1) qualifies → count=1 ≠ 3 → false
```

---

## Example 5: Kruskal's for MST Forest (Disconnected Graph)

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func mstForest(n int, edges []Edge) (int, int) {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	total, edgesUsed := 0, 0
	for _, e := range edges {
		px, py := find(e.u), find(e.v)
		if px != py {
			parent[px] = py
			total += e.w
			edgesUsed++
		}
	}
	components := n - edgesUsed
	return total, components
}

func main() {
	// Two components: {0,1,2} and {3,4}
	edges := []Edge{{0,1,2},{1,2,3},{3,4,1}}
	total, comp := mstForest(5, edges)
	fmt.Printf("MST forest weight: %d, components: %d\n", total, comp) // 6, 2
}
```

**Textual Figure:**
```
  Disconnected Graph:

    Component A              Component B
    0 ──(2)── 1 ──(3)── 2    3 ──(1)── 4

  Kruskal's on disconnected graph:
  ┌──────┬────────────┬────────┬───────────────────────┐
  │ Step │ Edge       │ Action │ Components              │
  ├──────┼────────────┼────────┼───────────────────────┤
  │  1   │ 3─4 (w=1) │ ADD    │ {0}{1}{2}{3,4}          │
  │  2   │ 0─1 (w=2) │ ADD    │ {0,1}{2}{3,4}           │
  │  3   │ 1─2 (w=3) │ ADD    │ {0,1,2}{3,4}            │
  └──────┴────────────┴────────┴───────────────────────┘

  edgesUsed = 3,  components = n - edgesUsed = 5 - 3 = 2
  Total MST forest weight = 1 + 2 + 3 = 6

  MST Forest (two separate trees):
    Tree A: 0─(2)─1─(3)─2     Tree B: 3─(1)─4
```

---

## Example 6: Minimum Spanning Subgraph (Remove Extra Edges)

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w, idx int }

func minEdgesToRemove(n int, edges []Edge) []int {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	inMST := map[int]bool{}
	for _, e := range edges {
		px, py := find(e.u), find(e.v)
		if px != py {
			parent[px] = py
			inMST[e.idx] = true
		}
	}

	removed := []int{}
	for _, e := range edges {
		if !inMST[e.idx] { removed = append(removed, e.idx) }
	}
	return removed
}

func main() {
	edges := []Edge{
		{0, 1, 1, 0}, {1, 2, 2, 1}, {0, 2, 3, 2},
		{2, 3, 4, 3}, {1, 3, 5, 4},
	}
	removed := minEdgesToRemove(4, edges)
	fmt.Println("Remove edge indices:", removed) // [2 4]
}
```

**Textual Figure:**
```
  Graph with indexed edges:

       0 ──(1,idx0)── 1           Sorted by weight:
       │╲            │            idx0: 0─1 w=1
      (3,idx2)  (5,idx4)          idx1: 1─2 w=2
       │    ╲        │            idx2: 0─2 w=3
       2 ─(2,idx1)─ 1            idx3: 2─3 w=4
       │            │            idx4: 1─3 w=5
       └─(4,idx3)── 3

  Kruskal's MST selection:
  ┌─────┬───────────────┬────────┬───────────┐
  │ idx │ Edge          │ Action │ In MST?   │
  ├─────┼───────────────┼────────┼───────────┤
  │  0  │ 0─1 (w=1)     │ ADD    │ ✓ Yes     │
  │  1  │ 1─2 (w=2)     │ ADD    │ ✓ Yes     │
  │  2  │ 0─2 (w=3)     │ SKIP   │ ✗ Remove  │
  │  3  │ 2─3 (w=4)     │ ADD    │ ✓ Yes     │
  │  4  │ 1─3 (w=5)     │ SKIP   │ ✗ Remove  │
  └─────┴───────────────┴────────┴───────────┘

  Removed edge indices: [2, 4] (redundant edges not in MST)
```

---

## Example 7: Critical and Pseudo-Critical MST Edges (LeetCode 1489)

```go
package main

import (
	"fmt"
	"sort"
)

func findCriticalAndPseudoCriticalEdges(n int, edges [][]int) [][]int {
	m := len(edges)
	indexed := make([][]int, m)
	for i, e := range edges {
		indexed[i] = []int{e[0], e[1], e[2], i}
	}
	sort.Slice(indexed, func(i, j int) bool { return indexed[i][2] < indexed[j][2] })

	parent := make([]int, n)
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	mstWeight := func(skip, force int) int {
		for i := range parent { parent[i] = i }
		w, cnt := 0, 0
		if force >= 0 {
			e := indexed[force]
			parent[find(e[0])] = find(e[1])
			w += e[2]; cnt++
		}
		for i, e := range indexed {
			if i == skip { continue }
			px, py := find(e[0]), find(e[1])
			if px != py {
				parent[px] = py; w += e[2]; cnt++
			}
		}
		if cnt != n-1 { return 1<<31 - 1 }
		return w
	}

	base := mstWeight(-1, -1)
	var critical, pseudo []int
	for i := range indexed {
		if mstWeight(i, -1) > base {
			critical = append(critical, indexed[i][3])
		} else if mstWeight(-1, i) == base {
			pseudo = append(pseudo, indexed[i][3])
		}
	}
	return [][]int{critical, pseudo}
}

func main() {
	edges := [][]int{{0,1,1},{1,2,1},{2,3,2},{0,3,2},{0,2,2},{1,3,1}}
	result := findCriticalAndPseudoCriticalEdges(4, edges)
	fmt.Println("Critical:", result[0])
	fmt.Println("Pseudo-critical:", result[1])
}
```

**Textual Figure:**
```
  Graph: edges = {0-1(1), 1-2(1), 2-3(2), 0-3(2), 0-2(2), 1-3(1)}

       0 ──(1)── 1       Base MST weight:
       │╲      │╲        1+1+1 = 3 (using 3 weight-1 edges)
      (2)(2) (1)(1)
       │    ╳    │
       3 ──(2)── 2

  Critical/Pseudo-Critical Analysis:
  ┌─────┬────────────┬──────┬────────────────────────────────┐
  │ idx │ Edge       │ Type │ Reason                         │
  ├─────┼────────────┼──────┼────────────────────────────────┤
  │  0  │ 0─1 (w=1) │ Crit │ Skip → MST > 3 (no alt w=1)   │
  │  1  │ 1─2 (w=1) │ Crit │ Skip → MST > 3                │
  │  5  │ 1─3 (w=1) │ Crit │ Skip → MST > 3                │
  │  2  │ 2─3 (w=2) │ Pse  │ Force → MST still = 3         │
  │  3  │ 0─3 (w=2) │ Pse  │ Force → MST still = 3         │
  │  4  │ 0─2 (w=2) │ Pse  │ Force → MST still = 3         │
  └─────┴────────────┴──────┴────────────────────────────────┘

  Critical: must exist in every MST (removing increases cost)
  Pseudo-critical: can appear in some MST (forcing doesn't increase cost)
```

---

## Example 8: Kruskal's Step-by-Step Trace

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func kruskalTrace(n int, edges []Edge) {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	total := 0
	for _, e := range edges {
		px, py := find(e.u), find(e.v)
		if px == py {
			fmt.Printf("  SKIP %d--%d (w=%d) — creates cycle\n", e.u, e.v, e.w)
		} else {
			parent[px] = py
			total += e.w
			fmt.Printf("  ADD  %d--%d (w=%d) — total=%d\n", e.u, e.v, e.w, total)
		}
	}
	fmt.Println("MST weight:", total)
}

func main() {
	edges := []Edge{{0,1,4},{0,2,8},{1,2,2},{1,3,5},{2,3,6},{2,4,7},{3,4,3}}
	fmt.Println("Kruskal's trace:")
	kruskalTrace(5, edges)
}
```

**Textual Figure:**
```
  Graph: 0─1(4)  0─2(8)  1─2(2)  1─3(5)  2─3(6)  2─4(7)  3─4(3)

  Sorted edges: 1─2(2), 3─4(3), 0─1(4), 1─3(5), 2─3(6), 2─4(7), 0─2(8)

  Trace Output:
  ┌──────┬─────────────┬────────┬─────────┬────────────────────┐
  │ Step │ Edge        │ Action │ Running │ Components         │
  ├──────┼─────────────┼────────┼─────────┼────────────────────┤
  │  1   │ 1─2 (w=2)  │ ADD    │    2    │ {1,2}{0}{3}{4}     │
  │  2   │ 3─4 (w=3)  │ ADD    │    5    │ {1,2}{3,4}{0}      │
  │  3   │ 0─1 (w=4)  │ ADD    │    9    │ {0,1,2}{3,4}       │
  │  4   │ 1─3 (w=5)  │ ADD    │   14    │ {0,1,2,3,4}        │
  │  5   │ 2─3 (w=6)  │ SKIP   │   14    │ creates cycle       │
  │  6   │ 2─4 (w=7)  │ SKIP   │   14    │ creates cycle       │
  │  7   │ 0─2 (w=8)  │ SKIP   │   14    │ creates cycle       │
  └──────┴─────────────┴────────┴─────────┴────────────────────┘

  MST weight = 14.  Edges used: 1─2(2), 3─4(3), 0─1(4), 1─3(5)
```

---

## Example 9: Kruskal's for Maximum Spanning Tree

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func maxSpanningTree(n int, edges []Edge) int {
	// Sort in DESCENDING order for max spanning tree
	sort.Slice(edges, func(i, j int) bool { return edges[i].w > edges[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	total, count := 0, 0
	for _, e := range edges {
		px, py := find(e.u), find(e.v)
		if px != py {
			parent[px] = py
			total += e.w
			count++
			if count == n-1 { break }
		}
	}
	return total
}

func main() {
	edges := []Edge{{0,1,1},{0,2,4},{1,2,2},{1,3,6},{2,3,3}}
	fmt.Println("Max spanning tree:", maxSpanningTree(4, edges)) // 12
}
```

**Textual Figure:**
```
  Graph (undirected, weighted):         Max Spanning Tree:

     0 ─(1)─ 1                            0       1
     │       │                            │       │
    (4)     (6)                          (4)     (6)
     │       │                            │       │
     2 ─(2)─ 1    2 ─(3)─ 3               2       3
                                          │
                                         (3)
                                          │
                                          3

  Kruskal's Max (edges sorted DESCENDING):
  ┌──────┬────────────┬────────┬─────────┬───────────────────┐
  │ Step │ Edge       │ Action │ Running │ Components        │
  ├──────┼────────────┼────────┼─────────┼───────────────────┤
  │  1   │ 1─3 (w=6) │ ADD    │    6    │ {1,3}{0}{2}       │
  │  2   │ 0─2 (w=4) │ ADD    │   10    │ {0,2}{1,3}        │
  │  3   │ 2─3 (w=3) │ ADD    │   13    │ {0,1,2,3} done    │
  │  4   │ 1─2 (w=2) │ SKIP   │   13    │ cycle             │
  │  5   │ 0─1 (w=1) │ SKIP   │   13    │ cycle             │
  └──────┴────────────┴────────┴─────────┴───────────────────┘

  Max ST = 1─3(6) + 0─2(4) + 2─3(3) = 13
  Key: sort edges DESCENDING instead of ascending
```

---

## Example 10: Time Complexity Analysis

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"time"
)

type Edge struct{ u, v, w int }

func benchKruskal(n, e int) time.Duration {
	edges := make([]Edge, e)
	for i := 0; i < e; i++ {
		edges[i] = Edge{rand.Intn(n), rand.Intn(n), rand.Intn(1000)}
	}

	start := time.Now()

	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })
	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, ed := range edges {
		px, py := find(ed.u), find(ed.v)
		if px != py { parent[px] = py }
	}

	return time.Since(start)
}

func main() {
	sizes := [][2]int{{100, 500}, {1000, 5000}, {10000, 50000}, {100000, 500000}}
	for _, s := range sizes {
		d := benchKruskal(s[0], s[1])
		fmt.Printf("V=%6d, E=%7d → %v\n", s[0], s[1], d)
	}
	fmt.Println("\nKruskal's: O(E log E) dominated by sorting")
}
```

**Textual Figure:**
```
  Kruskal's Time Complexity Breakdown:

  Total: O(E log E) + O(E · α(V)) ≈ O(E log E)
         │                │
         │                └─ Union-Find operations
         │                   (nearly constant per op)
         └─ Sorting edges
            (dominates)

  Scaling behavior:
  ┌─────────┬──────────┬───────────────────────────┐
  │ V       │ E        │ Relative time               │
  ├─────────┼──────────┼───────────────────────────┤
  │     100 │      500 │ ▓░░░                        │
  │   1,000 │    5,000 │ ▓▓▓░░░░░                  │
  │  10,000 │   50,000 │ ▓▓▓▓▓▓▓░░░░░░            │
  │ 100,000 │  500,000 │ ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░  │
  └─────────┴──────────┴───────────────────────────┘

  Sorting (O(E log E)) dominates total runtime.
  Union-Find with path compression + union by rank ≈ O(1) amortized.
```

---

## Key Takeaways

1. Sort all edges by weight → greedily add if no cycle (Union-Find)
2. Time: O(E log E) — dominated by sorting
3. Space: O(V) for Union-Find parent/rank arrays
4. Best for sparse graphs (E ≈ V)
5. Reverse sort gives Maximum Spanning Tree
6. If edges used < V-1, graph is disconnected

> **Next up:** Prim's Algorithm →
