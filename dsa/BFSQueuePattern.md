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

**Textual Figure:**
```
         3          Level 0:  Queue: [3]
        / \                   Process 3 → enqueue 9, 20
       9   20        Level 1:  Queue: [9, 20]
          / \                  Process 9 → no children
         15  7                 Process 20 → enqueue 15, 7
                     Level 2:  Queue: [15, 7]
                               Process 15, 7 → no children

BFS Queue Trace:
┌─────────┬─────────────────┬───────────────┐
│  Step   │  Queue           │  Output        │
├─────────┼─────────────────┼───────────────┤
│  Init   │  [3]             │               │
│  Lvl 0  │  [] → [9,20]     │  [[3]]        │
│  Lvl 1  │  [] → [15,7]     │  [[3],[9,20]] │
│  Lvl 2  │  []              │  +[15,7]      │
└─────────┴─────────────────┴───────────────┘

Key: Process ALL nodes at current level before moving to next.
     levelSize = len(queue) at start of each level.
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

**Textual Figure:**
```
Graph:              0 ─── 1 ─── 3
                    |             |
                    2 ─── 4 ─── 5

BFS from 0 to 5:
┌───────┬─────────────────┬─────────────┐
│ Dist  │ Queue            │ Visited      │
├───────┼─────────────────┼─────────────┤
│   0   │ [(0)]            │ {0}          │
│   1   │ [(1),(2)]         │ {0,1,2}      │
│   2   │ [(3),(4)]         │ {0,1,2,3,4}  │
│   3   │ [(5)] ← found!   │ {0..5}       │
└───────┴─────────────────┴─────────────┘

Path: 0 → 2 → 4 → 5  (distance = 3)  ✓
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
**Textual Figure:**
```
Grid evolution (2=rotten, 1=fresh, 0=empty):

Minute 0 (initial):    Minute 1:         Minute 2:
┌───┬───┬───┐        ┌───┬───┬───┐     ┌───┬───┬───┐
│ 2 │ 1 │ 1 │        │ 2 │ 2 │ 1 │     │ 2 │ 2 │ 2 │
├───┼───┼───┤        ├───┼───┼───┤     ├───┼───┼───┤
│ 1 │ 1 │ 0 │        │ 2 │ 1 │ 0 │     │ 2 │ 2 │ 0 │
├───┼───┼───┤        ├───┼───┼───┤     ├───┼───┼───┤
│ 0 │ 1 │ 1 │        │ 0 │ 1 │ 1 │     │ 0 │ 1 │ 1 │
└───┴───┴───┘        └───┴───┴───┘     └───┴───┴───┘

Minute 3:             Minute 4:
┌───┬───┬───┐        ┌───┬───┬───┐
│ 2 │ 2 │ 2 │        │ 2 │ 2 │ 2 │     All rotten!
├───┼───┼───┤        ├───┼───┼───┤
│ 2 │ 2 │ 0 │        │ 2 │ 2 │ 0 │
├───┼───┼───┤        ├───┼───┼───┤
│ 0 │ 2 │ 1 │        │ 0 │ 2 │ 2 │
└───┴───┴───┘        └───┴───┴───┘

Multi-source BFS: start with ALL rotten in queue
Queue: [(0,0)] → spread to neighbors each minute
Answer = 4 minutes  ✓
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

**Textual Figure:**
```
Grid (0=passable, 1=blocked):     BFS 8-directional:
┌───┬───┬───┐                     ┌───┬───┬───┐
│ 0 │ 0 │ 0 │                     │ 1 │ 2 │ 3 │
├───┼───┼───┤                     ├───┼───┼───┤
│ 1 │ 1 │ 0 │                     │ █ │ █ │ 3 │
├───┼───┼───┤                     ├───┼───┼───┤
│ 1 │ 1 │ 0 │                     │ █ │ █ │ 4 │  ← goal
└───┴───┴───┘                     └───┴───┴───┘

Path: (0,0)→(0,1)→(0,2)→(1,2)→(2,2)
       1      2      3      3      4

8 directions: ← → ↑ ↓ ↖ ↗ ↙ ↘
Answer = 4  ✓
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

**Textual Figure:**
```
Word Ladder: "hit" → "cog" (change 1 letter at a time)

  Level 0:  hit
             │
  Level 1:  hot     (h→h, i→o, t→t)
            / \
  Level 2: dot  lot  (h→d/l)
            |    |
  Level 3: dog  log  (t→g)
             \ /
  Level 4:  cog     (d/l→c)

BFS Queue Trace:
┌───────┬──────────────────────┬─────────────┐
│ Step  │ Queue                 │ Visited      │
├───────┼──────────────────────┼─────────────┤
│   1   │ [hit]                 │ {hit}        │
│   2   │ [hot]                 │ +{hot}       │
│   3   │ [dot, lot]            │ +{dot,lot}   │
│   4   │ [dog, log]            │ +{dog,log}   │
│   5   │ [cog] ← found!       │ +{cog}       │
└───────┴──────────────────────┴─────────────┘

Transformation length = 5 (counting both start and end words)  ✓
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

**Textual Figure:**
```
Bidirectional BFS: search from BOTH ends, meet in the middle.

  Forward set (from "hit"):     Backward set (from "cog"):
  ────────────────────────       ────────────────────────
  Step 1: {hit}                  Step 1: {cog}
  Step 2: {hot}                  Step 2: {dog, log}
  Step 3: {dot, lot}             
           ↓   ↓                    ↑    ↑
        MEET: "dot" ∈ backSet or "lot" ∈ backSet?
              Not yet → swap (always expand smaller set)

  After swap, front={dog,log}, back={dot,lot}
  Step 3: {dog,log} → try transforms → "dot" ∈ backSet!
  Found! steps = 2 + 2 + 1 = 5

Search space comparison:
┌───────────────────┬────────────────────┐
│  Standard BFS       │  Bidirectional BFS    │
├───────────────────┼────────────────────┤
│  O(b^d)              │  O(b^(d/2))           │
│  One big circle      │  Two small circles    │
└───────────────────┴────────────────────┘
  hit○────────○ cog      hit○─○─○─○cog
      (expand all)            ↑     ↑
                            meet in middle!
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

**Textual Figure:**
```
0-1 BFS uses a DEQUE instead of a queue:
  - Cost 0 edge → push to FRONT
  - Cost 1 edge → push to BACK

Grid (arrows = signs):        Distance grid:
┌───┬───┬───┬───┐             ┌───┬───┬───┬───┐
│ → │ → │ → │ → │             │ 0 │ 0 │ 0 │ 1 │
├───┼───┼───┼───┤             ├───┼───┼───┼───┤
│ ← │ ← │ ← │ ← │             │ 1 │ 1 │ 1 │ 1 │
├───┼───┼───┼───┤             ├───┼───┼───┼───┤
│ → │ → │ → │ → │             │ 2 │ 2 │ 2 │ 2 │
├───┼───┼───┼───┤             ├───┼───┼───┼───┤
│ ← │ ← │ ← │ ← │             │ 3 │ 3 │ 3 │ 3 │
└───┴───┴───┴───┘             └───┴───┴───┴───┘

Deque behavior:
  Follow sign → cost 0 → push FRONT (like free move)
  Change sign → cost 1 → push BACK  (like paying)

  Deque: [front] ── 0-cost ── ... ── 1-cost ── [back]

Min cost to reach (3,3) = 3  ✓
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

**Textual Figure:**
```
Lock: 4 wheels, each 0-9 (wraps around: 9→0, 0→9)

Target: "0202",  Deadends: {0201, 0101, 0102, 1212, 2002}

BFS on state space (each state = 4-digit string):

  Step 0:  "0000"
            │
  Step 1:  "1000" "9000" "0100" "0900" "0010" "0090" "0001" "0009"
            │                                          │
  Step 2:  "1100" "1001" ...                  "0002" "0009"→"0019"...
            │                                   │
  Step 3:  ...                              "0012" "0003"...
            │
  ...continues...
            │
  Step 6:  "0202" ← found!

Each state has 8 neighbors (4 wheels × 2 directions).
Avoid deadends: skip {0201, 0101, 0102, 1212, 2002}

  State space:  10^4 = 10,000 possible states
  BFS guarantees shortest path = 6 turns  ✓
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

**Textual Figure:**
```
Multi-source BFS from ALL gates simultaneously:

Initial (W=wall, G=gate, ∞=empty):     After BFS:
┌───┬───┬───┬───┐                       ┌───┬───┬───┬───┐
│ ∞ │ W │ G │ ∞ │                       │ 3 │ W │ 0 │ 1 │
├───┼───┼───┼───┤                       ├───┼───┼───┼───┤
│ ∞ │ ∞ │ ∞ │ W │                       │ 2 │ 2 │ 1 │ W │
├───┼───┼───┼───┤                       ├───┼───┼───┼───┤
│ ∞ │ W │ ∞ │ W │                       │ 1 │ W │ 2 │ W │
├───┼───┼───┼───┤                       ├───┼───┼───┼───┤
│ G │ W │ ∞ │ ∞ │                       │ 0 │ W │ 3 │ 4 │
└───┴───┴───┴───┘                       └───┴───┴───┴───┘

BFS levels (start from gates at (0,2) and (3,0)):
  Level 0: Queue = [(0,2), (3,0)]        → gates = 0
  Level 1: Queue = [(0,3), (1,2), (2,0)] → dist = 1
  Level 2: Queue = [(1,1), (1,0)]        → dist = 2
  Level 3: Queue = [(0,0), (2,2)]        → dist = 3
  Level 4: Queue = [(3,3)]               → dist = 4
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

**Textual Figure:**
```
Kahn's Algorithm (BFS-based topological sort):

Prerequisites: [[1,0],[2,0],[3,1],[3,2]]
  0 → 1, 0 → 2, 1 → 3, 2 → 3

Graph:        In-degree:
  0 ──→ 1     0: 0  ← start
  │     │     1: 1
  └─→ 2 │     2: 1
       │ │     3: 2
       └─┴→ 3

BFS Trace:
┌───────┬───────┬─────────────┬──────────────┐
│ Step  │ Queue │  In-degrees   |  Order        │
├───────┼───────┼─────────────┼──────────────┤
│ Init  │ [0]   │ [0,1,1,2]     │ []            │
│  1    │ [1,2] │ [_,0,0,2]     │ [0]           │
│  2    │ [2]   │ [_,_,0,1]     │ [0,1]         │
│  3    │ [3]   │ [_,_,_,0]     │ [0,1,2]       │
│  4    │ []    │ all done      │ [0,1,2,3]  ✓  │
└───────┴───────┴─────────────┴──────────────┘

len(order)==numCourses → no cycle → can finish!  ✓
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

**Textual Figure:**
```
Graph:       0 ─── 1 ─── 3
             |   |       |
             2 ── 4 ─── 5

BFS from 0 to 5 (tracking parent[]):

┌──────┬─────────────┬─────────────┬──────────────────────┐
│ Step │ Process     │ Queue       │ parent[]              │
├──────┼─────────────┼─────────────┼──────────────────────┤
│ Init │ -           │ [0]         │ [-1,-1,-1,-1,-1,-1]   │
│  1   │ node=0      │ [1,2]       │ [-1, 0, 0,-1,-1,-1]   │
│  2   │ node=1      │ [2,3,4]     │ [-1, 0, 0, 1, 1,-1]   │
│  3   │ node=2      │ [3,4]       │ (4 already visited)   │
│  4   │ node=3      │ [4,5]       │ [-1, 0, 0, 1, 1, 3]   │
│  5   │ node=5 ✓    │ found!      │                       │
└──────┴─────────────┴─────────────┴──────────────────────┘

Path reconstruction (follow parent[] backwards):
  5 → parent[5]=3 → parent[3]=1 → parent[1]=0 → parent[0]=-1 (stop)
  Reverse: [0, 1, 3, 5]

  0 ──▶ 1 ──▶ 3 ──▶ 5
  distance = 3  ✓
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
