# Phase 12: Graphs — BFS (Breadth-First Search)

## Overview

**BFS** explores a graph level by level using a **queue**. It finds the **shortest path** in unweighted graphs.

```
Start at node 0:
Level 0: [0]
Level 1: [1, 2]
Level 2: [3, 4]

BFS order: 0, 1, 2, 3, 4
```

| Property | Value |
|----------|-------|
| Time | O(V + E) |
| Space | O(V) for queue + visited |
| Shortest path? | Yes (unweighted) |
| Data structure | Queue |

---

## Example 1: Basic BFS

```go
package main

import "fmt"

func bfs(adj [][]int, start int) []int {
	visited := make([]bool, len(adj))
	queue := []int{start}
	visited[start] = true
	var order []int

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
	adj := [][]int{{1,2},{0,3,4},{0,4},{1},{1,2}}
	fmt.Println(bfs(adj, 0)) // [0 1 2 3 4]
}
```

---

## Example 2: BFS Shortest Path (Unweighted)

```go
package main

import "fmt"

func shortestPath(adj [][]int, src, dst int) int {
	if src == dst { return 0 }
	visited := make([]bool, len(adj))
	queue := []int{src}
	visited[src] = true
	dist := 0

	for len(queue) > 0 {
		dist++
		size := len(queue)
		for i := 0; i < size; i++ {
			v := queue[0]; queue = queue[1:]
			for _, u := range adj[v] {
				if u == dst { return dist }
				if !visited[u] {
					visited[u] = true
					queue = append(queue, u)
				}
			}
		}
	}
	return -1 // unreachable
}

func main() {
	adj := [][]int{{1,2},{0,3},{0,3},{1,2,4},{3}}
	fmt.Println(shortestPath(adj, 0, 4)) // 3
}
```

---

## Example 3: Number of Islands BFS (LeetCode 200)

```go
package main

import "fmt"

func numIslands(grid [][]byte) int {
	if len(grid) == 0 { return 0 }
	rows, cols := len(grid), len(grid[0])
	count := 0
	dirs := [][2]int{{0,1},{0,-1},{1,0},{-1,0}}

	bfs := func(sr, sc int) {
		queue := [][2]int{{sr, sc}}
		grid[sr][sc] = '0'
		for len(queue) > 0 {
			p := queue[0]; queue = queue[1:]
			for _, d := range dirs {
				nr, nc := p[0]+d[0], p[1]+d[1]
				if nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == '1' {
					grid[nr][nc] = '0'
					queue = append(queue, [2]int{nr, nc})
				}
			}
		}
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if grid[r][c] == '1' {
				count++
				bfs(r, c)
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

## Example 4: Word Ladder (LeetCode 127)

```go
package main

import "fmt"

func ladderLength(beginWord string, endWord string, wordList []string) int {
	wordSet := map[string]bool{}
	for _, w := range wordList { wordSet[w] = true }
	if !wordSet[endWord] { return 0 }

	queue := []string{beginWord}
	visited := map[string]bool{beginWord: true}
	steps := 1

	for len(queue) > 0 {
		steps++
		size := len(queue)
		for i := 0; i < size; i++ {
			word := queue[0]; queue = queue[1:]
			buf := []byte(word)
			for j := 0; j < len(buf); j++ {
				orig := buf[j]
				for c := byte('a'); c <= 'z'; c++ {
					buf[j] = c
					next := string(buf)
					if next == endWord { return steps }
					if wordSet[next] && !visited[next] {
						visited[next] = true
						queue = append(queue, next)
					}
				}
				buf[j] = orig
			}
		}
	}
	return 0
}

func main() {
	fmt.Println(ladderLength("hit", "cog", []string{"hot","dot","dog","lot","log","cog"})) // 5
	fmt.Println(ladderLength("hit", "cog", []string{"hot","dot","dog","lot","log"}))        // 0
}
```

---

## Example 5: Multi-Source BFS (Walls and Gates LC 286)

```go
package main

import "fmt"

func wallsAndGates(rooms [][]int) {
	if len(rooms) == 0 { return }
	INF := 1<<31 - 1
	rows, cols := len(rooms), len(rooms[0])
	queue := [][2]int{}

	// Start from ALL gates simultaneously
	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			if rooms[r][c] == 0 { queue = append(queue, [2]int{r, c}) }
		}
	}

	dirs := [][2]int{{0,1},{0,-1},{1,0},{-1,0}}
	for len(queue) > 0 {
		p := queue[0]; queue = queue[1:]
		for _, d := range dirs {
			nr, nc := p[0]+d[0], p[1]+d[1]
			if nr < 0 || nr >= rows || nc < 0 || nc >= cols || rooms[nr][nc] != INF { continue }
			rooms[nr][nc] = rooms[p[0]][p[1]] + 1
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
	for _, r := range rooms { fmt.Println(r) }
}
```

**Why multi-source?** BFS from all sources simultaneously finds the shortest distance from any source to every cell.

---

## Example 6: BFS Level Order (Track Levels)

```go
package main

import "fmt"

func bfsLevels(adj [][]int, start int) [][]int {
	visited := make([]bool, len(adj))
	queue := []int{start}
	visited[start] = true
	var levels [][]int

	for len(queue) > 0 {
		size := len(queue)
		level := make([]int, 0, size)
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
	adj := [][]int{{1,2},{0,3,4},{0,5},{1},{1},{2}}
	levels := bfsLevels(adj, 0)
	for i, l := range levels {
		fmt.Printf("Level %d: %v\n", i, l)
	}
	// Level 0: [0]
	// Level 1: [1 2]
	// Level 2: [3 4 5]
}
```

---

## Example 7: Bidirectional BFS

```go
package main

import "fmt"

func biDirectionalBFS(adj [][]int, src, dst int) int {
	if src == dst { return 0 }
	n := len(adj)
	visitedS := make([]bool, n)
	visitedD := make([]bool, n)
	queueS := []int{src}
	queueD := []int{dst}
	visitedS[src] = true
	visitedD[dst] = true
	steps := 0

	for len(queueS) > 0 && len(queueD) > 0 {
		steps++
		// Expand the smaller frontier
		if len(queueS) > len(queueD) {
			queueS, queueD = queueD, queueS
			visitedS, visitedD = visitedD, visitedS
		}

		var next []int
		for _, v := range queueS {
			for _, u := range adj[v] {
				if visitedD[u] { return steps }
				if !visitedS[u] {
					visitedS[u] = true
					next = append(next, u)
				}
			}
		}
		queueS = next
	}
	return -1
}

func main() {
	adj := [][]int{{1,2},{0,3},{0,4},{1,5},{2,5},{3,4}}
	fmt.Println(biDirectionalBFS(adj, 0, 5)) // 3
}
```

---

## Example 8: 01-BFS (Deque BFS)

```go
package main

import "fmt"

// BFS on graph with edge weights 0 or 1
// Use deque: push weight-0 to front, weight-1 to back
func bfs01(n int, adj [][]struct{ to, w int }, src int) []int {
	dist := make([]int, n)
	for i := range dist { dist[i] = 1<<31 - 1 }
	dist[src] = 0

	deque := []int{src}
	for len(deque) > 0 {
		v := deque[0]; deque = deque[1:]
		for _, e := range adj[v] {
			nd := dist[v] + e.w
			if nd < dist[e.to] {
				dist[e.to] = nd
				if e.w == 0 {
					deque = append([]int{e.to}, deque...) // front
				} else {
					deque = append(deque, e.to) // back
				}
			}
		}
	}
	return dist
}

func main() {
	type E = struct{ to, w int }
	adj := [][]E{
		{{1, 1}, {2, 0}},
		{{3, 0}},
		{{1, 1}, {3, 1}},
		{},
	}
	fmt.Println(bfs01(4, adj, 0)) // [0 1 0 1]
}
```

---

## Example 9: Minimum Height Trees (LeetCode 310)

```go
package main

import "fmt"

func findMinHeightTrees(n int, edges [][]int) []int {
	if n == 1 { return []int{0} }

	adj := make([][]int, n)
	degree := make([]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
		degree[e[0]]++
		degree[e[1]]++
	}

	// Start with all leaves
	queue := []int{}
	for i := 0; i < n; i++ {
		if degree[i] == 1 { queue = append(queue, i) }
	}

	remaining := n
	for remaining > 2 {
		size := len(queue)
		remaining -= size
		var next []int
		for _, v := range queue {
			for _, u := range adj[v] {
				degree[u]--
				if degree[u] == 1 { next = append(next, u) }
			}
		}
		queue = next
	}
	return queue
}

func main() {
	fmt.Println(findMinHeightTrees(4, [][]int{{1,0},{1,2},{1,3}}))         // [1]
	fmt.Println(findMinHeightTrees(6, [][]int{{3,0},{3,1},{3,2},{3,4},{5,4}})) // [3 4]
}
```

---

## Example 10: BFS Pattern Template

```go
package main

import "fmt"

func main() {
	fmt.Println("=== BFS Template ===")
	fmt.Println(`
// Standard BFS
func bfs(adj [][]int, start int) {
    visited := make([]bool, len(adj))
    queue := []int{start}
    visited[start] = true
    
    for len(queue) > 0 {
        v := queue[0]; queue = queue[1:]
        // process v
        for _, u := range adj[v] {
            if !visited[u] {
                visited[u] = true
                queue = append(queue, u)
            }
        }
    }
}

// Level-by-level BFS
for len(queue) > 0 {
    size := len(queue)
    for i := 0; i < size; i++ {
        v := queue[0]; queue = queue[1:]
        // process v at current level
    }
    level++
}

// Multi-source BFS
// Initialize queue with ALL sources
for _, src := range sources {
    queue = append(queue, src)
    visited[src] = true
}
// Then standard BFS
`)

	fmt.Println("When to use BFS:")
	fmt.Println("  • Shortest path in unweighted graph")
	fmt.Println("  • Level-order traversal")
	fmt.Println("  • Multi-source shortest distance")
	fmt.Println("  • Topological sort (Kahn's)")
	fmt.Println("  • Finding all nodes at distance K")
}
```

---

## Key Takeaways

1. BFS = queue-based, explores level by level → shortest path in unweighted graphs
2. Mark visited **when enqueuing**, not when dequeuing (avoids duplicates)
3. Multi-source BFS: start from all sources simultaneously
4. Level tracking: use `size := len(queue)` inner loop
5. 0-1 BFS uses a deque for graphs with 0/1 weights — O(V+E)

> **Next up:** DFS (Graph) →
