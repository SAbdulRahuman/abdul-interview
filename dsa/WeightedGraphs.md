# Phase 12: Graphs — Weighted Graphs

## Overview

A **weighted graph** assigns a numerical **weight** (cost, distance, capacity) to each edge. Weights can be positive, negative, or zero.

```
     4       2
  0 ——→ 1 ——→ 3
  |           ↑
  |1          |5
  ↓           |
  2 ——————————┘

type Edge struct { To, Weight int }
adj[0] = [{1,4}, {2,1}]
adj[1] = [{3,2}]
adj[2] = [{3,5}]
```

| Algorithm | Handles Negative? | Time |
|-----------|------------------|------|
| Dijkstra | No | O((V+E) log V) |
| Bellman-Ford | Yes | O(VE) |
| Floyd-Warshall | Yes | O(V³) |

---

## Example 1: Weighted Adjacency List

```go
package main

import "fmt"

type Edge struct {
	To, Weight int
}

func main() {
	n := 5
	adj := make([][]Edge, n)

	add := func(u, v, w int) {
		adj[u] = append(adj[u], Edge{v, w})
		adj[v] = append(adj[v], Edge{u, w}) // undirected
	}

	add(0, 1, 4)
	add(0, 2, 1)
	add(1, 3, 2)
	add(2, 3, 5)
	add(3, 4, 3)

	for v, edges := range adj {
		fmt.Printf("%d:", v)
		for _, e := range edges {
			fmt.Printf(" →(%d, w=%d)", e.To, e.Weight)
		}
		fmt.Println()
	}
}
```

---

## Example 2: Dijkstra's Algorithm

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ To, Wt int }
type State struct{ node, dist int }

type PQ []State
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(State)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func dijkstra(adj [][]Edge, src int) []int {
	n := len(adj)
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[src] = 0

	pq := &PQ{}
	heap.Push(pq, State{src, 0})

	for pq.Len() > 0 {
		s := heap.Pop(pq).(State)
		if s.dist > dist[s.node] { continue } // lazy skip
		for _, e := range adj[s.node] {
			nd := s.dist + e.Wt
			if nd < dist[e.To] {
				dist[e.To] = nd
				heap.Push(pq, State{e.To, nd})
			}
		}
	}
	return dist
}

func main() {
	adj := make([][]Edge, 5)
	for _, e := range [][3]int{{0,1,4},{0,2,1},{1,3,2},{2,1,2},{2,3,5},{3,4,3}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
	}
	fmt.Println(dijkstra(adj, 0)) // [0 3 1 5 8]
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
type State struct{ node, dist int }
type PQ []State
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(State)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func networkDelayTime(times [][]int, n int, k int) int {
	adj := make([][]Edge, n+1)
	for _, t := range times {
		adj[t[0]] = append(adj[t[0]], Edge{t[1], t[2]})
	}

	dist := make([]int, n+1)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[k] = 0

	pq := &PQ{}
	heap.Push(pq, State{k, 0})
	for pq.Len() > 0 {
		s := heap.Pop(pq).(State)
		if s.dist > dist[s.node] { continue }
		for _, e := range adj[s.node] {
			nd := s.dist + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				heap.Push(pq, State{e.to, nd})
			}
		}
	}

	maxTime := 0
	for i := 1; i <= n; i++ {
		if dist[i] == math.MaxInt32 { return -1 }
		if dist[i] > maxTime { maxTime = dist[i] }
	}
	return maxTime
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

import "fmt"

func findCheapestPrice(n int, flights [][]int, src int, dst int, k int) int {
	// Bellman-Ford with k+1 iterations
	INF := 1<<31 - 1
	dist := make([]int, n)
	for i := range dist { dist[i] = INF }
	dist[src] = 0

	for i := 0; i <= k; i++ {
		tmp := make([]int, n)
		copy(tmp, dist)
		for _, f := range flights {
			u, v, w := f[0], f[1], f[2]
			if dist[u] != INF && dist[u]+w < tmp[v] {
				tmp[v] = dist[u] + w
			}
		}
		dist = tmp
	}

	if dist[dst] == INF { return -1 }
	return dist[dst]
}

func main() {
	flights := [][]int{{0,1,100},{1,2,100},{2,0,100},{1,3,600},{2,3,200}}
	fmt.Println(findCheapestPrice(4, flights, 0, 3, 1)) // 700
	fmt.Println(findCheapestPrice(4, flights, 0, 3, 2)) // 400
}
```

---

## Example 5: Bellman-Ford Algorithm

```go
package main

import (
	"fmt"
	"math"
)

type Edge struct{ from, to, weight int }

func bellmanFord(n int, edges []Edge, src int) ([]int, bool) {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt32 }
	dist[src] = 0

	// Relax all edges V-1 times
	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			if dist[e.from] != math.MaxInt32 && dist[e.from]+e.weight < dist[e.to] {
				dist[e.to] = dist[e.from] + e.weight
			}
		}
	}

	// Check for negative cycles
	for _, e := range edges {
		if dist[e.from] != math.MaxInt32 && dist[e.from]+e.weight < dist[e.to] {
			return dist, true // negative cycle
		}
	}
	return dist, false
}

func main() {
	edges := []Edge{{0,1,4},{0,2,5},{1,2,-3},{2,3,4},{1,3,6}}
	dist, negCycle := bellmanFord(4, edges, 0)
	fmt.Println("Distances:", dist)    // [0 4 1 5]
	fmt.Println("Neg cycle:", negCycle) // false
}
```

---

## Example 6: Minimum Spanning Tree — Kruskal's

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

	totalWeight := 0
	var mst []Edge
	for _, e := range edges {
		if union(e.u, e.v) {
			totalWeight += e.w
			mst = append(mst, e)
		}
	}
	return totalWeight, mst
}

func main() {
	edges := []Edge{{0,1,4},{0,2,8},{1,2,2},{1,3,5},{2,3,5},{2,4,7},{3,4,2}}
	total, mst := kruskal(5, edges)
	fmt.Println("MST weight:", total) // 13
	for _, e := range mst {
		fmt.Printf("  %d — %d (w=%d)\n", e.u, e.v, e.w)
	}
}
```

---

## Example 7: Swim in Rising Water (LeetCode 778)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Cell struct{ time, r, c int }
type PQ []Cell
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].time < h[j].time }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(Cell)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func swimInWater(grid [][]int) int {
	n := len(grid)
	visited := make([][]bool, n)
	for i := range visited { visited[i] = make([]bool, n) }

	pq := &PQ{}
	heap.Push(pq, Cell{grid[0][0], 0, 0})
	visited[0][0] = true
	dirs := [][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	for pq.Len() > 0 {
		cur := heap.Pop(pq).(Cell)
		if cur.r == n-1 && cur.c == n-1 { return cur.time }
		for _, d := range dirs {
			nr, nc := cur.r+d[0], cur.c+d[1]
			if nr < 0 || nr >= n || nc < 0 || nc >= n || visited[nr][nc] { continue }
			visited[nr][nc] = true
			t := cur.time
			if grid[nr][nc] > t { t = grid[nr][nc] }
			heap.Push(pq, Cell{t, nr, nc})
		}
	}
	return -1
}

func main() {
	grid := [][]int{{0,2},{1,3}}
	fmt.Println(swimInWater(grid)) // 3

	grid2 := [][]int{
		{0, 1, 2, 3, 4},
		{24,23,22,21,5},
		{12,13,14,15,16},
		{11,17,18,19,20},
		{10, 9, 8, 7, 6},
	}
	fmt.Println(swimInWater(grid2)) // 16
}
```

---

## Example 8: Shortest Path with Dijkstra + Path Reconstruction

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct{ to, w int }
type State struct{ node, dist int }
type PQ []State
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].dist < h[j].dist }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(State)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func shortestPath(adj [][]Edge, src, dst int) (int, []int) {
	n := len(adj)
	dist := make([]int, n)
	prev := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt32; prev[i] = -1 }
	dist[src] = 0

	pq := &PQ{}
	heap.Push(pq, State{src, 0})
	for pq.Len() > 0 {
		s := heap.Pop(pq).(State)
		if s.dist > dist[s.node] { continue }
		for _, e := range adj[s.node] {
			nd := s.dist + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				prev[e.to] = s.node
				heap.Push(pq, State{e.to, nd})
			}
		}
	}

	// Reconstruct path
	path := []int{}
	for cur := dst; cur != -1; cur = prev[cur] {
		path = append([]int{cur}, path...)
	}
	return dist[dst], path
}

func main() {
	adj := make([][]Edge, 5)
	for _, e := range [][3]int{{0,1,4},{0,2,1},{2,1,2},{1,3,1},{2,3,5},{3,4,3}} {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
	}
	d, path := shortestPath(adj, 0, 4)
	fmt.Println("Distance:", d)  // 7
	fmt.Println("Path:", path)    // [0 2 1 3 4]
}
```

---

## Example 9: Minimum Effort Path (LeetCode 1631)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type State struct{ effort, r, c int }
type PQ []State
func (h PQ) Len() int           { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].effort < h[j].effort }
func (h PQ) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x interface{}) { *h = append(*h, x.(State)) }
func (h *PQ) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minimumEffortPath(heights [][]int) int {
	rows, cols := len(heights), len(heights[0])
	dist := make([][]int, rows)
	for i := range dist {
		dist[i] = make([]int, cols)
		for j := range dist[i] { dist[i][j] = math.MaxInt32 }
	}
	dist[0][0] = 0

	pq := &PQ{}
	heap.Push(pq, State{0, 0, 0})
	dirs := [][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	for pq.Len() > 0 {
		s := heap.Pop(pq).(State)
		if s.r == rows-1 && s.c == cols-1 { return s.effort }
		if s.effort > dist[s.r][s.c] { continue }
		for _, d := range dirs {
			nr, nc := s.r+d[0], s.c+d[1]
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
			diff := heights[nr][nc] - heights[s.r][s.c]
			if diff < 0 { diff = -diff }
			maxE := s.effort
			if diff > maxE { maxE = diff }
			if maxE < dist[nr][nc] {
				dist[nr][nc] = maxE
				heap.Push(pq, State{maxE, nr, nc})
			}
		}
	}
	return 0
}

func main() {
	fmt.Println(minimumEffortPath([][]int{{1,2,2},{3,8,2},{5,3,5}})) // 2
	fmt.Println(minimumEffortPath([][]int{{1,2,3},{3,8,4},{5,3,5}})) // 1
}
```

---

## Example 10: Weighted Graph Algorithm Selection

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Weighted Graph Algorithm Selection Guide ===")
	fmt.Println()
	fmt.Println("| Problem Type                  | Algorithm      | Time           |")
	fmt.Println("|-------------------------------|----------------|----------------|")
	fmt.Println("| SSSP, non-negative weights    | Dijkstra       | O((V+E)log V)  |")
	fmt.Println("| SSSP, negative weights        | Bellman-Ford   | O(VE)          |")
	fmt.Println("| SSSP, K stops limit           | BF (K iters)   | O(KE)          |")
	fmt.Println("| All-pairs shortest path       | Floyd-Warshall | O(V³)          |")
	fmt.Println("| Minimum spanning tree         | Kruskal/Prim   | O(E log E)     |")
	fmt.Println("| Min bottleneck path           | Modified Dijk.  | O((V+E)log V)  |")
	fmt.Println("| Shortest path in DAG          | Topo sort + DP | O(V+E)         |")
	fmt.Println()
	fmt.Println("Key patterns:")
	fmt.Println("  • Dijkstra = BFS with priority queue (greedy)")
	fmt.Println("  • Bellman-Ford = relax all edges V-1 times")
	fmt.Println("  • Grid shortest path = Dijkstra on implicit graph")
	fmt.Println("  • 'Minimize maximum edge' = modified Dijkstra or binary search")
}
```

---

## Key Takeaways

1. Use `type Edge struct { To, Weight int }` for weighted adjacency list
2. **Dijkstra**: non-negative weights, O((V+E) log V) with min-heap
3. **Bellman-Ford**: handles negative weights, detects negative cycles
4. **Floyd-Warshall**: all-pairs, O(V³), use adjacency matrix
5. Grid problems are weighted graphs — Dijkstra works on them directly

> **Next up:** BFS (Graph) →
