# Phase 8: Binary Trees — Tree Terminology

## Overview

Before diving into tree algorithms, master the vocabulary. Every tree concept builds on these terms.

---

## Core Definitions

| Term | Definition |
|------|-----------|
| **Node** | An element containing data and references to children |
| **Root** | The topmost node (no parent) |
| **Edge** | Connection between parent and child |
| **Parent** | Node directly above in the hierarchy |
| **Child** | Node directly below a parent |
| **Leaf** | Node with no children |
| **Internal node** | Node with at least one child |
| **Sibling** | Nodes sharing the same parent |
| **Depth** | Number of edges from root to node |
| **Height** | Number of edges on the longest path from node to a leaf |
| **Level** | Depth + 1 (root is level 1) |
| **Subtree** | A node and all its descendants |
| **Degree** | Number of children a node has |
| **Path** | Sequence of nodes connected by edges |
| **Ancestor** | Any node on the path from root to n |
| **Descendant** | Any node in the subtree of n |

---

## Example 1: Define a Tree Node in Go

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func main() {
    // Build:     1
    //           / \
    //          2   3
    //         / \
    //        4   5
    root := &TreeNode{Val: 1}
    root.Left = &TreeNode{Val: 2}
    root.Right = &TreeNode{Val: 3}
    root.Left.Left = &TreeNode{Val: 4}
    root.Left.Right = &TreeNode{Val: 5}

    fmt.Println("Root:", root.Val)              // 1
    fmt.Println("Left child:", root.Left.Val)    // 2
    fmt.Println("Right child:", root.Right.Val)  // 3
}
```

**Textual Figure:**

```
  Tree Structure with Terminology Labels
  ═══════════════════════════════════════

        ┌───┐
        │ 1 │ ← Root (no parent)
        └─┬─┘
         ╱ ╲
      ┌───┐ ┌───┐
      │ 2 │ │ 3 │ ← Children of 1 (siblings)
      └─┬─┘ └───┘
       ╱ ╲       ↑ Leaf (no children)
    ┌───┐ ┌───┐
    │ 4 │ │ 5 │ ← Leaves (no children)
    └───┘ └───┘

  Node Roles:
  ┌──────────────┬───────────────────────┐
  │ Root         │ Node 1                │
  │ Internal     │ Nodes 1, 2            │
  │ Leaves       │ Nodes 3, 4, 5         │
  │ Siblings     │ (2,3), (4,5)          │
  │ Parent of 4  │ Node 2                │
  │ Edges        │ 1→2, 1→3, 2→4, 2→5   │
  │ Total edges  │ 4 (= 5 nodes − 1)     │
  └──────────────┴───────────────────────┘

  Output:
    Root: 1
    Left child: 2
    Right child: 3
```

---

## Example 2: Count All Terminology Properties

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool {
    return n != nil && n.Left == nil && n.Right == nil
}

func isInternal(n *TreeNode) bool {
    return n != nil && (n.Left != nil || n.Right != nil)
}

func degree(n *TreeNode) int {
    d := 0
    if n.Left != nil  { d++ }
    if n.Right != nil { d++ }
    return d
}

func countNodes(root *TreeNode) int {
    if root == nil { return 0 }
    return 1 + countNodes(root.Left) + countNodes(root.Right)
}

func countLeaves(root *TreeNode) int {
    if root == nil { return 0 }
    if isLeaf(root) { return 1 }
    return countLeaves(root.Left) + countLeaves(root.Right)
}

func countInternal(root *TreeNode) int {
    if root == nil || isLeaf(root) { return 0 }
    return 1 + countInternal(root.Left) + countInternal(root.Right)
}

func countEdges(root *TreeNode) int {
    // In a tree with N nodes, there are N-1 edges
    n := countNodes(root)
    if n == 0 { return 0 }
    return n - 1
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \   \
    //     4   5   6
    //    /
    //   7
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, &TreeNode{7, nil, nil}, nil},
            &TreeNode{5, nil, nil},
        },
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }

    fmt.Println("Total nodes:   ", countNodes(root))    // 7
    fmt.Println("Leaves:        ", countLeaves(root))   // 3 (5,6,7)
    fmt.Println("Internal nodes:", countInternal(root)) // 4 (1,2,3,4)
    fmt.Println("Edges:         ", countEdges(root))    // 6
    fmt.Println("Root degree:   ", degree(root))         // 2
    fmt.Println("Node 3 degree: ", degree(root.Right))  // 1
}
```

**Textual Figure:**

```
  Tree with Terminology Properties
  ═════════════════════════════════

         ┌───┐
         │ 1 │  degree=2, internal
         └─┬─┘
          ╱ ╲
       ┌───┐ ┌───┐
       │ 2 │ │ 3 │  degree=2, degree=1
       └─┬─┘ └─┬─┘
        ╱ ╲     ╲
     ┌───┐┌───┐ ┌───┐
     │ 4 ││ 5 │ │ 6 │  leaves (degree=0)
     └─┬─┘└───┘ └───┘
      ╱              except node 4:
   ┌───┐             degree=1, internal
   │ 7 │  leaf
   └───┘

  Counting:
  ┌──────────────────┬─────────────────────┐
  │ Total nodes      │ 7                   │
  │ Leaves           │ 3 → {5, 6, 7}       │
  │ Internal nodes   │ 4 → {1, 2, 3, 4}    │
  │ Edges            │ 6 (= 7 − 1)         │
  │ Degree of root 1 │ 2 (left=2, right=3) │
  │ Degree of node 3 │ 1 (right=6 only)    │
  └──────────────────┴─────────────────────┘
```

---

## Example 3: Depth of Every Node

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func printDepths(root *TreeNode, depth int) {
    if root == nil {
        return
    }
    fmt.Printf("Node %d → depth %d\n", root.Val, depth)
    printDepths(root.Left, depth+1)
    printDepths(root.Right, depth+1)
}

func main() {
    //         1        depth 0
    //        / \
    //       2   3      depth 1
    //      / \
    //     4   5        depth 2
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    printDepths(root, 0)
}
```

**Textual Figure:**

```
  Depth of Each Node (edges from root)
  ═════════════════════════════════════

  depth=0      ┌───┐
               │ 1 │  ← root, depth = 0
               └─┬─┘
                ╱ ╲
  depth=1   ┌───┐ ┌───┐
            │ 2 │ │ 3 │  ← depth = 1
            └─┬─┘ └───┘
             ╱ ╲
  depth=2 ┌───┐ ┌───┐
          │ 4 │ │ 5 │  ← depth = 2
          └───┘ └───┘

  Depth = number of edges from root to node:
    Node 1 → depth 0  (root)
    Node 2 → depth 1  (1 edge: 1→2)
    Node 3 → depth 1  (1 edge: 1→3)
    Node 4 → depth 2  (2 edges: 1→2→4)
    Node 5 → depth 2  (2 edges: 1→2→5)
```

---

## Example 4: Height of Every Node

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func height(node *TreeNode) int {
    if node == nil {
        return -1 // convention: height of nil = -1
    }
    lh := height(node.Left)
    rh := height(node.Right)
    if lh > rh {
        return lh + 1
    }
    return rh + 1
}

func printHeights(root *TreeNode) {
    if root == nil {
        return
    }
    fmt.Printf("Node %d → height %d\n", root.Val, height(root))
    printHeights(root.Left)
    printHeights(root.Right)
}

func main() {
    //         1        height 2
    //        / \
    //       2   3      height 1, height 0
    //      / \
    //     4   5        height 0, height 0
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    printHeights(root)
}
```

**Textual Figure:**

```
  Height of Each Node (longest path to leaf)
  ═══════════════════════════════════════════

           ┌───┐
           │ 1 │  height=2 (path: 1→2→4 or 1→2→5)
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌───┐
  h=1 ←  │ 2 │ │ 3 │  → h=0 (leaf)
         └─┬─┘ └───┘
          ╱ ╲
       ┌───┐ ┌───┐
 h=0 ← │ 4 │ │ 5 │ → h=0  (leaves)
       └───┘ └───┘

  Height = edges on longest downward path to leaf:
    Node 1 → height 2  (1→2→4)
    Node 2 → height 1  (2→4 or 2→5)
    Node 3 → height 0  (leaf)
    Node 4 → height 0  (leaf)
    Node 5 → height 0  (leaf)

  Note: height(nil) = −1 by convention
```

---

## Example 5: Level of Every Node

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func printLevels(root *TreeNode) {
    if root == nil {
        return
    }
    queue := []*TreeNode{root}
    level := 1

    for len(queue) > 0 {
        size := len(queue)
        fmt.Printf("Level %d: ", level)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            fmt.Printf("%d ", node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        fmt.Println()
        level++
    }
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \   \
    //     4   5   6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }
    printLevels(root)
    // Level 1: 1
    // Level 2: 2 3
    // Level 3: 4 5 6
}
```

**Textual Figure:**

```
  Level-by-Level View (Level = Depth + 1)
  ════════════════════════════════════════

  Level 1:          ┌───┐
                    │ 1 │
                    └─┬─┘
                     ╱ ╲
  Level 2:       ┌───┐ ┌───┐
                 │ 2 │ │ 3 │
                 └─┬─┘ └─┬─┘
                  ╱ ╲     ╲
  Level 3:     ┌───┐┌───┐ ┌───┐
               │ 4 ││ 5 │ │ 6 │
               └───┘└───┘ └───┘

  BFS traversal (queue-based):
    Step 1: Enqueue [1]         → print Level 1: 1
    Step 2: Dequeue 1, enqueue [2,3] → print Level 2: 2 3
    Step 3: Dequeue 2,3, enqueue [4,5,6] → print Level 3: 4 5 6

  Output:
    Level 1: 1
    Level 2: 2 3
    Level 3: 4 5 6
```

---

## Example 6: Find Ancestors of a Node

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func findAncestors(root *TreeNode, target int, path *[]int) bool {
    if root == nil {
        return false
    }
    if root.Val == target {
        return true
    }

    *path = append(*path, root.Val)
    if findAncestors(root.Left, target, path) || findAncestors(root.Right, target, path) {
        return true
    }
    *path = (*path)[:len(*path)-1] // backtrack
    return false
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \
    //     4   5
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }

    for _, target := range []int{4, 5, 3, 1} {
        path := []int{}
        findAncestors(root, target, &path)
        fmt.Printf("Ancestors of %d: %v\n", target, path)
    }
    // Ancestors of 4: [1 2]
    // Ancestors of 5: [1 2]
    // Ancestors of 3: [1]
    // Ancestors of 1: []
}
```

**Textual Figure:**

```
  Finding Ancestors (path from root to target)
  ═══════════════════════════════════════════

           ┌───┐
           │ 1 │  ← ancestor of all nodes
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌───┐
         │ 2 │ │ 3 │
         └─┬─┘ └───┘
          ╱ ╲
       ┌───┐ ┌───┐
       │ 4 │ │ 5 │
       └───┘ └───┘

  Ancestor paths (root → target, excluding target):
    Target 4: 1 → 2 → [4]   ancestors = [1, 2]
    Target 5: 1 → 2 → [5]   ancestors = [1, 2]
    Target 3: 1 → [3]        ancestors = [1]
    Target 1: [1]             ancestors = [] (root has none)

  Backtracking search: DFS appends to path,
  removes on backtrack if target not in subtree.
```

---

## Example 7: Find All Descendants of a Node

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func findNode(root *TreeNode, target int) *TreeNode {
    if root == nil || root.Val == target {
        return root
    }
    left := findNode(root.Left, target)
    if left != nil {
        return left
    }
    return findNode(root.Right, target)
}

func getDescendants(node *TreeNode, result *[]int) {
    if node == nil {
        return
    }
    if node.Left != nil {
        *result = append(*result, node.Left.Val)
        getDescendants(node.Left, result)
    }
    if node.Right != nil {
        *result = append(*result, node.Right.Val)
        getDescendants(node.Right, result)
    }
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \   \
    //     4   5   6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }

    for _, target := range []int{1, 2, 3, 4} {
        node := findNode(root, target)
        desc := []int{}
        getDescendants(node, &desc)
        fmt.Printf("Descendants of %d: %v\n", target, desc)
    }
    // 1: [2 4 5 3 6]
    // 2: [4 5]
    // 3: [6]
    // 4: []
}
```

**Textual Figure:**

```
  Finding Descendants (all nodes in subtree)
  ═════════════════════════════════════════

            ┌───┐
            │ 1 │  descendants: {2,4,5,3,6}
            └─┬─┘
             ╱ ╲
          ┌───┐ ┌───┐
          │ 2 │ │ 3 │  desc of 2: {4,5}
          └─┬─┘ └─┬─┘  desc of 3: {6}
           ╱ ╲     ╲
        ┌───┐┌───┐ ┌───┐
        │ 4 ││ 5 │ │ 6 │  leaves: desc = {}
        └───┘└───┘ └───┘

  Subtree view for each target:
    Node 1: entire tree   → [2, 4, 5, 3, 6]
    Node 2: left subtree  → [4, 5]
    Node 3: right subtree → [6]
    Node 4: leaf node     → [] (no children)
```

---

## Example 8: Check if Nodes are Siblings

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func areSiblings(root *TreeNode, a, b int) bool {
    if root == nil {
        return false
    }
    if root.Left != nil && root.Right != nil {
        leftVal, rightVal := root.Left.Val, root.Right.Val
        if (leftVal == a && rightVal == b) || (leftVal == b && rightVal == a) {
            return true
        }
    }
    return areSiblings(root.Left, a, b) || areSiblings(root.Right, a, b)
}

func areCousins(root *TreeNode, a, b int) bool {
    // Cousins = same depth, different parent
    var depthA, depthB int
    var parentA, parentB int

    var dfs func(node *TreeNode, parent, depth int)
    dfs = func(node *TreeNode, parent, depth int) {
        if node == nil {
            return
        }
        if node.Val == a {
            depthA = depth
            parentA = parent
        }
        if node.Val == b {
            depthB = depth
            parentB = parent
        }
        dfs(node.Left, node.Val, depth+1)
        dfs(node.Right, node.Val, depth+1)
    }
    dfs(root, -1, 0)

    return depthA == depthB && parentA != parentB
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \   \
    //     4   5   6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }

    fmt.Println("4 & 5 siblings?", areSiblings(root, 4, 5)) // true
    fmt.Println("4 & 6 siblings?", areSiblings(root, 4, 6)) // false
    fmt.Println("4 & 6 cousins?", areCousins(root, 4, 6))   // true
    fmt.Println("4 & 5 cousins?", areCousins(root, 4, 5))   // false (same parent)
}
```

**Textual Figure:**

```
  Siblings vs Cousins
  ════════════════════

            ┌───┐
            │ 1 │  depth=0
            └─┬─┘
             ╱ ╲
          ┌───┐ ┌───┐
          │ 2 │ │ 3 │  depth=1, parent=1
          └─┬─┘ └─┬─┘
           ╱ ╲     ╲
        ┌───┐┌───┐ ┌───┐
        │ 4 ││ 5 │ │ 6 │  depth=2
        └───┘└───┘ └───┘

  Siblings (same parent):
    4 & 5 → YES (parent = 2)
    4 & 6 → NO  (parent of 4=2, parent of 6=3)

  Cousins (same depth, different parent):
    4 & 6 → YES (depth=2, parents: 2 ≠ 3)
    5 & 6 → YES (depth=2, parents: 2 ≠ 3)
    4 & 5 → NO  (same parent = 2)
```

---

## Example 9: Tree Properties — Full, Complete, Perfect

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Full: every node has 0 or 2 children
func isFull(root *TreeNode) bool {
    if root == nil {
        return true
    }
    if (root.Left == nil) != (root.Right == nil) {
        return false // exactly one child
    }
    return isFull(root.Left) && isFull(root.Right)
}

// Perfect: all internal nodes have 2 children AND all leaves at same depth
func isPerfect(root *TreeNode) bool {
    h := height(root)
    return checkPerfect(root, 0, h)
}

func checkPerfect(node *TreeNode, depth, h int) bool {
    if node == nil {
        return true
    }
    if node.Left == nil && node.Right == nil {
        return depth == h // leaf must be at max depth
    }
    if node.Left == nil || node.Right == nil {
        return false
    }
    return checkPerfect(node.Left, depth+1, h) && checkPerfect(node.Right, depth+1, h)
}

// Complete: all levels full except possibly last, which is filled left to right
func isComplete(root *TreeNode) bool {
    if root == nil {
        return true
    }
    queue := []*TreeNode{root}
    foundNull := false

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if node == nil {
            foundNull = true
            continue
        }
        if foundNull {
            return false // non-null after null → not complete
        }
        queue = append(queue, node.Left, node.Right)
    }
    return true
}

func height(node *TreeNode) int {
    if node == nil {
        return -1
    }
    lh := height(node.Left)
    rh := height(node.Right)
    if lh > rh {
        return lh + 1
    }
    return rh + 1
}

func main() {
    // Full & Perfect:  1
    //                 / \
    //                2   3
    full := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println("Full:", isFull(full))       // true
    fmt.Println("Perfect:", isPerfect(full)) // true
    fmt.Println("Complete:", isComplete(full)) // true

    // Complete but not full:  1
    //                        / \
    //                       2   3
    //                      /
    //                     4
    comp := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println("\nFull:", isFull(comp))       // false
    fmt.Println("Perfect:", isPerfect(comp)) // false
    fmt.Println("Complete:", isComplete(comp)) // true
}
```

**Textual Figure:**

```
  Tree Type Classification
  ═══════════════════════════

  Full & Perfect:          Complete but NOT Full:
       ┌───┐                      ┌───┐
       │ 1 │                      │ 1 │
       └─┬─┘                      └─┬─┘
        ╱ ╲                        ╱ ╲
     ┌───┐ ┌───┐              ┌───┐ ┌───┐
     │ 2 │ │ 3 │              │ 2 │ │ 3 │
     └───┘ └───┘              └─┬─┘ └───┘
                                 ╱
  ✔ Full (0 or 2 children)     ┌───┐
  ✔ Perfect (all leaves same)  │ 4 │
  ✔ Complete (filled L→R)      └───┘
                              ✗ Full (node 2 has 1 child)
                              ✗ Perfect (leaves at diff depths)
                              ✔ Complete (last level L→R)

  Rule summary:
  ┌──────────┬─────────────────────────────────┐
  │ Full     │ Every node: 0 or 2 children    │
  │ Complete │ All levels full except last     │
  │          │ (last filled left → right)      │
  │ Perfect  │ Full + all leaves same depth    │
  └──────────┴─────────────────────────────────┘
```

---

## Example 10: Build Tree from Slice (Array Representation)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func buildTree(arr []int, i int) *TreeNode {
    if i >= len(arr) || arr[i] == -1 {
        return nil
    }
    node := &TreeNode{Val: arr[i]}
    node.Left = buildTree(arr, 2*i+1)
    node.Right = buildTree(arr, 2*i+2)
    return node
}

func printTree(root *TreeNode, prefix string, isLeft bool) {
    if root == nil {
        return
    }
    connector := "└── "
    if isLeft {
        connector = "├── "
    }
    fmt.Println(prefix + connector + fmt.Sprint(root.Val))

    newPrefix := prefix
    if isLeft {
        newPrefix += "│   "
    } else {
        newPrefix += "    "
    }
    printTree(root.Left, newPrefix, true)
    printTree(root.Right, newPrefix, false)
}

func main() {
    // Array: [1, 2, 3, 4, 5, -1, 6]  (-1 = null)
    // Tree:      1
    //           / \
    //          2   3
    //         / \   \
    //        4   5   6
    arr := []int{1, 2, 3, 4, 5, -1, 6}
    root := buildTree(arr, 0)
    printTree(root, "", false)
}
```

**Textual Figure:**

```
  Array to Tree Mapping
  ═══════════════════════

  Array: [1, 2, 3, 4, 5, -1, 6]
  Index:  0  1  2  3  4   5  6

  Index formula: left = 2i+1, right = 2i+2

  i=0 → val=1, left=i1, right=i2
  i=1 → val=2, left=i3, right=i4
  i=2 → val=3, left=i5(-1=nil), right=i6

  Resulting tree:
           ┌───┐
     i=0   │ 1 │
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌───┐
   i=1   │ 2 │ │ 3 │  i=2
         └─┬─┘ └─┬─┘
          ╱ ╲     ╲
       ┌───┐┌───┐ ┌───┐
 i=3   │ 4 ││ 5 │ │ 6 │  i=6
       └───┘└───┘ └───┘
         i=4   (i=5 is -1, skipped)

  printTree output:
  └── 1
      ├── 2
      │   ├── 4
      │   └── 5
      └── 3
          └── 6
```

---

## Tree Terminology Quick Reference

```
          1          ← Root (depth=0, height=3)
         / \
        2   3        ← Internal nodes (depth=1)
       / \   \
      4   5   6      ← 5,6 are leaves (depth=2)
     /
    7                ← Leaf (depth=3, height=0)

Path: 1→2→4→7 (length=3)
Ancestors of 7: [1, 2, 4]
Descendants of 2: [4, 5, 7]
Siblings: (2,3), (4,5)
Cousins: (5,6) — same depth, different parent
Degree of node 2: 2
Tree degree: max(2,2,1,1,0,0,0) = 2 (binary)
```

## Key Takeaways

1. **Height vs Depth**: Depth counts DOWN from root; Height counts UP from leaves
2. **N nodes → N-1 edges** in any tree
3. **Full binary tree**: every node has 0 or 2 children
4. **Complete**: all levels full except last, filled left-to-right (used by heaps)
5. **Perfect**: all leaves at same level and all internal nodes have 2 children
6. A binary tree with height h has at most 2^(h+1) - 1 nodes
7. The tree is Go's `nil`-pointer-terminated structure — base case is always `node == nil`

> **Next up:** Binary Tree Representation →
