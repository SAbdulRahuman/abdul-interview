# Phase 12: Graphs — Undirected Graphs

## Overview

An **undirected graph** has edges without direction: edge (u, v) means both u→v and v→u. Every edge appears in both adjacency lists.

```
0 — 1 — 3
|       |
2 ------┘

adj[0] = [1, 2]
adj[1] = [0, 3]
adj[2] = [0, 3]
adj[3] = [1, 2]
```

Key differences from directed:
- **Degree** = number of edges (no in/out distinction)
- **Cycle detection** is different (need parent tracking)
- **Connected components** instead of strongly connected components

---

## Example 1: Build Undirected Graph

```go
package main

import "fmt"

func main() {
	n := 5
	adj := make([][]int, n)
	edges := [][2]int{{0,1},{0,4},{1,2},{1,3},{3,4}}

	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0]) // both directions
	}

	for v, nb := range adj {
		fmt.Printf("%d → %v (degree %d)\n", v, nb, len(nb))
	}
}
```

---

## Example 2: Graph Valid Tree (LeetCode 261)

A graph is a valid tree if: connected + no cycles + V = E + 1.

```go
package main

import "fmt"

func validTree(n int, edges [][]int) bool {
	if len(edges) != n-1 { return false } // tree has exactly n-1 edges

	adj := make([][]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	visited := make([]bool, n)
	queue := []int{0}
	visited[0] = true
	count := 0

	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		count++
		for _, u := range adj[v] {
			if !visited[u] {
				visited[u] = true
				queue = append(queue, u)
			}
		}
	}
	return count == n // must be connected
}

func main() {
	fmt.Println(validTree(5, [][]int{{0,1},{0,2},{0,3},{1,4}})) // true
	fmt.Println(validTree(5, [][]int{{0,1},{1,2},{2,3},{1,3},{1,4}})) // false
}
```

---

## Example 3: Cycle Detection in Undirected Graph (DFS)

```go
package main

import "fmt"

func hasCycle(n int, adj [][]int) bool {
	visited := make([]bool, n)

	var dfs func(v, parent int) bool
	dfs = func(v, parent int) bool {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] {
				if dfs(u, v) { return true }
			} else if u != parent {
				return true // visited neighbor that isn't parent = cycle
			}
		}
		return false
	}

	for i := 0; i < n; i++ {
		if !visited[i] && dfs(i, -1) {
			return true
		}
	}
	return false
}

func main() {
	// Tree (no cycle)
	adj1 := make([][]int, 4)
	for _, e := range [][2]int{{0,1},{1,2},{2,3}} {
		adj1[e[0]] = append(adj1[e[0]], e[1])
		adj1[e[1]] = append(adj1[e[1]], e[0])
	}
	fmt.Println("Tree has cycle?", hasCycle(4, adj1)) // false

	// Has cycle: 0-1-2-0
	adj2 := make([][]int, 3)
	for _, e := range [][2]int{{0,1},{1,2},{2,0}} {
		adj2[e[0]] = append(adj2[e[0]], e[1])
		adj2[e[1]] = append(adj2[e[1]], e[0])
	}
	fmt.Println("Triangle has cycle?", hasCycle(3, adj2)) // true
}
```

---

## Example 4: Connected Components

```go
package main

import "fmt"

func countComponents(n int, edges [][]int) int {
	adj := make([][]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	visited := make([]bool, n)
	components := 0

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
	}

	for i := 0; i < n; i++ {
		if !visited[i] {
			components++
			dfs(i)
		}
	}
	return components
}

func main() {
	fmt.Println(countComponents(5, [][]int{{0,1},{1,2},{3,4}})) // 2
	fmt.Println(countComponents(5, [][]int{{0,1},{1,2},{2,3},{3,4}})) // 1
}
```

---

## Example 5: Redundant Connection (LeetCode 684)

```go
package main

import "fmt"

func findRedundantConnection(edges [][]int) []int {
	n := len(edges)
	parent := make([]int, n+1)
	rank := make([]int, n+1)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) bool {
		px, py := find(x), find(y)
		if px == py { return false } // cycle
		if rank[px] < rank[py] { px, py = py, px }
		parent[py] = px
		if rank[px] == rank[py] { rank[px]++ }
		return true
	}

	for _, e := range edges {
		if !union(e[0], e[1]) {
			return e // this edge creates a cycle
		}
	}
	return nil
}

func main() {
	fmt.Println(findRedundantConnection([][]int{{1,2},{1,3},{2,3}})) // [2 3]
	fmt.Println(findRedundantConnection([][]int{{1,2},{2,3},{3,4},{1,4},{1,5}})) // [1 4]
}
```

---

## Example 6: Bridges in Undirected Graph

```go
package main

import "fmt"

func findBridges(n int, adj [][]int) [][2]int {
	disc := make([]int, n)
	low := make([]int, n)
	visited := make([]bool, n)
	timer := 0
	var bridges [][2]int

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		visited[v] = true
		timer++
		disc[v] = timer
		low[v] = timer

		for _, u := range adj[v] {
			if u == parent { continue }
			if visited[u] {
				if disc[u] < low[v] { low[v] = disc[u] }
			} else {
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }
				if low[u] > disc[v] {
					bridges = append(bridges, [2]int{v, u})
				}
			}
		}
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i, -1) }
	}
	return bridges
}

func main() {
	adj := make([][]int, 5)
	for _, e := range [][2]int{{0,1},{1,2},{2,0},{1,3},{3,4}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("Bridges:", findBridges(5, adj)) // [[1 3] [3 4]]
}
```

---

## Example 7: Is Graph Bipartite? (LeetCode 785)

```go
package main

import "fmt"

func isBipartite(graph [][]int) bool {
	n := len(graph)
	color := make([]int, n)
	for i := range color { color[i] = -1 }

	for i := 0; i < n; i++ {
		if color[i] != -1 { continue }
		queue := []int{i}
		color[i] = 0
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range graph[v] {
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
	fmt.Println(isBipartite([][]int{{1,3},{0,2},{1,3},{0,2}})) // true
	fmt.Println(isBipartite([][]int{{1,2,3},{0,2},{0,1,3},{0,2}})) // false
}
```

---

## Example 8: Articulation Points

```go
package main

import "fmt"

func findArticulationPoints(n int, adj [][]int) []int {
	disc := make([]int, n)
	low := make([]int, n)
	visited := make([]bool, n)
	isAP := make([]bool, n)
	timer := 0

	var dfs func(v, parent int)
	dfs = func(v, parent int) {
		visited[v] = true
		timer++
		disc[v] = timer
		low[v] = timer
		children := 0

		for _, u := range adj[v] {
			if !visited[u] {
				children++
				dfs(u, v)
				if low[u] < low[v] { low[v] = low[u] }
				// Root with 2+ children
				if parent == -1 && children > 1 { isAP[v] = true }
				// Non-root where child can't reach above
				if parent != -1 && low[u] >= disc[v] { isAP[v] = true }
			} else if u != parent {
				if disc[u] < low[v] { low[v] = disc[u] }
			}
		}
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i, -1) }
	}

	var result []int
	for i, ap := range isAP {
		if ap { result = append(result, i) }
	}
	return result
}

func main() {
	adj := make([][]int, 5)
	for _, e := range [][2]int{{0,1},{1,2},{2,0},{1,3},{3,4}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("Articulation points:", findArticulationPoints(5, adj)) // [1 3]
}
```

---

## Example 9: Pacific Atlantic Water Flow (LeetCode 417)

```go
package main

import "fmt"

func pacificAtlantic(heights [][]int) [][]int {
	if len(heights) == 0 { return nil }
	rows, cols := len(heights), len(heights[0])

	pacific := make([][]bool, rows)
	atlantic := make([][]bool, rows)
	for i := range pacific {
		pacific[i] = make([]bool, cols)
		atlantic[i] = make([]bool, cols)
	}

	dirs := [][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	var dfs func(r, c int, visited [][]bool)
	dfs = func(r, c int, visited [][]bool) {
		visited[r][c] = true
		for _, d := range dirs {
			nr, nc := r+d[0], c+d[1]
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
			if visited[nr][nc] || heights[nr][nc] < heights[r][c] { continue }
			dfs(nr, nc, visited)
		}
	}

	for r := 0; r < rows; r++ {
		dfs(r, 0, pacific)
		dfs(r, cols-1, atlantic)
	}
	for c := 0; c < cols; c++ {
		dfs(0, c, pacific)
		dfs(rows-1, c, atlantic)
	}

	var result [][]int
	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if pacific[r][c] && atlantic[r][c] {
				result = append(result, []int{r, c})
			}
		}
	}
	return result
}

func main() {
	heights := [][]int{
		{1,2,2,3,5},
		{3,2,3,4,4},
		{2,4,5,3,1},
		{6,7,1,4,5},
		{5,1,1,2,4},
	}
	fmt.Println(pacificAtlantic(heights))
}
```

---

## Example 10: Undirected vs Directed Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Undirected vs Directed Graphs ===")
	fmt.Println()
	fmt.Println("| Feature           | Undirected        | Directed          |")
	fmt.Println("|-------------------|-------------------|-------------------|")
	fmt.Println("| Edge storage      | Both adj lists    | One adj list      |")
	fmt.Println("| Degree            | Single degree     | In-deg + Out-deg  |")
	fmt.Println("| Cycle detection   | Parent tracking   | 3-state coloring  |")
	fmt.Println("| Components        | Connected comp.   | SCC (Tarjan/Kos.) |")
	fmt.Println("| Tree check        | E=V-1, connected  | Not applicable    |")
	fmt.Println("| Topological sort  | Not applicable    | Only for DAGs     |")
	fmt.Println("| Bridges/APs       | Tarjan's low-link | Not same concept  |")
	fmt.Println()
	fmt.Println("Common undirected problems:")
	fmt.Println("  • Number of Islands, Valid Tree, Redundant Connection")
	fmt.Println("  • Bipartite check, Bridges, Connected Components")
}
```

---

## Key Takeaways

1. Undirected edges: add to **both** adjacency lists
2. Cycle detection needs **parent tracking** (not 3-state like directed)
3. Tree = connected undirected graph with exactly V-1 edges
4. Bridges and articulation points use Tarjan's low-link values
5. Bipartite check: BFS/DFS with 2-coloring

> **Next up:** Weighted Graphs →
