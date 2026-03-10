# Phase 12: Graphs — Bipartite Graphs

## Overview

A graph is **bipartite** if its vertices can be divided into two disjoint sets such that every edge connects a vertex from one set to the other. Equivalently: a graph is bipartite **if and only if it contains no odd-length cycles**.

**Check:** 2-color the graph using BFS/DFS. If successful → bipartite.

---

## Example 1: Bipartite Check with BFS (LeetCode 785)

```go
package main

import "fmt"

func isBipartite(graph [][]int) bool {
	n := len(graph)
	color := make([]int, n) // 0=uncolored, 1, -1

	for i := 0; i < n; i++ {
		if color[i] != 0 { continue }
		queue := []int{i}
		color[i] = 1
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range graph[v] {
				if color[u] == 0 {
					color[u] = -color[v]
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
Graph 1: Even cycle 0─1─2─3─0 → Bipartite ✓

    Set A          Set B
   ┌─────┐       ┌─────┐
   │  0  │───────│  1  │
   │     │       │     │
   │  2  │───────│  3  │
   └─────┘       └─────┘

BFS 2-Coloring from node 0:
┌──────┬─────────┬────────────────────┬──────────────┐
│ Step │ Dequeue │ Assign Colors      │ Queue        │
├──────┼─────────┼────────────────────┼──────────────┤
│  1   │    0    │ color[0] =  1      │ [1, 3]       │
│  2   │    1    │ color[1] = -1      │ [3, 2]       │
│  3   │    3    │ color[3] = -1      │ [2]          │
│  4   │    2    │ color[2] =  1      │ []           │
└──────┴─────────┴────────────────────┴──────────────┘
All neighbors have opposite colors → true ✓

Graph 2: Has triangle 0─1─2 → NOT Bipartite ✗

       0 (color= 1)
      /│\
     / │ \
    1  2  3     ← all get color = -1
    └──┘        ← 1 & 2 are neighbors with SAME color
                   → conflict! return false ✗
```

---

## Example 2: Bipartite Check with DFS

```go
package main

import "fmt"

func isBipartiteDFS(graph [][]int) bool {
	n := len(graph)
	color := make([]int, n)

	var dfs func(v, c int) bool
	dfs = func(v, c int) bool {
		color[v] = c
		for _, u := range graph[v] {
			if color[u] == 0 {
				if !dfs(u, -c) { return false }
			} else if color[u] == c {
				return false
			}
		}
		return true
	}

	for i := 0; i < n; i++ {
		if color[i] == 0 && !dfs(i, 1) { return false }
	}
	return true
}

func main() {
	// Bipartite: 0-1-2-3-0 (even cycle)
	graph := [][]int{{1,3},{0,2},{1,3},{0,2}}
	fmt.Println("Bipartite:", isBipartiteDFS(graph)) // true

	// Not bipartite: triangle 0-1-2-0
	graph2 := [][]int{{1,2},{0,2},{0,1}}
	fmt.Println("Bipartite:", isBipartiteDFS(graph2)) // false
}
```

**Textual Figure:**

```
DFS 2-Coloring on graph 0─1─2─3─0 (even cycle):

  dfs(0, +1)
   │
   ├─→ dfs(1, -1)
   │    │
   │    └─→ dfs(2, +1)
   │         │
   │         └─→ neighbor 3: color[3]=0 → dfs(3, -1)
   │              │
   │              └─→ neighbor 0: color[0]=+1 ≠ -1 ✓
   │
   └─→ neighbor 3: color[3]=-1 ≠ +1 ✓

  Result: color = [+1, -1, +1, -1] → Bipartite ✓

DFS on triangle 0─1─2─0:

  dfs(0, +1)
   │
   ├─→ dfs(1, -1)
   │    │
   │    └─→ dfs(2, +1)
   │         │
   │         └─→ neighbor 0: color[0]=+1 ≠ +1 ✓
   │
   └─→ neighbor 2: color[2]=+1 == color[0]=+1
        → CONFLICT! return false ✗
```

---

## Example 3: Get the Two Sets

```go
package main

import "fmt"

func bipartiteSets(n int, adj [][]int) ([]int, []int, bool) {
	color := make([]int, n)
	queue := []int{}

	for i := 0; i < n; i++ {
		if color[i] != 0 { continue }
		color[i] = 1
		queue = append(queue, i)
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range adj[v] {
				if color[u] == 0 {
					color[u] = -color[v]
					queue = append(queue, u)
				} else if color[u] == color[v] {
					return nil, nil, false
				}
			}
		}
	}

	setA, setB := []int{}, []int{}
	for i, c := range color {
		if c == 1 { setA = append(setA, i) } else { setB = append(setB, i) }
	}
	return setA, setB, true
}

func main() {
	adj := [][]int{{1,3},{0,2},{1,3},{0,2}}
	a, b, ok := bipartiteSets(4, adj)
	fmt.Println("Bipartite:", ok) // true
	fmt.Println("Set A:", a)      // [0 2]
	fmt.Println("Set B:", b)      // [1 3]
}
```

**Textual Figure:**

```
Graph: 0─1─2─3─0  (square / even cycle)

   0 ─── 1
   │     │
   3 ─── 2

BFS coloring from node 0:
  color[0]= 1 → color[1]=-1 → color[2]= 1 → color[3]=-1

Partition into two sets:
  ┌───────────┐     ┌───────────┐
  │  Set A    │     │  Set B    │
  │  color=1  │     │  color=-1 │
  │           │     │           │
  │  0,  2    │─────│  1,  3    │
  └───────────┘     └───────────┘

  Every edge crosses from Set A to Set B ✓
  Set A = [0, 2]   Set B = [1, 3]
```

---

## Example 4: Possible Bipartition (LeetCode 886)

```go
package main

import "fmt"

func possibleBipartition(n int, dislikes [][]int) bool {
	adj := make([][]int, n+1)
	for _, d := range dislikes {
		adj[d[0]] = append(adj[d[0]], d[1])
		adj[d[1]] = append(adj[d[1]], d[0])
	}

	color := make([]int, n+1)
	for i := 1; i <= n; i++ {
		if color[i] != 0 { continue }
		queue := []int{i}
		color[i] = 1
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range adj[v] {
				if color[u] == 0 {
					color[u] = -color[v]
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
	fmt.Println(possibleBipartition(4, [][]int{{1,2},{1,3},{2,4}})) // true
	fmt.Println(possibleBipartition(3, [][]int{{1,2},{1,3},{2,3}})) // false
}
```

**Textual Figure:**

```
Case 1: n=4, dislikes=[(1,2),(1,3),(2,4)]

  Dislike graph (1-indexed):
    1 ─── 2
    │     │
    3     4

  BFS coloring:
    color[1]= 1 → color[2]=-1 → color[3]=-1 → color[4]= 1

   Group A: {1, 4}     Group B: {2, 3}
    1─2 ✓ (diff groups)   1─3 ✓   2─4 ✓
    → Possible bipartition = true ✓

Case 2: n=3, dislikes=[(1,2),(1,3),(2,3)]

  Dislike graph:
    1 ─── 2
     \   /
      \ /
       3

  BFS: color[1]=1 → color[2]=-1, color[3]=-1
       but 2─3 are neighbors with SAME color → conflict!
    → Possible bipartition = false ✗
```

---

## Example 5: Bipartite Check with Union-Find

```go
package main

import "fmt"

func isBipartiteUF(graph [][]int) bool {
	n := len(graph)
	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		parent[find(x)] = find(y)
	}

	for v := 0; v < n; v++ {
		for _, u := range graph[v] {
			if find(v) == find(u) { return false }
			union(graph[v][0], u) // all neighbors in same group
		}
	}
	return true
}

func main() {
	fmt.Println(isBipartiteUF([][]int{{1,3},{0,2},{1,3},{0,2}}))       // true
	fmt.Println(isBipartiteUF([][]int{{1,2,3},{0,2},{0,1,3},{0,2}}))   // false
}
```

**Textual Figure:**

```
Union-Find Bipartite Check:

For each vertex v, union ALL its neighbors together.
Then check: if v is in the same component as any neighbor → NOT bipartite.

Graph 1: 0─1─2─3─0  (adj = {{1,3},{0,2},{1,3},{0,2}})

  v=0: neighbors={1,3} → union(1,3)    parent: 0 1 2 1
  v=1: neighbors={0,2} → union(0,2)    parent: 0 1 0 1
  v=2: neighbors={1,3} → union(1,3)    (already same set)
  v=3: neighbors={0,2} → union(0,2)    (already same set)

  Check: find(0)=0, find(1)=1 → diff ✓
         find(1)=1, find(2)=0 → diff ✓
         find(2)=0, find(3)=1 → diff ✓
  → Bipartite = true ✓

Graph 2: Triangle 0─1, 0─2, 1─2
  v=0: neighbors={1,2,3} → union(1,2), union(2,3)
  Check: find(0) vs find(1) → 0 == union'd group?
         0 is in same component as neighbor → false ✗
```

---

## Example 6: Odd Cycle Detection

```go
package main

import "fmt"

func findOddCycle(n int, adj [][]int) []int {
	color := make([]int, n)
	parent := make([]int, n)
	for i := range parent { parent[i] = -1 }

	for start := 0; start < n; start++ {
		if color[start] != 0 { continue }
		color[start] = 1
		queue := []int{start}
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range adj[v] {
				if color[u] == 0 {
					color[u] = -color[v]
					parent[u] = v
					queue = append(queue, u)
				} else if color[u] == color[v] {
					// Found odd cycle — reconstruct
					pathV, pathU := []int{}, []int{}
					a, b := v, u
					for a != b {
						pathV = append(pathV, a); a = parent[a]
						pathU = append(pathU, b); b = parent[b]
					}
					cycle := append(pathV, a)
					for i := len(pathU) - 1; i >= 0; i-- {
						cycle = append(cycle, pathU[i])
					}
					return cycle
				}
			}
		}
	}
	return nil // bipartite
}

func main() {
	adj := [][]int{{1,2},{0,2},{0,1}} // triangle
	cycle := findOddCycle(3, adj)
	fmt.Println("Odd cycle:", cycle)
}
```

**Textual Figure:**

```
Triangle graph: adj = {{1,2},{0,2},{0,1}}

       0
      / \
     1───2

BFS from node 0:
  color[0] =  1   queue = [0]
  pop 0: color[1] = -1, color[2] = -1   queue = [1, 2]
  pop 1: neighbor 2 has color[2] = -1 == color[1] = -1
         → CONFLICT at edge 1─2 (same color)

Odd Cycle Reconstruction:
  v = 1, u = 2  (both color = -1)
  Trace parents back to common ancestor:
    pathV: 1 → parent[1]=0
    pathU: 2 → parent[2]=0
    Meet at node 0

  Cycle: [1, 0, 2] → odd length (3) ✓

       0 ←── start
      ↙ ↘
     1 → 2    (odd cycle of length 3)
```

---

## Example 7: Maximum Bipartite Matching (Hungarian/Hopcroft-Karp Simplified)

```go
package main

import "fmt"

func maxMatching(n, m int, adj [][]int) int {
	matchL := make([]int, n) // left match
	matchR := make([]int, m) // right match
	for i := range matchL { matchL[i] = -1 }
	for i := range matchR { matchR[i] = -1 }

	var dfs func(u int, visited []bool) bool
	dfs = func(u int, visited []bool) bool {
		for _, v := range adj[u] {
			if visited[v] { continue }
			visited[v] = true
			if matchR[v] == -1 || dfs(matchR[v], visited) {
				matchL[u] = v
				matchR[v] = u
				return true
			}
		}
		return false
	}

	result := 0
	for u := 0; u < n; u++ {
		visited := make([]bool, m)
		if dfs(u, visited) { result++ }
	}
	return result
}

func main() {
	// Left: {0,1,2}, Right: {0,1,2}
	// Edges: L0-R0, L0-R1, L1-R0, L2-R2
	adj := [][]int{{0, 1}, {0}, {2}}
	fmt.Println("Max matching:", maxMatching(3, 3, adj)) // 3
}
```

**Textual Figure:**

```
Bipartite Matching: Left = {L0, L1, L2}, Right = {R0, R1, R2}
Edges: L0→{R0,R1}, L1→{R0}, L2→{R2}

  Left        Right
  ┌────┐      ┌────┐
  │ L0 │─────→│ R0 │
  │    │──┐   │    │
  └────┘  │   └────┘
          │
  ┌────┐  │   ┌────┐
  │ L1 │──┼──→│ R1 │
  └────┘  │   └────┘
          │
  ┌────┐  │   ┌────┐
  │ L2 │  └──→│ R2 │
  └────┘      └────┘

Augmenting path DFS:
  Step 1: L0 → R0 (free)        match: L0─R0
  Step 2: L1 → R0 (taken by L0)
          → augment: L0 → R1    match: L0─R1, L1─R0
  Step 3: L2 → R2 (free)        match: L0─R1, L1─R0, L2─R2

  Final Matching (size = 3):
    L0 ═══ R1
    L1 ═══ R0
    L2 ═══ R2
```

---

## Example 8: Graph Coloring (2-Colorable = Bipartite)

```go
package main

import "fmt"

func twoColor(n int, adj [][]int) ([]int, bool) {
	colors := make([]int, n)

	var dfs func(v, c int) bool
	dfs = func(v, c int) bool {
		colors[v] = c
		for _, u := range adj[v] {
			if colors[u] == 0 {
				if !dfs(u, 3-c) { return false } // alternate 1,2
			} else if colors[u] == c {
				return false
			}
		}
		return true
	}

	for i := 0; i < n; i++ {
		if colors[i] == 0 {
			if !dfs(i, 1) { return nil, false }
		}
	}
	return colors, true
}

func main() {
	adj := [][]int{{1,3},{0,2},{1,3},{0,2}}
	colors, ok := twoColor(4, adj)
	fmt.Println("2-colorable:", ok) // true
	fmt.Println("Colors:", colors)  // [1 2 1 2]
}
```

**Textual Figure:**

```
Graph: 0─1─2─3─0  (square / even cycle)

  2-Coloring via DFS (colors 1 and 2):

    dfs(0, color=1)
     ├→ dfs(1, color=2)
     │   └→ dfs(2, color=1)
     │       └→ neighbor 3: color=2
     └→ neighbor 3: already color=2 ≠ 1 ✓

  Result:
    ┌─────────┬─────────┐
    │ Color 1 │ Color 2 │
    ├─────────┼─────────┤
    │  0, 2   │  1, 3   │
    └─────────┴─────────┘

    0(1) ─── 1(2)
    │         │
    3(2) ─── 2(1)

  No two adjacent nodes share the same color → 2-colorable ✓
  colors = [1, 2, 1, 2]
```

---

## Example 9: Bipartite in Grid (Chessboard Pattern)

```go
package main

import "fmt"

func isGridBipartite(grid [][]int) bool {
	rows, cols := len(grid), len(grid[0])
	color := make([][]int, rows)
	for i := range color { color[i] = make([]int, cols) }
	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	for sr := 0; sr < rows; sr++ {
		for sc := 0; sc < cols; sc++ {
			if grid[sr][sc] == 0 || color[sr][sc] != 0 { continue }
			color[sr][sc] = 1
			queue := [][2]int{{sr, sc}}
			for len(queue) > 0 {
				cell := queue[0]; queue = queue[1:]
				r, c := cell[0], cell[1]
				for _, d := range dirs {
					nr, nc := r+d[0], c+d[1]
					if nr < 0 || nr >= rows || nc < 0 || nc >= cols || grid[nr][nc] == 0 { continue }
					if color[nr][nc] == 0 {
						color[nr][nc] = -color[r][c]
						queue = append(queue, [2]int{nr, nc})
					} else if color[nr][nc] == color[r][c] {
						return false
					}
				}
			}
		}
	}
	return true
}

func main() {
	grid := [][]int{{1,1,1},{1,1,1},{1,1,1}}
	fmt.Println("Grid bipartite:", isGridBipartite(grid)) // true (chessboard)
}
```

**Textual Figure:**

```
3×3 Grid (all cells = 1) with chessboard coloring:

  ┌─────┬─────┬─────┐
  │ +1  │ -1  │ +1  │   row 0
  ├─────┼─────┼─────┤
  │ -1  │ +1  │ -1  │   row 1
  ├─────┼─────┼─────┤
  │ +1  │ -1  │ +1  │   row 2
  └─────┴─────┴─────┘

  Every horizontal neighbor: +1 ↔ -1  ✓
  Every vertical   neighbor: +1 ↔ -1  ✓

  BFS from (0,0) assigns alternating colors:
    (0,0)=+1 → (0,1)=-1 → (0,2)=+1
                  ↓
              (1,1)=+1 → (1,0)=-1, (1,2)=-1
                           ↓
                       (2,0)=+1 → (2,1)=-1 → (2,2)=+1

  No adjacent cells share the same color → Grid is bipartite ✓
```

---

## Example 10: Bipartite Properties Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bipartite Graph Properties ===")
	fmt.Println()

	props := []struct{ prop, desc string }{
		{"Definition", "Vertices split into 2 sets, edges only between sets"},
		{"Equivalent", "2-colorable / no odd-length cycles"},
		{"Check", "BFS/DFS coloring: O(V+E)"},
		{"Trees", "Always bipartite"},
		{"Even cycles", "Bipartite"},
		{"Odd cycles", "NOT bipartite"},
		{"Complete bipartite", "K(m,n): all edges between 2 sets"},
		{"Applications", "Matching, scheduling, conflict resolution"},
		{"Max matching", "König's theorem: max matching = min vertex cover"},
		{"Chromatic number", "χ(G) = 2 for bipartite (with ≥1 edge)"},
	}

	for i, p := range props {
		fmt.Printf("%2d. %-20s %s\n", i+1, p.prop, p.desc)
	}
}
```

**Textual Figure:**

```
Bipartite Graph Properties at a Glance:

┌──────────────────────────────────────────────────────────┐
│                  BIPARTITE GRAPH                         │
│                                                          │
│  Set A ●───────● Set B     Every edge crosses the cut    │
│        ●───────●                                         │
│        ●───────●           No edges within a set         │
└──────────────────────────────────────────────────────────┘

┌────────────────────┬──────────────────────────────┐
│ Property           │ Detail                       │
├────────────────────┼──────────────────────────────┤
│ Odd cycles         │ None (iff bipartite)          │
│ Chromatic number   │ χ(G) = 2                     │
│ Check algorithm    │ BFS/DFS coloring O(V+E)      │
│ Trees              │ Always bipartite             │
│ Max matching       │ = min vertex cover (König)   │
│ Applications       │ Matching, scheduling         │
└────────────────────┴──────────────────────────────┘

  Bipartite:          NOT Bipartite:
  0 ─ 1               0 ─ 1
  │   │                 \ │
  3 ─ 2                  2
  (even cycle)        (odd cycle / triangle)
```

---

## Key Takeaways

1. Bipartite ↔ 2-colorable ↔ no odd-length cycles
2. Check via BFS/DFS coloring in O(V+E)
3. All trees are bipartite
4. Union-Find alternative: union all neighbors of each vertex
5. Applications: matching problems, scheduling with conflicts

> **Next up:** Strongly Connected Components →
