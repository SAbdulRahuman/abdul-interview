# Phase 12: Graphs — Minimum Spanning Tree

## Overview

A **Minimum Spanning Tree (MST)** connects all V vertices using V-1 edges with minimum total weight.

| Property | Guarantee |
|----------|-----------|
| Connects all vertices | Yes (spanning) |
| No cycles | Yes (tree) |
| Minimum total weight | Yes |
| Unique? | Only if all edge weights are distinct |

Two main algorithms:
- **Kruskal's** — sort edges, add cheapest non-cycle edge (Union-Find)
- **Prim's** — grow tree from vertex, add cheapest crossing edge (Min-Heap)

---

## Example 1: MST Concept — Cut Property

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Cut Property ===")
	fmt.Println("For any cut of a graph, the minimum-weight edge")
	fmt.Println("crossing the cut belongs to some MST.")
	fmt.Println()
	fmt.Println("This is the foundation for both Kruskal's and Prim's:")
	fmt.Println("- Kruskal's: considers global cheapest edges")
	fmt.Println("- Prim's: considers cheapest edge crossing current cut")
	fmt.Println()

	// Demonstrate: graph with 4 nodes
	// Edges: (0,1,1), (0,2,4), (1,2,2), (1,3,6), (2,3,3)
	// MST: (0,1,1), (1,2,2), (2,3,3) = total 6
	fmt.Println("Example graph:")
	fmt.Println("  0--1(1)  1--2(2)  2--3(3)  = MST total: 6")
	fmt.Println("  0--2(4) and 1--3(6) are NOT in MST")
}
```

---

## Example 2: Kruskal's Algorithm

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

	mst := []Edge{}
	total := 0
	for _, e := range edges {
		px, py := find(e.u), find(e.v)
		if px == py { continue }
		if rank[px] < rank[py] { px, py = py, px }
		parent[py] = px
		if rank[px] == rank[py] { rank[px]++ }
		mst = append(mst, e)
		total += e.w
		if len(mst) == n-1 { break }
	}
	return mst, total
}

func main() {
	edges := []Edge{{0,1,4},{0,2,8},{1,2,2},{1,3,5},{2,3,5},{2,4,7},{3,4,6}}
	mst, total := kruskal(5, edges)
	fmt.Println("MST edges:")
	for _, e := range mst { fmt.Printf("  %d--%d (w=%d)\n", e.u, e.v, e.w) }
	fmt.Println("Total:", total) // 17
}
```

---

## Example 3: Prim's Algorithm

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, w int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].w < h[j].w }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func prim(n int, adj [][]Edge) int {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[0] = 0

	h := &PQ{{0, 0}}
	heap.Init(h)
	total := 0

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if inMST[cur.node] { continue }
		inMST[cur.node] = true
		total += cur.w

		for _, e := range adj[cur.node] {
			if !inMST[e.to] && e.w < key[e.to] {
				key[e.to] = e.w
				heap.Push(h, Item{e.to, e.w})
			}
		}
	}
	return total
}

func main() {
	adj := make([][]Edge, 5)
	for _, e := range [][3]int{{0,1,4},{0,2,8},{1,2,2},{1,3,5},{2,3,5},{2,4,7},{3,4,6}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	fmt.Println("MST weight:", prim(5, adj)) // 17
}
```

---

## Example 4: Min Cost to Connect All Points (LeetCode 1584)

```go
package main

import (
	"fmt"
	"sort"
)

func minCostConnectPoints(points [][]int) int {
	n := len(points)
	edges := []struct{ u, v, w int }{}

	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			dx := points[i][0] - points[j][0]
			dy := points[i][1] - points[j][1]
			if dx < 0 { dx = -dx }
			if dy < 0 { dy = -dy }
			edges = append(edges, struct{ u, v, w int }{i, j, dx + dy})
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

## Example 5: MST vs Shortest Path

```go
package main

import "fmt"

func main() {
	fmt.Println("=== MST vs Shortest Path Tree ===")
	fmt.Println()
	fmt.Println("MST: Minimizes TOTAL edge weight (connects all nodes)")
	fmt.Println("SPT: Minimizes distance from SOURCE to each node")
	fmt.Println()
	fmt.Println("Example graph: 0--1(1), 1--2(1), 0--2(3)")
	fmt.Println("MST edges: 0-1(1), 1-2(1) → total = 2")
	fmt.Println("SPT from 0: 0-1(1), 0-2(3) if 0-2 is direct")
	fmt.Println("   or: 0-1(1), 1-2(1) if via 1 is shorter")
	fmt.Println()
	fmt.Println("MST ≠ SPT in general!")
	fmt.Println("MST: global minimum weight")
	fmt.Println("SPT: local optimal from source")
}
```

---

## Example 6: Connecting Cities with Minimum Cost (LeetCode 1135)

```go
package main

import (
	"fmt"
	"sort"
)

func minimumCost(n int, connections [][]int) int {
	sort.Slice(connections, func(i, j int) bool {
		return connections[i][2] < connections[j][2]
	})

	parent := make([]int, n+1)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	total, edges := 0, 0
	for _, c := range connections {
		px, py := find(c[0]), find(c[1])
		if px == py { continue }
		parent[px] = py
		total += c[2]
		edges++
		if edges == n-1 { return total }
	}
	return -1 // not connected
}

func main() {
	connections := [][]int{{1,2,5},{1,3,6},{2,3,1}}
	fmt.Println(minimumCost(3, connections)) // 6

	connections2 := [][]int{{1,2,3},{3,4,4}}
	fmt.Println(minimumCost(4, connections2)) // -1 (not connected)
}
```

---

## Example 7: Second Best MST

```go
package main

import (
	"fmt"
	"math"
	"sort"
)

type Edge struct{ u, v, w int }

func secondBestMST(n int, edges []Edge) int {
	sort.Slice(edges, func(i, j int) bool { return edges[i].w < edges[j].w })

	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	inMST := make([]bool, len(edges))
	mstWeight := 0
	for i, e := range edges {
		px, py := find(e.u), find(e.v)
		if px == py { continue }
		if rank[px] < rank[py] { px, py = py, px }
		parent[py] = px
		if rank[px] == rank[py] { rank[px]++ }
		inMST[i] = true
		mstWeight += e.w
	}

	// Try removing each MST edge and rebuilding
	best := math.MaxInt64
	for skip := 0; skip < len(edges); skip++ {
		if !inMST[skip] { continue }

		p := make([]int, n)
		r := make([]int, n)
		for i := range p { p[i] = i }
		findLocal := func(x int) int {
			for p[x] != x { p[x] = p[p[x]]; x = p[x] }
			return x
		}

		w, cnt := 0, 0
		for i, e := range edges {
			if i == skip { continue }
			px, py := findLocal(e.u), findLocal(e.v)
			if px == py { continue }
			if r[px] < r[py] { px, py = py, px }
			p[py] = px
			if r[px] == r[py] { r[px]++ }
			w += e.w
			cnt++
		}
		if cnt == n-1 && w < best { best = w }
	}
	return best
}

func main() {
	edges := []Edge{{0,1,1},{1,2,2},{0,2,3},{2,3,4},{1,3,5}}
	fmt.Println("2nd best MST:", secondBestMST(4, edges)) // 8
}
```

---

## Example 8: MST Edge Classification

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func classifyEdges(n int, edges []Edge) {
	sorted := make([]Edge, len(edges))
	copy(sorted, edges)
	sort.Slice(sorted, func(i, j int) bool { return sorted[i].w < sorted[j].w })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	mstEdges := map[[2]int]bool{}
	for _, e := range sorted {
		px, py := find(e.u), find(e.v)
		if px == py { continue }
		parent[px] = py
		u, v := e.u, e.v
		if u > v { u, v = v, u }
		mstEdges[[2]int{u, v}] = true
	}

	for _, e := range edges {
		u, v := e.u, e.v
		if u > v { u, v = v, u }
		if mstEdges[[2]int{u, v}] {
			fmt.Printf("  %d--%d (w=%d): MST EDGE\n", e.u, e.v, e.w)
		} else {
			fmt.Printf("  %d--%d (w=%d): non-MST (would create cycle)\n", e.u, e.v, e.w)
		}
	}
}

func main() {
	edges := []Edge{{0,1,1},{0,2,4},{1,2,2},{1,3,6},{2,3,3}}
	fmt.Println("Edge classification:")
	classifyEdges(4, edges)
}
```

---

## Example 9: Kruskal's with Edge Count Check

```go
package main

import (
	"fmt"
	"sort"
)

type Edge struct{ u, v, w int }

func kruskalFull(n int, edges []Edge) (int, bool) {
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
	}

	if count != n-1 {
		return -1, false // graph not connected
	}
	return total, true
}

func main() {
	edges := []Edge{{0,1,3},{1,2,1},{2,3,4},{0,3,2},{1,3,5}}
	w, ok := kruskalFull(4, edges)
	fmt.Printf("MST weight: %d, connected: %v\n", w, ok) // 6, true

	edges2 := []Edge{{0,1,1},{2,3,2}} // disconnected
	w2, ok2 := kruskalFull(4, edges2)
	fmt.Printf("MST weight: %d, connected: %v\n", w2, ok2) // -1, false
}
```

---

## Example 10: Kruskal vs Prim Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Kruskal's vs Prim's ===")
	fmt.Println()

	type Algo struct {
		name, approach, ds, time, best string
	}

	algos := []Algo{
		{"Kruskal's", "Sort edges, Union-Find", "Union-Find", "O(E log E)", "Sparse graphs (E ≈ V)"},
		{"Prim's", "Grow from vertex, min-heap", "Min-Heap", "O(E log V)", "Dense graphs (E ≈ V²)"},
	}

	for _, a := range algos {
		fmt.Printf("%-10s: %s\n", a.name, a.approach)
		fmt.Printf("  DS: %s | Time: %s\n", a.ds, a.time)
		fmt.Printf("  Best for: %s\n\n", a.best)
	}

	fmt.Println("Both produce the same MST (if unique)")
	fmt.Println("Both are greedy — they pick locally optimal edges")
}
```

---

## Key Takeaways

1. MST connects all vertices with minimum total weight using exactly V-1 edges
2. Kruskal's: sort edges + Union-Find; O(E log E); best for sparse
3. Prim's: grow from vertex + min-heap; O(E log V); best for dense
4. Cut property: cheapest edge crossing any cut is in the MST
5. If graph is disconnected, no MST exists → detect via edge count

> **Next up:** Kruskal's Algorithm (Deep Dive) →
