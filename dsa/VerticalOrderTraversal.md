# Phase 8: Binary Trees — Vertical Order Traversal

## Overview

**Vertical order traversal** groups tree nodes by their **column** (x-coordinate). The root is at column 0, left child at column-1, right child at column+1.

```
        1 (col=0)
       / \
      2   3 (col=-1, col=1)
     / \ / \
    4  5 6  7 (col=-2, -0, 0, 2)
              ↑ nodes 5,6 share col=0
```

Vertical lines: col=-2: [4], col=-1: [2], col=0: [1,5,6], col=1: [3], col=2: [7]

---

## Example 1: Vertical Order Traversal (LeetCode 314)

BFS-based — nodes at same row+column appear left-to-right.

```go
package main

import (
    "fmt"
    "sort"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func verticalOrder(root *TreeNode) [][]int {
    if root == nil { return nil }

    type Item struct {
        Node *TreeNode
        Col  int
    }

    colMap := map[int][]int{}
    minCol, maxCol := 0, 0
    queue := []Item{{root, 0}}

    for len(queue) > 0 {
        item := queue[0]
        queue = queue[1:]

        colMap[item.Col] = append(colMap[item.Col], item.Node.Val)
        if item.Col < minCol { minCol = item.Col }
        if item.Col > maxCol { maxCol = item.Col }

        if item.Node.Left != nil {
            queue = append(queue, Item{item.Node.Left, item.Col - 1})
        }
        if item.Node.Right != nil {
            queue = append(queue, Item{item.Node.Right, item.Col + 1})
        }
    }

    result := [][]int{}
    for col := minCol; col <= maxCol; col++ {
        result = append(result, colMap[col])
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
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    for i, col := range verticalOrder(root) {
        fmt.Printf("Column %d: %v\n", i, col)
    }
    // Column 0: [9]
    // Column 1: [3 15]
    // Column 2: [20]
    // Column 3: [7]
}
```

**Textual Figure – Example 1:**
```
 Vertical Order Traversal (BFS):
 ──────────────────────────────
  col: -1   0   1   2
        │   │   │   │
        │   3   │   │      BFS with column tracking:
        │  / \  │   │      root(col=0) → L(col-1), R(col+1)
        9   │  20  │
   (col=-1) │ / \ │
            15  │  7
       (col=0) (col=2)

 ┌────────┬───────┬────────────────┐
 │ Column │ Nodes │ Output          │
 ├────────┼───────┼────────────────┤
 │   -1   │  [9]  │ Column 0: [9]   │
 │    0   │ [3,15]│ Column 1: [3,15]│
 │    1   │ [20]  │ Column 2: [20]  │
 │    2   │  [7]  │ Column 3: [7]   │
 └────────┴───────┴────────────────┘
 BFS ensures nodes at same column appear
 top-to-bottom, left-to-right.
```

---

## Example 2: Vertical Order with Sorting (LeetCode 987)

When two nodes share the same (row, col), sort them by value.

```go
package main

import (
    "fmt"
    "sort"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type Entry struct {
    Row, Col, Val int
}

func verticalTraversal(root *TreeNode) [][]int {
    if root == nil { return nil }

    entries := []Entry{}
    var dfs func(node *TreeNode, row, col int)
    dfs = func(node *TreeNode, row, col int) {
        if node == nil { return }
        entries = append(entries, Entry{row, col, node.Val})
        dfs(node.Left, row+1, col-1)
        dfs(node.Right, row+1, col+1)
    }
    dfs(root, 0, 0)

    // Sort by col, then row, then value
    sort.Slice(entries, func(i, j int) bool {
        if entries[i].Col != entries[j].Col { return entries[i].Col < entries[j].Col }
        if entries[i].Row != entries[j].Row { return entries[i].Row < entries[j].Row }
        return entries[i].Val < entries[j].Val
    })

    result := [][]int{}
    prevCol := entries[0].Col - 1
    for _, e := range entries {
        if e.Col != prevCol {
            result = append(result, []int{})
            prevCol = e.Col
        }
        result[len(result)-1] = append(result[len(result)-1], e.Val)
    }
    return result
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \ / \
    //   4  6 5  7
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{6, nil, nil}},
        &TreeNode{3, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(verticalTraversal(root))
    // [[-2:[4]], [-1:[2]], [0:[1,5,6]], [1:[3]], [2:[7]]]
}
```

**Textual Figure – Example 2:**
```
 Vertical Order with Sorting (LeetCode 987):
 ─────────────────────────────────
 col: -2  -1   0   1   2
      │   │   │   │   │
      │   │   1   │   │    row=0
      │   │  / \  │   │
      │   2   │   3   │    row=1
      │  / \  │  / \  │
      4   6   │  5   7       row=2
  (r2,c-2)    │
         nodes 6 and 5 share col=0, row=2
         → sort by value: [5, 6]

 Sort order: (col, row, val)
 ┌──────┬─────┬──────┬─────────────────┐
 │ col  │ row │ vals │ group           │
 ├──────┼─────┼──────┼─────────────────┤
 │  -2  │  2  │  4   │ [4]             │
 │  -1  │  1  │  2   │ [2]             │
 │   0  │  0  │  1   │ [1, 5, 6]       │
 │   0  │  2  │ 5, 6 │ (sorted by val) │
 │   1  │  1  │  3   │ [3]             │
 │   2  │  2  │  7   │ [7]             │
 └──────┴─────┴──────┴─────────────────┘
```

---

## Example 3: Vertical Sum

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func verticalSum(root *TreeNode) map[int]int {
    sums := map[int]int{}
    var dfs func(node *TreeNode, col int)
    dfs = func(node *TreeNode, col int) {
        if node == nil { return }
        sums[col] += node.Val
        dfs(node.Left, col-1)
        dfs(node.Right, col+1)
    }
    dfs(root, 0)
    return sums
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \ / \
    //   4  5 6  7
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    sums := verticalSum(root)

    // Find range
    minC, maxC := 0, 0
    for c := range sums {
        if c < minC { minC = c }
        if c > maxC { maxC = c }
    }
    for c := minC; c <= maxC; c++ {
        fmt.Printf("Col %d: sum=%d\n", c, sums[c])
    }
    // Col -2: sum=4
    // Col -1: sum=2
    // Col  0: sum=1+5+6=12
    // Col  1: sum=3
    // Col  2: sum=7
}
```

**Textual Figure – Example 3:**
```
 Vertical Column Sums:
 ────────────────────
 col: -2  -1   0   1   2
      │   │   │   │   │
      │   │   1   │   │
      │   │  / \  │   │
      │   2   │   3   │
      │  / \  │  / \  │
      4   5   │  6   7

 DFS tracks col for each node:
 ┌────────┬──────────────┬─────────┐
 │ Column │ Nodes        │ Sum     │
 ├────────┼──────────────┼─────────┤
 │   -2   │ 4            │  4      │
 │   -1   │ 2            │  2      │
 │    0   │ 1, 5, 6      │ 12      │
 │    1   │ 3            │  3      │
 │    2   │ 7            │  7      │
 └────────┴──────────────┴─────────┘
 Total = 4+2+12+3+7 = 28 (sum of all nodes)
```

---

## Example 4: Top View of Binary Tree

First node seen at each column when viewed from top.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func topView(root *TreeNode) []int {
    if root == nil { return nil }

    type Item struct {
        Node *TreeNode
        Col  int
    }

    colMap := map[int]int{} // col → first node val
    minCol, maxCol := 0, 0
    queue := []Item{{root, 0}}

    for len(queue) > 0 {
        item := queue[0]
        queue = queue[1:]

        if _, exists := colMap[item.Col]; !exists {
            colMap[item.Col] = item.Node.Val
        }
        if item.Col < minCol { minCol = item.Col }
        if item.Col > maxCol { maxCol = item.Col }

        if item.Node.Left != nil {
            queue = append(queue, Item{item.Node.Left, item.Col - 1})
        }
        if item.Node.Right != nil {
            queue = append(queue, Item{item.Node.Right, item.Col + 1})
        }
    }

    result := []int{}
    for c := minCol; c <= maxCol; c++ {
        result = append(result, colMap[c])
    }
    return result
}

func main() {
    //        1
    //       / \
    //      2   3
    //     / \   \
    //    4   5   6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }
    fmt.Println("Top view:", topView(root)) // [4 2 1 3 6]
}
```

**Textual Figure – Example 4:**
```
 Top View — First node seen at each column:
 ──────────────────────────────
  col: -2  -1   0   1   2
        │   │   │   │   │
        │   │   1   │   │    ← first at col 0
        │   │  / \  │   │
        │   2   │   3   │    ← first at col -1, 1
        │  / \  │    \  │
        4   5   │    6       ← first at col -2, 2

 BFS processes level by level:
 ┌────────┬───────────────┬─────────────────────┐
 │ Column │ First node    │ Already seen?       │
 ├────────┼───────────────┼─────────────────────┤
 │   -2   │  4 (row 2)    │ No → record 4      │
 │   -1   │  2 (row 1)    │ No → record 2      │
 │    0   │  1 (row 0)    │ No → record 1      │
 │    0   │  5 (row 2)    │ Yes → skip (hidden) │
 │    1   │  3 (row 1)    │ No → record 3      │
 │    2   │  6 (row 2)    │ No → record 6      │
 └────────┴───────────────┴─────────────────────┘

 Top view (left to right): [4, 2, 1, 3, 6]
 Node 5 hidden behind 1 at col 0.
```

---

## Example 5: Bottom View of Binary Tree

Last node seen at each column.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func bottomView(root *TreeNode) []int {
    if root == nil { return nil }

    type Item struct {
        Node *TreeNode
        Col  int
    }

    colMap := map[int]int{} // col → last node val
    minCol, maxCol := 0, 0
    queue := []Item{{root, 0}}

    for len(queue) > 0 {
        item := queue[0]
        queue = queue[1:]

        colMap[item.Col] = item.Node.Val // overwrite = keep last
        if item.Col < minCol { minCol = item.Col }
        if item.Col > maxCol { maxCol = item.Col }

        if item.Node.Left != nil {
            queue = append(queue, Item{item.Node.Left, item.Col - 1})
        }
        if item.Node.Right != nil {
            queue = append(queue, Item{item.Node.Right, item.Col + 1})
        }
    }

    result := []int{}
    for c := minCol; c <= maxCol; c++ {
        result = append(result, colMap[c])
    }
    return result
}

func main() {
    //        20
    //       /  \
    //      8   22
    //     / \    \
    //    5  3    25
    //      / \
    //     10  14
    root := &TreeNode{20,
        &TreeNode{8,
            &TreeNode{5, nil, nil},
            &TreeNode{3, &TreeNode{10, nil, nil}, &TreeNode{14, nil, nil}},
        },
        &TreeNode{22, nil, &TreeNode{25, nil, nil}},
    }
    fmt.Println("Bottom view:", bottomView(root)) // [5 10 3 14 22 25]
}
```

**Textual Figure – Example 5:**
```
 Bottom View — Last node seen at each column:
 ──────────────────────────────────
        20 (col 0)
       /  \
      8   22 (col 1)
     / \    \
    5  3    25 (col 2)
      / \
     10  14

 col: -2  -1   0   1   2
      │   │   │   │   │
      5   │  20   22  │
      │   8   │   │   25
      │  │ \  3   │
      │  │  │ / \  │
      │  │ 10  14  │

 BFS overwrites (keeps last per column):
 ┌────────┬────────────────┬─────────────┐
 │ Column │ Last node      │ Value       │
 ├────────┼────────────────┼─────────────┤
 │   -2   │ 5 → 5         │  5          │
 │   -1   │ 8 → 10        │ 10          │
 │    0   │ 20 → 3        │  3          │
 │    1   │ 22 → 14       │ 14          │
 │    2   │ 25            │ 22 (no ow.) │
 │    3   │ 25            │ 25          │
 └────────┴────────────────┴─────────────┘
 Bottom view: [5, 10, 3, 14, 22, 25]
```

---

## Example 6: Diagonal Traversal

Diagonal traversal: nodes at same (col - row) or same diagonal.

```go
package main

import (
    "fmt"
    "sort"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func diagonalTraversal(root *TreeNode) [][]int {
    if root == nil { return nil }

    diagMap := map[int][]int{}
    var dfs func(node *TreeNode, diag int)
    dfs = func(node *TreeNode, diag int) {
        if node == nil { return }
        diagMap[diag] = append(diagMap[diag], node.Val)
        dfs(node.Left, diag+1)  // going left increases diagonal
        dfs(node.Right, diag)   // going right stays on same diagonal
    }
    dfs(root, 0)

    // Sort diagonals
    keys := []int{}
    for k := range diagMap { keys = append(keys, k) }
    sort.Ints(keys)

    result := [][]int{}
    for _, k := range keys {
        result = append(result, diagMap[k])
    }
    return result
}

func main() {
    //       8
    //      / \
    //     3  10
    //    / \   \
    //   1  6   14
    //     / \  /
    //    4  7 13
    root := &TreeNode{8,
        &TreeNode{3,
            &TreeNode{1, nil, nil},
            &TreeNode{6, &TreeNode{4, nil, nil}, &TreeNode{7, nil, nil}},
        },
        &TreeNode{10, nil,
            &TreeNode{14, &TreeNode{13, nil, nil}, nil},
        },
    }
    diags := diagonalTraversal(root)
    for i, d := range diags {
        fmt.Printf("Diagonal %d: %v\n", i, d)
    }
    // Diagonal 0: [8 10 14]
    // Diagonal 1: [3 6 7 13]
    // Diagonal 2: [1 4]
}
```

**Textual Figure – Example 6:**
```
 Diagonal Traversal:
 ──────────────────
        8                  Going right: same diagonal
       / \                 Going left:  diagonal + 1
      3  10
     / \   \              ╱╱╱ Diagonal 0: 8 → 10 → 14
    1   6  14             ╲╲╲ Diagonal 1: 3 → 6 → 7 → 13
       / \  /                  Diagonal 2: 1 → 4
      4  7 13

 Diagonal lines visualization:
  d=0: 8 ──────→ 10 ───→ 14
        \                \
  d=1:   3 ────→ 6 ──→ 7    13
          \       \
  d=2:     1       4

 ┌───────────┬────────────────┐
 │ Diagonal  │ Nodes          │
 ├───────────┼────────────────┤
 │    0      │ [8, 10, 14]    │
 │    1      │ [3, 6, 7, 13]  │
 │    2      │ [1, 4]         │
 └───────────┴────────────────┘
```

---

## Example 7: Column Width (Nodes per Column)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func columnWidths(root *TreeNode) (map[int]int, int, int) {
    widths := map[int]int{} // col → count
    minCol, maxCol := 0, 0

    var dfs func(node *TreeNode, col int)
    dfs = func(node *TreeNode, col int) {
        if node == nil { return }
        widths[col]++
        if col < minCol { minCol = col }
        if col > maxCol { maxCol = col }
        dfs(node.Left, col-1)
        dfs(node.Right, col+1)
    }
    dfs(root, 0)
    return widths, minCol, maxCol
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \ / \
    //   4  5 6  7
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    widths, minC, maxC := columnWidths(root)
    for c := minC; c <= maxC; c++ {
        fmt.Printf("Col %d: %d nodes\n", c, widths[c])
    }
    // Col -2: 1 node  (4)
    // Col -1: 1 node  (2)
    // Col  0: 3 nodes (1,5,6)
    // Col  1: 1 node  (3)
    // Col  2: 1 node  (7)
}
```

**Textual Figure – Example 7:**
```
 Column Width (Nodes per Column):
 ──────────────────────────
 col: -2  -1   0   1   2
      │   │   │   │   │
      │   │  [1]  │   │
      │   │  / \  │   │
      │  [2]  │  [3]  │
      │  / \  │  / \  │
     [4] [5]  │ [6] [7]

 ┌────────┬────────┬─────────────┐
 │ Column │ Count  │ Nodes       │
 ├────────┼────────┼─────────────┤
 │   -2   │   1    │ 4           │
 │   -1   │   1    │ 2           │
 │    0   │   3    │ 1, 5, 6     │
 │    1   │   1    │ 3           │
 │    2   │   1    │ 7           │
 └────────┴────────┴─────────────┘
 Col 0 is widest with 3 nodes.
```

---

## Example 8: Print Tree Column by Column

```go
package main

import (
    "fmt"
    "sort"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type NodeInfo struct {
    Val, Row int
}

func printVertical(root *TreeNode) {
    if root == nil { return }

    cols := map[int][]NodeInfo{}
    var dfs func(node *TreeNode, row, col int)
    dfs = func(node *TreeNode, row, col int) {
        if node == nil { return }
        cols[col] = append(cols[col], NodeInfo{node.Val, row})
        dfs(node.Left, row+1, col-1)
        dfs(node.Right, row+1, col+1)
    }
    dfs(root, 0, 0)

    keys := []int{}
    for k := range cols { keys = append(keys, k) }
    sort.Ints(keys)

    maxRow := 0
    for _, nodes := range cols {
        for _, n := range nodes {
            if n.Row > maxRow { maxRow = n.Row }
        }
    }

    for _, col := range keys {
        nodes := cols[col]
        sort.Slice(nodes, func(i, j int) bool { return nodes[i].Row < nodes[j].Row })
        vals := []string{}
        for _, n := range nodes {
            vals = append(vals, fmt.Sprint(n.Val))
        }
        fmt.Printf("Col %2d: %s\n", col, strings.Join(vals, " → "))
    }
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    printVertical(root)
}
```

**Textual Figure – Example 8:**
```
 Print Tree Column by Column:
 ────────────────────────
       1                     DFS collects (val, row)
      / \                    per column, then sorts by row.
     2   3
    / \ / \
   4  5 6  7              Output:
                           Col -2: 4
 col: -2  -1   0   1   2   Col -1: 2
       │   │   │   │   │   Col  0: 1 → 5 → 6
       │   │  r0   │   │   Col  1: 3
       │  r1   │  r1   │   Col  2: 7
      r2  r2   │  r2  r2
               │            Nodes at same col sorted by row,
              r2            then by value if same (col, row).
         (5 and 6)
```

---

## Example 9: Vertical Order Traversal – Optimized with TreeMap-like Approach

```go
package main

import (
    "fmt"
    "sort"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func verticalTraversalOptimized(root *TreeNode) [][]int {
    if root == nil { return nil }

    type Item struct {
        Node     *TreeNode
        Row, Col int
    }

    // BFS to maintain natural left-to-right order within same row
    type Entry struct {
        Row, Val int
    }

    colEntries := map[int][]Entry{}
    minCol, maxCol := 0, 0

    queue := []Item{{root, 0, 0}}
    for len(queue) > 0 {
        item := queue[0]
        queue = queue[1:]

        colEntries[item.Col] = append(colEntries[item.Col], Entry{item.Row, item.Node.Val})
        if item.Col < minCol { minCol = item.Col }
        if item.Col > maxCol { maxCol = item.Col }

        if item.Node.Left != nil {
            queue = append(queue, Item{item.Node.Left, item.Row + 1, item.Col - 1})
        }
        if item.Node.Right != nil {
            queue = append(queue, Item{item.Node.Right, item.Row + 1, item.Col + 1})
        }
    }

    result := [][]int{}
    for col := minCol; col <= maxCol; col++ {
        entries := colEntries[col]
        // Sort by row, then by value for same row
        sort.Slice(entries, func(i, j int) bool {
            if entries[i].Row != entries[j].Row {
                return entries[i].Row < entries[j].Row
            }
            return entries[i].Val < entries[j].Val
        })
        vals := []int{}
        for _, e := range entries {
            vals = append(vals, e.Val)
        }
        result = append(result, vals)
    }
    return result
}

func main() {
    //       3
    //      / \
    //     9   8
    //    / \  / \
    //   4  0 1   7
    root := &TreeNode{3,
        &TreeNode{9, &TreeNode{4, nil, nil}, &TreeNode{0, nil, nil}},
        &TreeNode{8, &TreeNode{1, nil, nil}, &TreeNode{7, nil, nil}},
    }
    result := verticalTraversalOptimized(root)
    for i, col := range result {
        fmt.Printf("Column %d: %v\n", i, col)
    }
}
```

**Textual Figure – Example 9:**
```
 Optimized Vertical Traversal (BFS + Sort per column):
 ────────────────────────────────────────
       3                 BFS preserves natural
      / \                left-to-right order.
     9   8
    / \ / \              Column entries sorted by
   4  0 1  7             (row, value) within each col.

 col: -2  -1   0   1   2
       │   │   │   │   │
       │   │   3   │   │   row=0
       │   9   │   8   │   row=1
       4   0   │   1   7   row=2
               │
          nodes 0,1 at col=0, row=2
          → sorted by val: [0, 1]

 ┌────────┬─────────────┐
 │ Column │ Result      │
 ├────────┼─────────────┤
 │   0    │ [4]         │
 │   1    │ [9, 0]      │
 │   2    │ [3, 0, 1]   │
 │   3    │ [8, 1]      │
 │   4    │ [7]         │
 └────────┴─────────────┘
```

---

## Example 10: Vertical Width of Binary Tree

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func verticalWidth(root *TreeNode) int {
    if root == nil { return 0 }
    minCol, maxCol := 0, 0

    var dfs func(node *TreeNode, col int)
    dfs = func(node *TreeNode, col int) {
        if node == nil { return }
        if col < minCol { minCol = col }
        if col > maxCol { maxCol = col }
        dfs(node.Left, col-1)
        dfs(node.Right, col+1)
    }
    dfs(root, 0)
    return maxCol - minCol + 1
}

func main() {
    //       1
    //      / \
    //     2   3
    //    /     \
    //   4       5
    //  /         \
    // 6           7
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, &TreeNode{6, nil, nil}, nil}, nil},
        &TreeNode{3, nil, &TreeNode{5, nil, &TreeNode{7, nil, nil}}},
    }
    fmt.Println("Vertical width:", verticalWidth(root)) // 7 (cols -3 to 3)

    single := &TreeNode{Val: 1}
    fmt.Println("Single node width:", verticalWidth(single)) // 1
}
```

**Textual Figure – Example 10:**
```
 Vertical Width = maxCol - minCol + 1:
 ────────────────────────────────
       1 (col 0)
      / \                      Columns used:
     2   3 (col -1, 1)         -3, -2, -1, 0, 1, 2, 3
    /     \
   4       5 (col -2, 2)       Width = 3 - (-3) + 1 = 7
  /         \
 6           7 (col -3, 3)

 Column spread:
 ─────────────────────────────────────────
 col: -3  -2  -1   0   1   2   3
      6    4   2   1   3   5   7
      │    │   │   │   │   │   │
      └────┴───┴───┴───┴───┴───┘
      │─────── width = 7 ───────│

 Single node: minCol=maxCol=0 → width=1
```

---

## Key Takeaways

| View | Algorithm | Column Tracking |
|------|-----------|-----------------|
| **Vertical order** | BFS, col ± 1 | Map col → values |
| **Top view** | BFS, keep first per col | Map col → first val |
| **Bottom view** | BFS, keep last per col | Map col → last val (overwrite) |
| **Diagonal** | DFS, right same diag, left +1 | Map diag → values |

1. **BFS preserves natural left-to-right order** within same level — preferred for vertical order
2. **LeetCode 987** requires sorting by (col, row, val) — use sort after collecting
3. Track `minCol` and `maxCol` to iterate columns in order without sorting keys
4. Top/bottom views are simplified vertical traversals keeping first/last per column

> **Next up:** Boundary Traversal →
