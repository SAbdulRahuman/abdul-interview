# Phase 12: Graphs — Cycle Detection in Graphs

## Overview

Detecting cycles is different for directed vs. undirected graphs:

| Graph Type | Method | Key Idea |
|-----------|--------|----------|
| Undirected | DFS + parent tracking | Back edge to non-parent = cycle |
| Directed | DFS + 3-state coloring | Back edge to gray node = cycle |
| Either | Union-Find | Merging same-component nodes = cycle |

---

## Example 1: Undirected Cycle Detection (DFS + Parent)

```go
package main

import "fmt"

func hasCycleUndirected(n int, adj [][]int) bool {
	visited := make([]bool, n)

	var dfs func(v, parent int) bool
	dfs = func(v, parent int) bool {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] {
				if dfs(u, v) { return true }
			} else if u != parent {
				return true // back edge → cycle
			}
		}
		return false
	}

	for i := 0; i < n; i++ {
		if !visited[i] && dfs(i, -1) { return true }
	}
	return false
}

func main() {
	adj := make([][]int, 4)
	for _, e := range [][2]int{{0,1},{1,2},{2,3},{3,1}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("Has cycle:", hasCycleUndirected(4, adj)) // true
}
```

---

## Example 2: Directed Cycle Detection (3-State DFS)

```go
package main

import "fmt"

const (
	WHITE = 0 // unvisited
	GRAY  = 1 // in current path
	BLACK = 2 // fully processed
)

func hasCycleDirected(n int, adj [][]int) bool {
	color := make([]int, n)

	var dfs func(v int) bool
	dfs = func(v int) bool {
		color[v] = GRAY
		for _, u := range adj[v] {
			if color[u] == GRAY { return true } // back edge
			if color[u] == WHITE && dfs(u) { return true }
		}
		color[v] = BLACK
		return false
	}

	for i := 0; i < n; i++ {
		if color[i] == WHITE && dfs(i) { return true }
	}
	return false
}

func main() {
	adj := [][]int{{1}, {2}, {0}} // 0→1→2→0
	fmt.Println("Has cycle:", hasCycleDirected(3, adj)) // true

	adj2 := [][]int{{1}, {2}, {}} // 0→1→2
	fmt.Println("Has cycle:", hasCycleDirected(3, adj2)) // false
}
```

---

## Example 3: Cycle Detection with Union-Find (Undirected)

```go
package main

import "fmt"

func hasCycleUF(n int, edges [][2]int) bool {
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, e := range edges {
		px, py := find(e[0]), find(e[1])
		if px == py { return true } // same component → cycle
		if rank[px] < rank[py] { px, py = py, px }
		parent[py] = px
		if rank[px] == rank[py] { rank[px]++ }
	}
	return false
}

func main() {
	fmt.Println(hasCycleUF(4, [][2]int{{0,1},{1,2},{2,3},{3,1}})) // true
	fmt.Println(hasCycleUF(4, [][2]int{{0,1},{1,2},{2,3}}))        // false
}
```

---

## Example 4: Course Schedule (LeetCode 207)

```go
package main

import "fmt"

func canFinish(numCourses int, prerequisites [][]int) bool {
	adj := make([][]int, numCourses)
	for _, p := range prerequisites {
		adj[p[1]] = append(adj[p[1]], p[0])
	}

	color := make([]int, numCourses)
	var dfs func(v int) bool
	dfs = func(v int) bool {
		color[v] = 1 // gray
		for _, u := range adj[v] {
			if color[u] == 1 { return false }
			if color[u] == 0 && !dfs(u) { return false }
		}
		color[v] = 2 // black
		return true
	}

	for i := 0; i < numCourses; i++ {
		if color[i] == 0 && !dfs(i) { return false }
	}
	return true
}

func main() {
	fmt.Println(canFinish(2, [][]int{{1,0}}))       // true
	fmt.Println(canFinish(2, [][]int{{1,0},{0,1}}))  // false (cycle)
}
```

---

## Example 5: Cycle Detection Using BFS (Kahn's — Directed)

```go
package main

import "fmt"

func hasCycleBFS(n int, adj [][]int) bool {
	indegree := make([]int, n)
	for v := 0; v < n; v++ {
		for _, u := range adj[v] { indegree[u]++ }
	}

	queue := []int{}
	for i := 0; i < n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	processed := 0
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		processed++
		for _, u := range adj[v] {
			indegree[u]--
			if indegree[u] == 0 { queue = append(queue, u) }
		}
	}
	return processed != n // not all processed → cycle
}

func main() {
	fmt.Println(hasCycleBFS(3, [][]int{{1},{2},{0}})) // true
	fmt.Println(hasCycleBFS(3, [][]int{{1},{2},{}}))   // false
}
```

---

## Example 6: Find All Nodes in a Cycle (Directed)

```go
package main

import "fmt"

func nodesInCycle(n int, adj [][]int) []int {
	indegree := make([]int, n)
	for v := 0; v < n; v++ {
		for _, u := range adj[v] { indegree[u]++ }
	}

	queue := []int{}
	for i := 0; i < n; i++ {
		if indegree[i] == 0 { queue = append(queue, i) }
	}

	removed := make([]bool, n)
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		removed[v] = true
		for _, u := range adj[v] {
			indegree[u]--
			if indegree[u] == 0 { queue = append(queue, u) }
		}
	}

	cycleNodes := []int{}
	for i := 0; i < n; i++ {
		if !removed[i] { cycleNodes = append(cycleNodes, i) }
	}
	return cycleNodes
}

func main() {
	// 0→1→2→0 (cycle), 3→1
	adj := [][]int{{1},{2},{0},{1}}
	fmt.Println("Nodes in cycle:", nodesInCycle(4, adj)) // [0 1 2]
}
```

---

## Example 7: Redundant Connection (LeetCode 684)

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

	for _, e := range edges {
		px, py := find(e[0]), find(e[1])
		if px == py { return e }
		if rank[px] < rank[py] { px, py = py, px }
		parent[py] = px
		if rank[px] == rank[py] { rank[px]++ }
	}
	return nil
}

func main() {
	fmt.Println(findRedundantConnection([][]int{{1,2},{1,3},{2,3}}))   // [2 3]
	fmt.Println(findRedundantConnection([][]int{{1,2},{2,3},{3,4},{1,4},{1,5}})) // [1 4]
}
```

---

## Example 8: Graph Valid Tree (LeetCode 261)

```go
package main

import "fmt"

func validTree(n int, edges [][]int) bool {
	// Tree: connected + no cycles ⟹ n-1 edges + connected
	if len(edges) != n-1 { return false }

	adj := make([][]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	visited := make([]bool, n)
	count := 0
	queue := []int{0}
	visited[0] = true
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		count++
		for _, u := range adj[v] {
			if !visited[u] { visited[u] = true; queue = append(queue, u) }
		}
	}
	return count == n
}

func main() {
	fmt.Println(validTree(5, [][]int{{0,1},{0,2},{0,3},{1,4}}))     // true
	fmt.Println(validTree(5, [][]int{{0,1},{1,2},{2,3},{1,3},{1,4}})) // false
}
```

---

## Example 9: Detect Cycle and Return the Cycle Path (Directed)

```go
package main

import "fmt"

func findCycle(n int, adj [][]int) []int {
	color := make([]int, n)
	par := make([]int, n)
	for i := range par { par[i] = -1 }
	cycleStart, cycleEnd := -1, -1

	var dfs func(v int) bool
	dfs = func(v int) bool {
		color[v] = 1 // gray
		for _, u := range adj[v] {
			if color[u] == 1 {
				cycleEnd = v; cycleStart = u
				return true
			}
			if color[u] == 0 {
				par[u] = v
				if dfs(u) { return true }
			}
		}
		color[v] = 2 // black
		return false
	}

	for i := 0; i < n; i++ {
		if color[i] == 0 && dfs(i) { break }
	}
	if cycleStart == -1 { return nil }

	cycle := []int{cycleStart}
	for v := cycleEnd; v != cycleStart; v = par[v] {
		cycle = append(cycle, v)
	}
	cycle = append(cycle, cycleStart)
	// Reverse
	for i, j := 0, len(cycle)-1; i < j; i, j = i+1, j-1 {
		cycle[i], cycle[j] = cycle[j], cycle[i]
	}
	return cycle
}

func main() {
	adj := [][]int{{1}, {2}, {3}, {1}} // 0→1→2→3→1
	fmt.Println("Cycle:", findCycle(4, adj)) // [1 2 3 1]
}
```

---

## Example 10: Detect Cycle in Grid (LeetCode 1559)

```go
package main

import "fmt"

func containsCycle(grid [][]byte) bool {
	rows, cols := len(grid), len(grid[0])
	visited := make([][]bool, rows)
	for i := range visited { visited[i] = make([]bool, cols) }
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	var dfs func(r, c, pr, pc int) bool
	dfs = func(r, c, pr, pc int) bool {
		visited[r][c] = true
		for _, d := range dirs {
			nr, nc := r+d[0], c+d[1]
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
			if grid[nr][nc] != grid[r][c] { continue }
			if nr == pr && nc == pc { continue }
			if visited[nr][nc] { return true }
			if dfs(nr, nc, r, c) { return true }
		}
		return false
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if !visited[r][c] && dfs(r, c, -1, -1) { return true }
		}
	}
	return false
}

func main() {
	grid := [][]byte{
		{'a','a','a','a'},
		{'a','b','b','a'},
		{'a','b','b','a'},
		{'a','a','a','a'},
	}
	fmt.Println(containsCycle(grid)) // true
}
```

---

## Summary Table

| Method | Graph Type | Complexity | Finds Cycle Path? |
|--------|-----------|-----------|-------------------|
| DFS + parent | Undirected | O(V+E) | Yes |
| 3-state DFS | Directed | O(V+E) | Yes |
| Kahn's BFS | Directed | O(V+E) | Nodes, not path |
| Union-Find | Undirected | O(Eα(V)) | Extra edge only |

## Key Takeaways

1. Undirected: back edge to non-parent = cycle
2. Directed: back edge to gray (in-stack) node = cycle; 3-state coloring is essential
3. Kahn's BFS: if processed ≠ n → cycle exists (but doesn't give path)
4. Union-Find: detect cycle incrementally while adding edges
5. Tree = connected + acyclic = exactly V-1 edges

> **Next up:** Topological Sorting →
