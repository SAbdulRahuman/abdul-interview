# Phase 12: Graphs — Dijkstra's Algorithm

## Overview

**Dijkstra's algorithm** finds the shortest path from a source to all other vertices in a graph with **non-negative edge weights**.

- **Time:** O((V + E) log V) with min-heap
- **Space:** O(V)
- **Cannot handle negative weights** — use Bellman-Ford instead

```
Algorithm:
1. Set dist[source] = 0, all others = ∞
2. Push source into min-heap
3. Pop minimum, relax neighbors
4. Repeat until heap is empty
```

---

## Example 1: Basic Dijkstra's

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, weight int }
type Item struct{ node, dist int }
type MinHeap []Item

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool   { return h[i].dist < h[j].dist }
func (h MinHeap) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func dijkstra(n int, adj [][]Edge, src int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	h := &MinHeap{{src, 0}}
	heap.Init(h)

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if cur.dist > dist[cur.node] { continue }
		for _, e := range adj[cur.node] {
			newDist := dist[cur.node] + e.weight
			if newDist < dist[e.to] {
				dist[e.to] = newDist
				heap.Push(h, Item{e.to, newDist})
			}
		}
	}
	return dist
}

func main() {
	adj := make([][]Edge, 5)
	edges := [][3]int{{0,1,4},{0,2,1},{2,1,2},{1,3,1},{2,3,5},{3,4,3}}
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
	}
	fmt.Println(dijkstra(5, adj, 0)) // [0 3 1 4 7]
}
```

**Textual Figure:**

```
Graph (directed, weighted):

       0
      / \
   4/    \1
    /      \
   1───────2
   │  2↑    │
  1│       │5
   │       │
   3──────4
      3

Dijkstra from source 0:
┌──────┬─────────────┬─────────────┬──────────────────────────────┐
│ Step │ Pop (node,d) │ Relax edges │ dist = [0, 1, 2, 3, 4]       │
├──────┼─────────────┼─────────────┼──────────────────────────────┤
│ init │             │             │ [0, ∞, ∞, ∞, ∞]             │
│  1   │   (0, 0)    │ 0→1:4 0→2:1 │ [0, 4, 1, ∞, ∞]             │
│  2   │   (2, 1)    │ 2→1:3 2→3:6 │ [0, 3, 1, 6, ∞]             │
│  3   │   (1, 3)    │ 1→3:4       │ [0, 3, 1, 4, ∞]             │
│  4   │   (3, 4)    │ 3→4:7       │ [0, 3, 1, 4, 7]             │
│  5   │   (4, 7)    │ (none)      │ [0, 3, 1, 4, 7]  ← final    │
└──────┴─────────────┴─────────────┴──────────────────────────────┘

Shortest paths from 0:
  0→0: 0    0→1: 0→2→1 = 3    0→2: 1    0→3: 0→2→1→3 = 4    0→4: 7
```

---

## Example 2: Dijkstra with Path Reconstruction

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, dist int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func dijkstraPath(n int, adj [][]Edge, src, dst int) (int, []int) {
	dist := make([]int, n)
	prev := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64; prev[i] = -1 }
	dist[src] = 0

	h := &PQ{{src, 0}}
	heap.Init(h)

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if cur.node == dst { break }
		if cur.dist > dist[cur.node] { continue }
		for _, e := range adj[cur.node] {
			nd := dist[cur.node] + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				prev[e.to] = cur.node
				heap.Push(h, Item{e.to, nd})
			}
		}
	}

	if dist[dst] == math.MaxInt64 { return -1, nil }

	path := []int{}
	for v := dst; v != -1; v = prev[v] {
		path = append(path, v)
	}
	for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
		path[i], path[j] = path[j], path[i]
	}
	return dist[dst], path
}

func main() {
	adj := make([][]Edge, 5)
	for _, e := range [][3]int{{0,1,4},{0,2,1},{2,1,2},{1,3,1},{2,3,5},{3,4,3}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
	}
	d, p := dijkstraPath(5, adj, 0, 4)
	fmt.Println("Dist:", d, "Path:", p) // Dist: 7 Path: [0 2 1 3 4]
}
```

**Textual Figure:**

```
Same graph, source=0, destination=4:

       0
      / \
   4/    \1
    /      \
   1───────2
   │  2↑    │
  1│       │5
   │       │
   3──────4
      3

Path reconstruction via prev[] array:
┌──────┬───────┬────────────────────────┐
│ Node │ dist  │ prev                   │
├──────┼───────┼────────────────────────┤
│  0   │   0   │ -1  (source)           │
│  1   │   3   │  2  (via 0→2→1)       │
│  2   │   1   │  0  (via 0→2)          │
│  3   │   4   │  1  (via 0→2→1→3)    │
│  4   │   7   │  3  (via 0→2→1→3→4) │
└──────┴───────┴────────────────────────┘

Trace back from dst=4:
  4 → prev[4]=3 → prev[3]=1 → prev[1]=2 → prev[2]=0 → prev[0]=-1

Reverse: Path = [0, 2, 1, 3, 4]   Distance = 7

  0 ──1─→ 2 ──2─→ 1 ──1─→ 3 ──3─→ 4
```

---

## Example 3: Network Delay Time (LeetCode 743)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, dist int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func networkDelayTime(times [][]int, n, k int) int {
	adj := make([][]Edge, n+1)
	for _, t := range times {
		adj[t[0]] = append(adj[t[0]], Edge{t[1], t[2]})
	}

	dist := make([]int, n+1)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[k] = 0

	h := &PQ{{k, 0}}
	heap.Init(h)

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if cur.dist > dist[cur.node] { continue }
		for _, e := range adj[cur.node] {
			nd := dist[cur.node] + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				heap.Push(h, Item{e.to, nd})
			}
		}
	}

	maxDelay := 0
	for i := 1; i <= n; i++ {
		if dist[i] == math.MaxInt64 { return -1 }
		if dist[i] > maxDelay { maxDelay = dist[i] }
	}
	return maxDelay
}

func main() {
	fmt.Println(networkDelayTime([][]int{{2,1,1},{2,3,1},{3,4,1}}, 4, 2)) // 2
	fmt.Println(networkDelayTime([][]int{{1,2,1}}, 2, 2))                  // -1
}
```

**Textual Figure:**

```
Network: n=4, k=2 (source), times = [(2,1,1),(2,3,1),(3,4,1)]

       2
      / \
   1/    \1
    /      \
   1       3
            \
           1 \
              \
               4

Dijkstra from node 2:
┌──────┬───────────┬─────────────────┐
│ Step │ Pop       │ dist[1..4]        │
├──────┼───────────┼─────────────────┤
│ init │           │ [∞, 0, ∞, ∞]     │
│  1   │ (2, 0)    │ [1, 0, 1, ∞]     │
│  2   │ (1, 1)    │ [1, 0, 1, ∞]     │
│  3   │ (3, 1)    │ [1, 0, 1, 2]     │
│  4   │ (4, 2)    │ [1, 0, 1, 2]     │
└──────┴───────────┴─────────────────┘

Max delay = max(1, 0, 1, 2) = 2  (all nodes reached)
Answer: 2
```

---

## Example 4: Cheapest Flights Within K Stops (LeetCode 787)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Edge struct{ to, cost int }
type State struct{ node, cost, stops int }
type PQ []State
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].cost < h[j].cost }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(State)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func findCheapestPrice(n int, flights [][]int, src, dst, k int) int {
	adj := make([][]Edge, n)
	for _, f := range flights {
		adj[f[0]] = append(adj[f[0]], Edge{f[1], f[2]})
	}

	// Track best stops to reach each node
	bestStops := make([]int, n)
	for i := range bestStops { bestStops[i] = 1<<31 - 1 }

	h := &PQ{{src, 0, 0}}
	heap.Init(h)

	for h.Len() > 0 {
		cur := heap.Pop(h).(State)
		if cur.node == dst { return cur.cost }
		if cur.stops > k { continue }
		if cur.stops >= bestStops[cur.node] { continue }
		bestStops[cur.node] = cur.stops

		for _, e := range adj[cur.node] {
			heap.Push(h, State{e.to, cur.cost + e.cost, cur.stops + 1})
		}
	}
	return -1
}

func main() {
	flights := [][]int{{0,1,100},{1,2,100},{0,2,500}}
	fmt.Println(findCheapestPrice(3, flights, 0, 2, 1)) // 200
	fmt.Println(findCheapestPrice(3, flights, 0, 2, 0)) // 500
}
```

**Textual Figure:**

```
Flights: 0──100─▒1, 1──100─▒2, 0──500─▒2
src=0, dst=2

       0
      / \
  100/   \500
    /     \
   1─────2
     100

Case 1: k=1 (at most 1 stop)
┌──────┬───────────────┬─────────┬────────┐
│ Step │ State(n,c,s)  │ Action  │ Result │
├──────┼───────────────┼─────────┼────────┤
│  1   │ (0, 0, 0)     │ expand  │        │
│  2   │ (1, 100, 1)   │ expand  │        │
│  3   │ (2, 200, 2)   │ dst!    │  200   │
└──────┴───────────────┴─────────┴────────┘
  Path: 0 → 1 → 2, cost = 200  (1 stop ≤ k=1) ✓

Case 2: k=0 (no stops allowed, must be direct)
  Only direct flight: 0 → 2 = 500
  Answer: 500
```

---

## Example 5: Minimum Effort Path (LeetCode 1631)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Cell struct{ r, c, effort int }
type PQ []Cell
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].effort < h[j].effort }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Cell)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minimumEffortPath(heights [][]int) int {
	rows, cols := len(heights), len(heights[0])
	dist := make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] { dist[i][j] = math.MaxInt64 }
	}
	dist[0][0] = 0

	h := &PQ{{0, 0, 0}}
	heap.Init(h)
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	for h.Len() > 0 {
		cur := heap.Pop(h).(Cell)
		if cur.r == rows-1 && cur.c == cols-1 { return cur.effort }
		if cur.effort > dist[cur.r][cur.c] { continue }
		for _, d := range dirs {
			nr, nc := cur.r+d[0], cur.c+d[1]
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
			diff := heights[nr][nc] - heights[cur.r][cur.c]
			if diff < 0 { diff = -diff }
			newEffort := diff
			if cur.effort > newEffort { newEffort = cur.effort }
			if newEffort < dist[nr][nc] {
				dist[nr][nc] = newEffort
				heap.Push(h, Cell{nr, nc, newEffort})
			}
		}
	}
	return 0
}

func main() {
	heights := [][]int{{1,2,2},{3,8,2},{5,3,5}}
	fmt.Println(minimumEffortPath(heights)) // 2
}
```

**Textual Figure:**

```
Grid heights:
  ┌───┬───┬───┐
  │ 1 │ 2 │ 2 │
  ├───┼───┼───┤
  │ 3 │ 8 │ 2 │
  ├───┼───┼───┤
  │ 5 │ 3 │ 5 │
  └───┴───┴───┘

Effort = max absolute height difference along path
Modified Dijkstra (minimize max-edge):

  Path: (0,0)→(0,1)→(0,2)→(1,2)→(2,2)
  Diffs:  |1-2|=1  |2-2|=0  |2-2|=0  |2-5|=3
  Max effort = 3

  Path: (0,0)→(0,1)→(0,2)→(1,2)→(1,1)→(2,1)→(2,2)
  Diffs:    1      0       0       6      5      2
  Max effort = 6

  Optimal: (0,0)→(0,1)→(0,2)→(1,2)→(2,2)
  But also: (0,0)→(1,0)→(2,0)→(2,1)→(2,2)
  Diffs:       2      2       2      2
  Max effort = 2  ← optimal!

  Answer: 2
```

---

## Example 6: Swim in Rising Water (LeetCode 778)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Cell struct{ r, c, t int }
type PQ []Cell
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].t < h[j].t }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Cell)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func swimInWater(grid [][]int) int {
	n := len(grid)
	visited := make([][]bool, n)
	for i := range visited { visited[i] = make([]bool, n) }

	h := &PQ{{0, 0, grid[0][0]}}
	heap.Init(h)
	visited[0][0] = true
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	for h.Len() > 0 {
		cur := heap.Pop(h).(Cell)
		if cur.r == n-1 && cur.c == n-1 { return cur.t }
		for _, d := range dirs {
			nr, nc := cur.r+d[0], cur.c+d[1]
			if nr < 0 || nr >= n || nc < 0 || nc >= n || visited[nr][nc] { continue }
			visited[nr][nc] = true
			t := cur.t
			if grid[nr][nc] > t { t = grid[nr][nc] }
			heap.Push(h, Cell{nr, nc, t})
		}
	}
	return -1
}

func main() {
	grid := [][]int{{0,2},{1,3}}
	fmt.Println(swimInWater(grid)) // 3
}
```

**Textual Figure:**

```
Grid elevations:
  ┌───┬───┐
  │ 0 │ 2 │
  ├───┼───┤
  │ 1 │ 3 │
  └───┴───┘

Swim at time t: can traverse cells with elevation ≤ t

Modified Dijkstra (minimize max elevation along path):
┌──────┬────────────┬────────────────┐
│ Step │ Pop (r,c,t) │ Action          │
├──────┼────────────┼────────────────┤
│  1   │ (0,0, 0)   │ push neighbors  │
│  2   │ (1,0, 1)   │ push (1,1,t=3)  │
│  3   │ (0,1, 2)   │ push (1,1,t=3)  │
│  4   │ (1,1, 3)   │ destination!    │
└──────┴────────────┴────────────────┘

  Path: (0,0) → (1,0) → (1,1)
  Max elevation = max(0, 1, 3) = 3
  Answer: 3  (must wait until t=3 to swim through)
```

---

## Example 7: Dijkstra on Undirected Graph

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, dist int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func dijkstraUndirected(n int, edges [][3]int, src int) []int {
	adj := make([][]Edge, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]}) // undirected
	}

	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	h := &PQ{{src, 0}}
	heap.Init(h)

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if cur.dist > dist[cur.node] { continue }
		for _, e := range adj[cur.node] {
			nd := dist[cur.node] + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				heap.Push(h, Item{e.to, nd})
			}
		}
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,7},{0,2,9},{0,5,14},{1,2,10},{1,3,15},{2,3,11},{2,5,2},{3,4,6},{4,5,9}}
	dist := dijkstraUndirected(6, edges, 0)
	fmt.Println(dist) // [0 7 9 20 26 11]
}
```

**Textual Figure:**

```
Undirected weighted graph (6 nodes):

        0
       /|\
    7/ 9| \14
     /  |  \
    1   2   5
    |\  |  /
  15| \2| /2
    |  \|/
    3  (2)──
    |   |  \
   11  11  2\
    |   |    5
    3────
     6\
       4──9──5

Dijkstra from source 0:
┌──────┬───────────┬───────────────────────────────┐
│ Step │ Pop       │ dist = [0, 1, 2, 3, 4, 5]    │
├──────┼───────────┼───────────────────────────────┤
│ init │           │ [0, ∞, ∞, ∞, ∞, ∞]          │
│  1   │ (0, 0)    │ [0, 7, 9, ∞, ∞, 14]         │
│  2   │ (1, 7)    │ [0, 7, 9, 22, ∞, 14]        │
│  3   │ (2, 9)    │ [0, 7, 9, 20, ∞, 11]        │
│  4   │ (5, 11)   │ [0, 7, 9, 20, 20, 11]       │
│  5   │ (3, 20)   │ [0, 7, 9, 20, 26, 11]       │
│  6   │ (4, 26)   │ [0, 7, 9, 20, 26, 11] final │
└──────┴───────────┴───────────────────────────────┘

Result: [0, 7, 9, 20, 26, 11]
```

---

## Example 8: Multi-Source Dijkstra

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type Item struct{ node, dist int }
type PQ []Item
func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool   { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{})  { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func multiSourceDijkstra(n int, adj [][]Edge, sources []int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }

	h := &PQ{}
	heap.Init(h)
	for _, s := range sources {
		dist[s] = 0
		heap.Push(h, Item{s, 0})
	}

	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if cur.dist > dist[cur.node] { continue }
		for _, e := range adj[cur.node] {
			nd := dist[cur.node] + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				heap.Push(h, Item{e.to, nd})
			}
		}
	}
	return dist
}

func main() {
	adj := make([][]Edge, 5)
	for _, e := range [][3]int{{0,1,3},{1,2,1},{2,3,4},{3,4,2},{0,4,10}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}
	// Sources: 0 and 4
	dist := multiSourceDijkstra(5, adj, []int{0, 4})
	fmt.Println(dist) // [0 3 4 2 0]  — min distance from {0, 4}
}
```

**Textual Figure:**

```
Undirected graph:

  0 ──3── 1 ──1── 2 ──4── 3 ──2── 4
  │                             │
  └──────────10────────────┘

Multi-source Dijkstra with sources = {0, 4}:
  Initialize: dist[0]=0, dist[4]=0, others=∞
  Heap: [(0,0), (4,0)]

┌──────┬─────────┬────────────────────────┐
│ Step │ Pop     │ dist = [0,1,2,3,4]     │
├──────┼─────────┼────────────────────────┤
│ init │         │ [0, ∞, ∞, ∞, 0]       │
│  1   │ (0, 0)  │ [0, 3, ∞, ∞, 0]       │
│  2   │ (4, 0)  │ [0, 3, ∞, 2, 0]        │
│  3   │ (3, 2)  │ [0, 3, 6, 2, 0]        │
│  4   │ (1, 3)  │ [0, 3, 4, 2, 0]        │
│  5   │ (2, 4)  │ [0, 3, 4, 2, 0] final  │
└──────┴─────────┴────────────────────────┘

Each node's distance = min dist to nearest source:
  Node 2: dist=4 (closer to source 0 via 0→1→2)
  Node 3: dist=2 (closer to source 4 via 4→3)
```

---

## Example 9: 0-1 BFS (Deque-based Dijkstra for 0/1 weights)

```go
package main

import "fmt"

type Edge struct{ to, w int }

func bfs01(n int, adj [][]Edge, src int) []int {
	const INF = 1<<31 - 1
	dist := make([]int, n)
	for i := range dist { dist[i] = INF }
	dist[src] = 0

	deque := []int{src}
	for len(deque) > 0 {
		v := deque[0]; deque = deque[1:]
		for _, e := range adj[v] {
			nd := dist[v] + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				if e.w == 0 {
					deque = append([]int{e.to}, deque...) // front
				} else {
					deque = append(deque, e.to) // back
				}
			}
		}
	}
	return dist
}

func main() {
	// Grid: 0-weight = same region, 1-weight = cross boundary
	adj := make([][]Edge, 4)
	adj[0] = []Edge{{1, 0}, {2, 1}}
	adj[1] = []Edge{{0, 0}, {3, 1}}
	adj[2] = []Edge{{0, 1}, {3, 0}}
	adj[3] = []Edge{{1, 1}, {2, 0}}
	fmt.Println(bfs01(4, adj, 0)) // [0 0 1 1]
}
```

**Textual Figure:**

```
0-1 BFS graph (edge weights are 0 or 1):

   0 ──0── 1
   │         │
  1│        1│
   │         │
   2 ──0── 3

Deque-based Dijkstra (0-weight → push front, 1-weight → push back):
┌──────┬──────────────┬────────────────────┬────────────────┐
│ Step │ Pop          │ Deque              │ dist           │
├──────┼──────────────┼────────────────────┼────────────────┤
│ init │              │ [0]                │ [0,∞,∞,∞]     │
│  1   │ 0            │ [1, 2] (1→front)  │ [0, 0, 1, ∞]   │
│  2   │ 1 (w=0)      │ [2, 3]             │ [0, 0, 1, 1]   │
│  3   │ 2 (w=1)      │ [3]                │ [0, 0, 1, 1]   │
│  4   │ 3            │ []                 │ [0, 0, 1, 1]   │
└──────┴──────────────┴────────────────────┴────────────────┘

Result: [0, 0, 1, 1]
  0→1: weight 0 (same region)
  0→2: weight 1 (cross boundary)
  0→3: weight 1 (via 0→1→3 = 0+1 = 1)
```

---

## Example 10: When NOT to Use Dijkstra (Negative Weights)

```go
package main

import (
	"fmt"
	"math"
)

// Dijkstra gives WRONG result with negative edges
// Bellman-Ford gives correct result
func bellmanFord(n int, edges [][3]int, src int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
			}
		}
	}
	return dist
}

func main() {
	// Edge 1→2 has weight -3 (negative)
	edges := [][3]int{{0,1,4},{0,2,5},{1,2,-3}}

	fmt.Println("Bellman-Ford:", bellmanFord(3, edges, 0)) // [0 4 1] ← correct
	fmt.Println("(Dijkstra would give [0 4 5] — WRONG)")
}
```

**Textual Figure:**

```
Graph with negative edge:

  0 ──4─→ 1
  │       │
  5\    /-3    (negative!)
   \ \ / /
    \v/v
     2

Dijkstra (WRONG with negative edges):
  Pop (0,0): dist[1]=4, dist[2]=5
  Pop (1,4): no improvement to 2 (4+(-3)=1 but 1 already finalized ✗)
  Pop (2,5): finalized
  Result: [0, 4, 5] ← WRONG (dist[2] should be 1)

Bellman-Ford (CORRECT):
┌───────┬───────────────┬───────────────────────┐
│ Pass  │ Relax         │ dist = [0, 1, 2]      │
├───────┼───────────────┼───────────────────────┤
│ init  │               │ [0, ∞, ∞]             │
│   1   │ 0→1:4, 0→2:5 │ [0, 4, 5]              │
│       │ 1→2: 4-3=1   │ [0, 4, 1]  ← updated! │
│   2   │ no changes    │ [0, 4, 1]  final      │
└───────┴───────────────┴───────────────────────┘

Bellman-Ford correctly finds: [0, 4, 1]
  0→2: via 0→1→2 = 4+(-3) = 1  (not direct 5)
```

---

## Summary Table

| Variant | Time | Use Case |
|---------|------|----------|
| Basic Dijkstra | O((V+E) log V) | Single-source, non-negative weights |
| With path reconstruction | O((V+E) log V) | Need actual path |
| Multi-source | O((V+E) log V) | Nearest facility |
| 0-1 BFS | O(V+E) | Edge weights 0 or 1 only |
| Grid Dijkstra | O(RC log(RC)) | Min-cost/effort path in grid |

## Key Takeaways

1. Dijkstra = greedy BFS with min-heap for weighted graphs
2. **Only works with non-negative weights** — use Bellman-Ford for negative edges
3. Skip stale entries: `if cur.dist > dist[cur.node] { continue }`
4. 0-1 BFS: O(V+E) alternative when weights are only 0 or 1
5. Modified Dijkstra handles diverse problems: k-stops limit, max-of-edges, etc.

> **Next up:** Bellman-Ford Algorithm →
