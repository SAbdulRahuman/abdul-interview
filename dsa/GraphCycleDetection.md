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

**Textual Figure:**
```
Undirected Graph (4 vertices):
  Edges: {0,1}, {1,2}, {2,3}, {3,1}

    ┌───┐     ┌───┐
    │ 0 │─────│ 1 │
    └───┘     └─┬─┘
               │ │
             ┌─┘ └─┐
             │     │
           ┌─┴─┐ ┌─┴─┐
           │ 2 │─│ 3 │  ← edge 2─3 completes cycle
           └───┘ └───┘

DFS Cycle Detection (parent tracking):
  dfs(0, parent=-1):
    └→ dfs(1, parent=0):
        └→ dfs(2, parent=1):
            └→ dfs(3, parent=2):
                neighbor 1: visited AND 1 ≠ parent(2)
                → BACK EDGE to non-parent → CYCLE!

Result: true
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

**Textual Figure:**
```
Test 1: Directed cycle  0→1→2→0

    ┌───┐     ┌───┐     ┌───┐
    │ 0 │────→│ 1 │────→│ 2 │
    └───┘     └───┘     └─┬─┘
      ↑                   │
      └───────────────────┘  back edge!

  3-State Coloring:  W=white, G=gray, B=black
    dfs(0): 0=G → dfs(1): 1=G → dfs(2): 2=G
            neighbor 0 is GRAY → back edge → CYCLE!
    Result: true

Test 2: No cycle  0→1→2
    0 ──→ 1 ──→ 2
    All nodes finish: 2=B, 1=B, 0=B
    No gray→gray edge found
    Result: false
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

**Textual Figure:**
```
Test 1: Edges {0,1}, {1,2}, {2,3}, {3,1}

  Union-Find process:
  ┌──────┬───────────┬───────────────────────────────┐
  │ Edge │ find(u,v) │ Action                        │
  ├──────┼───────────┼───────────────────────────────┤
  │ 0─1 │ 0 ≠ 1     │ Union(0,1) → {0,1}           │
  │ 1─2 │ 0 ≠ 2     │ Union(0,2) → {0,1,2}         │
  │ 2─3 │ 0 ≠ 3     │ Union(0,3) → {0,1,2,3}       │
  │ 3─1 │ 0 == 0    │ Same component → CYCLE!      │
  └──────┴───────────┴───────────────────────────────┘
  Result: true

Test 2: Edges {0,1}, {1,2}, {2,3}  (tree)
  All unions succeed, no same-component edge
  Result: false
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

**Textual Figure:**
```
Test 1: prerequisites=[[1,0]]  (take 0 before 1)
    Directed graph:  0 → 1
    Kahn's: indegree=[0,1] → queue=[0]
      Process 0 → indeg[1]-- → queue=[1]
      Process 1 → done. count=2 == numCourses ✓
    Result: true (can finish)

Test 2: prerequisites=[[1,0],[0,1]]  (mutual dependency)
    ┌───┐     ┌───┐
    │ 0 │────→│ 1 │
    └───┘←────└───┘
           cycle!

    3-State DFS:
      dfs(0): 0=gray → dfs(1): 1=gray
              neighbor 0 is gray → CYCLE!
    Result: false (cannot finish)
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

**Textual Figure:**
```
Kahn's BFS Cycle Detection:

Test 1: adj = [[1],[2],[0]]   (0→1→2→0)
    ┌───┐     ┌───┐     ┌───┐
    │ 0 │────→│ 1 │────→│ 2 │
    └───┘     └───┘     └─┬─┘
      ↑                   │
      └───────────────────┘

    indegree: [1, 1, 1]  → No node with indegree 0!
    queue starts empty → processed=0 ≠ 3
    Result: true (cycle exists)

Test 2: adj = [[1],[2],[]]   (0→1→2)
    0 ──→ 1 ──→ 2

    indegree: [0, 1, 1]  → queue=[0]
    Process 0 → 1 → 2 → processed=3 == n
    Result: false (no cycle)
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

**Textual Figure:**
```
Directed Graph: 0→1, 1→2, 2→0, 3→1

     ┌───┐
     │ 3 │
     └─┬─┘
       │
       ↓
    ┌───┐     ┌───┐     ┌───┐
    │ 0 │────→│ 1 │────→│ 2 │
    └───┘     └───┘     └─┬─┘
      ↑                   │
      └───────────────────┘  cycle: 0→1→2→0

Kahn's algorithm to find non-cycle nodes:
  indegree: [1, 2, 1, 0]  → queue=[3]
  Process 3 → indeg[1]-- → indeg=[1,1,1,0]
  No more indeg-0 nodes → stop
  removed = {3}
  Remaining (in cycle): {0, 1, 2}

Result: [0, 1, 2]
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

**Textual Figure:**
```
Test 1: edges [[1,2],[1,3],[2,3]]

  Union-Find process:
  ┌───────┬───────────┬──────────────────────────┐
  │ Edge  │ find(u,v) │ Action                   │
  ├───────┼───────────┼──────────────────────────┤
  │ [1,2] │ 1 ≠ 2     │ Union → {1,2}           │
  │ [1,3] │ 1 ≠ 3     │ Union → {1,2,3}         │
  │ [2,3] │ 1 == 1    │ Same root → REDUNDANT!  │
  └───────┴───────────┴──────────────────────────┘

     1 ── 2       1 ── 2
     |         →   | / |
     3             3  redundant [2,3]

  Result: [2, 3]
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

**Textual Figure:**
```
Tree conditions: connected + no cycles = exactly n-1 edges

Test 1: n=5, edges=[[0,1],[0,2],[0,3],[1,4]]  (4 edges = n-1 ✓)
    ┌───┐
    │ 0 │
    └─┬─┘
   ┌──┼──┐
   ↓  ↓  ↓
  ┌┴┐┌┴┐┌┴┐
  │1││2││3│
  └┬┘└─┘└─┘
   ↓
  ┌┴┐
  │4│
  └─┘
  BFS from 0: visits all 5 nodes → connected ✓
  Result: true (valid tree)

Test 2: n=5, edges=[[0,1],[1,2],[2,3],[1,3],[1,4]]  (5 edges ≠ n-1)
  len(edges)=5 ≠ 4 → immediately false
  Result: false
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

**Textual Figure:**
```
Directed Graph: 0→1→2→3→1

    ┌───┐     ┌───┐     ┌───┐     ┌───┐
    │ 0 │────→│ 1 │────→│ 2 │────→│ 3 │
    └───┘     └───┘     └───┘     └─┬─┘
               ↑                   │
               └───────────────────┘  back edge!

DFS 3-State Trace (finding cycle path):
  dfs(0): 0=gray → dfs(1): 1=gray → dfs(2): 2=gray → dfs(3): 3=gray
          neighbor 1 is GRAY → cycleStart=1, cycleEnd=3

Reconstructing cycle path via parent array:
  par: [_, 0, 1, 2]  (par[1]=0, par[2]=1, par[3]=2)
  Start from cycleEnd=3, trace back to cycleStart=1:
    3 → par[3]=2 → par[2]=1 = cycleStart → stop
  Cycle: [1, 2, 3, 1]  (reversed)

Result: [1, 2, 3, 1]
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

**Textual Figure:**
```
Input Grid (4×4):
┌───┬───┬───┬───┐
│ a │ a │ a │ a │
├───┼───┼───┼───┤
│ a │ b │ b │ a │
├───┼───┼───┼───┤
│ a │ b │ b │ a │
├───┼───┼───┼───┤
│ a │ a │ a │ a │
└───┴───┴───┴───┘

Cycle Detection in 'a' component (DFS + parent):
  The 'a' cells form a ring around the border:
  (0,0)→(0,1)→(0,2)→(0,3)→(1,3)→(2,3)→(3,3)
    ↑                                       │
  (1,0)←(2,0)←(3,0)←(3,1)←(3,2)←─────────┘

  DFS from (0,0) eventually reaches an already-visited
  cell that isn’t the parent → CYCLE found!

Result: true
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
