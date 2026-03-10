# Phase 8: Binary Trees — Tree Height

## Overview

**Tree height** = length of the longest path from root to a leaf (measured in edges or nodes depending on convention). **Depth** of a node = distance from root. **Height** of a node = longest path to a leaf descendant.

```
         1          Height: 3 (edges) or 4 (nodes)
        / \
       2   3        Depth of node 5: 2
      / \
     4   5
    /
   6                Height of node 2: 2 (edges 2→4→6)
```

> Convention used here: **height = number of edges** unless stated otherwise (LeetCode uses node count, i.e., height + 1 = maxDepth).

---

## Example 1: Maximum Depth / Height (LeetCode 104)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Returns max depth (node count convention = height + 1)
func maxDepth(root *TreeNode) int {
    if root == nil { return 0 }
    left := maxDepth(root.Left)
    right := maxDepth(root.Right)
    if left > right {
        return left + 1
    }
    return right + 1
}

// Height in edges
func height(root *TreeNode) int {
    if root == nil { return -1 } // empty tree has height -1
    left := height(root.Left)
    right := height(root.Right)
    if left > right { return left + 1 }
    return right + 1
}

func main() {
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println("Max depth (nodes):", maxDepth(root))  // 3
    fmt.Println("Height (edges):", height(root))        // 2
}
```

**Textual Figure:**
```
         3          depth(nodes)=1, height(edges)=0
        / \
       9  20         depth=2, height=1
         /  \
        15   7       depth=3, height=2

  maxDepth (node count convention):
  ┌──────┬───────┬────────┬─────────────┐
  │ Node │ left  │ right  │ maxDepth    │
  ├──────┼───────┼────────┼─────────────┤
  │   9  │   0   │    0   │ 1           │
  │  15  │   0   │    0   │ 1           │
  │   7  │   0   │    0   │ 1           │
  │  20  │   1   │    1   │ 2           │
  │   3  │   1   │    2   │ 3 ★         │
  └──────┴───────┴────────┴─────────────┘

  maxDepth (nodes) = 3, height (edges) = 2
```

---

## Example 2: Minimum Depth (LeetCode 111)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func minDepth(root *TreeNode) int {
    if root == nil { return 0 }

    // If one child is nil, we must go to the other side
    if root.Left == nil { return 1 + minDepth(root.Right) }
    if root.Right == nil { return 1 + minDepth(root.Left) }

    left := minDepth(root.Left)
    right := minDepth(root.Right)
    if left < right { return left + 1 }
    return right + 1
}

func main() {
    //     2
    //      \
    //       3
    //        \
    //         4
    root := &TreeNode{2, nil, &TreeNode{3, nil, &TreeNode{4, nil, nil}}}
    fmt.Println(minDepth(root)) // 3 (not 1!)

    //     1
    //    / \
    //   2   3
    //  /
    // 4
    root2 := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(minDepth(root2)) // 2 (path 1→3)
}
```

**Textual Figure:**
```
  Skewed tree:          Balanced tree:
  2                       1
   \                     / \
    3                   2   3   ← leaf at depth 2
     \                 /
      4               4

  Common bug vs correct:
  ┌──────────────────────────────────────────┐
  │ Skewed (2→3→4):                            │
  │   ✗ Bug: min(0,3)+1 = 1 (nil left child) │
  │   ✓ Fix: left=nil → return 1+minDepth(R)  │
  │         = 1 + 1 + 1 = 3                   │
  ├──────────────────────────────────────────┤
  │ Balanced (1→2+3):                          │
  │   min(2, 1) + 1 = 2  (path 1→3)           │
  └──────────────────────────────────────────┘
```

**Why?** A common bug: returning `min(left, right) + 1` when one child is nil gives depth 1 (counting the nil child), which is wrong.

---

## Example 3: Iterative Height (BFS)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func maxDepthBFS(root *TreeNode) int {
    if root == nil { return 0 }
    queue := []*TreeNode{root}
    depth := 0

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        depth++
    }
    return depth
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(maxDepthBFS(root)) // 3
}
```

**Textual Figure:**
```
       1           Level 1
      / \
     2   3         Level 2
    /
   4               Level 3

  BFS level counting:
  ┌───────┬───────────┬─────────┬─────────┐
  │ Level │ Queue     │ Size    │ depth++ │
  ├───────┼───────────┼─────────┼─────────┤
  │   1   │ [1]       │   1     │ d=1     │
  │   2   │ [2, 3]    │   2     │ d=2     │
  │   3   │ [4]       │   1     │ d=3 ★   │
  └───────┴───────────┴─────────┴─────────┘

  maxDepth = 3
```

---

## Example 4: Height of Every Node

```go
package main

import "fmt"

type TreeNode struct {
    Val    int
    Left   *TreeNode
    Right  *TreeNode
    Height int
}

func computeHeights(node *TreeNode) int {
    if node == nil { return -1 }
    lh := computeHeights(node.Left)
    rh := computeHeights(node.Right)
    if lh > rh {
        node.Height = lh + 1
    } else {
        node.Height = rh + 1
    }
    return node.Height
}

func printPreorder(node *TreeNode) {
    if node == nil { return }
    fmt.Printf("Node %d → height %d\n", node.Val, node.Height)
    printPreorder(node.Left)
    printPreorder(node.Right)
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \
    //   4   5
    //  /
    // 6
    n6 := &TreeNode{Val: 6}
    n4 := &TreeNode{Val: 4, Left: n6}
    n5 := &TreeNode{Val: 5}
    n2 := &TreeNode{Val: 2, Left: n4, Right: n5}
    n3 := &TreeNode{Val: 3}
    root := &TreeNode{Val: 1, Left: n2, Right: n3}

    computeHeights(root)
    printPreorder(root)
    // Node 1 → height 3
    // Node 2 → height 2
    // Node 4 → height 1
    // Node 6 → height 0
    // Node 5 → height 0
    // Node 3 → height 0
}
```

**Textual Figure:**
```
       1  (h=3)
      / \
     2   3  (h=2, h=0)
    / \
   4   5    (h=1, h=0)
  /
 6          (h=0)

  Bottom-up height computation:
  ┌──────┬────────┬─────────┬───────────────────┐
  │ Node │ leftH  │ rightH  │ height = max+1 │
  ├──────┼────────┼─────────┼───────────────────┤
  │  6   │   -1   │   -1    │ 0              │
  │  4   │    0   │   -1    │ 1              │
  │  5   │   -1   │   -1    │ 0              │
  │  2   │    1   │    0    │ 2              │
  │  3   │   -1   │   -1    │ 0              │
  │  1   │    2   │    0    │ 3              │
  └──────┴────────┴─────────┴───────────────────┘
```

---

## Example 5: Balanced Binary Tree (LeetCode 110)

A tree is balanced if for every node, |height(left) - height(right)| ≤ 1.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// O(n) solution: combine height calculation with balance check
func isBalanced(root *TreeNode) bool {
    return checkHeight(root) != -1
}

func checkHeight(node *TreeNode) int {
    if node == nil { return 0 }

    left := checkHeight(node.Left)
    if left == -1 { return -1 } // left subtree unbalanced

    right := checkHeight(node.Right)
    if right == -1 { return -1 } // right subtree unbalanced

    diff := left - right
    if diff < 0 { diff = -diff }
    if diff > 1 { return -1 } // this node unbalanced

    if left > right { return left + 1 }
    return right + 1
}

func main() {
    //     3
    //    / \
    //   9  20
    //     /  \
    //    15   7
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(isBalanced(root)) // true

    //       1
    //      / \
    //     2   2
    //    / \
    //   3   3
    //  / \
    // 4   4
    root2 := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{4, nil, nil}},
            &TreeNode{3, nil, nil},
        },
        &TreeNode{2, nil, nil},
    }
    fmt.Println(isBalanced(root2)) // false
}
```

**Textual Figure:**
```
  Balanced:               Unbalanced:
     3                       1
    / \                     / \
   9  20                   2   2
     /  \                 / \
    15   7               3   3
                        / \
                       4   4

  Height check with -1 sentinel:
  ┌──────┬─────┬─────┬───────┐ ┌──────┬─────┬─────┬───────┐
  │ Node │ lH  │ rH  │ diff  │ │ Node │ lH  │ rH  │ diff  │
  ├──────┼─────┼─────┼───────┤ ├──────┼─────┼─────┼───────┤
  │  9   │  0  │  0  │  0 ✓ │ │  4   │  0  │  0  │  0 ✓ │
  │  15  │  0  │  0  │  0 ✓ │ │  3   │  1  │  1  │  0 ✓ │
  │   7  │  0  │  0  │  0 ✓ │ │  3   │  0  │  0  │  0 ✓ │
  │  20  │  1  │  1  │  0 ✓ │ │  2   │  2  │  1  │  1 ✓ │
  │   3  │  1  │  2  │  1 ✓ │ │  2   │  0  │  0  │  0 ✓ │
  └──────┴─────┴─────┴───────┘ │  1   │  3  │  1  │  2 ✗ │
  Balanced ✓                  └──────┴─────┴─────┴───────┘
                             |3-1|=2 > 1 → unbalanced!
```

---

## Example 6: Maximum Depth of N-ary Tree (LeetCode 559)

```go
package main

import "fmt"

type Node struct {
    Val      int
    Children []*Node
}

func maxDepthNary(root *Node) int {
    if root == nil { return 0 }
    maxChild := 0
    for _, child := range root.Children {
        d := maxDepthNary(child)
        if d > maxChild { maxChild = d }
    }
    return maxChild + 1
}

func main() {
    //       1
    //      /|\
    //     3 2 4
    //    / \
    //   5   6
    root := &Node{1, []*Node{
        {3, []*Node{{5, nil}, {6, nil}}},
        {2, nil},
        {4, nil},
    }}
    fmt.Println(maxDepthNary(root)) // 3
}
```

---

## Example 7: Depth of All Nodes

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func allDepths(root *TreeNode) map[int]int {
    depths := map[int]int{} // val → depth
    var dfs func(node *TreeNode, d int)
    dfs = func(node *TreeNode, d int) {
        if node == nil { return }
        depths[node.Val] = d
        dfs(node.Left, d+1)
        dfs(node.Right, d+1)
    }
    dfs(root, 0)
    return depths
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }
    for val, depth := range allDepths(root) {
        fmt.Printf("Node %d: depth %d\n", val, depth)
    }
}
```

---

## Example 8: Sum of Depths of All Nodes

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func sumOfDepths(root *TreeNode) int {
    total := 0
    var dfs func(node *TreeNode, depth int)
    dfs = func(node *TreeNode, depth int) {
        if node == nil { return }
        total += depth
        dfs(node.Left, depth+1)
        dfs(node.Right, depth+1)
    }
    dfs(root, 0)
    return total
}

// Also compute average depth
func avgDepth(root *TreeNode) float64 {
    totalDepth, count := 0, 0
    var dfs func(node *TreeNode, d int)
    dfs = func(node *TreeNode, d int) {
        if node == nil { return }
        totalDepth += d
        count++
        dfs(node.Left, d+1)
        dfs(node.Right, d+1)
    }
    dfs(root, 0)
    if count == 0 { return 0 }
    return float64(totalDepth) / float64(count)
}

func main() {
    //       1        depth 0
    //      / \
    //     2   3      depth 1
    //    / \
    //   4   5        depth 2
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println("Sum of depths:", sumOfDepths(root))      // 0+1+1+2+2 = 6
    fmt.Printf("Average depth: %.2f\n", avgDepth(root))   // 6/5 = 1.20
}
```

---

## Example 9: Height-Balanced BST from Sorted Array (LeetCode 108)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func sortedArrayToBST(nums []int) *TreeNode {
    if len(nums) == 0 { return nil }
    mid := len(nums) / 2
    return &TreeNode{
        Val:   nums[mid],
        Left:  sortedArrayToBST(nums[:mid]),
        Right: sortedArrayToBST(nums[mid+1:]),
    }
}

func maxDepth(root *TreeNode) int {
    if root == nil { return 0 }
    l, r := maxDepth(root.Left), maxDepth(root.Right)
    if l > r { return l + 1 }
    return r + 1
}

func printInorder(root *TreeNode) {
    if root == nil { return }
    printInorder(root.Left)
    fmt.Printf("%d ", root.Val)
    printInorder(root.Right)
}

func main() {
    nums := []int{-10, -3, 0, 5, 9}
    root := sortedArrayToBST(nums)
    fmt.Print("Inorder: "); printInorder(root); fmt.Println()
    fmt.Println("Height:", maxDepth(root)) // 3 — balanced!
}
```

---

## Example 10: Compute Height Without Recursion (Iterative DFS)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func maxDepthIterative(root *TreeNode) int {
    if root == nil { return 0 }

    type Pair struct {
        Node  *TreeNode
        Depth int
    }

    stack := []Pair{{root, 1}}
    maxD := 0

    for len(stack) > 0 {
        p := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if p.Depth > maxD { maxD = p.Depth }

        if p.Node.Left != nil {
            stack = append(stack, Pair{p.Node.Left, p.Depth + 1})
        }
        if p.Node.Right != nil {
            stack = append(stack, Pair{p.Node.Right, p.Depth + 1})
        }
    }
    return maxD
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, &TreeNode{7, nil, nil}, nil},
            nil,
        },
        &TreeNode{3, nil, nil},
    }
    fmt.Println("Max depth (iterative):", maxDepthIterative(root)) // 4
}
```

---

## Key Takeaways

| Concept | Detail |
|---------|--------|
| **maxDepth** | `1 + max(left, right)`, base nil → 0 |
| **height (edges)** | same formula but base nil → -1 |
| **minDepth** | Must handle single-child nodes carefully |
| **Balanced check** | Combine with height in one O(n) pass, return -1 sentinel |
| **Iterative** | BFS counts levels; DFS tracks depth in stack items |
| **N-ary** | Same pattern: max over all children |

> **Next up:** Tree Diameter →
