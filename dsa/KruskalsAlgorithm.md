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

---

## Key Takeaways

1. Sort all edges by weight → greedily add if no cycle (Union-Find)
2. Time: O(E log E) — dominated by sorting
3. Space: O(V) for Union-Find parent/rank arrays
4. Best for sparse graphs (E ≈ V)
5. Reverse sort gives Maximum Spanning Tree
6. If edges used < V-1, graph is disconnected

> **Next up:** Prim's Algorithm →
