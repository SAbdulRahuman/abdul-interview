# Phase 12: Graphs — Adjacency Matrix

## Overview

An **adjacency matrix** is a 2D array where `matrix[i][j]` indicates an edge from vertex `i` to `j`.

```
     0  1  2  3
  0 [0  1  1  0]
  1 [1  0  0  1]
  2 [1  0  0  1]
  3 [0  1  1  0]
```

| Operation | Time |
|-----------|------|
| Check edge (u,v) | O(1) |
| Iterate neighbors | O(V) |
| Space | O(V²) |

**Use when:** V is small, graph is dense, or O(1) edge lookup is needed.

---

## Example 1: Basic Adjacency Matrix

```go
package main

import "fmt"

func main() {
	n := 4
	matrix := make([][]int, n)
	for i := range matrix {
		matrix[i] = make([]int, n)
	}

	// Undirected edges
	edges := [][2]int{{0,1},{0,2},{1,3},{2,3}}
	for _, e := range edges {
		matrix[e[0]][e[1]] = 1
		matrix[e[1]][e[0]] = 1
	}

	for _, row := range matrix {
		fmt.Println(row)
	}
}
```

---

## Example 2: Weighted Adjacency Matrix

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	n := 4
	INF := math.MaxInt32
	matrix := make([][]int, n)
	for i := range matrix {
		matrix[i] = make([]int, n)
		for j := range matrix[i] {
			if i != j { matrix[i][j] = INF }
		}
	}

	// Weighted edges
	edges := [][3]int{{0,1,4},{0,2,8},{1,2,2},{1,3,5},{2,3,1}}
	for _, e := range edges {
		matrix[e[0]][e[1]] = e[2]
		matrix[e[1]][e[0]] = e[2]
	}

	for i, row := range matrix {
		fmt.Printf("%d: ", i)
		for _, v := range row {
			if v == INF { fmt.Printf(" INF") } else { fmt.Printf(" %3d", v) }
		}
		fmt.Println()
	}
}
```

---

## Example 3: Floyd-Warshall on Matrix

```go
package main

import (
	"fmt"
	"math"
)

func floydWarshall(n int, edges [][3]int) [][]int {
	INF := math.MaxInt32 / 2
	dist := make([][]int, n)
	for i := range dist {
		dist[i] = make([]int, n)
		for j := range dist[i] {
			if i == j { dist[i][j] = 0 } else { dist[i][j] = INF }
		}
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
	edges := [][3]int{{0,1,3},{0,3,7},{1,2,2},{2,3,1},{3,0,2}}
	dist := floydWarshall(4, edges)
	for i, row := range dist {
		fmt.Printf("%d: %v\n", i, row)
	}
}
```

---

## Example 4: Matrix BFS

```go
package main

import "fmt"

func bfsMatrix(matrix [][]int, start int) []int {
	n := len(matrix)
	visited := make([]bool, n)
	order := []int{}
	queue := []int{start}
	visited[start] = true

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		order = append(order, v)

		for u := 0; u < n; u++ {
			if matrix[v][u] != 0 && !visited[u] {
				visited[u] = true
				queue = append(queue, u)
			}
		}
	}
	return order
}

func main() {
	matrix := [][]int{
		{0, 1, 1, 0, 0},
		{1, 0, 0, 1, 0},
		{1, 0, 0, 1, 1},
		{0, 1, 1, 0, 1},
		{0, 0, 1, 1, 0},
	}
	fmt.Println("BFS from 0:", bfsMatrix(matrix, 0)) // [0 1 2 3 4]
}
```

---

## Example 5: Check Bipartite Using Matrix

```go
package main

import "fmt"

func isBipartite(matrix [][]int) bool {
	n := len(matrix)
	color := make([]int, n)
	for i := range color { color[i] = -1 }

	for start := 0; start < n; start++ {
		if color[start] != -1 { continue }
		queue := []int{start}
		color[start] = 0

		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for u := 0; u < n; u++ {
				if matrix[v][u] == 0 { continue }
				if color[u] == -1 {
					color[u] = 1 - color[v]
					queue = append(queue, u)
				} else if color[u] == color[v] {
					return false
				}
			}
		}
	}
	return true
}

func main() {
	// Bipartite: 0-1, 0-3, 2-1, 2-3
	m1 := [][]int{
		{0,1,0,1},
		{1,0,1,0},
		{0,1,0,1},
		{1,0,1,0},
	}

	// Not bipartite: triangle 0-1-2
	m2 := [][]int{
		{0,1,1},
		{1,0,1},
		{1,1,0},
	}

	fmt.Println("Bipartite?", isBipartite(m1)) // true
	fmt.Println("Bipartite?", isBipartite(m2)) // false
}
```

---

## Example 6: Transitive Closure (Reachability)

```go
package main

import "fmt"

func transitiveClosure(n int, adj [][]int) [][]bool {
	reach := make([][]bool, n)
	for i := range reach {
		reach[i] = make([]bool, n)
		reach[i][i] = true
	}
	for u, nb := range adj {
		for _, v := range nb {
			reach[u][v] = true
		}
	}

	// Warshall's algorithm
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
	adj := [][]int{{1}, {2}, {}, {0}} // 0→1→2, 3→0
	reach := transitiveClosure(4, adj)
	for i, row := range reach {
		fmt.Printf("%d can reach: ", i)
		for j, r := range row {
			if r { fmt.Printf("%d ", j) }
		}
		fmt.Println()
	}
	// 0 can reach: 0 1 2
	// 3 can reach: 0 1 2 3
}
```

---

## Example 7: Count Paths of Length K

```go
package main

import "fmt"

// Matrix multiplication counts paths
func matMul(a, b [][]int) [][]int {
	n := len(a)
	c := make([][]int, n)
	for i := range c {
		c[i] = make([]int, n)
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
	result := make([][]int, n) // identity
	for i := range result {
		result[i] = make([]int, n)
		result[i][i] = 1
	}
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
		{1, 0, 1, 1},
		{1, 1, 0, 0},
		{0, 1, 0, 0},
	}

	// A^k[i][j] = number of paths of length k from i to j
	for k := 1; k <= 3; k++ {
		result := matPow(adj, k)
		fmt.Printf("Paths of length %d from 0 to 3: %d\n", k, result[0][3])
	}
}
```

---

## Example 8: Rotate Image (Matrix) — LeetCode 48

```go
package main

import "fmt"

func rotate(matrix [][]int) {
	n := len(matrix)
	// Transpose
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
		}
	}
	// Reverse each row
	for i := 0; i < n; i++ {
		for l, r := 0, n-1; l < r; l, r = l+1, r-1 {
			matrix[i][l], matrix[i][r] = matrix[i][r], matrix[i][l]
		}
	}
}

func main() {
	m := [][]int{{1,2,3},{4,5,6},{7,8,9}}
	fmt.Println("Before:")
	for _, r := range m { fmt.Println(r) }
	rotate(m)
	fmt.Println("After 90° rotation:")
	for _, r := range m { fmt.Println(r) }
	// [7 4 1] [8 5 2] [9 6 3]
}
```

---

## Example 9: Adjacency Matrix from Grid (Walls and Gates LC 286)

```go
package main

import "fmt"

func wallsAndGates(rooms [][]int) {
	if len(rooms) == 0 { return }
	INF := 1<<31 - 1
	rows, cols := len(rooms), len(rooms[0])
	queue := [][2]int{}

	// Start BFS from all gates simultaneously
	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if rooms[r][c] == 0 { // gate
				queue = append(queue, [2]int{r, c})
			}
		}
	}

	dirs := [][2]int{{0,1},{0,-1},{1,0},{-1,0}}
	for len(queue) > 0 {
		pos := queue[0]; queue = queue[1:]
		r, c := pos[0], pos[1]
		for _, d := range dirs {
			nr, nc := r+d[0], c+d[1]
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
			if rooms[nr][nc] != INF { continue }
			rooms[nr][nc] = rooms[r][c] + 1
			queue = append(queue, [2]int{nr, nc})
		}
	}
}

func main() {
	INF := 1<<31 - 1
	rooms := [][]int{
		{INF, -1,  0, INF},
		{INF, INF, INF, -1},
		{INF, -1, INF, -1},
		{0,   -1, INF, INF},
	}
	wallsAndGates(rooms)
	for _, row := range rooms {
		fmt.Println(row)
	}
	// [3 -1 0 1]
	// [2 2 1 -1]
	// [1 -1 2 -1]
	// [0 -1 3 4]
}
```

---

## Example 10: When to Use Matrix vs List

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Adjacency Matrix vs Adjacency List ===")
	fmt.Println()
	fmt.Println("Use MATRIX when:")
	fmt.Println("  • V ≤ 1000 (O(V²) space is acceptable)")
	fmt.Println("  • Need O(1) edge lookup")
	fmt.Println("  • Floyd-Warshall all-pairs shortest path")
	fmt.Println("  • Matrix exponentiation (counting paths)")
	fmt.Println("  • Graph is dense (E ≈ V²)")
	fmt.Println()
	fmt.Println("Use LIST when:")
	fmt.Println("  • V > 1000 (sparse graph)")
	fmt.Println("  • BFS/DFS traversal (only visit neighbors)")
	fmt.Println("  • Most interview problems")
	fmt.Println("  • Dijkstra's, BFS, DFS, Topological sort")
	fmt.Println()
	fmt.Println("| V=1000 | Matrix Space | List Space (E=3000) |")
	fmt.Println("|--------|-------------|---------------------|")
	fmt.Println("| Space  | 1,000,000   | ~7,000              |")
}
```

---

## Key Takeaways

1. Adjacency matrix: `[][]int` — O(V²) space, O(1) edge lookup
2. Best for dense graphs, Floyd-Warshall, matrix exponentiation
3. Wasteful for sparse graphs (most interview problems)
4. Used for problems involving path counting (A^k) or transitive closure
5. Grid problems use the matrix directly — no explicit conversion needed

> **Next up:** Directed Graphs →
