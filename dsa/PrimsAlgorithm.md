# Phase 12: Graphs — Prim's Algorithm

## Overview

**Prim's** builds an MST by growing a tree from an arbitrary vertex, always adding the cheapest edge that connects the tree to a new vertex.

- **Time:** O(E log V) with binary heap / O(V²) with array
- **Space:** O(V + E)
- Best for **dense graphs** (E close to V²)

---

## Example 1: Prim's with Min-Heap

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
	for _, e := range [][3]int{{0,1,2},{0,3,6},{1,2,3},{1,3,8},{1,4,5},{2,4,7},{3,4,9}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	fmt.Println("MST weight:", prim(5, adj)) // 16
}
```

---

## Example 2: Prim's with Edge Tracking

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, w, from int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].w < h[j].w }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func primEdges(n int, adj [][]Edge) ([][3]int, int) {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[0] = 0

	h := &PQ{{0, 0, -1}}
	heap.Init(h)
	total := 0
	mstEdges := [][3]int{}

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if inMST[cur.node] { continue }
		inMST[cur.node] = true
		total += cur.w
		if cur.from >= 0 {
			mstEdges = append(mstEdges, [3]int{cur.from, cur.node, cur.w})
		}

		for _, e := range adj[cur.node] {
			if !inMST[e.to] && e.w < key[e.to] {
				key[e.to] = e.w
				heap.Push(h, Item{e.to, e.w, cur.node})
			}
		}
	}
	return mstEdges, total
}

func main() {
	adj := make([][]Edge, 4)
	for _, e := range [][3]int{{0,1,10},{0,2,6},{0,3,5},{1,3,15},{2,3,4}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	edges, total := primEdges(4, adj)
	for _, e := range edges { fmt.Printf("%d--%d (w=%d)\n", e[0], e[1], e[2]) }
	fmt.Println("Total:", total) // 19
}
```

---

## Example 3: Prim's O(V²) Array Version (Dense Graphs)

```go
package main

import (
	"fmt"
	"math"
)

func primArray(n int, cost [][]int) int {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[0] = 0
	total := 0

	for i := 0; i < n; i++ {
		// Find min key not in MST
		u := -1
		for v := 0; v < n; v++ {
			if !inMST[v] && (u == -1 || key[v] < key[u]) { u = v }
		}

		inMST[u] = true
		total += key[u]

		for v := 0; v < n; v++ {
			if !inMST[v] && cost[u][v] > 0 && cost[u][v] < key[v] {
				key[v] = cost[u][v]
			}
		}
	}
	return total
}

func main() {
	// Adjacency matrix (0 = no edge)
	cost := [][]int{
		{0, 2, 0, 6, 0},
		{2, 0, 3, 8, 5},
		{0, 3, 0, 0, 7},
		{6, 8, 0, 0, 9},
		{0, 5, 7, 9, 0},
	}
	fmt.Println("MST weight:", primArray(5, cost)) // 16
}
```

---

## Example 4: Min Cost to Connect All Points (LeetCode 1584 — Prim's)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Item struct{ node, w int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].w < h[j].w }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minCostConnectPoints(points [][]int) int {
	n := len(points)
	abs := func(x int) int { if x < 0 { return -x }; return x }

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

		for j := 0; j < n; j++ {
			if inMST[j] { continue }
			d := abs(points[cur.node][0]-points[j][0]) + abs(points[cur.node][1]-points[j][1])
			if d < key[j] {
				key[j] = d
				heap.Push(h, Item{j, d})
			}
		}
	}
	return total
}

func main() {
	points := [][]int{{0,0},{2,2},{3,10},{5,2},{7,0}}
	fmt.Println(minCostConnectPoints(points)) // 20
}
```

---

## Example 5: Prim's from Any Starting Vertex

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

func primFrom(n int, adj [][]Edge, start int) int {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[start] = 0

	h := &PQ{{start, 0}}
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
	adj := make([][]Edge, 4)
	for _, e := range [][3]int{{0,1,1},{1,2,2},{2,3,3},{0,3,4}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	// MST weight is the same regardless of starting vertex
	fmt.Println("Start 0:", primFrom(4, adj, 0)) // 6
	fmt.Println("Start 2:", primFrom(4, adj, 2)) // 6
	fmt.Println("Start 3:", primFrom(4, adj, 3)) // 6
}
```

---

## Example 6: Prim's Detecting Disconnected Graph

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

func primCheck(n int, adj [][]Edge) (int, bool) {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[0] = 0

	h := &PQ{{0, 0}}
	heap.Init(h)
	total, count := 0, 0

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if inMST[cur.node] { continue }
		inMST[cur.node] = true
		total += cur.w
		count++
		for _, e := range adj[cur.node] {
			if !inMST[e.to] && e.w < key[e.to] {
				key[e.to] = e.w
				heap.Push(h, Item{e.to, e.w})
			}
		}
	}
	return total, count == n
}

func main() {
	adj := make([][]Edge, 4)
	adj[0] = []Edge{{1, 1}}; adj[1] = []Edge{{0, 1}}
	adj[2] = []Edge{{3, 2}}; adj[3] = []Edge{{2, 2}}
	w, connected := primCheck(4, adj)
	fmt.Printf("Weight: %d, Connected: %v\n", w, connected) // 1, false
}
```

---

## Example 7: Prim's Step-by-Step Trace

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, w, from int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].w < h[j].w }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func primTrace(n int, adj [][]Edge) {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[0] = 0

	h := &PQ{{0, 0, -1}}
	heap.Init(h)
	total := 0

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if inMST[cur.node] { continue }
		inMST[cur.node] = true
		total += cur.w
		if cur.from >= 0 {
			fmt.Printf("  Add edge %d--%d (w=%d), total=%d\n", cur.from, cur.node, cur.w, total)
		} else {
			fmt.Printf("  Start at node %d\n", cur.node)
		}
		for _, e := range adj[cur.node] {
			if !inMST[e.to] && e.w < key[e.to] {
				key[e.to] = e.w
				heap.Push(h, Item{e.to, e.w, cur.node})
			}
		}
	}
	fmt.Println("MST total:", total)
}

func main() {
	adj := make([][]Edge, 5)
	for _, e := range [][3]int{{0,1,2},{0,3,6},{1,2,3},{1,3,8},{1,4,5},{2,4,7},{3,4,9}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	primTrace(5, adj)
}
```

---

## Example 8: Prim's for Maximum Spanning Tree

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, w int }
type MaxPQ []Item
func (h MaxPQ) Len() int            { return len(h) }
func (h MaxPQ) Less(i, j int) bool   { return h[i].w > h[j].w } // Max heap
func (h MaxPQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *MaxPQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *MaxPQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func primMax(n int, adj [][]Edge) int {
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MinInt64 }
	key[0] = 0

	h := &MaxPQ{{0, 0}}
	heap.Init(h)
	total := 0

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if inMST[cur.node] { continue }
		inMST[cur.node] = true
		total += cur.w

		for _, e := range adj[cur.node] {
			if !inMST[e.to] && e.w > key[e.to] {
				key[e.to] = e.w
				heap.Push(h, Item{e.to, e.w})
			}
		}
	}
	return total
}

func main() {
	adj := make([][]Edge, 4)
	for _, e := range [][3]int{{0,1,1},{0,2,4},{1,2,2},{1,3,6},{2,3,3}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	fmt.Println("Max spanning tree:", primMax(4, adj)) // 12
}
```

---

## Example 9: Prim's with Adjacency Matrix

```go
package main

import (
	"fmt"
	"math"
)

func primMatrix(cost [][]int) int {
	n := len(cost)
	inMST := make([]bool, n)
	key := make([]int, n)
	for i := range key { key[i] = math.MaxInt64 }
	key[0] = 0
	total := 0

	for count := 0; count < n; count++ {
		u := -1
		for v := 0; v < n; v++ {
			if !inMST[v] && (u == -1 || key[v] < key[u]) { u = v }
		}
		inMST[u] = true
		total += key[u]

		for v := 0; v < n; v++ {
			if !inMST[v] && cost[u][v] > 0 && cost[u][v] < key[v] {
				key[v] = cost[u][v]
			}
		}
	}
	return total
}

func main() {
	cost := [][]int{
		{0, 4, 0, 0, 0, 0, 0, 8, 0},
		{4, 0, 8, 0, 0, 0, 0, 11, 0},
		{0, 8, 0, 7, 0, 4, 0, 0, 2},
		{0, 0, 7, 0, 9, 14, 0, 0, 0},
		{0, 0, 0, 9, 0, 10, 0, 0, 0},
		{0, 0, 4, 14, 10, 0, 2, 0, 0},
		{0, 0, 0, 0, 0, 2, 0, 1, 6},
		{8, 11, 0, 0, 0, 0, 1, 0, 7},
		{0, 0, 2, 0, 0, 0, 6, 7, 0},
	}
	fmt.Println("MST weight:", primMatrix(cost)) // 37
}
```

---

## Example 10: Prim's vs Kruskal's Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Prim's vs Kruskal's ===")
	fmt.Println()

	type Row struct{ feature, prim, kruskal string }
	rows := []Row{
		{"Approach", "Grow from vertex", "Sort edges globally"},
		{"Data Structure", "Min-Heap / Array", "Union-Find"},
		{"Time (Binary Heap)", "O(E log V)", "O(E log E)"},
		{"Time (Array)", "O(V²)", "N/A"},
		{"Best for", "Dense (E ≈ V²)", "Sparse (E ≈ V)"},
		{"Implementation", "Like Dijkstra", "Sort + UF"},
		{"Works on disconnected?", "No (finds 1 component)", "Yes (forest)"},
	}

	fmt.Printf("%-25s %-22s %-22s\n", "Feature", "Prim's", "Kruskal's")
	fmt.Println("-------------------------------------------------------------------")
	for _, r := range rows {
		fmt.Printf("%-25s %-22s %-22s\n", r.feature, r.prim, r.kruskal)
	}
}
```

---

## Key Takeaways

1. Prim's grows tree from a vertex, always adding cheapest crossing edge
2. O(E log V) with binary heap; O(V²) with array (better for dense)
3. Very similar to Dijkstra's — same template, different relaxation
4. MST weight is the same regardless of starting vertex
5. Use max-heap for Maximum Spanning Tree variant

> **Next up:** Bipartite Graphs →
