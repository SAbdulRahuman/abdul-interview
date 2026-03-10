# Phase 12: Graphs — Graph Traversal Patterns

## Overview

Graph traversal = systematically visiting all vertices/edges. The two fundamental strategies:

| Strategy | Data Structure | Order | Use Case |
|----------|---------------|-------|----------|
| BFS | Queue | Level by level | Shortest path (unweighted), level-order |
| DFS | Stack / Recursion | Depth first | Connectivity, cycle detection, topological sort |

---

## Example 1: Generic BFS Template

```go
package main

import "fmt"

func bfs(adj [][]int, start int) []int {
	visited := make([]bool, len(adj))
	order := []int{}
	queue := []int{start}
	visited[start] = true

	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		order = append(order, v)
		for _, u := range adj[v] {
			if !visited[u] {
				visited[u] = true
				queue = append(queue, u)
			}
		}
	}
	return order
}

func main() {
	adj := make([][]int, 6)
	for _, e := range [][2]int{{0,1},{0,2},{1,3},{2,4},{3,5}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("BFS:", bfs(adj, 0)) // [0 1 2 3 4 5]
}
```

**Textual Figure:**
```
    Graph (Undirected, 6 vertices):
    Edges: 0-1, 0-2, 1-3, 2-4, 3-5

        ┌───┐         ┌───┐         ┌───┐
        │ 0 │─────────│ 1 │─────────│ 3 │
        └─┬─┘         └───┘         └─┬─┘
          │                              │
        ┌─┴─┐         ┌───┐         ┌─┴─┐
        │ 2 │─────────│ 4 │         │ 5 │
        └───┘         └───┘         └───┘

    BFS from 0 (Queue-based, level by level):
    ┌──────┬─────────┬───────────────┐
    │ Step │ Dequeue │ Enqueue       │
    ├──────┼─────────┼───────────────┤
    │  1   │   0     │ [1, 2]        │
    │  2   │   1     │ [2, 3]        │
    │  3   │   2     │ [3, 4]        │
    │  4   │   3     │ [4, 5]        │
    │  5   │   4     │ [5]           │
    │  6   │   5     │ []            │
    └──────┴─────────┴───────────────┘
    Result: [0, 1, 2, 3, 4, 5]
```

---

## Example 2: Generic DFS Template (Iterative)

```go
package main

import "fmt"

func dfsIterative(adj [][]int, start int) []int {
	visited := make([]bool, len(adj))
	order := []int{}
	stack := []int{start}

	for len(stack) > 0 {
		v := stack[len(stack)-1]; stack = stack[:len(stack)-1]
		if visited[v] { continue }
		visited[v] = true
		order = append(order, v)
		// Push neighbors in reverse for consistent ordering
		for i := len(adj[v]) - 1; i >= 0; i-- {
			if !visited[adj[v][i]] { stack = append(stack, adj[v][i]) }
		}
	}
	return order
}

func main() {
	adj := make([][]int, 6)
	for _, e := range [][2]int{{0,1},{0,2},{1,3},{2,4},{3,5}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("DFS:", dfsIterative(adj, 0)) // [0 1 3 5 2 4]
}
```

**Textual Figure:**
```
    Same Graph (Undirected, 6 vertices):

        ┌───┐         ┌───┐         ┌───┐
        │ 0 │─────────│ 1 │─────────│ 3 │
        └─┬─┘         └───┘         └─┬─┘
          │                              │
        ┌─┴─┐         ┌───┐         ┌─┴─┐
        │ 2 │─────────│ 4 │         │ 5 │
        └───┘         └───┘         └───┘

    DFS from 0 (Stack-based, depth first):
    ┌──────┬─────────┬───────────────┐
    │ Step │ Pop     │ Stack         │
    ├──────┼─────────┼───────────────┤
    │  1   │   0     │ [1, 2]        │
    │  2   │   1     │ [2, 3, 0*]    │
    │  3   │   3     │ [2, 5, 1*]    │
    │  4   │   5     │ [2, 3*]       │
    │  5   │   2     │ [4, 0*]       │
    │  6   │   4     │ []            │
    └──────┴─────────┴───────────────┘
    * = already visited, skipped
    Result: [0, 1, 3, 5, 2, 4]
```

---

## Example 3: DFS with Pre/Post Order Timestamps

```go
package main

import "fmt"

func dfsTimestamps(adj [][]int) ([]int, []int) {
	n := len(adj)
	pre, post := make([]int, n), make([]int, n)
	visited := make([]bool, n)
	timer := 0

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		pre[v] = timer; timer++
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
		post[v] = timer; timer++
	}

	for i := 0; i < n; i++ {
		if !visited[i] { dfs(i) }
	}
	return pre, post
}

func main() {
	adj := make([][]int, 5)
	for _, e := range [][2]int{{0,1},{0,2},{1,3},{1,4}} {
		adj[e[0]] = append(adj[e[0]], e[1])
	}
	pre, post := dfsTimestamps(adj)
	fmt.Println("Pre: ", pre)  // Discovery times
	fmt.Println("Post:", post) // Finish times
}
```

**Textual Figure:**
```
    Directed Graph (DAG, 5 vertices):
    Edges: 0→1, 0→2, 1→3, 1→4

            ┌───┐
            │ 0 │
            └─┬─┘
           ╱   ╲
          ↓     ↓
        ┌───┐ ┌───┐
        │ 1 │ │ 2 │
        └─┬─┘ └───┘
       ╱   ╲
      ↓     ↓
    ┌───┐ ┌───┐
    │ 3 │ │ 4 │
    └───┘ └───┘

    DFS Timestamps (pre=discover, post=finish):
    ┌────────┬─────┬──────┐
    │ Vertex │ Pre │ Post │
    ├────────┼─────┼──────┤
    │   0    │  0  │   9  │
    │   1    │  1  │   6  │
    │   2    │  7  │   8  │
    │   3    │  2  │   3  │
    │   4    │  4  │   5  │
    └────────┴─────┴──────┘
    DFS order: 0→1→3(back)→4(back)→2(back)
```

---

## Example 4: BFS Level-by-Level

```go
package main

import "fmt"

func bfsLevels(adj [][]int, start int) [][]int {
	visited := make([]bool, len(adj))
	levels := [][]int{}
	queue := []int{start}
	visited[start] = true

	for len(queue) > 0 {
		size := len(queue)
		level := []int{}
		for i := 0; i < size; i++ {
			v := queue[0]; queue = queue[1:]
			level = append(level, v)
			for _, u := range adj[v] {
				if !visited[u] {
					visited[u] = true
					queue = append(queue, u)
				}
			}
		}
		levels = append(levels, level)
	}
	return levels
}

func main() {
	adj := make([][]int, 7)
	for _, e := range [][2]int{{0,1},{0,2},{1,3},{1,4},{2,5},{2,6}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	for i, level := range bfsLevels(adj, 0) {
		fmt.Printf("Level %d: %v\n", i, level)
	}
}
```

**Textual Figure:**
```
    Tree Graph (Undirected, 7 vertices):

                ┌───┐              Level 0
                │ 0 │
                └─┬─┘
               ╱   ╲
        ┌───┐       ┌───┐        Level 1
        │ 1 │       │ 2 │
        └─┬─┘       └─┬─┘
       ╱   ╲       ╱   ╲
    ┌───┐ ┌───┐ ┌───┐ ┌───┐  Level 2
    │ 3 │ │ 4 │ │ 5 │ │ 6 │
    └───┘ └───┘ └───┘ └───┘

    BFS Level-by-Level:
    Level 0: [0]
    Level 1: [1, 2]
    Level 2: [3, 4, 5, 6]

    Queue at each level boundary:
    Start → [0] → [1,2] → [3,4,5,6] → []
```

---

## Example 5: Multi-Source BFS (01-Matrix LeetCode 542)

```go
package main

import "fmt"

func updateMatrix(mat [][]int) [][]int {
	rows, cols := len(mat), len(mat[0])
	dist := make([][]int, rows)
	queue := [][2]int{}

	for r := 0; r < rows; r++ {
		dist[r] = make([]int, cols)
		for c := 0; c < cols; c++ {
			if mat[r][c] == 0 {
				queue = append(queue, [2]int{r, c})
			} else {
				dist[r][c] = 1<<31 - 1
			}
		}
	}

	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}
	for len(queue) > 0 {
		cell := queue[0]; queue = queue[1:]
		for _, d := range dirs {
			nr, nc := cell[0]+d[0], cell[1]+d[1]
			if nr >= 0 && nr < rows && nc >= 0 && nc < cols && dist[nr][nc] > dist[cell[0]][cell[1]]+1 {
				dist[nr][nc] = dist[cell[0]][cell[1]] + 1
				queue = append(queue, [2]int{nr, nc})
			}
		}
	}
	return dist
}

func main() {
	mat := [][]int{{0,0,0},{0,1,0},{1,1,1}}
	for _, row := range updateMatrix(mat) {
		fmt.Println(row)
	}
}
```

**Textual Figure:**
```
    Input Matrix:           Output (Distance to nearest 0):

    ┌───┬───┬───┐          ┌───┬───┬───┐
    │ 0 │ 0 │ 0 │          │ 0 │ 0 │ 0 │
    ├───┼───┼───┤          ├───┼───┼───┤
    │ 0 │ 1 │ 0 │    →     │ 0 │ 1 │ 0 │
    ├───┼───┼───┤          ├───┼───┼───┤
    │ 1 │ 1 │ 1 │          │ 1 │ 2 │ 1 │
    └───┴───┴───┘          └───┴───┴───┘

    Multi-source BFS: start from all 0-cells
    simultaneously, expand outward.
    Cell (2,1) is distance 2 from nearest 0.
```

---

## Example 6: DFS for All Paths (LeetCode 797)

```go
package main

import "fmt"

func allPathsSourceTarget(graph [][]int) [][]int {
	result := [][]int{}
	target := len(graph) - 1

	var dfs func(node int, path []int)
	dfs = func(node int, path []int) {
		path = append(path, node)
		if node == target {
			tmp := make([]int, len(path))
			copy(tmp, path)
			result = append(result, tmp)
			return
		}
		for _, next := range graph[node] {
			dfs(next, path)
		}
	}

	dfs(0, []int{})
	return result
}

func main() {
	graph := [][]int{{1,2},{3},{3},{}}
	fmt.Println(allPathsSourceTarget(graph))
	// [[0 1 3] [0 2 3]]
}
```

**Textual Figure:**
```
    DAG (4 vertices, 0 to 3):
    graph = [[1,2],[3],[3],[]]

        ┌───┐
        │ 0 │ (source)
        └─┬─┘
       ╱   ╲
      ↓     ↓
    ┌───┐ ┌───┐
    │ 1 │ │ 2 │
    └─┬─┘ └─┬─┘
      └───┬───┘
          ↓
        ┌───┐
        │ 3 │ (target)
        └───┘

    DFS explores all paths:
    Path 1: 0 → 1 → 3  ✓
    Path 2: 0 → 2 → 3  ✓
    Result: [[0,1,3], [0,2,3]]
```

---

## Example 7: Grid Traversal (Flood Fill LeetCode 733)

```go
package main

import "fmt"

func floodFill(image [][]int, sr, sc, color int) [][]int {
	orig := image[sr][sc]
	if orig == color { return image }

	rows, cols := len(image), len(image[0])
	var dfs func(r, c int)
	dfs = func(r, c int) {
		if r < 0 || r >= rows || c < 0 || c >= cols || image[r][c] != orig { return }
		image[r][c] = color
		dfs(r+1, c); dfs(r-1, c); dfs(r, c+1); dfs(r, c-1)
	}

	dfs(sr, sc)
	return image
}

func main() {
	image := [][]int{{1,1,1},{1,1,0},{1,0,1}}
	result := floodFill(image, 1, 1, 2)
	for _, row := range result {
		fmt.Println(row) // [[2 2 2] [2 2 0] [2 0 1]]
	}
}
```

**Textual Figure:**
```
    Flood Fill from (1,1), newColor=2:

    Before:             After:
    ┌───┬───┬───┐       ┌───┬───┬───┐
    │ 1 │ 1 │ 1 │       │ 2 │ 2 │ 2 │
    ├───┼───┼───┤       ├───┼───┼───┤
    │ 1 │*1*│ 0 │  →    │ 2 │ 2 │ 0 │
    ├───┼───┼───┤       ├───┼───┼───┤
    │ 1 │ 0 │ 1 │       │ 2 │ 0 │ 1 │
    └───┴───┴───┘       └───┴───┴───┘
    * = start cell

    DFS spreads to all 4-connected cells
    with same original color (1).
    Cells (2,2) isolated — blocked by 0s.
```

---

## Example 8: BFS Shortest Path (Minimum Knight Moves)

```go
package main

import "fmt"

func minKnightMoves(x, y int) int {
	if x < 0 { x = -x }
	if y < 0 { y = -y }
	if x == 0 && y == 0 { return 0 }

	visited := map[[2]int]bool{{0, 0}: true}
	queue := [][2]int{{0, 0}}
	moves := [8][2]int{{1,2},{2,1},{-1,2},{-2,1},{1,-2},{2,-1},{-1,-2},{-2,-1}}
	steps := 0

	for len(queue) > 0 {
		steps++
		size := len(queue)
		for i := 0; i < size; i++ {
			cur := queue[0]; queue = queue[1:]
			for _, m := range moves {
				nx, ny := cur[0]+m[0], cur[1]+m[1]
				if nx == x && ny == y { return steps }
				key := [2]int{nx, ny}
				if !visited[key] && nx >= -2 && ny >= -2 {
					visited[key] = true
					queue = append(queue, key)
				}
			}
		}
	}
	return -1
}

func main() {
	fmt.Println(minKnightMoves(2, 1)) // 1
	fmt.Println(minKnightMoves(5, 5)) // 4
}
```

**Textual Figure:**
```
    Knight Moves (8 possible L-shaped moves):

         ┌───┬───┬───┬───┬───┐
         │   │ X │   │ X │   │
         ├───┼───┼───┼───┼───┤
         │ X │   │   │   │ X │
         ├───┼───┼───┼───┼───┤
         │   │   │ K │   │   │
         ├───┼───┼───┼───┼───┤
         │ X │   │   │   │ X │
         ├───┼───┼───┼───┼───┤
         │   │ X │   │ X │   │
         └───┴───┴───┴───┴───┘
    K = Knight, X = reachable in 1 move

    minKnightMoves(2,1) = 1 (direct L-move)
    minKnightMoves(5,5) = 4 (BFS finds shortest)
    BFS is ideal: implicit graph, unweighted edges
```

---

## Example 9: DFS Traversal Order Comparison

```go
package main

import "fmt"

func traversalOrders(adj [][]int, root int) ([]int, []int) {
	n := len(adj)
	visited := make([]bool, n)
	preOrder, postOrder := []int{}, []int{}

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		preOrder = append(preOrder, v)
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
		postOrder = append(postOrder, v)
	}

	dfs(root)
	return preOrder, postOrder
}

func main() {
	// Tree: 0→{1,2}, 1→{3,4}, 2→{5}
	adj := [][]int{{1,2},{3,4},{5},{},{},{}}
	pre, post := traversalOrders(adj, 0)
	fmt.Println("Pre-order: ", pre)  // [0 1 3 4 2 5]
	fmt.Println("Post-order:", post) // [3 4 1 5 2 0]
}
```

**Textual Figure:**
```
    Tree (Directed): 0→{1,2}, 1→{3,4}, 2→{5}

            ┌───┐
            │ 0 │
            └─┬─┘
           ╱   ╲
        ┌───┐ ┌───┐
        │ 1 │ │ 2 │
        └─┬─┘ └─┬─┘
       ╱   ╲     │
    ┌───┐ ┌───┐ ┌───┐
    │ 3 │ │ 4 │ │ 5 │
    └───┘ └───┘ └───┘

    Pre-order  (visit on entry): [0, 1, 3, 4, 2, 5]
    Post-order (visit on exit):  [3, 4, 1, 5, 2, 0]

    DFS trace: enter 0 → enter 1 → enter 3 → leave 3
    → enter 4 → leave 4 → leave 1 → enter 2
    → enter 5 → leave 5 → leave 2 → leave 0
```

---

## Example 10: BFS vs DFS Comparison on Same Graph

```go
package main

import "fmt"

func bfsOrder(adj [][]int, start int) []int {
	visited := make([]bool, len(adj))
	order := []int{}
	queue := []int{start}
	visited[start] = true
	for len(queue) > 0 {
		v := queue[0]; queue = queue[1:]
		order = append(order, v)
		for _, u := range adj[v] {
			if !visited[u] { visited[u] = true; queue = append(queue, u) }
		}
	}
	return order
}

func dfsOrder(adj [][]int, start int) []int {
	visited := make([]bool, len(adj))
	order := []int{}
	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		order = append(order, v)
		for _, u := range adj[v] {
			if !visited[u] { dfs(u) }
		}
	}
	dfs(start)
	return order
}

func main() {
	adj := make([][]int, 7)
	for _, e := range [][2]int{{0,1},{0,2},{1,3},{1,4},{2,5},{2,6}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("BFS:", bfsOrder(adj, 0)) // [0 1 2 3 4 5 6]
	fmt.Println("DFS:", dfsOrder(adj, 0)) // [0 1 3 4 2 5 6]
}
```

**Textual Figure:**
```
    Tree (Undirected, 7 vertices):

                ┌───┐
                │ 0 │
                └─┬─┘
               ╱   ╲
        ┌───┐       ┌───┐
        │ 1 │       │ 2 │
        └─┬─┘       └─┬─┘
       ╱   ╲       ╱   ╲
    ┌───┐ ┌───┐ ┌───┐ ┌───┐
    │ 3 │ │ 4 │ │ 5 │ │ 6 │
    └───┘ └───┘ └───┘ └───┘

    BFS (level by level):  [0, 1, 2, 3, 4, 5, 6]
      0 ─→ 1,2 ─→ 3,4,5,6

    DFS (depth first):     [0, 1, 3, 4, 2, 5, 6]
      0 ─→ 1 ─→ 3 (backtrack) ─→ 4 (backtrack)
      ─→ 2 ─→ 5 (backtrack) ─→ 6

    Same graph, different visit order!
```

---

## Summary Table

| Traversal | DS | Space | Use Case |
|-----------|-----|-------|----------|
| BFS | Queue | O(V) | Shortest path, level order |
| DFS | Stack | O(V) | Connectivity, cycles, topo sort |
| Multi-src BFS | Queue | O(V) | Distance from multiple sources |
| DFS timestamps | Stack | O(V) | Edge classification, SCC |

## Key Takeaways

1. BFS explores level by level → naturally finds shortest path in unweighted graphs
2. DFS explores deep → good for connectivity, cycles, topological sort
3. Both are O(V + E) time, O(V) space
4. Mark visited BEFORE enqueue in BFS to avoid duplicates
5. Pre/post timestamps in DFS enable edge classification

> **Next up:** Cycle Detection →
