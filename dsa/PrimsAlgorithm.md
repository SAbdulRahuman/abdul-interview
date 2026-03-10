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

**Textual Figure:**
```
  Graph (undirected, weighted):                MST Result (weight = 16):

       0 ───(2)─── 1 ───(5)─── 4                  0 ───(2)─── 1
       │         ╱ │             │                  │         ╱   ╲
      (6)     (3) (8)          (7)                (6)     (3)     (5)
       │    ╱      │             │                  │    ╱           ╲
       3          1 ───(8)───── 3                  3   2              4
       │                        │
       └────────(9)─────────── 4

  Prim's Step-by-Step (start at vertex 0):
  ┌──────┬────────────────┬─────────┬─────────────────┐
  │ Step │ Edge Added     │ Running │ MST Vertices    │
  │      │                │ Total   │                 │
  ├──────┼────────────────┼─────────┼─────────────────┤
  │  1   │ start at 0     │    0    │ {0}             │
  │  2   │ 0──1  (w=2)    │    2    │ {0,1}           │
  │  3   │ 1──2  (w=3)    │    5    │ {0,1,2}         │
  │  4   │ 1──4  (w=5)    │   10    │ {0,1,2,4}       │
  │  5   │ 0──3  (w=6)    │   16    │ {0,1,2,3,4}     │
  └──────┴────────────────┴─────────┴─────────────────┘

  MST Edges: 0─1(2), 1─2(3), 1─4(5), 0─3(6)  →  Total = 16
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

**Textual Figure:**
```
  Graph (undirected, weighted):              MST Result (weight = 19):

       0 ──(10)── 1                              0 ──(10)── 1
       │╲         │                              │
      (6) (5)   (15)                            (5)
       │    ╲     │                               │
       2 ──(4)── 3                              3 ──(4)── 2

  Prim's with Edge Tracking (start at vertex 0):
  ┌──────┬────────────────┬─────────┬─────────────────┐
  │ Step │ Edge Added     │ Running │ MST Vertices    │
  │      │ (from → to)    │ Total   │                 │
  ├──────┼────────────────┼─────────┼─────────────────┤
  │  1   │ start at 0     │    0    │ {0}             │
  │  2   │ 0→3  (w=5)     │    5    │ {0,3}           │
  │  3   │ 3→2  (w=4)     │    9    │ {0,2,3}         │
  │  4   │ 0→1  (w=10)    │   19    │ {0,1,2,3}       │
  └──────┴────────────────┴─────────┴─────────────────┘

  Output:  0──3 (w=5)  │  3──2 (w=4)  │  0──1 (w=10)  →  Total = 19
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

**Textual Figure:**
```
  Adjacency Matrix (0 = no edge):
  ┌───┬───┬───┬───┬───┬───┐
  │   │ 0 │ 1 │ 2 │ 3 │ 4 │
  ├───┼───┼───┼───┼───┼───┤
  │ 0 │ 0 │ 2 │ 0 │ 6 │ 0 │
  │ 1 │ 2 │ 0 │ 3 │ 8 │ 5 │
  │ 2 │ 0 │ 3 │ 0 │ 0 │ 7 │
  │ 3 │ 6 │ 8 │ 0 │ 0 │ 9 │
  │ 4 │ 0 │ 5 │ 7 │ 9 │ 0 │
  └───┴───┴───┴───┴───┴───┘

  O(V²) Array Prim's — no heap, scan all vertices each step:
  ┌──────┬──────────────────────────┬─────────┬─────────────────┐
  │ Step │ Pick vertex u (min key)  │ Running │ key[] updates    │
  │      │                          │ Total   │                 │
  ├──────┼──────────────────────────┼─────────┼─────────────────┤
  │  1   │ u=0, key[0]=0            │    0    │ key[1]=2,key[3]=6│
  │  2   │ u=1, key[1]=2            │    2    │ key[2]=3,key[4]=5│
  │  3   │ u=2, key[2]=3            │    5    │ key[4]=min(5,7)=5│
  │  4   │ u=4, key[4]=5            │   10    │ key[3]=min(6,9)=6│
  │  5   │ u=3, key[3]=6            │   16    │ (all in MST)    │
  └──────┴──────────────────────────┴─────────┴─────────────────┘

  Same graph as Example 1 → same MST weight = 16
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

**Textual Figure:**
```
  Points on 2D plane (Manhattan distance):

        10│       P2(3,10)
          │        ╎
          │        ╎ d=9
          │        ╎
         2│ P1(2,2)╎·····P3(5,2)                     P0─P1: 4
          │  ╎  d=3   d=4 ╎                          P1─P3: 3
         0│P0(0,0)·····╌╌╌╌P4(7,0)                   P3─P4: 4
          └──────────────────────                     P1─P2: 9
           0  1  2  3  4  5  6  7

  Prim's MST Construction:
  ┌──────┬─────────────────┬─────────┬───────────────────┐
  │ Step │ Edge Added      │ Running │ MST Vertices      │
  ├──────┼─────────────────┼─────────┼───────────────────┤
  │  1   │ start at P0     │    0    │ {P0}              │
  │  2   │ P0→P1 (d=4)     │    4    │ {P0,P1}           │
  │  3   │ P1→P3 (d=3)     │    7    │ {P0,P1,P3}        │
  │  4   │ P3→P4 (d=4)     │   11    │ {P0,P1,P3,P4}     │
  │  5   │ P1→P2 (d=9)     │   20    │ {P0,P1,P2,P3,P4}  │
  └──────┴─────────────────┴─────────┴───────────────────┘

  MST total cost = 20
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

**Textual Figure:**
```
  Graph: 0 ──(1)── 1 ──(2)── 2 ──(3)── 3
         │                             │
         └──────────(4)────────────────┘

  MST = {0─1(1), 1─2(2), 2─3(3)} = 6  (edge 0─3(4) excluded)

  Proof: MST weight is identical regardless of start vertex:
  ┌─────────┬────────────────────────────┬───────┐
  │ Start   │ Edges added (in order)     │ Total │
  ├─────────┼────────────────────────────┼───────┤
  │ start=0 │ 0─1(1) → 1─2(2) → 2─3(3)  │   6   │
  │ start=2 │ 2─1(2) → 1─0(1) → 2─3(3)  │   6   │
  │ start=3 │ 3─2(3) → 2─1(2) → 1─0(1)  │   6   │
  └─────────┴────────────────────────────┴───────┘

  Key insight: MST is a property of the graph,
  not the starting vertex — only traversal order changes.
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

**Textual Figure:**
```
  Disconnected Graph:

    Component A        Component B
    0 ──(1)── 1        2 ──(2)── 3

  Prim's from vertex 0:
  ┌──────┬────────────────┬─────────┬─────────────┐
  │ Step │ Action         │ Running │ inMST count │
  ├──────┼────────────────┼─────────┼─────────────┤
  │  1   │ start at 0     │    0    │  1 / 4      │
  │  2   │ 0→1  (w=1)     │    1    │  2 / 4      │
  │      │ (heap empty)   │         │             │
  └──────┴────────────────┴─────────┴─────────────┘

  count=2 ≠ n=4  →  Connected = false
  Nodes 2, 3 unreachable from component containing 0
  Weight returned = 1 (partial MST of reachable component)
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

**Textual Figure:**
```
  Same graph as Example 1:
       0 ───(2)─── 1 ───(5)─── 4
       │         ╱ │             │
      (6)     (3) (8)          (7)
       │    ╱     │              │
       3         1 ───(8)─── 3  2
       │                    │
       └────────(9)─────── 4

  Console Trace Output:
  ┌─────────────────────────────────────────────┐
  │  Start at node 0                          │
  │  Add edge 0──1 (w=2),  total=2             │
  │  Add edge 1──2 (w=3),  total=5             │
  │  Add edge 1──4 (w=5),  total=10            │
  │  Add edge 0──3 (w=6),  total=16            │
  │  MST total: 16                             │
  └─────────────────────────────────────────────┘

  The `from` field tracks which vertex brought each node
  into the MST, enabling reconstruction of the tree edges.
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

  Prim's Max-Heap (start at 0, pick HEAVIEST edges):
  ┌──────┬────────────────┬─────────┬───────────────┐
  │ Step │ Edge Added     │ Running │ MST Vertices  │
  ├──────┼────────────────┼─────────┼───────────────┤
  │  1   │ start at 0     │    0    │ {0}           │
  │  2   │ 0→2  (w=4)     │    4    │ {0,2}         │
  │  3   │ 2→3  (w=3)     │    7    │ {0,2,3}       │
  │  4   │ 3→1  (w=6)     │   13    │ {0,1,2,3}     │
  └──────┴────────────────┴─────────┴───────────────┘

  Key difference from min-MST:
  • Uses Max-Heap (Less returns h[i].w > h[j].w)
  • key[] initialized to MinInt64 (not MaxInt64)
  • Update condition: e.w > key[e.to] (not <)
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

**Textual Figure:**
```
  Classic 9-node graph (adjacency matrix):

       0 ─(4)─ 1 ─(8)─ 2 ─(7)─ 3 ─(9)─ 4
       │              │         │        │
      (8)            (2)      (14)     (10)
       │              │         │        │
       7 ─(1)─ 6 ─(2)─ 5 ───────────── 4
       │        │      │
      (11)     (6)    (4)
       │        │      │
       1        8 ─(7)─ 7
    (connects  (connects
     to 1)      to 7)

  Prim's O(V²) trace (start at 0):
  ┌──────┬────────────────┬─────────┬───────────────────────┐
  │ Step │ Edge Added     │ Running │ MST Vertices          │
  ├──────┼────────────────┼─────────┼───────────────────────┤
  │  1   │ start at 0     │    0    │ {0}                   │
  │  2   │ 0→1  (w=4)     │    4    │ {0,1}                 │
  │  3   │ 0→7  (w=8)     │   12    │ {0,1,7}               │
  │  4   │ 7→6  (w=1)     │   13    │ {0,1,6,7}             │
  │  5   │ 6→5  (w=2)     │   15    │ {0,1,5,6,7}           │
  │  6   │ 5→2  (w=4)     │   19    │ {0,1,2,5,6,7}         │
  │  7   │ 2→8  (w=2)     │   21    │ {0,1,2,5,6,7,8}       │
  │  8   │ 2→3  (w=7)     │   28    │ {0,1,2,3,5,6,7,8}     │
  │  9   │ 3→4  (w=9)     │   37    │ {0,1,2,3,4,5,6,7,8}   │
  └──────┴────────────────┴─────────┴───────────────────────┘

  MST weight = 37
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

**Textual Figure:**
```
  Prim’s vs Kruskal’s — Visual Comparison:

  PRIM’S (grow from vertex):        KRUSKAL’S (sort all edges):

  Step 1: start at (0)               Sort: e1 ≤ e2 ≤ e3 ≤ ... ≤ eM
           ↓                                  ↓
  Step 2: (0)───(1)                  Take e1: no cycle? → ADD
           ↓                                  ↓
  Step 3: (0)───(1)───(2)            Take e2: no cycle? → ADD
           ↓                                  ↓
  Step N: full MST                   Take eK: cycle? → SKIP
                                              ↓
  Uses: Min-Heap (priority queue)    Uses: Union-Find (DSU)

  ┌─────────────────────┬───────────────┬───────────────┐
  │                     │ Prim’s         │ Kruskal’s      │
  ├─────────────────────┼───────────────┼───────────────┤
  │ Best for            │ Dense (E≈V²)   │ Sparse (E≈V)  │
  │ Time (heap)         │ O(E log V)     │ O(E log E)    │
  │ Disconnected graph? │ No (1 comp)    │ Yes (forest)  │
  └─────────────────────┴───────────────┴───────────────┘
```

---

## Key Takeaways

1. Prim's grows tree from a vertex, always adding cheapest crossing edge
2. O(E log V) with binary heap; O(V²) with array (better for dense)
3. Very similar to Dijkstra's — same template, different relaxation
4. MST weight is the same regardless of starting vertex
5. Use max-heap for Maximum Spanning Tree variant

> **Next up:** Bipartite Graphs →
