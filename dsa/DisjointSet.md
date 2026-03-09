# Phase 13: Union Find — Disjoint Set

## Overview

A **Disjoint Set** (Union-Find) is a data structure that tracks a set of elements partitioned into non-overlapping subsets. It supports two primary operations:

| Operation | Description | Naive Time | Optimized Time |
|-----------|-------------|-----------|---------------|
| **Find(x)** | Return the representative (root) of x's set | O(n) | O(α(n)) ≈ O(1) |
| **Union(x,y)** | Merge the sets containing x and y | O(n) | O(α(n)) ≈ O(1) |

α(n) = inverse Ackermann function, effectively constant for all practical input sizes.

---

## Example 1: Basic Disjoint Set (Naive)

```go
package main

import "fmt"

type DisjointSet struct {
	parent []int
}

func NewDisjointSet(n int) *DisjointSet {
	parent := make([]int, n)
	for i := range parent { parent[i] = i }
	return &DisjointSet{parent: parent}
}

func (ds *DisjointSet) Find(x int) int {
	for ds.parent[x] != x {
		x = ds.parent[x]
	}
	return x
}

func (ds *DisjointSet) Union(x, y int) {
	rx, ry := ds.Find(x), ds.Find(y)
	if rx != ry { ds.parent[rx] = ry }
}

func (ds *DisjointSet) Connected(x, y int) bool {
	return ds.Find(x) == ds.Find(y)
}

func main() {
	ds := NewDisjointSet(6)
	ds.Union(0, 1)
	ds.Union(2, 3)
	ds.Union(1, 3)

	fmt.Println("0 and 3 connected:", ds.Connected(0, 3)) // true
	fmt.Println("0 and 4 connected:", ds.Connected(0, 4)) // false
}
```

---

## Example 2: Number of Connected Components

```go
package main

import "fmt"

func countComponents(n int, edges [][2]int) int {
	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	components := n
	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx != ry {
			parent[rx] = ry
			components--
		}
	}
	return components
}

func main() {
	edges := [][2]int{{0,1},{1,2},{3,4}}
	fmt.Println("Components:", countComponents(5, edges)) // 2
}
```

---

## Example 3: Number of Islands (LeetCode 200 — Union-Find Approach)

```go
package main

import "fmt"

func numIslands(grid [][]byte) int {
	if len(grid) == 0 { return 0 }
	rows, cols := len(grid), len(grid[0])

	parent := make([]int, rows*cols)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	count := 0
	for i := 0; i < rows; i++ {
		for j := 0; j < cols; j++ {
			if grid[i][j] == '1' { count++ }
		}
	}

	dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}
	for i := 0; i < rows; i++ {
		for j := 0; j < cols; j++ {
			if grid[i][j] == '0' { continue }
			for _, d := range dirs {
				ni, nj := i+d[0], j+d[1]
				if ni >= 0 && ni < rows && nj >= 0 && nj < cols && grid[ni][nj] == '1' {
					rx := find(i*cols + j)
					ry := find(ni*cols + nj)
					if rx != ry { parent[rx] = ry; count-- }
				}
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
	fmt.Println("Islands:", numIslands(grid)) // 3
}
```

---

## Example 4: Accounts Merge (LeetCode 721)

```go
package main

import (
	"fmt"
	"sort"
)

func accountsMerge(accounts [][]string) [][]string {
	parent := make([]int, len(accounts))
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	union := func(x, y int) { parent[find(x)] = find(y) }

	emailToID := map[string]int{} // email → account index
	for i, acc := range accounts {
		for _, email := range acc[1:] {
			if id, ok := emailToID[email]; ok {
				union(i, id)
			}
			emailToID[email] = i
		}
	}

	// Group emails by root
	groups := map[int]map[string]bool{}
	for email, id := range emailToID {
		root := find(id)
		if groups[root] == nil { groups[root] = map[string]bool{} }
		groups[root][email] = true
	}

	result := [][]string{}
	for root, emails := range groups {
		sorted := []string{}
		for e := range emails { sorted = append(sorted, e) }
		sort.Strings(sorted)
		merged := append([]string{accounts[root][0]}, sorted...)
		result = append(result, merged)
	}
	return result
}

func main() {
	accounts := [][]string{
		{"John", "j1@ex.com", "j2@ex.com"},
		{"John", "j2@ex.com", "j3@ex.com"},
		{"Mary", "m1@ex.com"},
	}
	merged := accountsMerge(accounts)
	for _, acc := range merged {
		fmt.Println(acc)
	}
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
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { return e } // cycle detected
		parent[rx] = ry
	}
	return nil
}

func main() {
	edges := [][]int{{1,2},{1,3},{2,3}}
	fmt.Println("Redundant:", findRedundantConnection(edges)) // [2 3]

	edges2 := [][]int{{1,2},{2,3},{3,4},{1,4},{1,5}}
	fmt.Println("Redundant:", findRedundantConnection(edges2)) // [1 4]
}
```

---

## Example 6: Graph Valid Tree (LeetCode 261)

```go
package main

import "fmt"

func validTree(n int, edges [][]int) bool {
	if len(edges) != n-1 { return false } // tree has exactly n-1 edges

	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	for _, e := range edges {
		rx, ry := find(e[0]), find(e[1])
		if rx == ry { return false } // cycle
		parent[rx] = ry
	}
	return true
}

func main() {
	fmt.Println(validTree(5, [][]int{{0,1},{0,2},{0,3},{1,4}})) // true
	fmt.Println(validTree(5, [][]int{{0,1},{1,2},{2,3},{1,3},{1,4}})) // false
}
```

---

## Example 7: Earliest Time When Everyone Becomes Friends (LeetCode 1101)

```go
package main

import (
	"fmt"
	"sort"
)

func earliestAcq(logs [][]int, n int) int {
	sort.Slice(logs, func(i, j int) bool { return logs[i][0] < logs[j][0] })

	parent := make([]int, n)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	components := n
	for _, log := range logs {
		t, a, b := log[0], log[1], log[2]
		rx, ry := find(a), find(b)
		if rx != ry {
			parent[rx] = ry
			components--
			if components == 1 { return t }
		}
	}
	return -1
}

func main() {
	logs := [][]int{{0,2,0},{1,0,1},{3,0,3},{4,1,2},{7,3,1}}
	fmt.Println("Earliest:", earliestAcq(logs, 4)) // 3? let's see...
	// Actually: t=0: {0,2}, t=1: {0,1,2}, t=3: {0,1,2,3} → answer = 3
}
```

---

## Example 8: Dynamic Connectivity

```go
package main

import "fmt"

type DSU struct {
	parent []int
	rank   []int
	count  int
}

func NewDSU(n int) *DSU {
	p := make([]int, n)
	r := make([]int, n)
	for i := range p { p[i] = i }
	return &DSU{parent: p, rank: r, count: n}
}

func (d *DSU) Find(x int) int {
	if d.parent[x] != x { d.parent[x] = d.Find(d.parent[x]) }
	return d.parent[x]
}

func (d *DSU) Union(x, y int) bool {
	rx, ry := d.Find(x), d.Find(y)
	if rx == ry { return false }
	if d.rank[rx] < d.rank[ry] { rx, ry = ry, rx }
	d.parent[ry] = rx
	if d.rank[rx] == d.rank[ry] { d.rank[rx]++ }
	d.count--
	return true
}

func main() {
	dsu := NewDSU(5)
	operations := [][2]int{{0,1},{2,3},{1,3},{0,4}}

	for _, op := range operations {
		merged := dsu.Union(op[0], op[1])
		fmt.Printf("Union(%d,%d): merged=%v, components=%d\n", op[0], op[1], merged, dsu.count)
	}
}
```

---

## Example 9: Satisfiability of Equality Equations (LeetCode 990)

```go
package main

import "fmt"

func equationsPossible(equations []string) bool {
	parent := make([]int, 26)
	for i := range parent { parent[i] = i }

	var find func(x int) int
	find = func(x int) int {
		if parent[x] != x { parent[x] = find(parent[x]) }
		return parent[x]
	}

	// Process '==' first
	for _, eq := range equations {
		if eq[1] == '=' {
			a, b := int(eq[0]-'a'), int(eq[3]-'a')
			parent[find(a)] = find(b)
		}
	}

	// Check '!='
	for _, eq := range equations {
		if eq[1] == '!' {
			a, b := int(eq[0]-'a'), int(eq[3]-'a')
			if find(a) == find(b) { return false }
		}
	}
	return true
}

func main() {
	fmt.Println(equationsPossible([]string{"a==b","b!=a"})) // false
	fmt.Println(equationsPossible([]string{"a==b","b==c","a==c"})) // true
	fmt.Println(equationsPossible([]string{"a==b","b!=c","c==a"})) // false
}
```

---

## Example 10: Disjoint Set Operations Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Disjoint Set (Union-Find) Summary ===")
	fmt.Println()

	rows := []struct{ topic, detail string }{
		{"Make-Set", "Initialize each element as its own parent"},
		{"Find", "Follow parent pointers to root representative"},
		{"Union", "Link root of one set to root of another"},
		{"Path Compression", "Point nodes directly to root during Find"},
		{"Union by Rank", "Attach shorter tree under taller tree"},
		{"Union by Size", "Attach smaller set under larger set"},
		{"Amortized Time", "O(α(n)) per operation ≈ O(1)"},
		{"Space", "O(n) for parent + rank/size arrays"},
		{"Key Applications", "Connected components, Kruskal's MST, cycle detection"},
		{"Cannot Do", "Split/disconnect sets (offline only)"},
	}

	for i, r := range rows {
		fmt.Printf("%2d. %-20s %s\n", i+1, r.topic, r.detail)
	}
}
```

---

## Key Takeaways

1. Disjoint Set supports near-constant-time union and find operations
2. Each element has a `parent` pointer; root points to itself
3. Without optimization, trees can degenerate to O(n) height
4. Core use: dynamic connectivity, cycle detection, component counting
5. Foundation for Kruskal's MST, graph connectivity, equivalence classes

> **Next up:** Path Compression →
