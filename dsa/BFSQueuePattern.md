# Phase 7: Queue — BFS Queue Pattern

## Overview

**Breadth-First Search (BFS)** uses a queue to explore nodes **level by level**. It guarantees the shortest path in unweighted graphs and is the foundation for many tree/graph traversal patterns.

```
BFS Template:
  1. Enqueue source, mark visited
  2. While queue not empty:
     a. Dequeue node
     b. Process node
     c. Enqueue all unvisited neighbors
```

| Use Case | Queue Stores | Terminates When |
|----------|-------------|-----------------|
| Shortest path | (node, distance) | Target found |
| Level-order traversal | node | Queue empty |
| Multi-source BFS | all sources initially | Queue empty |
| 0-1 BFS | node (deque-based) | All processed |

---

## Example 1: Basic BFS — Level-Order Traversal (LeetCode 102)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }

    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, 0, size)

        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}

func main() {
    //       3
    //      / \
    //     9  20
    //       /  \
    //      15   7
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20,
            &TreeNode{15, nil, nil},
            &TreeNode{7, nil, nil},
        },
    }

    for i, level := range levelOrder(root) {
        fmt.Printf("Level %d: %v\n", i, level)
    }
    // Level 0: [3]
    // Level 1: [9, 20]
    // Level 2: [15, 7]
}
```

---

## Example 2: Shortest Path in Unweighted Graph

```go
package main

import "fmt"

func shortestPath(graph [][]int, src, dst int) int {
    n := len(graph)
    visited := make([]bool, n)
    visited[src] = true

    type Pair struct {
        Node, Dist int
    }
    queue := []Pair{{src, 0}}

    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]

        if cur.Node == dst {
            return cur.Dist
        }

        for _, neighbor := range graph[cur.Node] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, Pair{neighbor, cur.Dist + 1})
            }
        }
    }
    return -1 // unreachable
}

func main() {
    // 0 - 1 - 3
    // |       |
    // 2 - 4 - 5
    graph := [][]int{
        {1, 2},    // 0
        {0, 3},    // 1
        {0, 4},    // 2
        {1, 5},    // 3
        {2, 5},    // 4
        {3, 4},    // 5
    }

    fmt.Println("0→5:", shortestPath(graph, 0, 5)) // 3
    fmt.Println("0→3:", shortestPath(graph, 0, 3)) // 2
    fmt.Println("2→3:", shortestPath(graph, 2, 3)) // 3
}
```

---

## Example 3: Multi-Source BFS — Rotting Oranges (LeetCode 994)

```go
package main

import "fmt"

func orangesRotting(grid [][]int) int {
    rows, cols := len(grid), len(grid[0])
    type Pos struct{ R, C int }
    queue := []Pos{}
    fresh := 0

    // Enqueue all rotten oranges
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if grid[r][c] == 2 {
                queue = append(queue, Pos{r, c})
            } else if grid[r][c] == 1 {
                fresh++
            }
        }
    }

    if fresh == 0 {
        return 0
    }

    dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}
    minutes := 0

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            pos := queue[0]
            queue = queue[1:]

            for _, d := range dirs {
                nr, nc := pos.R+d[0], pos.C+d[1]
                if nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] == 1 {
                    grid[nr][nc] = 2
                    fresh--
                    queue = append(queue, Pos{nr, nc})
                }
            }
        }
        if len(queue) > 0 {
            minutes++
        }
    }

    if fresh > 0 {
        return -1
    }
    return minutes
}

func main() {
    grid := [][]int{
        {2, 1, 1},
        {1, 1, 0},
        {0, 1, 1},
    }
    fmt.Println("Minutes:", orangesRotting(grid)) // 4
}
```

**Why?** Multi-source BFS starts with ALL rotten oranges in the queue simultaneously — each "level" = 1 minute of spread.

---

## Example 4: BFS on Grid — Shortest Path in Binary Matrix (LeetCode 1091)

```go
package main

import "fmt"

func shortestPathBinaryMatrix(grid [][]int) int {
    n := len(grid)
    if grid[0][0] == 1 || grid[n-1][n-1] == 1 {
        return -1
    }

    type Cell struct {
        R, C, Dist int
    }
    queue := []Cell{{0, 0, 1}}
    grid[0][0] = 1 // mark visited

    dirs := [][2]int{
        {0, 1}, {0, -1}, {1, 0}, {-1, 0},
        {1, 1}, {1, -1}, {-1, 1}, {-1, -1},
    }

    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]

        if cur.R == n-1 && cur.C == n-1 {
            return cur.Dist
        }

        for _, d := range dirs {
            nr, nc := cur.R+d[0], cur.C+d[1]
            if nr >= 0 && nr < n && nc >= 0 && nc < n && grid[nr][nc] == 0 {
                grid[nr][nc] = 1
                queue = append(queue, Cell{nr, nc, cur.Dist + 1})
            }
        }
    }
    return -1
}

func main() {
    grid := [][]int{
        {0, 0, 0},
        {1, 1, 0},
        {1, 1, 0},
    }
    fmt.Println("Shortest path:", shortestPathBinaryMatrix(grid)) // 4
}
```

---

## Example 5: Word Ladder (LeetCode 127)

```go
package main

import "fmt"

func ladderLength(beginWord string, endWord string, wordList []string) int {
    wordSet := map[string]bool{}
    for _, w := range wordList {
        wordSet[w] = true
    }
    if !wordSet[endWord] {
        return 0
    }

    queue := []string{beginWord}
    visited := map[string]bool{beginWord: true}
    steps := 1

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            word := queue[0]
            queue = queue[1:]

            if word == endWord {
                return steps
            }

            // Try all single-character transformations
            chars := []byte(word)
            for j := 0; j < len(chars); j++ {
                original := chars[j]
                for c := byte('a'); c <= 'z'; c++ {
                    if c == original {
                        continue
                    }
                    chars[j] = c
                    next := string(chars)
                    if wordSet[next] && !visited[next] {
                        visited[next] = true
                        queue = append(queue, next)
                    }
                }
                chars[j] = original
            }
        }
        steps++
    }
    return 0
}

func main() {
    fmt.Println(ladderLength("hit", "cog",
        []string{"hot", "dot", "dog", "lot", "log", "cog"})) // 5
    fmt.Println(ladderLength("hit", "cog",
        []string{"hot", "dot", "dog", "lot", "log"})) // 0
}
```

---

## Example 6: Bidirectional BFS (Optimized Word Ladder)

```go
package main

import "fmt"

func ladderLengthBidir(beginWord, endWord string, wordList []string) int {
    wordSet := map[string]bool{}
    for _, w := range wordList {
        wordSet[w] = true
    }
    if !wordSet[endWord] {
        return 0
    }

    frontSet := map[string]bool{beginWord: true}
    backSet := map[string]bool{endWord: true}
    visited := map[string]bool{beginWord: true, endWord: true}
    steps := 1

    for len(frontSet) > 0 && len(backSet) > 0 {
        // Always expand the smaller set
        if len(frontSet) > len(backSet) {
            frontSet, backSet = backSet, frontSet
        }

        nextSet := map[string]bool{}
        for word := range frontSet {
            chars := []byte(word)
            for i := 0; i < len(chars); i++ {
                orig := chars[i]
                for c := byte('a'); c <= 'z'; c++ {
                    chars[i] = c
                    next := string(chars)
                    if backSet[next] {
                        return steps + 1
                    }
                    if wordSet[next] && !visited[next] {
                        visited[next] = true
                        nextSet[next] = true
                    }
                }
                chars[i] = orig
            }
        }
        frontSet = nextSet
        steps++
    }
    return 0
}

func main() {
    fmt.Println(ladderLengthBidir("hit", "cog",
        []string{"hot", "dot", "dog", "lot", "log", "cog"})) // 5
}
```

**Why?** Bidirectional BFS explores from both ends. Search space: O(b^(d/2)) vs O(b^d), where b = branching factor, d = depth.

---

## Example 7: 0-1 BFS — Minimum Cost to Make At Least One Valid Path (LeetCode 1368)

0-1 BFS uses a deque: 0-cost edges push to front, 1-cost edges push to back.

```go
package main

import "fmt"

func minCost(grid [][]int) int {
    rows, cols := len(grid), len(grid[0])
    dist := make([][]int, rows)
    for i := range dist {
        dist[i] = make([]int, cols)
        for j := range dist[i] {
            dist[i][j] = 1<<31 - 1
        }
    }
    dist[0][0] = 0

    // Direction mapping: 1=right, 2=left, 3=down, 4=up
    dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}
    dirMap := map[int]int{1: 0, 2: 1, 3: 2, 4: 3}

    type Cell struct{ R, C int }
    deque := []Cell{{0, 0}}

    for len(deque) > 0 {
        cur := deque[0]
        deque = deque[1:]

        for i, d := range dirs {
            nr, nc := cur.R+d[0], cur.C+d[1]
            if nr < 0 || nr >= rows || nc < 0 || nc >= cols {
                continue
            }

            cost := 0
            if dirMap[grid[cur.R][cur.C]] != i {
                cost = 1 // need to change sign
            }

            newDist := dist[cur.R][cur.C] + cost
            if newDist < dist[nr][nc] {
                dist[nr][nc] = newDist
                if cost == 0 {
                    deque = append([]Cell{{nr, nc}}, deque...) // push front
                } else {
                    deque = append(deque, Cell{nr, nc}) // push back
                }
            }
        }
    }

    return dist[rows-1][cols-1]
}

func main() {
    grid := [][]int{
        {1, 1, 1, 1},
        {2, 2, 2, 2},
        {1, 1, 1, 1},
        {2, 2, 2, 2},
    }
    fmt.Println("Min cost:", minCost(grid)) // 3

    grid2 := [][]int{
        {1, 1, 3},
        {3, 2, 2},
        {1, 1, 4},
    }
    fmt.Println("Min cost:", minCost(grid2)) // 0
}
```

---

## Example 8: BFS with State — Open the Lock (LeetCode 752)

```go
package main

import "fmt"

func openLock(deadends []string, target string) int {
    dead := map[string]bool{}
    for _, d := range deadends {
        dead[d] = true
    }

    start := "0000"
    if dead[start] {
        return -1
    }
    if start == target {
        return 0
    }

    visited := map[string]bool{start: true}
    queue := []string{start}
    steps := 0

    for len(queue) > 0 {
        size := len(queue)
        steps++
        for i := 0; i < size; i++ {
            curr := queue[0]
            queue = queue[1:]

            // Try turning each of 4 wheels up or down
            for j := 0; j < 4; j++ {
                for _, delta := range []int{1, -1} {
                    chars := []byte(curr)
                    chars[j] = byte((int(chars[j]-'0')+delta+10)%10) + '0'
                    next := string(chars)

                    if next == target {
                        return steps
                    }
                    if !dead[next] && !visited[next] {
                        visited[next] = true
                        queue = append(queue, next)
                    }
                }
            }
        }
    }
    return -1
}

func main() {
    fmt.Println(openLock([]string{"0201", "0101", "0102", "1212", "2002"}, "0202")) // 6
    fmt.Println(openLock([]string{"8888"}, "0009"))                                   // 1
    fmt.Println(openLock([]string{"8887", "8889", "8878", "8898", "8788",
        "8988", "7888", "9888"}, "8888")) // -1
}
```

---

## Example 9: Multi-Source BFS — Walls and Gates (LeetCode 286)

```go
package main

import "fmt"

const (
    WALL  = -1
    GATE  = 0
    EMPTY = 1<<31 - 1
)

func wallsAndGates(rooms [][]int) {
    if len(rooms) == 0 {
        return
    }
    rows, cols := len(rooms), len(rooms[0])
    type Pos struct{ R, C int }
    queue := []Pos{}

    // Enqueue all gates
    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if rooms[r][c] == GATE {
                queue = append(queue, Pos{r, c})
            }
        }
    }

    dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

    for len(queue) > 0 {
        pos := queue[0]
        queue = queue[1:]

        for _, d := range dirs {
            nr, nc := pos.R+d[0], pos.C+d[1]
            if nr >= 0 && nr < rows && nc >= 0 && nc < cols && rooms[nr][nc] == EMPTY {
                rooms[nr][nc] = rooms[pos.R][pos.C] + 1
                queue = append(queue, Pos{nr, nc})
            }
        }
    }
}

func main() {
    rooms := [][]int{
        {EMPTY, WALL, GATE, EMPTY},
        {EMPTY, EMPTY, EMPTY, WALL},
        {EMPTY, WALL, EMPTY, WALL},
        {GATE, WALL, EMPTY, EMPTY},
    }

    wallsAndGates(rooms)

    for _, row := range rooms {
        for _, cell := range row {
            if cell == WALL {
                fmt.Printf("  W")
            } else {
                fmt.Printf("%3d", cell)
            }
        }
        fmt.Println()
    }
    // 3  W  0  1
    // 2  2  1  W
    // 1  W  2  W
    // 0  W  3  4
}
```

---

## Example 10: BFS Topological Sort — Course Schedule (LeetCode 207 / Kahn's Algorithm)

```go
package main

import "fmt"

func canFinish(numCourses int, prerequisites [][]int) bool {
    inDegree := make([]int, numCourses)
    graph := make([][]int, numCourses)

    for _, p := range prerequisites {
        graph[p[1]] = append(graph[p[1]], p[0])
        inDegree[p[0]]++
    }

    // Enqueue all nodes with in-degree 0
    queue := []int{}
    for i, d := range inDegree {
        if d == 0 {
            queue = append(queue, i)
        }
    }

    processed := 0
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        processed++

        for _, neighbor := range graph[node] {
            inDegree[neighbor]--
            if inDegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }

    return processed == numCourses
}

func findOrder(numCourses int, prerequisites [][]int) []int {
    inDegree := make([]int, numCourses)
    graph := make([][]int, numCourses)

    for _, p := range prerequisites {
        graph[p[1]] = append(graph[p[1]], p[0])
        inDegree[p[0]]++
    }

    queue := []int{}
    for i, d := range inDegree {
        if d == 0 {
            queue = append(queue, i)
        }
    }

    order := []int{}
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        order = append(order, node)

        for _, neighbor := range graph[node] {
            inDegree[neighbor]--
            if inDegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }

    if len(order) != numCourses {
        return nil // cycle exists
    }
    return order
}

func main() {
    fmt.Println("Can finish:", canFinish(4, [][]int{{1, 0}, {2, 0}, {3, 1}, {3, 2}})) // true
    fmt.Println("Can finish:", canFinish(2, [][]int{{1, 0}, {0, 1}}))                   // false (cycle)

    fmt.Println("Order:", findOrder(4, [][]int{{1, 0}, {2, 0}, {3, 1}, {3, 2}})) // [0 1 2 3] or [0 2 1 3]
}
```

---

## Example 11: BFS Shortest Path Reconstruction

```go
package main

import "fmt"

func shortestPathWithRoute(graph [][]int, src, dst int) (int, []int) {
    n := len(graph)
    visited := make([]bool, n)
    parent := make([]int, n)
    for i := range parent {
        parent[i] = -1
    }
    visited[src] = true

    queue := []int{src}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if node == dst {
            // Reconstruct path
            path := []int{}
            for cur := dst; cur != -1; cur = parent[cur] {
                path = append(path, cur)
            }
            // Reverse
            for i, j := 0, len(path)-1; i < j; i, j = i+1, j-1 {
                path[i], path[j] = path[j], path[i]
            }
            return len(path) - 1, path
        }

        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                visited[neighbor] = true
                parent[neighbor] = node
                queue = append(queue, neighbor)
            }
        }
    }
    return -1, nil
}

func main() {
    graph := [][]int{
        {1, 2},    // 0
        {0, 3, 4}, // 1
        {0, 4},    // 2
        {1, 5},    // 3
        {1, 2, 5}, // 4
        {3, 4},    // 5
    }

    dist, path := shortestPathWithRoute(graph, 0, 5)
    fmt.Printf("0→5: distance=%d, path=%v\n", dist, path)
    // distance=3, path=[0 1 3 5] or [0 1 4 5]
}
```

---

## BFS Pattern Summary

| Pattern | Queue Init | Key Feature |
|---------|-----------|-------------|
| Standard BFS | Single source | Shortest unweighted path |
| Multi-source BFS | All sources at once | Simultaneous spread |
| Level-by-level | Process `size` nodes per round | Level information |
| Bidirectional | Two frontiers | Halves search depth |
| 0-1 BFS | Deque (front/back) | Two edge weights (0/1) |
| Topological (Kahn's) | In-degree 0 nodes | DAG ordering |
| State BFS | Encode state as key | Complex state spaces |

## Key Takeaways

1. BFS guarantees the **shortest path** in unweighted graphs (or graphs with equal edge weights)
2. **Level-by-level** processing: capture `size := len(queue)` before inner loop
3. **Multi-source BFS**: enqueue ALL sources first → they expand in parallel
4. **Mark visited BEFORE enqueueing** (not when dequeuing) to avoid duplicates
5. **0-1 BFS** uses a deque: 0-cost → push front, 1-cost → push back → O(V+E)
6. **Bidirectional BFS** reduces search space exponentially — expand the smaller frontier
7. **Kahn's algorithm** is BFS-based topological sort — detects cycles when count ≠ n
8. BFS on grids: use `[][2]int` directions array for clean neighbor iteration

> **Phase 7 Complete!** Next up: Phase 8 — Binary Trees →
