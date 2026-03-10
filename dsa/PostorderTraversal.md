# Phase 8: Binary Trees — Postorder Traversal

## Overview

**Postorder** visits nodes in **Left → Right → Node** order. The root is always the **last** element. Postorder is the natural order for:
- Deleting a tree (children before parent)
- Evaluating expression trees (operands before operator)
- Computing subtree properties bottom-up (height, size)

```
         1
        / \
       2   3
      / \
     4   5

Postorder: 4, 5, 2, 3, 1
```

---

## Example 1: Recursive Postorder

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func postorderTraversal(root *TreeNode) []int {
    result := []int{}
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil { return }
        dfs(node.Left)
        dfs(node.Right)
        result = append(result, node.Val)
    }
    dfs(root)
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(postorderTraversal(root)) // [4 5 2 3 1]
}
```

**Textual Figure:**
```
         1
        / \
       2   3
      / \
     4   5

  Recursive Call Trace (Left → Right → Node):
  ┌──────────────────────────────────────────┐
  │ dfs(1)                                   │
  │  ├─ dfs(2)                               │
  │  │   ├─ dfs(4)                           │
  │  │   │   ├─ dfs(nil) → return            │
  │  │   │   ├─ dfs(nil) → return            │
  │  │   │   └─ append 4 ✓                   │
  │  │   ├─ dfs(5)                           │
  │  │   │   ├─ dfs(nil) → return            │
  │  │   │   ├─ dfs(nil) → return            │
  │  │   │   └─ append 5 ✓                   │
  │  │   └─ append 2 ✓                       │
  │  ├─ dfs(3)                               │
  │  │   ├─ dfs(nil) → return                │
  │  │   ├─ dfs(nil) → return                │
  │  │   └─ append 3 ✓                       │
  │  └─ append 1 ✓                           │
  └──────────────────────────────────────────┘

  Visit order: 4 → 5 → 2 → 3 → 1
  Result: [4, 5, 2, 3, 1]
```

---

## Example 2: Iterative Postorder (Two Stacks)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func postorderTwoStacks(root *TreeNode) []int {
    if root == nil { return nil }
    stack1 := []*TreeNode{root}
    stack2 := []*TreeNode{}

    for len(stack1) > 0 {
        node := stack1[len(stack1)-1]
        stack1 = stack1[:len(stack1)-1]
        stack2 = append(stack2, node)

        if node.Left != nil { stack1 = append(stack1, node.Left) }
        if node.Right != nil { stack1 = append(stack1, node.Right) }
    }

    result := make([]int, len(stack2))
    for i := len(stack2) - 1; i >= 0; i-- {
        result[len(stack2)-1-i] = stack2[i].Val
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, nil},
    }
    fmt.Println(postorderTwoStacks(root)) // [4 5 2 6 3 1]
}
```

**Textual Figure:**
```
         1
        / \
       2   3
      / \  /
     4   5 6

  Two-Stack Simulation:
  ┌──────┬─────────────────┬──────────────────────┐
  │ Step │ Stack1 (top→)   │ Stack2 (top→)        │
  ├──────┼─────────────────┼──────────────────────┤
  │  0   │ [1]             │ []                   │
  │  1   │ [2, 3]          │ [1]                  │
  │  2   │ [2, 6]          │ [1, 3]               │
  │  3   │ [2]             │ [1, 3, 6]            │
  │  4   │ [4, 5]          │ [1, 3, 6, 2]         │
  │  5   │ [4]             │ [1, 3, 6, 2, 5]      │
  │  6   │ []              │ [1, 3, 6, 2, 5, 4]   │
  └──────┴─────────────────┴──────────────────────┘

  Pop Stack2 top → bottom: 4 → 5 → 2 → 6 → 3 → 1
  Result: [4, 5, 2, 6, 3, 1]
```

---

## Example 3: Iterative Postorder (Single Stack)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func postorderOneStack(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    cur := root
    var lastVisited *TreeNode

    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }

        top := stack[len(stack)-1]
        // If right child exists and hasn't been visited
        if top.Right != nil && top.Right != lastVisited {
            cur = top.Right
        } else {
            result = append(result, top.Val)
            lastVisited = top
            stack = stack[:len(stack)-1]
        }
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(postorderOneStack(root)) // [4 5 2 3 1]
}
```

**Textual Figure:**
```
         1
        / \
       2   3
      / \
     4   5

  Single Stack + lastVisited Pointer:
  ┌──────┬────────────┬─────────────┬───────────────┐
  │ Step │ Stack      │ lastVisited │ Action        │
  ├──────┼────────────┼─────────────┼───────────────┤
  │  1   │ [1,2,4]    │ nil         │ push left     │
  │  2   │ [1,2]      │ 4           │ visit 4 ✓     │
  │  3   │ [1,2]      │ 4           │ cur = 5 (R)   │
  │  4   │ [1,2,5]    │ 4           │ push 5        │
  │  5   │ [1,2]      │ 5           │ visit 5 ✓     │
  │  6   │ [1]        │ 2           │ visit 2 ✓     │
  │  7   │ [1]        │ 2           │ cur = 3 (R)   │
  │  8   │ [1,3]      │ 2           │ push 3        │
  │  9   │ [1]        │ 3           │ visit 3 ✓     │
  │ 10   │ []         │ 1           │ visit 1 ✓     │
  └──────┴────────────┴─────────────┴───────────────┘

  Visit order: 4 → 5 → 2 → 3 → 1
  Result: [4, 5, 2, 3, 1]
```

---

## Example 4: Delete a Binary Tree (Postorder)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func deleteTree(root **TreeNode) {
    if *root == nil {
        return
    }
    deleteTree(&(*root).Left)
    deleteTree(&(*root).Right)
    fmt.Printf("Deleting node %d\n", (*root).Val)
    *root = nil // "free" in Go
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }

    deleteTree(&root)
    fmt.Println("Root is nil?", root == nil) // true
    // Order: 4, 5, 2, 3, 1 (children before parent)
}
```

**Textual Figure:**
```
         1
        / \
       2   3
      / \
     4   5

  Postorder Deletion (children before parent):
  ┌─────────────────────────────────────────┐
  │  Numbered deletion order:               │
  │                                         │
  │       1 ⑤        ← deleted last         │
  │      / \                                │
  │     2③  3④                              │
  │    / \                                  │
  │   4①  5②        ← deleted first         │
  │                                         │
  │  Step 1: Deleting node 4  (leaf)        │
  │  Step 2: Deleting node 5  (leaf)        │
  │  Step 3: Deleting node 2  (now leaf)    │
  │  Step 4: Deleting node 3  (leaf)        │
  │  Step 5: Deleting node 1  (now leaf)    │
  │                                         │
  │  Root is nil? → true                    │
  └─────────────────────────────────────────┘
```

---

## Example 5: Evaluate Expression Tree

```go
package main

import (
    "fmt"
    "strconv"
)

type ExprNode struct {
    Val   string
    Left  *ExprNode
    Right *ExprNode
}

func evaluate(root *ExprNode) int {
    if root == nil {
        return 0
    }
    // Leaf = operand
    if root.Left == nil && root.Right == nil {
        val, _ := strconv.Atoi(root.Val)
        return val
    }
    // Postorder: evaluate children first
    left := evaluate(root.Left)
    right := evaluate(root.Right)

    switch root.Val {
    case "+": return left + right
    case "-": return left - right
    case "*": return left * right
    case "/": return left / right
    }
    return 0
}

func postorderExpr(root *ExprNode) string {
    if root == nil { return "" }
    if root.Left == nil { return root.Val }
    return postorderExpr(root.Left) + " " + postorderExpr(root.Right) + " " + root.Val
}

func main() {
    //       *
    //      / \
    //     +   -
    //    / \ / \
    //   3  2 8  4
    // = (3+2) * (8-4) = 5 * 4 = 20
    root := &ExprNode{"*",
        &ExprNode{"+", &ExprNode{"3", nil, nil}, &ExprNode{"2", nil, nil}},
        &ExprNode{"-", &ExprNode{"8", nil, nil}, &ExprNode{"4", nil, nil}},
    }

    fmt.Println("Postfix:", postorderExpr(root)) // 3 2 + 8 4 - *
    fmt.Println("Result:", evaluate(root))        // 20
}
```

**Textual Figure:**
```
         *
        / \
       +   -
      / \ / \
     3  2 8  4

  Postorder Evaluation (operands before operator):
  ┌─────────────────────────────────────────┐
  │ eval(*)                                 │
  │  ├─ eval(+)                             │
  │  │   ├─ eval(3) → 3                     │
  │  │   ├─ eval(2) → 2                     │
  │  │   └─ 3 + 2 = 5  ←──┐                │
  │  ├─ eval(-)            │                │
  │  │   ├─ eval(8) → 8    │                │
  │  │   ├─ eval(4) → 4    │                │
  │  │   └─ 8 - 4 = 4  ←──┤                │
  │  └─ 5 * 4 = 20  ←─────┘                │
  └─────────────────────────────────────────┘

  Postfix: 3 2 + 8 4 - *
  Result:  20
```

---

## Example 6: Binary Tree Maximum Path Sum (LeetCode 124)

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

    var postorder func(node *TreeNode) int
    postorder = func(node *TreeNode) int {
        if node == nil { return 0 }

        // Postorder: compute children first
        leftGain := max(0, postorder(node.Left))
        rightGain := max(0, postorder(node.Right))

        // Path through this node
        currentPath := node.Val + leftGain + rightGain
        if currentPath > maxSum {
            maxSum = currentPath
        }

        // Return max single-side gain to parent
        if leftGain > rightGain {
            return node.Val + leftGain
        }
        return node.Val + rightGain
    }

    postorder(root)
    return maxSum
}

func max(a, b int) int {
    if a > b { return a }
    return b
}

func main() {
    //       -10
    //       /  \
    //      9   20
    //         /  \
    //        15   7
    root := &TreeNode{-10,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println("Max path sum:", maxPathSum(root)) // 42 (15→20→7)

    root2 := &TreeNode{1, &TreeNode{2, nil, nil}, &TreeNode{3, nil, nil}}
    fmt.Println("Max path sum:", maxPathSum(root2)) // 6 (2→1→3)
}
```

**Textual Figure:**
```
       -10
       /  \
      9   20
         /  \
        15   7

  Postorder gain computation (bottom-up):
  ┌────────────────────────────────────────────────┐
  │ Node │ leftGain │ rightGain │ pathThru │ return │
  ├──────┼──────────┼───────────┼──────────┼────────┤
  │  9   │    0     │     0     │    9     │   9    │
  │  15  │    0     │     0     │   15     │  15    │
  │   7  │    0     │     0     │    7     │   7    │
  │  20  │   15     │     7     │   42 ★   │  35    │
  │ -10  │    9     │    35     │   34     │  25    │
  └──────┴──────────┴───────────┴──────────┴────────┘

  Best path: 15 → 20 → 7 = 42
  maxSum = 42
```

---

## Example 7: Count Nodes in Complete Binary Tree (LeetCode 222)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func countNodes(root *TreeNode) int {
    if root == nil { return 0 }
    // Postorder: count left, count right, add 1
    return countNodes(root.Left) + countNodes(root.Right) + 1
}

// Optimized O(log²n) for complete binary trees
func countNodesOptimized(root *TreeNode) int {
    if root == nil { return 0 }

    leftHeight := 0
    node := root
    for node != nil {
        leftHeight++
        node = node.Left
    }

    rightHeight := 0
    node = root
    for node != nil {
        rightHeight++
        node = node.Right
    }

    if leftHeight == rightHeight {
        return (1 << leftHeight) - 1 // perfect tree: 2^h - 1
    }

    return 1 + countNodesOptimized(root.Left) + countNodesOptimized(root.Right)
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \  /
    //   4  5 6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, nil},
    }
    fmt.Println("Count:", countNodes(root))          // 6
    fmt.Println("Optimized:", countNodesOptimized(root)) // 6
}
```

**Textual Figure:**
```
       1
      / \
     2   3
    / \  /
   4  5 6

  Postorder count: left + right + 1
  ┌──────┬───────────┬────────────┬────────┐
  │ Node │ leftCount │ rightCount │ total  │
  ├──────┼───────────┼────────────┼────────┤
  │  4   │     0     │     0      │   1    │
  │  5   │     0     │     0      │   1    │
  │  6   │     0     │     0      │   1    │
  │  2   │     1     │     1      │   3    │
  │  3   │     1     │     0      │   2    │
  │  1   │     3     │     2      │   6 ★  │
  └──────┴───────────┴────────────┴────────┘

  Optimized: leftHeight=3, rightHeight=2 → not perfect
  → recurse: left subtree(2) is perfect (h=2, 3 nodes)
  → right subtree(3) has 2 nodes → total = 3 + 2 + 1 = 6
```

---

## Example 8: Lowest Common Ancestor (LeetCode 236)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }

    // Postorder: check children first
    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)

    if left != nil && right != nil {
        return root // p and q are in different subtrees
    }
    if left != nil {
        return left
    }
    return right
}

func main() {
    //         3
    //        / \
    //       5   1
    //      / \ / \
    //     6  2 0  8
    //       / \
    //      7   4
    n4 := &TreeNode{4, nil, nil}
    n7 := &TreeNode{7, nil, nil}
    n2 := &TreeNode{2, n7, n4}
    n6 := &TreeNode{6, nil, nil}
    n5 := &TreeNode{5, n6, n2}
    n0 := &TreeNode{0, nil, nil}
    n8 := &TreeNode{8, nil, nil}
    n1 := &TreeNode{1, n0, n8}
    root := &TreeNode{3, n5, n1}

    fmt.Println("LCA(5,1):", lowestCommonAncestor(root, n5, n1).Val) // 3
    fmt.Println("LCA(5,4):", lowestCommonAncestor(root, n5, n4).Val) // 5
    fmt.Println("LCA(6,4):", lowestCommonAncestor(root, n6, n4).Val) // 5
}
```

**Textual Figure:**
```
         3
        / \
       5   1
      / \ / \
     6  2 0  8
       / \
      7   4

  LCA(5,1): Postorder search
  ┌─────────────────────────────────────────────────┐
  │ lca(3)                                          │
  │  ├─ left  = lca(5) → found 5 itself → return 5  │
  │  ├─ right = lca(1) → found 1 itself → return 1  │
  │  └─ left≠nil AND right≠nil → return 3  ★        │
  └─────────────────────────────────────────────────┘

  LCA(5,4): Postorder search
  ┌─────────────────────────────────────────────────┐
  │ lca(3)                                          │
  │  ├─ left  = lca(5) → found 5 itself → return 5  │
  │  │   (4 is descendant of 5, but 5 found first)  │
  │  ├─ right = lca(1) → nil                        │
  │  └─ only left≠nil → return 5  ★                 │
  └─────────────────────────────────────────────────┘

  LCA(6,4): both under node 5 → LCA = 5
```

---

## Example 9: Balanced Binary Tree Check (LeetCode 110)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isBalanced(root *TreeNode) bool {
    return checkHeight(root) != -1
}

func checkHeight(root *TreeNode) int {
    if root == nil { return 0 }

    // Postorder: check children first
    left := checkHeight(root.Left)
    if left == -1 { return -1 }

    right := checkHeight(root.Right)
    if right == -1 { return -1 }

    diff := left - right
    if diff < -1 || diff > 1 {
        return -1 // unbalanced
    }

    if left > right {
        return left + 1
    }
    return right + 1
}

func main() {
    //   Balanced:     3        Unbalanced:  1
    //               / \                      \
    //              9  20                       2
    //                /  \                       \
    //               15   7                       3
    balanced := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    unbalanced := &TreeNode{1, nil,
        &TreeNode{2, nil, &TreeNode{3, nil, nil}},
    }

    fmt.Println("Balanced?", isBalanced(balanced))     // true
    fmt.Println("Balanced?", isBalanced(unbalanced))   // false
}
```

**Textual Figure:**
```
  Balanced:             Unbalanced:
       3                    1
      / \                    \
     9  20                    2
       /  \                    \
      15   7                    3

  Postorder height check (returns -1 if unbalanced):

  Balanced tree:                 Unbalanced tree:
  ┌──────┬────┬────┬─────┐      ┌──────┬────┬────┬─────┐
  │ Node │ lH │ rH │ ret │      │ Node │ lH │ rH │ ret │
  ├──────┼────┼────┼─────┤      ├──────┼────┼────┼─────┤
  │   9  │  0 │  0 │  1  │      │   3  │  0 │  0 │  1  │
  │  15  │  0 │  0 │  1  │      │   2  │  0 │  1 │  2  │
  │   7  │  0 │  0 │  1  │      │   1  │  0 │  2 │ -1  │
  │  20  │  1 │  1 │  2  │      │      │    │    │  ✗  │
  │   3  │  1 │  2 │  3  │      └──────┴────┴────┴─────┘
  │      │ |1-2|≤1 │ OK │      diff = |0-2| = 2 > 1
  └──────┴────┴────┴─────┘      → unbalanced!

  Balanced? true    Balanced? false
```

---

## Example 10: N-ary Tree Postorder (LeetCode 590)

```go
package main

import "fmt"

type Node struct {
    Val      int
    Children []*Node
}

// Recursive
func postorderRecursive(root *Node) []int {
    if root == nil { return nil }
    result := []int{}
    for _, child := range root.Children {
        result = append(result, postorderRecursive(child)...)
    }
    result = append(result, root.Val)
    return result
}

// Iterative (reverse of modified preorder)
func postorderIterative(root *Node) []int {
    if root == nil { return nil }
    result := []int{}
    stack := []*Node{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)

        // Push children left to right
        for _, child := range node.Children {
            stack = append(stack, child)
        }
    }

    // Reverse
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }
    return result
}

func main() {
    //       1
    //     / | \
    //    3  2  4
    //   / \
    //  5   6
    root := &Node{1, []*Node{
        {3, []*Node{{5, nil}, {6, nil}}},
        {2, nil},
        {4, nil},
    }}

    fmt.Println("Recursive:", postorderRecursive(root))  // [5 6 3 2 4 1]
    fmt.Println("Iterative:", postorderIterative(root))  // [5 6 3 2 4 1]
}
```

**Textual Figure:**
```
       1
     / | \
    3  2  4
   / \
  5   6

  Recursive postorder (children L→R, then node):
  ┌────────────────────────────────────────┐
  │ post(1)                                │
  │  ├─ post(3)                            │
  │  │   ├─ post(5) → visit 5 ✓           │
  │  │   ├─ post(6) → visit 6 ✓           │
  │  │   └─ visit 3 ✓                     │
  │  ├─ post(2) → visit 2 ✓  (leaf)       │
  │  ├─ post(4) → visit 4 ✓  (leaf)       │
  │  └─ visit 1 ✓                         │
  └────────────────────────────────────────┘

  Iterative: modified preorder (Node→R→L = 1,4,2,3,6,5)
             reverse → postorder: [5, 6, 3, 2, 4, 1]
```

---

## Postorder Patterns

| Problem | Why Postorder? |
|---------|---------------|
| Delete tree | Must delete children before parent |
| Evaluate expression | Need operand values before applying operator |
| Height/balance | Need subtree heights before checking current |
| Max path sum | Need subtree gains before computing path through node |
| LCA | Need to check both subtrees before deciding |

## Key Takeaways

1. **Postorder = bottom-up** — children are processed before the parent
2. **Iterative postorder** is the hardest — use two stacks or a `lastVisited` pointer
3. **Single stack trick**: reverse a "modified preorder" (Node→Right→Left) to get postorder
4. Most **bottom-up tree DP** problems use postorder naturally (height, diameter, max path)
5. **LCA** is a postorder problem — check left/right subtrees, then decide at current node
6. **Expression trees**: postorder → postfix notation (RPN)

> **Next up:** Level Order Traversal →
