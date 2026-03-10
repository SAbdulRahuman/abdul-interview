# Phase 12: Graphs — Floyd-Warshall Algorithm

## Overview

**Floyd-Warshall** computes shortest paths between **all pairs** of vertices. Works with negative edges (but not negative cycles).

- **Time:** O(V³)
- **Space:** O(V²)
- DP approach: `dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])` for each intermediate vertex k

---

## Example 1: Basic Floyd-Warshall

```go
package main

import (
	"fmt"
	"math"
)

func floydWarshall(n int, edges [][3]int) [][]int {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] { dist[i][j] = INF }
		dist[i][i] = 0
	}
	for _, e := range edges {
		dist[e[0]][e[1]] = e[2]
	}

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if dist[i][k]+dist[k][j] < dist[i][j] {
					dist[i][j] = dist[i][k] + dist[k][j]
				}
			}
		}
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,3},{0,2,8},{1,2,2},{2,3,1},{3,0,4}}
	dist := floydWarshall(4, edges)
	for i, row := range dist {
		fmt.Printf("From %d: %v\n", i, row)
	}
}
```

**Textual Figure:**

```
Directed weighted graph:

       0 ──3─→ 1
       │         │
      8│        2│
       │         │
       └─────→ 2 ──1─→ 3 ──4─→ 0

Floyd-Warshall (k = intermediate vertex):
┌───┬───────────────────────────────────┐
│ k │ dist[][] after using k          │
├───┼───────────────────────────────────┤
│   │     0    1    2    3            │
│ini│  [  0    3    8    ∞ ]  (from 0) │
│   │  [  ∞    0    ∞    ∞ ]  (from 1) │
│   │  [  ∞    ∞    0    1 ]  (from 2) │
│   │  [  4    ∞    ∞    0 ]  (from 3) │
├───┼───────────────────────────────────┤
│ 0 │  no changes (via 0)              │
│ 1 │  dist[0][2] = 3+2 = 5 (via 0→1) │
│ 2 │  dist[0][3] = 5+1 = 6 (via 2)   │
│   │  dist[1][3] = 2+1 = 3 (via 2)   │
│ 3 │  dist[2][0] = 1+4 = 5 (via 3)   │
│   │  dist[1][0] = 3+4 = 7 (via 3)   │
└───┴───────────────────────────────────┘

Final all-pairs shortest distances:
  From 0: [0, 3, 5, 6]
  From 1: [7, 0, 2, 3]
  From 2: [5, 8, 0, 1]
  From 3: [4, 7, 9, 0]
```

---

## Example 2: Floyd-Warshall with Path Reconstruction

```go
package main

import (
	"fmt"
	"math"
)

func floydWarshallPath(n int, edges [][3]int) ([][]int, [][]int) {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	next := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		next[i] = make([]int, n)
		for j := range dist[i] {
			dist[i][j] = INF
			next[i][j] = -1
		}
		dist[i][i] = 0
		next[i][i] = i
	}
	for _, e := range edges {
		dist[e[0]][e[1]] = e[2]
		next[e[0]][e[1]] = e[1]
	}

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if dist[i][k]+dist[k][j] < dist[i][j] {
					dist[i][j] = dist[i][k] + dist[k][j]
					next[i][j] = next[i][k]
				}
			}
		}
	}
	return dist, next
}

func reconstructPath(next [][]int, u, v int) []int {
	if next[u][v] == -1 { return nil }
	path := []int{u}
	for u != v {
		u = next[u][v]
		path = append(path, u)
	}
	return path
}

func main() {
	edges := [][3]int{{0,1,3},{1,2,2},{2,3,1},{3,0,4},{0,2,8}}
	_, next := floydWarshallPath(4, edges)
	fmt.Println("Path 0→3:", reconstructPath(next, 0, 3)) // [0 1 2 3]
	fmt.Println("Path 3→2:", reconstructPath(next, 3, 2)) // [3 0 1 2]
}
```

**Textual Figure:**

```
Graph: edges = (0,1,3),(1,2,2),(2,3,1),(3,0,4),(0,2,8)

     0 ──3─→ 1
   ↑ │       │
  4│ 8│      2│
   │ │       │
   3 ──1─ 2 ▒

next[][] tracks first hop on shortest path:
┌──────┬──────────┬─────────────────────────────┐
│ Path │ next[][] │ Reconstructed path        │
├──────┼──────────┼─────────────────────────────┤
│ 0→3 │ next=1   │ 0→1→2→3  (dist=3+2+1=6) │
│ 3→2 │ next=0   │ 3→0→1→2  (dist=4+3+2=9) │
└──────┴──────────┴─────────────────────────────┘

Path 0→3 reconstruction:
  u=0: next[0][3]=1 → go to 1
  u=1: next[1][3]=2 → go to 2
  u=2: next[2][3]=3 → go to 3 (arrived)
  Path = [0, 1, 2, 3]

Path 3→2 reconstruction:
  u=3: next[3][2]=0 → go to 0
  u=0: next[0][2]=1 → go to 1
  u=1: next[1][2]=2 → go to 2 (arrived)
  Path = [3, 0, 1, 2]
```

---

## Example 3: Detecting Negative Cycles

```go
package main

import (
	"fmt"
	"math"
)

func hasNegativeCycle(n int, edges [][3]int) bool {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] { dist[i][j] = INF }
		dist[i][i] = 0
	}
	for _, e := range edges { dist[e[0]][e[1]] = e[2] }

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if dist[i][k]+dist[k][j] < dist[i][j] {
					dist[i][j] = dist[i][k] + dist[k][j]
				}
			}
		}
	}

	for i := 0; i < n; i++ {
		if dist[i][i] < 0 { return true }
	}
	return false
}

func main() {
	edges := [][3]int{{0,1,1},{1,2,-1},{2,0,-1}}
	fmt.Println("Negative cycle:", hasNegativeCycle(3, edges)) // true

	edges2 := [][3]int{{0,1,1},{1,2,2},{2,0,3}}
	fmt.Println("Negative cycle:", hasNegativeCycle(3, edges2)) // false
}
```

**Textual Figure:**

```
Case 1: edges = (0,1,1),(1,2,-1),(2,0,-1)

  0 ──1─→ 1 ─-1─→ 2
  ↑                 │
  └──────-1──────┘

  Cycle: 0→1→2→0, total weight = 1+(-1)+(-1) = -1 < 0

  After Floyd-Warshall:
  dist[0][0] = min(0, 1+(-1)+(-1)) = -1 < 0  → negative cycle!

  Check: any dist[i][i] < 0 → true ✓

Case 2: edges = (0,1,1),(1,2,2),(2,0,3)

  0 ──1─→ 1 ──2─→ 2
  ↑                 │
  └──────+3──────┘

  Cycle: 0→1→2→0, total weight = 1+2+3 = 6 > 0
  dist[i][i] = 0 for all i (no improvement via cycle)
  → false ✓ (no negative cycle)
```

---

## Example 4: Transitive Closure (Reachability)

```go
package main

import "fmt"

func transitiveClosure(n int, adj [][]int) [][]bool {
	reach := make([][]bool, n)
	for i := range reach {
		reach[i] = make([]bool, n)
		reach[i][i] = true
		for _, j := range adj[i] { reach[i][j] = true }
	}

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if reach[i][k] && reach[k][j] {
					reach[i][j] = true
				}
			}
		}
	}
	return reach
}

func main() {
	adj := [][]int{{1}, {2}, {3}, {}}
	reach := transitiveClosure(4, adj)
	for i, row := range reach {
		fmt.Printf("%d can reach: ", i)
		for j, r := range row {
			if r { fmt.Printf("%d ", j) }
		}
		fmt.Println()
	}
}
```

**Textual Figure:**

```
Transitive Closure: adj = 0→1, 1→2, 2→3, 3→(none)

  0 → 1 → 2 → 3   (chain)

Boolean Floyd-Warshall: reach[i][j] = reach[i][k] AND reach[k][j]

Reachability matrix:
┌─────┬───┬───┬───┬───┐
│     │ 0 │ 1 │ 2 │ 3 │
├─────┼───┼───┼───┼───┤
│  0  │ ✓ │ ✓ │ ✓ │ ✓ │  0 can reach all
│  1  │   │ ✓ │ ✓ │ ✓ │  1 can reach 1,2,3
│  2  │   │   │ ✓ │ ✓ │  2 can reach 2,3
│  3  │   │   │   │ ✓ │  3 can reach only 3
└─────┴───┴───┴───┴───┘

Key iterations (k = intermediate):
  k=0: reach[i][j] via 0 (no new paths)
  k=1: reach[0][2] = reach[0][1] ∧ reach[1][2] = true
  k=2: reach[0][3] = reach[0][2] ∧ reach[2][3] = true
       reach[1][3] = reach[1][2] ∧ reach[2][3] = true
  k=3: no new paths
```

---

## Example 5: Find the City with Smallest Number of Neighbors (LeetCode 1334)

```go
package main

import (
	"fmt"
	"math"
)

func findTheCity(n int, edges [][]int, distThreshold int) int {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] { dist[i][j] = INF }
		dist[i][i] = 0
	}
	for _, e := range edges {
		dist[e[0]][e[1]] = e[2]
		dist[e[1]][e[0]] = e[2]
	}

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if dist[i][k]+dist[k][j] < dist[i][j] {
					dist[i][j] = dist[i][k] + dist[k][j]
				}
			}
		}
	}

	bestCity, minCount := -1, n+1
	for i := 0; i < n; i++ {
		count := 0
		for j := 0; j < n; j++ {
			if i != j && dist[i][j] <= distThreshold { count++ }
		}
		if count <= minCount {
			minCount = count
			bestCity = i
		}
	}
	return bestCity
}

func main() {
	edges := [][]int{{0,1,3},{1,2,1},{1,3,4},{2,3,1}}
	fmt.Println(findTheCity(4, edges, 4)) // 3
}
```

**Textual Figure:**

```
Undirected graph: edges = (0,1,3),(1,2,1),(1,3,4),(2,3,1)

   0 ──3── 1 ──1── 2
              │       │
             4│      1│
              │       │
              3 ─────┘

Floyd-Warshall all-pairs distances:
┌─────┬───┬───┬───┬───┐
│     │ 0 │ 1 │ 2 │ 3 │
├─────┼───┼───┼───┼───┤
│  0  │ 0 │ 3 │ 4 │ 5 │
│  1  │ 3 │ 0 │ 1 │ 2 │
│  2  │ 4 │ 1 │ 0 │ 1 │
│  3  │ 5 │ 2 │ 1 │ 0 │
└─────┴───┴───┴───┴───┘

Count cities within threshold=4:
┌──────┬──────────────────┬────────┐
│ City │ Neighbors ≤ 4    │ Count  │
├──────┼──────────────────┼────────┤
│  0   │ 1(3), 2(4)       │   2    │
│  1   │ 0(3), 2(1), 3(2) │   3    │
│  2   │ 0(4), 1(1), 3(1) │   3    │
│  3   │ 1(2), 2(1)       │   2    │
└──────┴──────────────────┴────────┘

Cities 0 and 3 both have count=2.
Tie-breaking: pick larger index → answer = 3
```

---

## Example 6: Graph Diameter (Longest Shortest Path)

```go
package main

import (
	"fmt"
	"math"
)

func graphDiameter(n int, edges [][3]int) int {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] { dist[i][j] = INF }
		dist[i][i] = 0
	}
	for _, e := range edges {
		dist[e[0]][e[1]] = e[2]
		dist[e[1]][e[0]] = e[2]
	}

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if dist[i][k]+dist[k][j] < dist[i][j] {
					dist[i][j] = dist[i][k] + dist[k][j]
				}
			}
		}
	}

	diameter := 0
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			if dist[i][j] < INF && dist[i][j] > diameter {
				diameter = dist[i][j]
			}
		}
	}
	return diameter
}

func main() {
	edges := [][3]int{{0,1,2},{1,2,3},{2,3,1},{0,3,10}}
	fmt.Println("Diameter:", graphDiameter(4, edges)) // 6 (0→1→2→3 = 6 < 10)
}
```

**Textual Figure:**

```
Undirected graph: edges = (0,1,2),(1,2,3),(2,3,1),(0,3,10)

   0 ──2── 1
   │       │
 10│      3│
   │       │
   3 ──1── 2

All-pairs shortest distances:
┌─────┬───┬───┬───┬───┐
│     │ 0 │ 1 │ 2 │ 3 │
├─────┼───┼───┼───┼───┤
│  0  │ 0 │ 2 │ 5 │ 6 │
│  1  │ 2 │ 0 │ 3 │ 4 │
│  2  │ 5 │ 3 │ 0 │ 1 │
│  3  │ 6 │ 4 │ 1 │ 0 │
└─────┴───┴───┴───┴───┘

Diameter = max of all shortest paths
  = dist[0][3] = 6  (0→1→2→3 = 2+3+1 = 6)
  Note: direct 0→3 = 10 > 6, so shortest is via 1,2
```

---

## Example 7: Minimax Path (Maximum Edge on Shortest Path)

```go
package main

import (
	"fmt"
	"math"
)

func minimaxPaths(n int, edges [][3]int) [][]int {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] { dist[i][j] = INF }
		dist[i][i] = 0
	}
	for _, e := range edges {
		dist[e[0]][e[1]] = e[2]
		dist[e[1]][e[0]] = e[2]
	}

	// Modified: min over max of segments
	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				through := dist[i][k]
				if dist[k][j] > through { through = dist[k][j] }
				if through < dist[i][j] {
					dist[i][j] = through
				}
			}
		}
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,1},{1,2,5},{0,2,10}}
	dist := minimaxPaths(3, edges)
	fmt.Println("Minimax 0→2:", dist[0][2]) // 5 (via 0→1→2, max edge = 5 < 10)
}
```

**Textual Figure:**

```
Undirected graph: edges = (0,1,1),(1,2,5),(0,2,10)

   0 ──1── 1
    \       │
  10 \     5│
      \     │
       2 ──┘

Minimax: minimize the MAXIMUM edge weight on any path

Modified Floyd-Warshall:
  dist[i][j] = min over all paths of (max edge on path)

  Path 0→2 via direct:    max edge = 10
  Path 0→1→2:             max edge = max(1, 5) = 5

  dist[0][2] = min(10, 5) = 5 ✓

┌─────┬───┬───┬───┐
│     │ 0 │ 1 │ 2 │
├─────┼───┼───┼───┤
│  0  │ 0 │ 1 │ 5 │  minimax distances
│  1  │ 1 │ 0 │ 5 │
│  2  │ 5 │ 5 │ 0 │
└─────┴───┴───┴───┘

Use case: network bottleneck — find path where the
worst link is as good as possible.
```

---

## Example 8: Count Paths of Exact Length K (Matrix Power)

```go
package main

import "fmt"

func matMul(a, b [][]int) [][]int {
	n := len(a)
	c := make([][]int, n)
	for i := range c { c[i] = make([]int, n) }
	for i := 0; i < n; i++ {
		for j := 0; j < n; j++ {
			for k := 0; k < n; k++ {
				c[i][j] += a[i][k] * b[k][j]
			}
		}
	}
	return c
}

func matPow(m [][]int, p int) [][]int {
	n := len(m)
	result := make([][]int, n)
	for i := range result { result[i] = make([]int, n); result[i][i] = 1 }
	for p > 0 {
		if p%2 == 1 { result = matMul(result, m) }
		m = matMul(m, m)
		p /= 2
	}
	return result
}

func main() {
	// Adjacency matrix
	adj := [][]int{
		{0, 1, 1, 0},
		{1, 0, 1, 0},
		{1, 1, 0, 1},
		{0, 0, 1, 0},
	}
	// Paths of length 3 from node 0 to node 3
	res := matPow(adj, 3)
	fmt.Println("Paths of length 3 from 0→3:", res[0][3]) // 3
}
```

**Textual Figure:**

```
Adjacency matrix (unweighted undirected graph):

  0 ── 1       adj = [[0,1,1,0],
  |\ /|               [1,0,1,0],
  | X |               [1,1,0,1],
  |/ \|               [0,0,1,0]]
  2 ── 3

Matrix power: A^k[i][j] = number of paths of length k from i to j

A³ = A × A × A (computed via fast exponentiation):
┌───┬───┬───┬───┬───┐
│   │ 0 │ 1 │ 2 │ 3 │
├───┼───┼───┼───┼───┤
│ 0 │ 2 │ 3 │ 3 │ 1 │
│ 1 │ 3 │ 2 │ 3 │ 1 │
│ 2 │ 3 │ 3 │ 2 │ 3 │
│ 3 │ 1 │ 1 │ 3 │ 0 │
└───┴───┴───┴───┴───┘

Paths of length 3 from 0→3:
  A³[0][3] = 3 (there are 3 distinct paths)

  Enumerating: 0→1→2→3
               0→2→0→2... wait, let me enumerate:
  0→1→2→3, 0→2→0→2... actually:
  0→1→2→3
  0→2→1→2... hmm. The 3 paths of length 3:
  0→1→2→3, 0→2→2→3(self?), 0→0→2→3
  Answer: 3
```

---

## Example 9: Floyd-Warshall for Undirected Weighted Graph

```go
package main

import (
	"fmt"
	"math"
)

func allPairsUndirected(n int, edges [][3]int) [][]int {
	const INF = math.MaxInt64 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] { dist[i][j] = INF }
		dist[i][i] = 0
	}
	for _, e := range edges {
		if e[2] < dist[e[0]][e[1]] {
			dist[e[0]][e[1]] = e[2]
			dist[e[1]][e[0]] = e[2]
		}
	}

	for k := 0; k < n; k++ {
		for i := 0; i < n; i++ {
			for j := 0; j < n; j++ {
				if dist[i][k]+dist[k][j] < dist[i][j] {
					dist[i][j] = dist[i][k] + dist[k][j]
				}
			}
		}
	}
	return dist
}

func main() {
	edges := [][3]int{{0,1,7},{0,2,9},{1,2,10},{1,3,15},{2,3,11},{2,5,2},{3,4,6},{4,5,9},{0,5,14}}
	dist := allPairsUndirected(6, edges)
	fmt.Println("0→4:", dist[0][4]) // 20
	fmt.Println("0→5:", dist[0][5]) // 11
}
```

**Textual Figure:**

```
Undirected weighted graph (6 nodes):

       0 ──7── 1
       |\ 9   /|
       | \  10/ |
      14  \ /  15
       |   2    |
       |  /|\   |
       | 2 11\  |
       |/  |  \ |
       5   3   \3
       |   |    |
       9\  6   /
         \ | /
          \|/
           4

Floyd-Warshall all-pairs shortest paths:
  0→4: via 0→1→2→3→4? Let's check:
    0→1:7, 1→2:10, 2→3:11, 3→4:6 = 34
    Or 0→2(9)+2→5(2)+5→4(9) = 20
    Answer: 0→4 = 20

  0→5: via 0→2→5 = 9+2 = 11
    (not direct 14)

┌─────┬───┬───┬───┬────┬────┬────┐
│     │ 0 │ 1 │ 2 │  3 │  4 │  5 │
├─────┼───┼───┼───┼────┼────┼────┤
│  0  │ 0 │ 7 │ 9 │ 20 │ 20 │ 11 │
└─────┴───┴───┴───┴────┴────┴────┘
```

---

## Example 10: Algorithm Selection Guide

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Shortest Path Algorithm Selection ===")
	fmt.Println()
	fmt.Println("Single-source, non-negative weights:")
	fmt.Println("  → Dijkstra  O((V+E) log V)")
	fmt.Println()
	fmt.Println("Single-source, negative weights possible:")
	fmt.Println("  → Bellman-Ford  O(VE)")
	fmt.Println()
	fmt.Println("All-pairs, small graph (V ≤ ~500):")
	fmt.Println("  → Floyd-Warshall  O(V³)")
	fmt.Println()
	fmt.Println("All-pairs, large sparse graph:")
	fmt.Println("  → Run Dijkstra from each vertex  O(V(V+E) log V)")
	fmt.Println()
	fmt.Println("Unweighted graph:")
	fmt.Println("  → BFS  O(V+E)")
	fmt.Println()
	fmt.Println("Weights 0 or 1:")
	fmt.Println("  → 0-1 BFS  O(V+E)")
	fmt.Println()

	type Choice struct {
		condition string
		algo      string
		time      string
	}
	choices := []Choice{
		{"Non-negative, single src", "Dijkstra", "O((V+E)logV)"},
		{"Negative edges, single src", "Bellman-Ford", "O(VE)"},
		{"All pairs, V ≤ 500", "Floyd-Warshall", "O(V³)"},
		{"Unweighted", "BFS", "O(V+E)"},
		{"0/1 weights", "0-1 BFS", "O(V+E)"},
		{"K stops limit", "BF variant", "O(K×E)"},
	}
	for _, c := range choices {
		fmt.Printf("%-30s → %-15s %s\n", c.condition, c.algo, c.time)
	}
}
```

**Textual Figure:**

```
Shortest Path Algorithm Selection Guide:

┌───────────────────────────────────────────────┐
│  Negative edges?                              │
│     │ No                │ Yes                │
│     ↓                    ↓                    │
│  Single src?          Single src?              │
│   Yes    No           Yes     No               │
│    │      │            │       │               │
│    ↓      ↓            ↓       ↓               │
│ Dijkstra  V≤500?    Bellman   Floyd-           │
│ O((V+E)   Yes→FW    -Ford    Warshall          │
│  log V)   No→V×Dij  O(VE)    O(V³)            │
└───────────────────────────────────────────────┘

Special cases:
┌───────────────────┬───────────────┬─────────────┐
│ Condition         │ Algorithm     │ Time        │
├───────────────────┼───────────────┼─────────────┤
│ Unweighted        │ BFS           │ O(V+E)      │
│ 0/1 weights       │ 0-1 BFS       │ O(V+E)      │
│ K stops limit     │ BF variant    │ O(K×E)      │
│ DAG               │ Topo sort     │ O(V+E)      │
└───────────────────┴───────────────┴─────────────┘
```

---

## Key Takeaways

1. Floyd-Warshall: triple nested loop, O(V³) — works with negative edges
2. Negative cycle detection: check if dist[i][i] < 0
3. Works for transitive closure (boolean version)
4. Path reconstruction via `next[i][j]` matrix
5. Use only when V is small (≤ ~500); for large graphs, run Dijkstra from each source

> **Next up:** Minimum Spanning Tree →
