# Phase 12: Graphs — Connected Components

## Overview

A **connected component** is a maximal set of vertices where every vertex is reachable from every other vertex.

```
Graph: 0-1-2   3-4   5
Components: {0,1,2}, {3,4}, {5}
Count: 3
```

**Methods:** DFS, BFS, or Union-Find — all O(V + E)

---

## Example 1: Count Connected Components (DFS)

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
	count := 0

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for _, u := range adj[v] { if !visited[u] { dfs(u) } }
	}

	for i := 0; i < n; i++ {
		if !visited[i] {
			count++
			dfs(i)
		}
	}
	return count
}

func main() {
	fmt.Println(countComponents(5, [][]int{{0,1},{1,2},{3,4}}))         // 2
	fmt.Println(countComponents(5, [][]int{{0,1},{1,2},{2,3},{3,4}}))   // 1
	fmt.Println(countComponents(5, [][]int{}))                           // 5
}
```

**Textual Figure:**
```
Test 1: n=5, edges=[[0,1],[1,2],[3,4]]

  Component 0       Component 1
  ┌───────────┐     ┌───────┐
  │ 0 ─ 1 ─ 2 │     │ 3 ─ 4 │
  └───────────┘     └───────┘

  DFS: visit 0→1→2 (comp 1), then 3→4 (comp 2)
  Result: 2 components

Test 2: n=5, edges=[[0,1],[1,2],[2,3],[3,4]]
  All connected: 0─1─2─3─4  → 1 component

Test 3: n=5, edges=[]  (no edges)
  Each vertex is its own component
  {0} {1} {2} {3} {4}  → 5 components
```

---

## Example 2: Label Each Component

```go
package main

import "fmt"

func labelComponents(n int, adj [][]int) ([]int, int) {
	comp := make([]int, n)
	for i := range comp { comp[i] = -1 }
	id := 0

	var dfs func(v int)
	dfs = func(v int) {
		comp[v] = id
		for _, u := range adj[v] {
			if comp[u] == -1 { dfs(u) }
		}
	}

	for i := 0; i < n; i++ {
		if comp[i] == -1 {
			dfs(i)
			id++
		}
	}
	return comp, id
}

func main() {
	adj := make([][]int, 7)
	for _, e := range [][2]int{{0,1},{1,2},{3,4},{5,6}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	comp, count := labelComponents(7, adj)
	fmt.Println("Labels:", comp)  // [0 0 0 1 1 2 2]
	fmt.Println("Count:", count)  // 3
}
```

**Textual Figure:**
```
Undirected Graph (7 vertices):
  Edges: 0─1, 1─2, 3─4, 5─6

  Component 0 (id=0)    Component 1 (id=1)   Component 2 (id=2)
  ┌───────────┐      ┌───────┐       ┌───────┐
  │ 0 ─ 1 ─ 2 │      │ 3 ─ 4 │       │ 5 ─ 6 │
  └───────────┘      └───────┘       └───────┘

Labeling process:
  i=0: comp[0]=-1 → dfs(0), assign id=0
       dfs(0)→dfs(1)→dfs(2): comp=[0,0,0,-1,-1,-1,-1]
  i=3: comp[3]=-1 → dfs(3), assign id=1
       dfs(3)→dfs(4): comp=[0,0,0,1,1,-1,-1]
  i=5: comp[5]=-1 → dfs(5), assign id=2
       dfs(5)→dfs(6): comp=[0,0,0,1,1,2,2]

Labels: [0, 0, 0, 1, 1, 2, 2]   Count: 3
```

---

## Example 3: Connected Components with BFS

```go
package main

import "fmt"

func componentsBFS(n int, adj [][]int) int {
	visited := make([]bool, n)
	count := 0

	for i := 0; i < n; i++ {
		if visited[i] { continue }
		count++
		queue := []int{i}
		visited[i] = true
		for len(queue) > 0 {
			v := queue[0]; queue = queue[1:]
			for _, u := range adj[v] {
				if !visited[u] {
					visited[u] = true
					queue = append(queue, u)
				}
			}
		}
	}
	return count
}

func main() {
	adj := make([][]int, 6)
	for _, e := range [][2]int{{0,1},{2,3},{4,5}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("Components:", componentsBFS(6, adj)) // 3
}
```

**Textual Figure:**
```
Undirected Graph (6 vertices):
  Edges: 0─1, 2─3, 4─5

  ┌───────┐  ┌───────┐  ┌───────┐
  │ 0 ─ 1 │  │ 2 ─ 3 │  │ 4 ─ 5 │
  └───────┘  └───────┘  └───────┘

BFS Component Discovery:
  i=0: not visited → BFS from 0
       queue=[0] → visit 0, enqueue 1
       queue=[1] → visit 1 → done. Component 1.
  i=2: not visited → BFS from 2
       queue=[2] → visit 2, enqueue 3
       queue=[3] → visit 3 → done. Component 2.
  i=4: not visited → BFS from 4
       queue=[4] → visit 4, enqueue 5
       queue=[5] → visit 5 → done. Component 3.

Result: 3 components
```

---

## Example 4: Connected Components with Union-Find

```go
package main

import "fmt"

type UnionFind struct {
	parent, rank []int
	count        int
}

func NewUF(n int) *UnionFind {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &UnionFind{p, make([]int, n), n}
}

func (uf *UnionFind) Find(x int) int {
	if uf.parent[x] != x { uf.parent[x] = uf.Find(uf.parent[x]) }
	return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) {
	px, py := uf.Find(x), uf.Find(y)
	if px == py { return }
	if uf.rank[px] < uf.rank[py] { px, py = py, px }
	uf.parent[py] = px
	if uf.rank[px] == uf.rank[py] { uf.rank[px]++ }
	uf.count--
}

func main() {
	uf := NewUF(5)
	edges := [][2]int{{0,1},{1,2},{3,4}}
	for _, e := range edges { uf.Union(e[0], e[1]) }
	fmt.Println("Components:", uf.count) // 2
}
```

**Textual Figure:**
```
Union-Find on n=5, edges=[[0,1],[1,2],[3,4]]

Initial: each node is its own component (count=5)
  parent: [0, 1, 2, 3, 4]

┌──────┬──────────────────┬───────┬─────────────────────┐
│ Edge │ Union            │ count │ Components            │
├──────┼──────────────────┼───────┼─────────────────────┤
│ 0─1 │ Union(0,1)       │   4   │ {0,1} {2} {3} {4}     │
│ 1─2 │ Union(1,2)       │   3   │ {0,1,2} {3} {4}       │
│ 3─4 │ Union(3,4)       │   2   │ {0,1,2} {3,4}         │
└──────┴──────────────────┴───────┴─────────────────────┘

Result: 2 components
```

---

## Example 5: Number of Provinces (LeetCode 547)

```go
package main

import "fmt"

func findCircleNum(isConnected [][]int) int {
	n := len(isConnected)
	visited := make([]bool, n)
	count := 0

	var dfs func(v int)
	dfs = func(v int) {
		visited[v] = true
		for u := 0; u < n; u++ {
			if isConnected[v][u] == 1 && !visited[u] {
				dfs(u)
			}
		}
	}

	for i := 0; i < n; i++ {
		if !visited[i] {
			count++
			dfs(i)
		}
	}
	return count
}

func main() {
	fmt.Println(findCircleNum([][]int{{1,1,0},{1,1,0},{0,0,1}})) // 2
	fmt.Println(findCircleNum([][]int{{1,0,0},{0,1,0},{0,0,1}})) // 3
}
```

**Textual Figure:**
```
Test 1: isConnected = [[1,1,0],[1,1,0],[0,0,1]]

  Adjacency matrix view:
       0  1  2
    0 [1, 1, 0]   → 0 connected to 1
    1 [1, 1, 0]   → 1 connected to 0
    2 [0, 0, 1]   → 2 alone

  Province 1: {0, 1}     Province 2: {2}
  ┌───────┐              ┌───┐
  │ 0 ─ 1 │              │ 2 │
  └───────┘              └───┘
  Result: 2

Test 2: isConnected = [[1,0,0],[0,1,0],[0,0,1]]
  No connections: {0} {1} {2}
  Result: 3
```

---

## Example 6: Largest Connected Component

```go
package main

import "fmt"

func largestComponent(n int, adj [][]int) int {
	visited := make([]bool, n)
	maxSize := 0

	var dfs func(v int) int
	dfs = func(v int) int {
		visited[v] = true
		size := 1
		for _, u := range adj[v] {
			if !visited[u] { size += dfs(u) }
		}
		return size
	}

	for i := 0; i < n; i++ {
		if !visited[i] {
			s := dfs(i)
			if s > maxSize { maxSize = s }
		}
	}
	return maxSize
}

func main() {
	adj := make([][]int, 8)
	for _, e := range [][2]int{{0,1},{1,2},{3,4},{4,5},{5,6},{6,7}} {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}
	fmt.Println("Largest component:", largestComponent(8, adj)) // 5
}
```

**Textual Figure:**
```
Undirected Graph (8 vertices):
  Edges: 0─1, 1─2, 3─4, 4─5, 5─6, 6─7

  Component A (size=3)    Component B (size=5)
  ┌───────────┐        ┌───────────────────┐
  │ 0 ─ 1 ─ 2 │        │ 3 ─ 4 ─ 5 ─ 6 ─ 7 │
  └───────────┘        └───────────────────┘

DFS finds each component and tracks size:
  dfs(0): visits {0,1,2} → size=3
  dfs(3): visits {3,4,5,6,7} → size=5
  maxSize = max(3, 5) = 5

Result: 5
```

---

## Example 7: Number of Islands (Grid Components) LeetCode 200

```go
package main

import "fmt"

func numIslands(grid [][]byte) int {
	if len(grid) == 0 { return 0 }
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
		{'1','1','0','0','0'},
		{'1','1','0','0','0'},
		{'0','0','1','0','0'},
		{'0','0','0','1','1'},
	}
	fmt.Println(numIslands(grid)) // 3
}
```

**Textual Figure:**
```
Input Grid (4×5, '1'=land, '0'=water):
┌───┬───┬───┬───┬───┐
│ 1 │ 1 │ 0 │ 0 │ 0 │  ← Island 1
├───┼───┼───┼───┼───┤
│ 1 │ 1 │ 0 │ 0 │ 0 │
├───┼───┼───┼───┼───┤
│ 0 │ 0 │ 1 │ 0 │ 0 │  ← Island 2
├───┼───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 1 │ 1 │  ← Island 3
└───┴───┴───┴───┴───┘

DFS Flood Fill per island:
  Island 1: start(0,0) → (0,1) → (1,0) → (1,1)  [4 cells]
  Island 2: start(2,2)                          [1 cell]
  Island 3: start(3,3) → (3,4)                  [2 cells]

Result: 3 islands
```

---

## Example 8: Remove Stones (LeetCode 947)

```go
package main

import "fmt"

func removeStones(stones [][]int) int {
	n := len(stones)
	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) {
		px, py := find(x), find(y)
		if px != py { parent[px] = py }
	}

	// Group stones sharing same row or column
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			if stones[i][0] == stones[j][0] || stones[i][1] == stones[j][1] {
				union(i, j)
			}
		}
	}

	// Count components
	roots := map[int]bool{}
	for i := 0; i < n; i++ {
		roots[find(i)] = true
	}
	return n - len(roots) // removable = total - components
}

func main() {
	fmt.Println(removeStones([][]int{{0,0},{0,1},{1,0},{1,2},{2,1},{2,2}})) // 5
}
```

**Textual Figure:**
```
Stones at coordinates (row, col):
  s0=(0,0)  s1=(0,1)  s2=(1,0)  s3=(1,2)  s4=(2,1)  s5=(2,2)

  Grid view:       col 0  col 1  col 2
  row 0:            s0     s1
  row 1:            s2            s3
  row 2:                   s4     s5

Union-Find groups stones sharing row or column:
  s0 & s1: same row 0    → union(0,1)
  s0 & s2: same col 0    → union(0,2)
  s1 & s4: same col 1    → union(1,4)
  s3 & s5: same col 2    → union(3,5)
  s2 & s3: same row 1    → union(2,3)
  s4 & s5: same row 2    → union(4,5)

  All stones end up in ONE component!
  Removable = total - components = 6 - 1 = 5

Result: 5
```

---

## Example 9: Accounts Merge (LeetCode 721)

```go
package main

import (
	"fmt"
	"sort"
)

func accountsMerge(accounts [][]string) [][]string {
	parent := map[string]string{}
	owner := map[string]string{}

	var find func(x string) string
	find = func(x string) string {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y string) {
		parent[find(x)] = find(y)
	}

	for _, acc := range accounts {
		name := acc[0]
		for _, email := range acc[1:] {
			if _, ok := parent[email]; !ok { parent[email] = email }
			owner[email] = name
			union(acc[1], email)
		}
	}

	groups := map[string][]string{}
	for email := range parent {
		root := find(email)
		groups[root] = append(groups[root], email)
	}

	var result [][]string
	for root, emails := range groups {
		sort.Strings(emails)
		result = append(result, append([]string{owner[root]}, emails...))
	}
	return result
}

func main() {
	accounts := [][]string{
		{"John", "john@mail.com", "john_newyork@mail.com"},
		{"John", "john@mail.com", "john00@mail.com"},
		{"Mary", "mary@mail.com"},
		{"John", "johnnybravo@mail.com"},
	}
	for _, acc := range accountsMerge(accounts) {
		fmt.Println(acc)
	}
}
```

**Textual Figure:**
```
Input Accounts:
  Acc 0: John  {john@mail.com, john_newyork@mail.com}
  Acc 1: John  {john@mail.com, john00@mail.com}
  Acc 2: Mary  {mary@mail.com}
  Acc 3: John  {johnnybravo@mail.com}

Union-Find on emails:
  john@mail.com = john_newyork@mail.com     (from Acc 0)
  john@mail.com = john00@mail.com           (from Acc 1)
  → All three merge into one component:

  Component 1 (John):
    ┌───────────────────────────────────────────┐
    │ john00  ─  john@mail  ─  john_newyork │
    └───────────────────────────────────────────┘
  Component 2 (Mary):    {mary@mail.com}
  Component 3 (John):    {johnnybravo@mail.com}

Result: 3 merged accounts (sorted emails within each)
```

---

## Example 10: Dynamic Connectivity (Online Components)

```go
package main

import "fmt"

type DSU struct {
	parent, rank []int
	count        int
}

func NewDSU(n int) *DSU {
	p := make([]int, n)
	for i := range p { p[i] = i }
	return &DSU{p, make([]int, n), n}
}

func (d *DSU) Find(x int) int {
	if d.parent[x] != x { d.parent[x] = d.Find(d.parent[x]) }
	return d.parent[x]
}

func (d *DSU) Union(x, y int) bool {
	px, py := d.Find(x), d.Find(y)
	if px == py { return false }
	if d.rank[px] < d.rank[py] { px, py = py, px }
	d.parent[py] = px
	if d.rank[px] == d.rank[py] { d.rank[px]++ }
	d.count--
	return true
}

func (d *DSU) Connected(x, y int) bool {
	return d.Find(x) == d.Find(y)
}

func main() {
	dsu := NewDSU(6)

	ops := [][2]int{{0,1},{2,3},{4,5},{1,2},{3,4}}
	for _, op := range ops {
		dsu.Union(op[0], op[1])
		fmt.Printf("Union(%d,%d) → %d components\n", op[0], op[1], dsu.count)
	}
	fmt.Println("0 and 5 connected?", dsu.Connected(0, 5)) // true
}
```

**Textual Figure:**
```
Dynamic Connectivity (6 vertices, edges added online):

Initial: 6 isolated nodes, count=6
  {0} {1} {2} {3} {4} {5}

┌────────────┬─────────┬───────────────────────────────┐
│ Operation  │  Count  │ Components                    │
├────────────┼─────────┼───────────────────────────────┤
│ Union(0,1) │    5    │ {0,1} {2} {3} {4} {5}         │
│ Union(2,3) │    4    │ {0,1} {2,3} {4} {5}           │
│ Union(4,5) │    3    │ {0,1} {2,3} {4,5}             │
│ Union(1,2) │    2    │ {0,1,2,3} {4,5}               │
│ Union(3,4) │    1    │ {0,1,2,3,4,5}                 │
└────────────┴─────────┴───────────────────────────────┘

Final: 0─1─2─3─4─5 all in one component
  Connected(0, 5)? → true
```

---

## Key Takeaways

1. Connected component = maximal set of mutually reachable vertices
2. Three methods: DFS, BFS, Union-Find — all O(V+E)
3. Union-Find is best for online/dynamic problems (edges added over time)
4. Grid components: DFS/BFS with flood fill
5. Removable count in stone/similar problems = total - components

> **Next up:** Graph Traversal →
