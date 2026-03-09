# Phase 12: Graphs — Graph Representation

## Overview

A **graph** G = (V, E) consists of **vertices** (nodes) and **edges** (connections). Graphs can be represented in multiple ways, each with trade-offs.

```
Common representations:
1. Adjacency List  — slice of slices / map of slices
2. Adjacency Matrix — 2D array
3. Edge List       — list of (u, v, weight) tuples
4. Implicit graph  — grid, state space (no explicit structure)
```

| Representation | Space | Edge Query | Iterate Neighbors | Add Edge |
|---------------|-------|------------|-------------------|----------|
| Adj List | O(V+E) | O(degree) | O(degree) | O(1) |
| Adj Matrix | O(V²) | O(1) | O(V) | O(1) |
| Edge List | O(E) | O(E) | O(E) | O(1) |

---

## Example 1: Adjacency List (Slice of Slices)

```go
package main

import "fmt"

func main() {
	n := 5 // vertices 0..4
	adj := make([][]int, n)

	// Add edges: undirected
	addEdge := func(u, v int) {
		adj[u] = append(adj[u], v)
		adj[v] = append(adj[v], u)
	}

	addEdge(0, 1)
	addEdge(0, 4)
	addEdge(1, 2)
	addEdge(1, 3)
	addEdge(3, 4)

	for v, neighbors := range adj {
		fmt.Printf("Vertex %d → %v\n", v, neighbors)
	}
	// Vertex 0 → [1 4]
	// Vertex 1 → [0 2 3]
	// Vertex 2 → [1]
	// Vertex 3 → [1 4]
	// Vertex 4 → [0 3]
}
```

---

## Example 2: Adjacency List with Map

```go
package main

import "fmt"

func main() {
	graph := map[string][]string{}

	addEdge := func(u, v string) {
		graph[u] = append(graph[u], v)
		graph[v] = append(graph[v], u)
	}

	addEdge("NYC", "LA")
	addEdge("NYC", "Chicago")
	addEdge("LA", "SF")
	addEdge("Chicago", "SF")

	for city, neighbors := range graph {
		fmt.Printf("%s → %v\n", city, neighbors)
	}
}
```

**Why map?** When vertices are strings, non-contiguous, or dynamically added.

---

## Example 3: Adjacency Matrix

```go
package main

import "fmt"

func main() {
	n := 4
	matrix := make([][]int, n)
	for i := range matrix {
		matrix[i] = make([]int, n)
	}

	// Add undirected edges
	addEdge := func(u, v int) {
		matrix[u][v] = 1
		matrix[v][u] = 1
	}

	addEdge(0, 1)
	addEdge(0, 2)
	addEdge(1, 3)
	addEdge(2, 3)

	fmt.Println("Adjacency Matrix:")
	for i, row := range matrix {
		fmt.Printf("  %d: %v\n", i, row)
	}

	// Check edge O(1)
	fmt.Printf("Edge (1,3)? %v\n", matrix[1][3] == 1) // true
	fmt.Printf("Edge (0,3)? %v\n", matrix[0][3] == 1) // false
}
```

---

## Example 4: Edge List

```go
package main

import "fmt"

type Edge struct {
	from, to, weight int
}

func main() {
	edges := []Edge{
		{0, 1, 4},
		{0, 2, 1},
		{1, 3, 2},
		{2, 3, 5},
		{3, 4, 3},
	}

	fmt.Println("Edge List:")
	for _, e := range edges {
		fmt.Printf("  %d → %d (weight %d)\n", e.from, e.to, e.weight)
	}

	// Convert edge list to adjacency list
	n := 5
	adj := make([][]Edge, n)
	for _, e := range edges {
		adj[e.from] = append(adj[e.from], e)
		adj[e.to] = append(adj[e.to], Edge{e.to, e.from, e.weight})
	}

	fmt.Println("\nAs Adjacency List:")
	for v, neighbors := range adj {
		fmt.Printf("  %d → ", v)
		for _, e := range neighbors {
			fmt.Printf("(%d,w=%d) ", e.to, e.weight)
		}
		fmt.Println()
	}
}
```

---

## Example 5: Grid as Implicit Graph

```go
package main

import "fmt"

func main() {
	grid := [][]int{
		{1, 1, 0},
		{1, 0, 1},
		{0, 1, 1},
	}

	rows, cols := len(grid), len(grid[0])
	dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

	// Each cell is a "vertex"; edges connect adjacent 1s
	fmt.Println("Implicit graph from grid:")
	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if grid[r][c] == 0 { continue }
			var neighbors [][2]int
			for _, d := range dirs {
				nr, nc := r+d[0], c+d[1]
				if nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == 1 {
					neighbors = append(neighbors, [2]int{nr, nc})
				}
			}
			fmt.Printf("  (%d,%d) → %v\n", r, c, neighbors)
		}
	}
}
```

---

## Example 6: Weighted Adjacency List

```go
package main

import "fmt"

type Edge struct {
	to, weight int
}

func main() {
	n := 5
	adj := make([][]Edge, n)

	addEdge := func(u, v, w int) {
		adj[u] = append(adj[u], Edge{v, w})
		adj[v] = append(adj[v], Edge{u, w})
	}

	addEdge(0, 1, 10)
	addEdge(0, 2, 3)
	addEdge(1, 2, 1)
	addEdge(1, 3, 2)
	addEdge(2, 3, 8)
	addEdge(3, 4, 7)

	for v, edges := range adj {
		fmt.Printf("Vertex %d:", v)
		for _, e := range edges {
			fmt.Printf(" →(%d, w=%d)", e.to, e.weight)
		}
		fmt.Println()
	}
}
```

---

## Example 7: Build Graph from LeetCode Input

```go
package main

import "fmt"

// LeetCode-style: edges as [][]int
func buildGraph(n int, edges [][]int) [][]int {
	adj := make([][]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	return adj
}

// LeetCode-style directed: prerequisites
func buildDAG(n int, prereqs [][]int) [][]int {
	adj := make([][]int, n)
	for _, p := range prereqs {
		adj[p[1]] = append(adj[p[1]], p[0]) // p[1] → p[0]
	}
	return adj
}

func main() {
	// Undirected
	adj := buildGraph(4, [][]int{{0,1},{1,2},{2,3},{0,3}})
	fmt.Println("Undirected:")
	for v, nb := range adj {
		fmt.Printf("  %d → %v\n", v, nb)
	}

	// DAG (Course Schedule style)
	dag := buildDAG(4, [][]int{{1,0},{2,0},{3,1},{3,2}})
	fmt.Println("DAG:")
	for v, nb := range dag {
		fmt.Printf("  %d → %v\n", v, nb)
	}
}
```

---

## Example 8: Degree Calculation

```go
package main

import "fmt"

func main() {
	n := 5
	adj := make([][]int, n)
	edges := [][2]int{{0,1},{0,2},{1,2},{1,3},{2,4},{3,4}}
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	fmt.Println("Vertex degrees:")
	for v, nb := range adj {
		fmt.Printf("  Vertex %d: degree = %d\n", v, len(nb))
	}

	// For directed graph: in-degree and out-degree
	dag := make([][]int, 4)
	dirEdges := [][2]int{{0,1},{0,2},{1,3},{2,3}}
	inDeg := make([]int, 4)
	for _, e := range dirEdges {
		dag[e[0]] = append(dag[e[0]], e[1])
		inDeg[e[1]]++
	}

	fmt.Println("\nDirected graph degrees:")
	for v := range dag {
		fmt.Printf("  Vertex %d: in-degree=%d, out-degree=%d\n", v, inDeg[v], len(dag[v]))
	}
}
```

---

## Example 9: Convert Between Representations

```go
package main

import "fmt"

func adjListToMatrix(n int, adj [][]int) [][]int {
	matrix := make([][]int, n)
	for i := range matrix { matrix[i] = make([]int, n) }
	for u, neighbors := range adj {
		for _, v := range neighbors {
			matrix[u][v] = 1
		}
	}
	return matrix
}

func matrixToAdjList(matrix [][]int) [][]int {
	n := len(matrix)
	adj := make([][]int, n)
	for u := 0; u < n; u++ {
		for v := 0; v < n; v++ {
			if matrix[u][v] != 0 {
				adj[u] = append(adj[u], v)
			}
		}
	}
	return adj
}

func main() {
	adj := [][]int{{1, 2}, {0, 3}, {0, 3}, {1, 2}}
	m := adjListToMatrix(4, adj)
	fmt.Println("Matrix:")
	for _, row := range m { fmt.Println(" ", row) }

	adj2 := matrixToAdjList(m)
	fmt.Println("Back to list:")
	for v, nb := range adj2 { fmt.Printf("  %d → %v\n", v, nb) }
}
```

---

## Example 10: Representation Trade-offs Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Graph Representation Trade-offs ===")
	fmt.Println()
	fmt.Println("| Feature          | Adj List        | Adj Matrix     | Edge List     |")
	fmt.Println("|-----------------|-----------------|----------------|---------------|")
	fmt.Println("| Space            | O(V+E)          | O(V²)          | O(E)          |")
	fmt.Println("| Edge exists?     | O(degree)       | O(1)           | O(E)          |")
	fmt.Println("| All neighbors    | O(degree)       | O(V)           | O(E)          |")
	fmt.Println("| Add edge         | O(1)            | O(1)           | O(1)          |")
	fmt.Println("| Remove edge      | O(degree)       | O(1)           | O(E)          |")
	fmt.Println("| Memory (sparse)  | Low             | High           | Low           |")
	fmt.Println("| Memory (dense)   | ~same           | Efficient      | Low           |")
	fmt.Println()
	fmt.Println("Best practices:")
	fmt.Println("  • Adjacency list: default for most problems (sparse graphs)")
	fmt.Println("  • Adjacency matrix: dense graphs, need O(1) edge lookup")
	fmt.Println("  • Edge list: Kruskal's MST, when iterating all edges")
	fmt.Println("  • Implicit (grid): BFS/DFS on 2D grids, no explicit graph needed")
}
```

---

## Key Takeaways

1. **Adjacency list** is the default for competitive programming and interviews
2. Use **slice of slices** when vertices are 0..n-1; **map** otherwise
3. **Adjacency matrix** only for dense graphs or O(1) edge queries
4. Grids are implicit graphs — no need to build explicit structure
5. Always clarify: directed vs undirected, weighted vs unweighted

> **Next up:** Adjacency List →
