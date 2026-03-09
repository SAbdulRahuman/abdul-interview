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
