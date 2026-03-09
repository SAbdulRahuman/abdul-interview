# Phase 8: Binary Trees — Depth First Search (DFS)

## Overview

**DFS** on trees explores as deep as possible before backtracking. Three orderings — preorder, inorder, postorder — are all DFS. This file focuses on **DFS patterns and techniques** applied to tree problems.

| Approach | Stack Space | Notes |
|----------|-------------|-------|
| Recursive | O(h) call stack | Simplest, risk stack overflow on deep trees |
| Iterative (explicit stack) | O(h) | Handles deep trees safely |
| Morris | O(1) | Modifies tree temporarily |

---

## Example 1: Recursive DFS — Path Sum (LeetCode 112)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func hasPathSum(root *TreeNode, targetSum int) bool {
    if root == nil { return false }
    targetSum -= root.Val
    // Leaf check
    if root.Left == nil && root.Right == nil {
        return targetSum == 0
    }
    return hasPathSum(root.Left, targetSum) || hasPathSum(root.Right, targetSum)
}

func main() {
    //        5
    //       / \
    //      4   8
    //     /   / \
    //    11  13  4
    //   / \       \
    //  7   2       1
    root := &TreeNode{5,
        &TreeNode{4, &TreeNode{11, &TreeNode{7, nil, nil}, &TreeNode{2, nil, nil}}, nil},
        &TreeNode{8, &TreeNode{13, nil, nil}, &TreeNode{4, nil, &TreeNode{1, nil, nil}}},
    }
    fmt.Println(hasPathSum(root, 22)) // true  (5→4→11→2)
    fmt.Println(hasPathSum(root, 26)) // true  (5→8→13)
    fmt.Println(hasPathSum(root, 99)) // false
}
```

---

## Example 2: Iterative DFS — Path Sum II (LeetCode 113)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func pathSum(root *TreeNode, targetSum int) [][]int {
    result := [][]int{}
    var dfs func(node *TreeNode, remain int, path []int)
    dfs = func(node *TreeNode, remain int, path []int) {
        if node == nil { return }
        path = append(path, node.Val)
        remain -= node.Val

        if node.Left == nil && node.Right == nil && remain == 0 {
            tmp := make([]int, len(path))
            copy(tmp, path)
            result = append(result, tmp)
            return
        }
        dfs(node.Left, remain, path)
        dfs(node.Right, remain, path)
    }
    dfs(root, targetSum, []int{})
    return result
}

func main() {
    root := &TreeNode{5,
        &TreeNode{4, &TreeNode{11, &TreeNode{7, nil, nil}, &TreeNode{2, nil, nil}}, nil},
        &TreeNode{8, &TreeNode{13, nil, nil}, &TreeNode{4, &TreeNode{5, nil, nil}, &TreeNode{1, nil, nil}}},
    }
    fmt.Println(pathSum(root, 22))
    // [[5,4,11,2], [5,8,4,5]]
}
```

---

## Example 3: Iterative DFS with Explicit Stack

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type StackItem struct {
    Node *TreeNode
    Sum  int
}

func hasPathSumIterative(root *TreeNode, targetSum int) bool {
    if root == nil { return false }
    stack := []StackItem{{root, root.Val}}

    for len(stack) > 0 {
        item := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        node, curSum := item.Node, item.Sum

        if node.Left == nil && node.Right == nil && curSum == targetSum {
            return true
        }
        if node.Right != nil {
            stack = append(stack, StackItem{node.Right, curSum + node.Right.Val})
        }
        if node.Left != nil {
            stack = append(stack, StackItem{node.Left, curSum + node.Left.Val})
        }
    }
    return false
}

func main() {
    root := &TreeNode{5,
        &TreeNode{4, &TreeNode{11, &TreeNode{7, nil, nil}, &TreeNode{2, nil, nil}}, nil},
        &TreeNode{8, &TreeNode{13, nil, nil}, &TreeNode{4, nil, &TreeNode{1, nil, nil}}},
    }
    fmt.Println(hasPathSumIterative(root, 22)) // true
    fmt.Println(hasPathSumIterative(root, 18)) // true  (5→8→4→1)
}
```

---

## Example 4: DFS to Collect All Root-to-Leaf Paths (LeetCode 257)

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func binaryTreePaths(root *TreeNode) []string {
    result := []string{}
    var dfs func(node *TreeNode, path []string)
    dfs = func(node *TreeNode, path []string) {
        if node == nil { return }
        path = append(path, strconv.Itoa(node.Val))

        if node.Left == nil && node.Right == nil {
            result = append(result, strings.Join(path, "->"))
            return
        }
        dfs(node.Left, path)
        dfs(node.Right, path)
    }
    dfs(root, nil)
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(binaryTreePaths(root)) // ["1->2->5", "1->3"]
}
```

---

## Example 5: DFS — Symmetric Tree Check (LeetCode 101)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isSymmetric(root *TreeNode) bool {
    if root == nil { return true }
    return isMirror(root.Left, root.Right)
}

func isMirror(t1, t2 *TreeNode) bool {
    if t1 == nil && t2 == nil { return true }
    if t1 == nil || t2 == nil { return false }
    return t1.Val == t2.Val &&
        isMirror(t1.Left, t2.Right) &&
        isMirror(t1.Right, t2.Left)
}

func main() {
    //     1
    //    / \
    //   2   2
    //  / \ / \
    // 3  4 4  3
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{3, nil, nil}, &TreeNode{4, nil, nil}},
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{3, nil, nil}},
    }
    fmt.Println(isSymmetric(root)) // true

    root2 := &TreeNode{1,
        &TreeNode{2, nil, &TreeNode{3, nil, nil}},
        &TreeNode{2, nil, &TreeNode{3, nil, nil}},
    }
    fmt.Println(isSymmetric(root2)) // false
}
```

---

## Example 6: DFS — Same Tree (LeetCode 100) & Subtree Check (LeetCode 572)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isSameTree(p, q *TreeNode) bool {
    if p == nil && q == nil { return true }
    if p == nil || q == nil { return false }
    return p.Val == q.Val &&
        isSameTree(p.Left, q.Left) &&
        isSameTree(p.Right, q.Right)
}

func isSubtree(root, subRoot *TreeNode) bool {
    if root == nil { return false }
    if isSameTree(root, subRoot) { return true }
    return isSubtree(root.Left, subRoot) || isSubtree(root.Right, subRoot)
}

func main() {
    root := &TreeNode{3,
        &TreeNode{4, &TreeNode{1, nil, nil}, &TreeNode{2, nil, nil}},
        &TreeNode{5, nil, nil},
    }
    sub := &TreeNode{4, &TreeNode{1, nil, nil}, &TreeNode{2, nil, nil}}
    fmt.Println(isSubtree(root, sub)) // true
}
```

---

## Example 7: DFS — Sum Root to Leaf Numbers (LeetCode 129)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func sumNumbers(root *TreeNode) int {
    var dfs func(node *TreeNode, num int) int
    dfs = func(node *TreeNode, num int) int {
        if node == nil { return 0 }
        num = num*10 + node.Val
        if node.Left == nil && node.Right == nil {
            return num
        }
        return dfs(node.Left, num) + dfs(node.Right, num)
    }
    return dfs(root, 0)
}

func main() {
    //     1
    //    / \
    //   2   3
    // Paths: 12 + 13 = 25
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(sumNumbers(root)) // 25

    //       4
    //      / \
    //     9   0
    //    / \
    //   5   1
    // Paths: 495 + 491 + 40 = 1026
    root2 := &TreeNode{4,
        &TreeNode{9, &TreeNode{5, nil, nil}, &TreeNode{1, nil, nil}},
        &TreeNode{0, nil, nil},
    }
    fmt.Println(sumNumbers(root2)) // 1026
}
```

---

## Example 8: DFS — Invert Binary Tree (LeetCode 226)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func invertTree(root *TreeNode) *TreeNode {
    if root == nil { return nil }
    root.Left, root.Right = invertTree(root.Right), invertTree(root.Left)
    return root
}

func invertTreeIterative(root *TreeNode) *TreeNode {
    if root == nil { return nil }
    stack := []*TreeNode{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        node.Left, node.Right = node.Right, node.Left
        if node.Left != nil { stack = append(stack, node.Left) }
        if node.Right != nil { stack = append(stack, node.Right) }
    }
    return root
}

func printInorder(root *TreeNode) {
    if root == nil { return }
    printInorder(root.Left)
    fmt.Printf("%d ", root.Val)
    printInorder(root.Right)
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{9, nil, nil}},
    }
    fmt.Print("Before: "); printInorder(root); fmt.Println()
    invertTree(root)
    fmt.Print("After:  "); printInorder(root); fmt.Println()
}
```

---

## Example 9: DFS with Return Value — Maximum Path Sum (LeetCode 124)

```go
package main

import (
    "fmt"
    "math"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func maxPathSum(root *TreeNode) int {
    maxSum := math.MinInt64

    var dfs func(node *TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil { return 0 }

        leftGain := max(0, dfs(node.Left))   // ignore negative paths
        rightGain := max(0, dfs(node.Right))

        // Path through this node as turning point
        pathSum := node.Val + leftGain + rightGain
        if pathSum > maxSum {
            maxSum = pathSum
        }

        // Return max single-branch gain to parent
        return node.Val + max(leftGain, rightGain)
    }
    dfs(root)
    return maxSum
}

func max(a, b int) int {
    if a > b { return a }
    return b
}

func main() {
    //     -10
    //     /  \
    //    9   20
    //       /  \
    //      15   7
    root := &TreeNode{-10,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(maxPathSum(root)) // 42 (15→20→7)
}
```

---

## Example 10: DFS — Count Good Nodes (LeetCode 1448)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func goodNodes(root *TreeNode) int {
    count := 0
    var dfs func(node *TreeNode, maxSoFar int)
    dfs = func(node *TreeNode, maxSoFar int) {
        if node == nil { return }
        if node.Val >= maxSoFar {
            count++
            maxSoFar = node.Val
        }
        dfs(node.Left, maxSoFar)
        dfs(node.Right, maxSoFar)
    }
    dfs(root, root.Val)
    return count
}

func main() {
    //       3
    //      / \
    //     1   4
    //    /   / \
    //   3   1   5
    root := &TreeNode{3,
        &TreeNode{1, &TreeNode{3, nil, nil}, nil},
        &TreeNode{4, &TreeNode{1, nil, nil}, &TreeNode{5, nil, nil}},
    }
    fmt.Println(goodNodes(root)) // 4 (nodes 3, 3, 4, 5)
}
```

---

## Example 11: DFS — Flatten Binary Tree (LeetCode 114)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Reverse postorder (right → left → root) approach
func flatten(root *TreeNode) {
    var prev *TreeNode
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil { return }
        dfs(node.Right)
        dfs(node.Left)
        node.Right = prev
        node.Left = nil
        prev = node
    }
    dfs(root)
}

func printList(root *TreeNode) {
    for root != nil {
        fmt.Printf("%d ", root.Val)
        root = root.Right
    }
    fmt.Println()
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{3, nil, nil}, &TreeNode{4, nil, nil}},
        &TreeNode{5, nil, &TreeNode{6, nil, nil}},
    }
    flatten(root)
    printList(root) // 1 2 3 4 5 6
}
```

---

## DFS Patterns Summary

| Pattern | When to Use | Example |
|---------|-------------|---------|
| **Top-down** | Pass info from root to leaves | Good nodes, path sum |
| **Bottom-up** | Aggregate results from leaves to root | Height, diameter, balanced check |
| **Path tracking** | Collect root-to-leaf paths | Path sum II, binary tree paths |
| **Return value** | Need subtree result for parent | Max path sum, LCA |
| **Two-tree DFS** | Compare two trees simultaneously | Same tree, symmetric, subtree |
| **Global variable** | Track best answer across all nodes | Max path sum, diameter |

## Key Takeaways

1. **Top-down DFS** passes accumulated state downward (e.g., current max, running sum)
2. **Bottom-up DFS** returns computed values upward (e.g., height, subtree sum)
3. Use **path slice + copy** to avoid shared-slice bugs when collecting paths
4. **Iterative DFS** with explicit stack avoids stack overflow on very deep trees
5. DFS naturally handles **divide and conquer** on trees: solve left, solve right, combine

> **Next up:** Breadth First Search (Trees) →
