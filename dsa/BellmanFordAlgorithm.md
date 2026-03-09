# Phase 12: Graphs — Bellman-Ford Algorithm

## Overview

**Bellman-Ford** finds shortest paths from a single source in graphs with **negative edge weights** and can **detect negative cycles**.

- **Time:** O(V × E)
- **Space:** O(V)
- Handles negative edges (unlike Dijkstra)
- Detects negative cycles (extra pass)

```
Algorithm:
1. Set dist[source] = 0, all others = ∞
2. Repeat V-1 times: relax all edges
3. Extra pass: if any edge still relaxes → negative cycle
```

---

## Example 1: Basic Bellman-Ford

```go
package main

import (
	"fmt"
	"math"
)

func bellmanFord(n int, edges [][3]int, src int) ([]int, bool) {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	// Relax V-1 times
	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
			}
		}
	}

	// Check for negative cycle
	for _, e := range edges {
		u, v, w := e[0], e[1], e[2]
		if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
			return dist, true // negative cycle
		}
	}
	return dist, false
}

func main() {
	edges := [][3]int{{0,1,4},{0,2,5},{1,2,-3},{2,3,3}}
	dist, hasCycle := bellmanFord(4, edges, 0)
	fmt.Println("Dist:", dist)          // [0 4 1 4]
	fmt.Println("Neg cycle:", hasCycle) // false
}
```

---

## Example 2: Detecting Negative Cycles

```go
package main

import (
	"fmt"
	"math"
)

func detectNegCycle(n int, edges [][3]int) bool {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[0] = 0

	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
			}
		}
	}

	for _, e := range edges {
		u, v, w := e[0], e[1], e[2]
		if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
			return true
		}
	}
	return false
}

func main() {
	// Negative cycle: 1→2 (-1), 2→3 (-1), 3→1 (-1) = total -3
	edges := [][3]int{{0,1,1},{1,2,-1},{2,3,-1},{3,1,-1}}
	fmt.Println("Has negative cycle:", detectNegCycle(4, edges)) // true

	// No negative cycle
	edges2 := [][3]int{{0,1,1},{1,2,-1},{2,3,2}}
	fmt.Println("Has negative cycle:", detectNegCycle(4, edges2)) // false
}
```

---

## Example 3: Bellman-Ford with Path Reconstruction

```go
package main

import (
	"fmt"
	"math"
)

func bellmanFordPath(n int, edges [][3]int, src, dst int) (int, []int) {
	dist := make([]int, n)
	prev := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64; prev[i] = -1 }
	dist[src] = 0

	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
				prev[v] = u
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
	edges := [][3]int{{0,1,4},{0,2,5},{1,2,-3},{2,3,3}}
	d, p := bellmanFordPath(4, edges, 0, 3)
	fmt.Println("Dist:", d, "Path:", p) // Dist: 4 Path: [0 1 2 3]
}
```

---

## Example 4: Cheapest Flights Within K Stops (LeetCode 787, BF Variant)

```go
package main

import (
	"fmt"
	"math"
)

func findCheapestPrice(n int, flights [][]int, src, dst, k int) int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	for i := 0; i <= k; i++ {
		temp := make([]int, n)
		copy(temp, dist)
		for _, f := range flights {
			u, v, w := f[0], f[1], f[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < temp[v] {
				temp[v] = dist[u] + w
			}
		}
		dist = temp
	}

	if dist[dst] == math.MaxInt64 { return -1 }
	return dist[dst]
}

func main() {
	flights := [][]int{{0,1,100},{1,2,100},{0,2,500}}
	fmt.Println(findCheapestPrice(3, flights, 0, 2, 1)) // 200
	fmt.Println(findCheapestPrice(3, flights, 0, 2, 0)) // 500
}
```

**Why temp copy?** Prevents using edges from the same iteration — ensures at most k+1 edges.

---

## Example 5: SPFA (Bellman-Ford with Queue Optimization)

```go
package main

import (
	"fmt"
	"math"
)

type Edge struct{ to, w int }

func spfa(n int, adj [][]Edge, src int) ([]int, bool) {
	dist := make([]int, n)
	inQueue := make([]bool, n)
	count := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	queue := []int{src}
	inQueue[src] = true
	count[src] = 1

	for len(queue) > 0 {
		u := queue[0]; queue = queue[1:]
		inQueue[u] = false
		for _, e := range adj[u] {
			if dist[u]+e.w < dist[e.to] {
				dist[e.to] = dist[u] + e.w
				if !inQueue[e.to] {
					queue = append(queue, e.to)
					inQueue[e.to] = true
					count[e.to]++
					if count[e.to] >= n { return dist, true } // neg cycle
				}
			}
		}
	}
	return dist, false
}

func main() {
	adj := make([][]Edge, 4)
	adj[0] = []Edge{{1, 4}, {2, 5}}
	adj[1] = []Edge{{2, -3}}
	adj[2] = []Edge{{3, 3}}

	dist, negCycle := spfa(4, adj, 0)
	fmt.Println("Dist:", dist)    // [0 4 1 4]
	fmt.Println("Neg cycle:", negCycle) // false
}
```

---

## Example 6: Bellman-Ford for Undirected Graph

```go
package main

import (
	"fmt"
	"math"
)

func bellmanFordUndirected(n int, edges [][3]int, src int) []int {
	// Convert undirected to directed (both directions)
	directed := make([][3]int, 0, len(edges)*2)
	for _, e := range edges {
		directed = append(directed, e)
		directed = append(directed, [3]int{e[1], e[0], e[2]})
	}

	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	for i := 0; i < n-1; i++ {
		for _, e := range directed {
			if dist[e[0]] != math.MaxInt64 && dist[e[0]]+e[2] < dist[e[1]] {
				dist[e[1]] = dist[e[0]] + e[2]
			}
		}
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,4},{0,2,1},{1,2,2},{1,3,5},{2,3,8}}
	fmt.Println(bellmanFordUndirected(4, edges, 0)) // [0 3 1 8]
}
```

---

## Example 7: Find Nodes in Negative Cycle

```go
package main

import (
	"fmt"
	"math"
)

func nodesInNegCycle(n int, edges [][3]int) []bool {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[0] = 0

	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
			}
		}
	}

	// Extra passes to find all affected nodes
	affected := make([]bool, n)
	for i := 0; i < n; i++ {
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
				affected[v] = true
				affected[u] = true
			}
		}
	}
	return affected
}

func main() {
	edges := [][3]int{{0,1,1},{1,2,-1},{2,3,-1},{3,1,-1},{3,4,2}}
	aff := nodesInNegCycle(5, edges)
	for i, a := range aff {
		if a { fmt.Printf("Node %d in/reachable from neg cycle\n", i) }
	}
}
```

---

## Example 8: Currency Arbitrage Detection

```go
package main

import (
	"fmt"
	"math"
)

func hasArbitrage(rates [][]float64) bool {
	n := len(rates)
	// Convert to -log weights so multiplication becomes addition
	edges := [][3]float64{}
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			if i != j && rates[i][j] > 0 {
				edges = append(edges, [3]float64{float64(i), float64(j), -math.Log(rates[i][j])})
			}
		}
	}

	dist := make([]float64, n)
	for i := range dist { dist[i] = 1e18 }
	dist[0] = 0

	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			u, v, w := int(e[0]), int(e[1]), e[2]
			if dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
			}
		}
	}

	for _, e := range edges {
		u, v, w := int(e[0]), int(e[1]), e[2]
		if dist[u]+w < dist[v]-1e-9 { return true }
	}
	return false
}

func main() {
	// USD, EUR, GBP — rates that create arbitrage
	rates := [][]float64{
		{1, 0.9, 0.75},
		{1.15, 1, 0.85},
		{1.4, 1.2, 1},
	}
	fmt.Println("Arbitrage?", hasArbitrage(rates)) // true
}
```

---

## Example 9: Bellman-Ford vs Dijkstra Comparison

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

func dijkstra(n int, adj [][]Edge, src int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0
	h := &PQ{{src, 0}}; heap.Init(h)
	for h.Len() > 0 {
		cur := heap.Pop(h).(Item)
		if cur.dist > dist[cur.node] { continue }
		for _, e := range adj[cur.node] {
			nd := dist[cur.node] + e.w
			if nd < dist[e.to] { dist[e.to] = nd; heap.Push(h, Item{e.to, nd}) }
		}
	}
	return dist
}

func bellmanFord(n int, edges [][3]int, src int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0
	for i := 0; i < n-1; i++ {
		for _, e := range edges {
			if dist[e[0]] != math.MaxInt64 && dist[e[0]]+e[2] < dist[e[1]] {
				dist[e[1]] = dist[e[0]] + e[2]
			}
		}
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,4},{0,2,1},{2,1,2},{1,3,1},{2,3,5},{3,4,3}}
	adj := make([][]Edge, 5)
	for _, e := range edges { adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]}) }

	fmt.Println("Dijkstra:    ", dijkstra(5, adj, 0))       // [0 3 1 4 7]
	fmt.Println("Bellman-Ford:", bellmanFord(5, edges, 0))   // [0 3 1 4 7]
}
```

---

## Example 10: Early Termination Optimization

```go
package main

import (
	"fmt"
	"math"
)

func bellmanFordEarly(n int, edges [][3]int, src int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = math.MaxInt64 }
	dist[src] = 0

	for i := 0; i < n-1; i++ {
		updated := false
		for _, e := range edges {
			u, v, w := e[0], e[1], e[2]
			if dist[u] != math.MaxInt64 && dist[u]+w < dist[v] {
				dist[v] = dist[u] + w
				updated = true
			}
		}
		if !updated { break } // converged early
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,1},{1,2,1},{2,3,1},{3,4,1}}
	fmt.Println(bellmanFordEarly(5, edges, 0)) // [0 1 2 3 4]
	// Converges in 4 passes (not worst case)
}
```

---

## Summary Table

| Feature | Bellman-Ford | Dijkstra | SPFA |
|---------|-------------|----------|------|
| Negative edges | ✅ | ❌ | ✅ |
| Negative cycle detection | ✅ | ❌ | ✅ |
| Time | O(VE) | O((V+E)logV) | O(VE) worst |
| Best for | Negative weights, k-stops | Non-negative | Sparse graphs |

## Key Takeaways

1. Relax all edges V-1 times → guarantees shortest paths
2. Extra pass detects negative cycles
3. K-stops variant: use temp array to prevent same-round relaxation
4. SPFA = queue-based BF; faster average case, same worst case
5. Choose Dijkstra for non-negative weights; Bellman-Ford when negatives possible

> **Next up:** Floyd-Warshall Algorithm →
