# Phase 12: Graphs — Adjacency List

## Overview

An **adjacency list** stores, for each vertex, a list of its neighbors. This is the most widely used graph representation for interview problems.

```
Vertex 0 → [1, 4]
Vertex 1 → [0, 2, 3]
Vertex 2 → [1]
Vertex 3 → [1, 4]
Vertex 4 → [0, 3]
```

**Time:** O(V + E) space, O(degree) to check neighbors  
**Go idiom:** `[][]int` or `[][]Edge` for weighted graphs

---

## Example 1: Basic Adjacency List Construction

```go
package main

import "fmt"

func main() {
	n := 6
	adj := make([][]int, n)

	edges := [][2]int{{0,1},{0,2},{1,3},{2,3},{3,4},{4,5}}
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0]) // undirected
	}

	for v := 0; v < n; v++ {
		fmt.Printf("%d → %v\n", v, adj[v])
	}
}
```

---

## Example 2: Directed Adjacency List

```go
package main

import "fmt"

func main() {
	n := 4
	adj := make([][]int, n)

	// Directed edges
	directed := [][2]int{{0,1},{0,2},{1,3},{2,3}}
	for _, e := range directed {
		adj[e[0]] = append(adj[e[0]], e[1]) // only one direction
	}

	for v := 0; v < n; v++ {
		fmt.Printf("%d → %v\n", v, adj[v])
	}
	// 0 → [1 2]
	// 1 → [3]
	// 2 → [3]
	// 3 → []
}
```

---

## Example 3: Weighted Adjacency List

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
		adj[v] = append(adj[v], Edge{u, w})
	}

	add(0, 1, 4)
	add(0, 2, 8)
	add(1, 2, 11)
	add(1, 3, 8)
	add(2, 4, 7)
	add(3, 4, 2)

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

## Example 4: Map-Based Adjacency List

```go
package main

import "fmt"

type Graph struct {
	adj map[string][]string
}

func NewGraph() *Graph {
	return &Graph{adj: map[string][]string{}}
}

func (g *Graph) AddEdge(u, v string) {
	g.adj[u] = append(g.adj[u], v)
	g.adj[v] = append(g.adj[v], u)
}

func (g *Graph) Neighbors(v string) []string {
	return g.adj[v]
}

func (g *Graph) HasEdge(u, v string) bool {
	for _, nb := range g.adj[u] {
		if nb == v { return true }
	}
	return false
}

func main() {
	g := NewGraph()
	g.AddEdge("A", "B")
	g.AddEdge("A", "C")
	g.AddEdge("B", "D")
	g.AddEdge("C", "D")

	fmt.Println("Neighbors of A:", g.Neighbors("A")) // [B C]
	fmt.Println("A-D edge?", g.HasEdge("A", "D"))     // false
	fmt.Println("B-D edge?", g.HasEdge("B", "D"))     // true
}
```

---

## Example 5: Number of Islands (LeetCode 200) — Grid as Adjacency List

```go
package main

import "fmt"

func numIslands(grid [][]byte) int {
	if len(grid) == 0 { return 0 }
	rows, cols := len(grid), len(grid[0])
	count := 0

	var dfs func(r, c int)
	dfs = func(r, c int) {
		if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] == '0' {
			return
		}
		grid[r][c] = '0' // mark visited
		dfs(r+1, c); dfs(r-1, c); dfs(r, c+1); dfs(r, c-1)
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if grid[r][c] == '1' {
				count++
				dfs(r, c)
			}
		}
	}
	return count
}

func main() {
	grid := [][]byte{
		{'1','1','0','0','0'},
		{'1','1','0','0','0'},
		{'0','0','1','0','0'},
		{'0','0','0','1','1'},
	}
	fmt.Println(numIslands(grid)) // 3
}
```

---

## Example 6: Clone Graph (LeetCode 133)

```go
package main

import "fmt"

type Node struct {
	Val       int
	Neighbors []*Node
}

func cloneGraph(node *Node) *Node {
	if node == nil { return nil }
	cloned := map[int]*Node{}

	var dfs func(n *Node) *Node
	dfs = func(n *Node) *Node {
		if c, ok := cloned[n.Val]; ok { return c }
		c := &Node{Val: n.Val}
		cloned[n.Val] = c
		for _, nb := range n.Neighbors {
			c.Neighbors = append(c.Neighbors, dfs(nb))
		}
		return c
	}
	return dfs(node)
}

func main() {
	// Build: 1-2, 1-4, 2-3, 3-4
	n1, n2, n3, n4 := &Node{Val: 1}, &Node{Val: 2}, &Node{Val: 3}, &Node{Val: 4}
	n1.Neighbors = []*Node{n2, n4}
	n2.Neighbors = []*Node{n1, n3}
	n3.Neighbors = []*Node{n2, n4}
	n4.Neighbors = []*Node{n3, n1}

	clone := cloneGraph(n1)
	fmt.Println("Clone val:", clone.Val)
	fmt.Printf("Same object? %v\n", clone == n1) // false

	for _, nb := range clone.Neighbors {
		fmt.Printf("  Neighbor: %d\n", nb.Val)
	}
}
```

---

## Example 7: Build Adjacency List from Prerequisites

```go
package main

import "fmt"

// Course Schedule style: prereqs[i] = [course, prereq]
func buildPrereqGraph(numCourses int, prereqs [][]int) ([][]int, []int) {
	adj := make([][]int, numCourses)
	inDeg := make([]int, numCourses)

	for _, p := range prereqs {
		course, prereq := p[0], p[1]
		adj[prereq] = append(adj[prereq], course)
		inDeg[course]++
	}
	return adj, inDeg
}

func main() {
	adj, inDeg := buildPrereqGraph(4, [][]int{{1,0},{2,0},{3,1},{3,2}})

	for v, nb := range adj {
		fmt.Printf("  %d → %v (in-degree: %d)\n", v, nb, inDeg[v])
	}
	// 0 → [1 2] (in-degree: 0)  ← start here
	// 1 → [3]   (in-degree: 1)
	// 2 → [3]   (in-degree: 1)
	// 3 → []    (in-degree: 2)
}
```

---

## Example 8: Adjacency List with Set (Avoid Duplicates)

```go
package main

import "fmt"

type Graph struct {
	adj map[int]map[int]bool
}

func NewGraph() *Graph {
	return &Graph{adj: map[int]map[int]bool{}}
}

func (g *Graph) AddEdge(u, v int) {
	if g.adj[u] == nil { g.adj[u] = map[int]bool{} }
	if g.adj[v] == nil { g.adj[v] = map[int]bool{} }
	g.adj[u][v] = true
	g.adj[v][u] = true
}

func (g *Graph) HasEdge(u, v int) bool {
	return g.adj[u][v]
}

func (g *Graph) RemoveEdge(u, v int) {
	delete(g.adj[u], v)
	delete(g.adj[v], u)
}

func (g *Graph) Neighbors(v int) []int {
	var result []int
	for nb := range g.adj[v] {
		result = append(result, nb)
	}
	return result
}

func main() {
	g := NewGraph()
	g.AddEdge(0, 1)
	g.AddEdge(0, 1) // duplicate — ignored by set
	g.AddEdge(1, 2)

	fmt.Println("0→1?", g.HasEdge(0, 1)) // true
	fmt.Println("Neighbors of 0:", g.Neighbors(0))

	g.RemoveEdge(0, 1)
	fmt.Println("After remove 0→1?", g.HasEdge(0, 1)) // false
}
```

---

## Example 9: Reverse a Directed Graph

```go
package main

import "fmt"

func reverseGraph(n int, adj [][]int) [][]int {
	rev := make([][]int, n)
	for u, neighbors := range adj {
		for _, v := range neighbors {
			rev[v] = append(rev[v], u)
		}
	}
	return rev
}

func main() {
	adj := [][]int{
		{1, 2}, // 0 → 1, 2
		{3},    // 1 → 3
		{3},    // 2 → 3
		{},     // 3 → (none)
	}

	fmt.Println("Original:")
	for v, nb := range adj { fmt.Printf("  %d → %v\n", v, nb) }

	rev := reverseGraph(4, adj)
	fmt.Println("Reversed:")
	for v, nb := range rev { fmt.Printf("  %d → %v\n", v, nb) }
	// 0 → []
	// 1 → [0]
	// 2 → [0]
	// 3 → [1 2]
}
```

---

## Example 10: Graph Density Check

```go
package main

import "fmt"

func main() {
	// Dense graph: V=4, E=6 (complete graph K4)
	// Sparse graph: V=1000, E=999 (tree)

	cases := []struct {
		name string
		v, e int
	}{
		{"K4 (complete)", 4, 6},
		{"Tree (1000)", 1000, 999},
		{"Social network", 1000000, 5000000},
		{"Dense mesh", 100, 4950},
	}

	for _, c := range cases {
		maxEdges := c.v * (c.v - 1) / 2
		density := float64(c.e) / float64(maxEdges)
		repr := "Adjacency List"
		if density > 0.5 {
			repr = "Adjacency Matrix"
		}
		fmt.Printf("%-20s V=%-8d E=%-10d density=%.4f → Use %s\n",
			c.name, c.v, c.e, density, repr)
	}

	fmt.Println("\nRule of thumb:")
	fmt.Println("  Sparse (E ≈ V):    Adjacency List")
	fmt.Println("  Dense (E ≈ V²):    Adjacency Matrix")
	fmt.Println("  Most interviews:   Adjacency List (sparse)")
}
```

---

## Key Takeaways

1. `[][]int` is the Go standard for adjacency lists — fast and simple
2. Use `[][]Edge` for weighted graphs with `type Edge struct{ To, Weight int }`
3. Map-based adjacency list for non-integer or dynamic vertices
4. Grids don't need explicit graph construction — use direction arrays
5. Always clarify directed vs undirected when building the graph

> **Next up:** Adjacency Matrix →
