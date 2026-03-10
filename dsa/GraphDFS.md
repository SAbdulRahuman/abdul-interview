# Phase 12: Graphs — DFS (Depth-First Search)

## Overview

**DFS** explores a graph by going as deep as possible before backtracking. Uses a **stack** (explicit or via recursion).

```
Start at 0:
0 → 1 → 3 (dead end, backtrack)
       → 4 (dead end, backtrack)
  → 2 → 5

DFS order: 0, 1, 3, 4, 2, 5
```

| Property | Value |
|----------|-------|
| Time | O(V + E) |
| Space | O(V) stack depth |
| Shortest path? | No |
| Good for | Cycle detection, topological sort, connected components |

---

## Example 1: Recursive DFS

```go
package main

import "fmt"

func dfs(adj [][]int, visited []bool, v int, order *[]int) {
	visited[v] = true
	*order = append(*order, v)
	for _, u := range adj[v] {
		if !visited[u] {
			dfs(adj, visited, u, order)
		}
	}
}

func main() {
	adj := [][]int{{1,2},{0,3,4},{0,5},{1},{1},{2}}
	visited := make([]bool, 6)
	var order []int
	dfs(adj, visited, 0, &order)
	fmt.Println("DFS:", order) // [0 1 3 4 2 5]
}
```

**Textual Figure:**
```
Undirected Graph (6 vertices):

        0
       / \
      1    2
     / \    \
    3   4    5

DFS Traversal (recursive, start=0):
┌──────┬───────┬──────────────────────────────┐
│ Step │ Visit │ Action                       │
├──────┼───────┼──────────────────────────────┤
│  1   │   0   │ Recurse to neighbor 1        │
│  2   │   1   │ Recurse to neighbor 3        │
│  3   │   3   │ Dead end → backtrack to 1    │
│  4   │   4   │ Dead end → backtrack to 0    │
│  5   │   2   │ Recurse to neighbor 5        │
│  6   │   5   │ Dead end → done              │
└──────┴───────┴──────────────────────────────┘

Call Stack Trace:
  dfs(0) → dfs(1) → dfs(3) ↩ → dfs(4) ↩ ↩ → dfs(2) → dfs(5) ↩ ↩

Result: [0, 1, 3, 4, 2, 5]
```

---

## Example 2: Iterative DFS with Stack

```go
package main

import "fmt"

func dfsIterative(adj [][]int, start int) []int {
	visited := make([]bool, len(adj))
	stack := []int{start}
	var order []int

	for len(stack) > 0 {
		v := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if visited[v] { continue }
		visited[v] = true
		order = append(order, v)
		// Push neighbors in reverse for same order as recursive
		for i := len(adj[v]) - 1; i >= 0; i-- {
			if !visited[adj[v][i]] {
				stack = append(stack, adj[v][i])
			}
		}
	}
	return order
}

func main() {
	adj := [][]int{{1,2},{0,3,4},{0,5},{1},{1},{2}}
	fmt.Println("DFS:", dfsIterative(adj, 0))
}
```

**Textual Figure:**
```
Same Graph as Example 1:

        0
       / \
      1    2
     / \    \
    3   4    5

Iterative DFS with Explicit Stack (start=0):
┌──────┬───────────────────┬───────┬────────────────────────┐
│ Step │ Stack (top→right) │  Pop  │ Push (reverse order)   │
├──────┼───────────────────┼───────┼────────────────────────┤
│  1   │ [0]               │   0   │ Push 2, 1              │
│  2   │ [2, 1]            │   1   │ Push 4, 3              │
│  3   │ [2, 4, 3]         │   3   │ (no unvisited)         │
│  4   │ [2, 4]            │   4   │ (no unvisited)         │
│  5   │ [2]               │   2   │ Push 5                 │
│  6   │ [5]               │   5   │ (no unvisited)         │
│  7   │ []                │       │ Done                   │
└──────┴───────────────────┴───────┴────────────────────────┘

Result: [0, 1, 3, 4, 2, 5]
```

---

## Example 3: DFS on Grid — Number of Islands (LeetCode 200)

```go
package main

import "fmt"

func numIslands(grid [][]byte) int {
	rows, cols := len(grid), len(grid[0])
	count := 0

	var dfs func(r, c int)
	dfs = func(r, c int) {
		if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] == '0' { return }
		grid[r][c] = '0'
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
		{'1','1','0','0'},
		{'1','0','0','1'},
		{'0','0','1','1'},
	}
	fmt.Println(numIslands(grid)) // 3
}
```

**Textual Figure:**
```
Input Grid (3×4, '1'=land, '0'=water):
┌───┬───┬───┬───┐
│ 1 │ 1 │ 0 │ 0 │  row 0
├───┼───┼───┼───┤
│ 1 │ 0 │ 0 │ 1 │  row 1
├───┼───┼───┼───┤
│ 0 │ 0 │ 1 │ 1 │  row 2
└───┴───┴───┴───┘

DFS Flood Fill identifies islands:
  Island A: (0,0)─(0,1)     ← connected horizontally
               │
            (1,0)            ← connected vertically

  Island B: (1,3)
               │
            (2,3)─(2,2)     ← connected vertically then left

After DFS marks visited cells as '0':
┌───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 0 │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
├───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┘

Result: 2 islands found
```

---

## Example 4: DFS Entry/Exit Times (Timestamps)

```go
package main

import "fmt"

func main() {
	adj := [][]int{{1,2},{3},{3},{}}
	n := len(adj)

	entry := make([]int, n)
	exit := make([]int, n)
	timer := 0

	var dfs func(v int)
	dfs = func(v int) {
		timer++
		entry[v] = timer
		for _, u := range adj[v] {
			if entry[u] == 0 { dfs(u) }
		}
		timer++
		exit[v] = timer
	}

	dfs(0)
	fmt.Println("Entry:", entry) // [1 2 5 3]
	fmt.Println("Exit: ", exit)  // [8 4 6 4] (varies)

	// u is ancestor of v iff entry[u] < entry[v] && exit[u] > exit[v]
	fmt.Println("0 is ancestor of 3?", entry[0] < entry[3] && exit[0] > exit[3]) // true
}
```

**Textual Figure:**
```
Directed Graph (4 vertices):

    ┌───┐     ┌───┐
    │ 0 │────→│ 1 │
    └─┬─┘     └─┬─┘
      │         │
      ↓         ↓
    ┌───┐     ┌───┐
    │ 2 │────→│ 3 │
    └───┘     └───┘

DFS Entry/Exit Timestamps (timer increments on enter & leave):
┌────────┬───────┬──────┬─────────────────────────────┐
│ Vertex │ Entry │ Exit │ Interval                    │
├────────┼───────┼──────┼─────────────────────────────┤
│   0    │   1   │  8   │ [1━━━━━━━━━━━━━━━━━━━━━━8]  │
│   1    │   2   │  5   │   [2━━━━━━━━━━5]             │
│   3    │   3   │  4   │     [3━━━4]                  │
│   2    │   6   │  7   │              [6━━━7]         │
└────────┴───────┴──────┴─────────────────────────────┘

Ancestor check: 0 is ancestor of 3?
  entry[0]=1 < entry[3]=3 ✓  AND  exit[0]=8 > exit[3]=4 ✓
  → Yes, 0 is an ancestor of 3
```

---

## Example 5: Cycle Detection — Directed (3-State)

```go
package main

import "fmt"

func hasCycle(n int, adj [][]int) bool {
	// 0=white, 1=gray (in stack), 2=black (done)
	color := make([]int, n)

	var dfs func(v int) bool
	dfs = func(v int) bool {
		color[v] = 1 // gray
		for _, u := range adj[v] {
			if color[u] == 1 { return true }  // back edge
			if color[u] == 0 && dfs(u) { return true }
		}
		color[v] = 2 // black
		return false
	}

	for i := 0; i < n; i++ {
		if color[i] == 0 && dfs(i) { return true }
	}
	return false
}

func main() {
	fmt.Println(hasCycle(4, [][]int{{1},{2},{3},{}}))   // false (DAG)
	fmt.Println(hasCycle(3, [][]int{{1},{2},{0}}))       // true (cycle)
}
```

**Textual Figure:**
```
Test 1: DAG (no cycle)
    0 → 1 → 2 → 3

  3-State DFS Coloring:
    dfs(0): 0=GRAY → dfs(1): 1=GRAY → dfs(2): 2=GRAY → dfs(3): 3=GRAY
            3=BLACK ← 2=BLACK ← 1=BLACK ← 0=BLACK
    No gray→gray back edge found → false

Test 2: Cycle exists
    ┌───┐     ┌───┐     ┌───┐
    │ 0 │────→│ 1 │────→│ 2 │
    └───┘     └───┘     └─┬─┘
      ↑                   │
      └───────────────────┘  ← back edge!

  3-State DFS Coloring:
    dfs(0): 0=GRAY → dfs(1): 1=GRAY → dfs(2): 2=GRAY
            neighbor 0 is GRAY → BACK EDGE → cycle!
    Result: true
```

---

## Example 6: DFS Topological Sort

```go
package main

import "fmt"

func topologicalSort(n int, adj [][]int) []int {
	visited := make([]bool, n)
	var stack []int

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
		stack = append(stack, v) // post-order
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i) }
	}

	// Reverse stack for topological order
	for l, r := 0, len(stack)-1; l < r; l, r = l+1, r-1 {
		stack[l], stack[r] = stack[r], stack[l]
	}
	return stack
}

func main() {
	adj := [][]int{{1,2},{3},{3},{4},{}}
	fmt.Println(topologicalSort(5, adj)) // [0 2 1 3 4] or [0 1 2 3 4]
}
```

**Textual Figure:**
```
Directed Acyclic Graph (5 vertices):

    ┌───┐     ┌───┐     ┌───┐     ┌───┐
    │ 0 │────→│ 1 │────→│ 3 │────→│ 4 │
    └─┬─┘     └───┘     └───┘     └───┘
      │         ┌───┐       ↑
      └────────→│ 2 │───────┘
                └───┘

DFS Post-order (reverse gives topological order):
  dfs(0) → dfs(1) → dfs(3) → dfs(4): post 4
                              post 3
                    post 1
           dfs(2) → (3 visited): post 2
           post 0

  Post-order stack: [4, 3, 1, 2, 0]
  Reversed:         [0, 2, 1, 3, 4]  ← valid topological order

  Verify: 0→1 ✓, 0→2 ✓, 1→3 ✓, 2→3 ✓, 3→4 ✓
```

---

## Example 7: Connected Components with DFS

```go
package main

import "fmt"

func connectedComponents(n int, adj [][]int) [][]int {
	visited := make([]bool, n)
	var components [][]int

	var dfs func(v int, comp *[]int)
	dfs = func(v int, comp *[]int) {
		visited[v] = true
		*comp = append(*comp, v)
		for _, u := range adj[v] {
			if !visited[u] { dfs(u, comp) }
		}
	}

	for i := 0; i < n; i++ {
		if !visited[i] {
			var comp []int
			dfs(i, &comp)
			components = append(components, comp)
		}
	}
	return components
}

func main() {
	adj := make([][]int, 7)
	for _, e := range [][2]int{{0,1},{1,2},{3,4},{5,6}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	comps := connectedComponents(7, adj)
	for i, c := range comps {
		fmt.Printf("Component %d: %v\n", i, c)
	}
	// Component 0: [0 1 2]
	// Component 1: [3 4]
	// Component 2: [5 6]
}
```

**Textual Figure:**
```
Undirected Graph (7 vertices):

  Component 0        Component 1     Component 2
  ┌───────────┐    ┌───────┐     ┌───────┐
  │ 0 ─ 1 ─ 2 │    │ 3 ─ 4 │     │ 5 ─ 6 │
  └───────────┘    └───────┘     └───────┘

  Edges: 0─1, 1─2, 3─4, 5─6

DFS Component Discovery:
  i=0: not visited → new component 0
       dfs(0) → dfs(1) → dfs(2)  → comp=[0,1,2]
  i=1,2: already visited, skip
  i=3: not visited → new component 1
       dfs(3) → dfs(4)            → comp=[3,4]
  i=4: already visited, skip
  i=5: not visited → new component 2
       dfs(5) → dfs(6)            → comp=[5,6]

Result: 3 components  [[0,1,2], [3,4], [5,6]]
```

---

## Example 8: DFS Path Finding

```go
package main

import "fmt"

func findPath(adj [][]int, src, dst int) []int {
	visited := make([]bool, len(adj))

	var dfs func(v int, path []int) []int
	dfs = func(v int, path []int) []int {
		path = append(path, v)
		if v == dst {
			result := make([]int, len(path))
			copy(result, path)
			return result
		}
		visited[v] = true
		for _, u := range adj[v] {
			if !visited[u] {
				if result := dfs(u, path); result != nil {
					return result
				}
			}
		}
		return nil
	}

	return dfs(src, nil)
}

func main() {
	adj := [][]int{{1,2},{0,3},{0,4},{1,5},{2,5},{3,4}}
	path := findPath(adj, 0, 5)
	fmt.Println("Path:", path) // [0 1 3 5]
}
```

**Textual Figure:**
```
Undirected Graph (6 vertices):
  adj[0]=[1,2]  adj[1]=[0,3]  adj[2]=[0,4]
  adj[3]=[1,5]  adj[4]=[2,5]  adj[5]=[3,4]

      0
     / \
    1   2
    |   |
    3   4
     \ /
      5

DFS Path Finding (src=0, dst=5):
  dfs(0, [0])
   └→ dfs(1, [0,1])
       └→ dfs(3, [0,1,3])
           └→ dfs(5, [0,1,3,5])  ← dst found! return path

  Path: 0 → 1 → 3 → 5

  Result: [0, 1, 3, 5]
```

---

## Example 9: Surrounded Regions (LeetCode 130)

```go
package main

import "fmt"

func solve(board [][]byte) {
	if len(board) == 0 { return }
	rows, cols := len(board), len(board[0])

	var dfs func(r, c int)
	dfs = func(r, c int) {
		if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] != 'O' { return }
		board[r][c] = '#' // mark as safe
		dfs(r+1, c); dfs(r-1, c); dfs(r, c+1); dfs(r, c-1)
	}

	// Mark border-connected O's as safe
	for r := 0; r < rows; r++ {
		dfs(r, 0); dfs(r, cols-1)
	}
	for c := 0; c < cols; c++ {
		dfs(0, c); dfs(rows-1, c)
	}

	// Flip: O→X (surrounded), #→O (safe)
	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if board[r][c] == 'O' { board[r][c] = 'X' }
			if board[r][c] == '#' { board[r][c] = 'O' }
		}
	}
}

func main() {
	board := [][]byte{
		{'X','X','X','X'},
		{'X','O','O','X'},
		{'X','X','O','X'},
		{'X','O','X','X'},
	}
	solve(board)
	for _, row := range board {
		fmt.Println(string(row))
	}
	// XXXX
	// XXXX
	// XXXX
	// XOXX
}
```

**Textual Figure:**
```
Input Board (4×4):
┌───┬───┬───┬───┐
│ X │ X │ X │ X │
├───┼───┼───┼───┤
│ X │ O │ O │ X │  ← surrounded O's
├───┼───┼───┼───┤
│ X │ X │ O │ X │  ← surrounded O
├───┼───┼───┼───┤
│ X │ O │ X │ X │  ← border-connected O (safe)
└───┴───┴───┴───┘

Step 1 ─ DFS from border O's, mark as '#' (safe):
  (3,1) is border O → mark '#'

Step 2 ─ Flip remaining:
  O → X (surrounded)    # → O (safe)

Output Board:
┌───┬───┬───┬───┐
│ X │ X │ X │ X │
├───┼───┼───┼───┤
│ X │ X │ X │ X │  ← O's flipped to X
├───┼───┼───┼───┤
│ X │ X │ X │ X │
├───┼───┼───┼───┤
│ X │ O │ X │ X │  ← border O preserved
└───┴───┴───┴───┘
```

---

## Example 10: BFS vs DFS Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== BFS vs DFS ===")
	fmt.Println()
	fmt.Println("| Feature           | BFS              | DFS              |")
	fmt.Println("|-------------------|------------------|------------------|")
	fmt.Println("| Data structure    | Queue            | Stack/Recursion  |")
	fmt.Println("| Order             | Level by level   | Go deep first    |")
	fmt.Println("| Shortest path?    | Yes (unweighted) | No               |")
	fmt.Println("| Space (tree)      | O(width)         | O(height)        |")
	fmt.Println("| Space (graph)     | O(V)             | O(V)             |")
	fmt.Println("| Cycle detection   | Less common      | Common (3-state) |")
	fmt.Println("| Topological sort  | Kahn's (in-deg)  | Post-order       |")
	fmt.Println("| Connected comp.   | Both work        | Both work        |")
	fmt.Println()
	fmt.Println("Use BFS when:")
	fmt.Println("  • Shortest path in unweighted graph")
	fmt.Println("  • Level-order traversal needed")
	fmt.Println("  • Multi-source problems")
	fmt.Println()
	fmt.Println("Use DFS when:")
	fmt.Println("  • Cycle detection, topological sort")
	fmt.Println("  • Path finding, backtracking")
	fmt.Println("  • Connected components")
	fmt.Println("  • Entry/exit timestamps needed")
}
```

**Textual Figure:**
```
BFS vs DFS — Visual Comparison on Same Graph:

        0
       / \
      1   2
     / \
    3   4

BFS (Queue → level-by-level):       DFS (Stack → depth-first):
  Level 0: [0]                       0 → 1 → 3 (backtrack)
  Level 1: [1, 2]                            → 4 (backtrack)
  Level 2: [3, 4]                       → 2
  Order: 0, 1, 2, 3, 4               Order: 0, 1, 3, 4, 2

┌───────────────────┬──────────────────┬──────────────────┐
│ Feature           │ BFS              │ DFS              │
├───────────────────┼──────────────────┼──────────────────┤
│ Data structure    │ Queue            │ Stack/Recursion  │
│ Shortest path?    │ Yes (unweighted) │ No               │
│ Cycle detection   │ Kahn's (indeg)   │ 3-state coloring │
│ Topological sort  │ Kahn's BFS       │ Post-order rev   │
└───────────────────┴──────────────────┴──────────────────┘
```

---

## Key Takeaways

1. DFS = recursion or explicit stack, goes deep before backtracking
2. 3-state coloring for directed cycle detection (white/gray/black)
3. Parent tracking for undirected cycle detection
4. Post-order DFS → topological sort (reverse the order)
5. DFS is preferred for backtracking, path finding, and component labeling

> **Next up:** Connected Components →
