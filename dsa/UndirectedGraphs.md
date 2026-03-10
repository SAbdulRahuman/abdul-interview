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

**Textual Figure:**

```
Graph Structure (5 nodes, 5 edges):

    0 ──── 1 ──── 2
    │      │
    │      │
    4 ──── 3

Edge processing (both directions):
  (0,1) → adj[0]←1,  adj[1]←0
  (0,4) → adj[0]←4,  adj[4]←0
  (1,2) → adj[1]←2,  adj[2]←1
  (1,3) → adj[1]←3,  adj[3]←1
  (3,4) → adj[3]←4,  adj[4]←3

Adjacency List Result:
┌───────┬─────────────┬────────┐
│ Node  │ Neighbors   │ Degree │
├───────┼─────────────┼────────┤
│   0   │ [1, 4]      │   2    │
│   1   │ [0, 2, 3]   │   3    │
│   2   │ [1]         │   1    │
│   3   │ [1, 4]      │   2    │
│   4   │ [0, 3]      │   2    │
└───────┴─────────────┴────────┘
  Total degree = 10 = 2 × 5 edges ✓
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

**Textual Figure:**

```
Test 1: n=5, edges=4 (n-1=4 ✓)

        0
      / | \
     1  2  3
     |
     4

  Step 1: edges == n-1? 4 == 4 ✓
  Step 2: BFS from 0:
    Queue: [0] → visit 0
    Queue: [1,2,3] → visit 1,2,3
    Queue: [4] → visit 4
    count=5 == n=5 → connected ✓
  Result: true

Test 2: n=5, edges=5 (n-1=4, 5≠4 ✗)

    0 ── 1 ── 2
         │\   │
         │ \ │
         4   3

  Step 1: edges == n-1? 5 ≠ 4 ✗
  → immediately return false
  (cycle exists: 1→2→3→1)
  Result: false
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

**Textual Figure:**

```
Test 1: Tree (0─1─2─3), no cycle

    0 ── 1 ── 2 ── 3

  DFS trace (parent tracking):
    dfs(0, parent=-1)
    ├── visit 0
    ├── neighbor 1: unvisited → dfs(1, parent=0)
    │   ├── visit 1
    │   ├── neighbor 0: visited, but 0==parent → skip
    │   ├── neighbor 2: unvisited → dfs(2, parent=1)
    │   │   ├── visit 2
    │   │   ├── neighbor 1: visited, but 1==parent → skip
    │   │   ├── neighbor 3: unvisited → dfs(3, parent=2)
    │   │   │   └── neighbor 2: visited, but 2==parent → skip
    │   │   │       return false
    │   │   └── return false
    │   └── return false
    └── return false
  Result: false ✓

Test 2: Triangle (0─1─2─0), has cycle

    0 ── 1
    │   /
    │  /
    2─┘

  DFS trace:
    dfs(0, parent=-1)
    ├── visit 0
    ├── neighbor 1 → dfs(1, parent=0)
    │   ├── visit 1
    │   ├── neighbor 0: visited, 0==parent → skip
    │   ├── neighbor 2 → dfs(2, parent=1)
    │   │   ├── visit 2
    │   │   ├── neighbor 0: visited AND 0≠parent(1)
    │   │   └── → CYCLE DETECTED! return true
    │   └── return true
    └── return true
  Result: true ✓
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

**Textual Figure:**

```
Test 1: n=5, edges [{0,1},{1,2},{3,4}]

  Component 1       Component 2
  ┌───────────┐     ┌─────────┐
  │  0 ─ 1 ─ 2│     │  3 ─ 4  │
  └───────────┘     └─────────┘

  DFS from node 0: visits {0, 1, 2} → components = 1
  Node 1: already visited → skip
  Node 2: already visited → skip
  DFS from node 3: visits {3, 4}   → components = 2
  Node 4: already visited → skip
  Result: 2

Test 2: n=5, edges [{0,1},{1,2},{2,3},{3,4}]

  ┌───────────────────┐
  │  0 ─ 1 ─ 2 ─ 3 ─ 4│
  └───────────────────┘

  DFS from node 0: visits {0,1,2,3,4} → components = 1
  Nodes 1-4: all visited → skip
  Result: 1
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

**Textual Figure:**

```
Test 1: edges [{1,2},{1,3},{2,3}]

  Union-Find steps:
  ┌─────────┬────────────────┬──────────────────┐
  │  Edge   │ Action         │ Result           │
  ├─────────┼────────────────┼──────────────────┤
  │ (1, 2)  │ union(1,2) ✓   │ parent: 2→1      │
  │ (1, 3)  │ union(1,3) ✓   │ parent: 3→1      │
  │ (2, 3)  │ find(2)=1,     │ same root →      │
  │         │ find(3)=1      │ CYCLE! return    │
  └─────────┴────────────────┴──────────────────┘

      1 ── 2
       \  /  ← edge (2,3) creates cycle
        3
  Result: [2 3]

Test 2: edges [{1,2},{2,3},{3,4},{1,4},{1,5}]

  ┌─────────┬────────────────┬──────────────────┐
  │  Edge   │ Action         │ Result           │
  ├─────────┼────────────────┼──────────────────┤
  │ (1, 2)  │ union(1,2) ✓   │ connected        │
  │ (2, 3)  │ union(2,3) ✓   │ connected        │
  │ (3, 4)  │ union(3,4) ✓   │ connected        │
  │ (1, 4)  │ find(1)=find(4)│ CYCLE! return    │
  └─────────┴────────────────┴──────────────────┘

      1 ── 2
     /|    |
    5 |    3     edge (1,4) creates cycle
      └── 4
  Result: [1 4]
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

**Textual Figure:**

```
Graph: edges {0,1},{1,2},{2,0},{1,3},{3,4}

    0 ── 1 ── 3 ── 4
     \  /
      2

DFS Tarjan’s low-link trace (start at 0):
┌──────┬──────┬─────┬───────────────────────┐
│ Node │ disc │ low │ Notes                 │
├──────┼──────┼─────┼───────────────────────┤
│  0   │  1   │  1  │                       │
│  1   │  2   │  1  │ back edge 2→0: low←1│
│  2   │  3   │  1  │ back edge to 0: low←1│
│  3   │  4   │  4  │ low[3] > disc[1]      │
│  4   │  5   │  5  │ low[4] > disc[3]      │
└──────┴──────┴─────┴───────────────────────┘

Bridge detection:
  Edge (1,3): low[3]=4 > disc[1]=2 → BRIDGE ✓
  Edge (3,4): low[4]=5 > disc[3]=4 → BRIDGE ✓

    0 ── 1 ┃┃ 3 ┃┃ 4
     \  /   ↑↑  ↑↑
      2    bridge bridge

Result: [[1 3] [3 4]]
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

**Textual Figure:**

```
Test 1: graph = [[1,3],[0,2],[1,3],[0,2]] → true

    0 ── 1        Color assignment:
    |    |        Set A (color 0): {0, 2}
    3 ── 2        Set B (color 1): {1, 3}

  BFS from 0 (color=0):
    0[0] → neighbors 1,3 → color 1
    1[1] → neighbor 2    → color 0
    3[1] → neighbor 2    → already color 0 ≠ 1 ✓
  No conflict → Bipartite ✓

Test 2: graph = [[1,2,3],[0,2],[0,1,3],[0,2]] → false

    0 ── 1        Attempted coloring:
    |\ /|        0[0], 1[1], 2[1], 3[1]
    | X  |
    |/ \|
    3 ── 2      0[0] → neighbor 2: color 1
                  1[1] → neighbor 2: color 1 == color 1
  CONFLICT at edge (1,2): same color!
  → Not bipartite ✗
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

**Textual Figure:**

```
Graph: edges {0,1},{1,2},{2,0},{1,3},{3,4}

    0 ── 1 ── 3 ── 4
     \  /
      2

DFS disc/low values:
┌──────┬──────┬─────┬───────────────────────────┐
│ Node │ disc │ low │ AP check                  │
├──────┼──────┼─────┼───────────────────────────┤
│  0   │  1   │  1  │ root, only 1 child → no  │
│  1   │  2   │  1  │ low[3]=4 >= disc[1]=2 ✓   │
│  2   │  3   │  1  │ back edge to 0            │
│  3   │  4   │  4  │ low[4]=5 >= disc[3]=4 ✓   │
│  4   │  5   │  5  │ leaf node                 │
└──────┴──────┴─────┴───────────────────────────┘

Articulation Point checks:
  Node 1: non-root, low[3]=4 >= disc[1]=2 → AP ✓
           (removing 1 disconnects {3,4} from {0,2})
  Node 3: non-root, low[4]=5 >= disc[3]=4 → AP ✓
           (removing 3 disconnects {4} from rest)

  Before removal    Remove node 1     Remove node 3
  0─1─3─4          0    3─4          0─1    4
   \ /              \                  \ /
    2                2                  2─3?✗

Result: [1 3]
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

**Textual Figure:**

```
Height grid (5×5):

  Pacific →   c0  c1  c2  c3  c4
           ┌────┬────┬────┬────┬────┐
  Pacific r0│  1 │  2 │  2 │  3 │  5 │
           ├────┼────┼────┼────┼────┤
  ↓     r1│  3 │  2 │  3 │  4 │  4 │
           ├────┼────┼────┼────┼────┤
        r2│  2 │  4 │  5 │  3 │  1 │
           ├────┼────┼────┼────┼────┤ Atlantic
        r3│  6 │  7 │  1 │  4 │  5 │     ↓
           ├────┼────┼────┼────┼────┤
        r4│  5 │  1 │  1 │  2 │  4 │
           └────┴────┴────┴────┴────┘
                 ← Atlantic

Reverse DFS from ocean borders (water flows uphill):

  Pacific reachable (P):    Atlantic reachable (A):
  P  P  P  P  P            .  .  .  .  A
  P  .  .  P  P            .  .  .  .  A
  P  P  P  .  .            .  A  A  .  A
  P  P  .  .  .            A  A  .  A  A
  P  .  .  .  .            A  A  A  A  A

  Both P and A (intersection):
  ┌────┬────┬────┬────┬────┐
  │    │    │    │    │ ★  │  (0,4)
  ├────┼────┼────┼────┼────┤
  │    │    │    │ ★  │ ★  │  (1,3),(1,4)
  ├────┼────┼────┼────┼────┤
  │    │ ★  │ ★  │    │    │  (2,1),(2,2)
  ├────┼────┼────┼────┼────┤
  │ ★  │ ★  │    │    │    │  (3,0),(3,1)
  ├────┼────┼────┼────┼────┤
  │ ★  │    │    │    │    │  (4,0)
  └────┴────┴────┴────┴────┘
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

**Textual Figure:**

```
Undirected vs Directed — Visual Comparison:

  Undirected:              Directed:
    0 ── 1                  0 → 1
    |    |                  ↑    ↓
    3 ── 2                  3 ← 2
  edge (0,1) = both ways   edge (0,1) = one way only

┌───────────────────┬───────────────────┬───────────────────┐
│ Feature           │ Undirected        │ Directed          │
├───────────────────┼───────────────────┼───────────────────┤
│ Edge storage      │ Both adj lists    │ One adj list      │
│ Degree            │ Single degree     │ In-deg + Out-deg  │
│ Cycle detection   │ Parent tracking   │ 3-state coloring  │
│ Components        │ Connected comp.   │ SCC (Tarjan/Kos.) │
│ Tree check        │ E=V-1, connected  │ Not applicable    │
│ Bridges/APs       │ Tarjan’s low-link │ Not same concept  │
└───────────────────┴───────────────────┴───────────────────┘
```

---

## Key Takeaways

1. Undirected edges: add to **both** adjacency lists
2. Cycle detection needs **parent tracking** (not 3-state like directed)
3. Tree = connected undirected graph with exactly V-1 edges
4. Bridges and articulation points use Tarjan's low-link values
5. Bipartite check: BFS/DFS with 2-coloring

> **Next up:** Weighted Graphs →
