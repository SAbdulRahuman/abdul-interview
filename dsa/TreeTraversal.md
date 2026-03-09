# Phase 8: Binary Trees — Tree Traversal (Overview)

## Overview

Tree traversal means visiting every node exactly once, in a specific order. There are two families:

| Family | Methods | Data Structure |
|--------|---------|---------------|
| **Depth-First (DFS)** | Preorder, Inorder, Postorder | Stack (or recursion) |
| **Breadth-First (BFS)** | Level-order | Queue |

```
         1
        / \
       2   3
      / \
     4   5

Preorder  (NLR): 1 2 4 5 3
Inorder   (LNR): 4 2 5 1 3
Postorder (LRN): 4 5 2 3 1
Level-order:     1 2 3 4 5
```

---

## Example 1: All Four Traversals Side-by-Side

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func preorder(root *TreeNode, result *[]int) {
    if root == nil { return }
    *result = append(*result, root.Val)
    preorder(root.Left, result)
    preorder(root.Right, result)
}

func inorder(root *TreeNode, result *[]int) {
    if root == nil { return }
    inorder(root.Left, result)
    *result = append(*result, root.Val)
    inorder(root.Right, result)
}

func postorder(root *TreeNode, result *[]int) {
    if root == nil { return }
    postorder(root.Left, result)
    postorder(root.Right, result)
    *result = append(*result, root.Val)
}

func levelOrder(root *TreeNode) []int {
    if root == nil { return nil }
    result := []int{}
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        result = append(result, node.Val)
        if node.Left != nil { queue = append(queue, node.Left) }
        if node.Right != nil { queue = append(queue, node.Right) }
    }
    return result
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

    var pre, in, post []int
    preorder(root, &pre)
    inorder(root, &in)
    postorder(root, &post)
    level := levelOrder(root)

    fmt.Println("Preorder: ", pre)   // [1 2 4 5 3 6]
    fmt.Println("Inorder:  ", in)    // [4 2 5 1 3 6]
    fmt.Println("Postorder:", post)  // [4 5 2 6 3 1]
    fmt.Println("Level:    ", level) // [1 2 3 4 5 6]
}
```

---

## Example 2: Traversal with Callbacks (Visitor Pattern)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type TraversalOrder int

const (
    Pre TraversalOrder = iota
    In
    Post
)

func traverse(root *TreeNode, order TraversalOrder, visit func(int)) {
    if root == nil {
        return
    }
    if order == Pre { visit(root.Val) }
    traverse(root.Left, order, visit)
    if order == In { visit(root.Val) }
    traverse(root.Right, order, visit)
    if order == Post { visit(root.Val) }
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }

    orders := []struct {
        name  string
        order TraversalOrder
    }{
        {"Preorder", Pre},
        {"Inorder", In},
        {"Postorder", Post},
    }

    for _, o := range orders {
        fmt.Printf("%-10s: ", o.name)
        traverse(root, o.order, func(val int) {
            fmt.Printf("%d ", val)
        })
        fmt.Println()
    }
}
```

---

## Example 3: Iterative DFS — Unified Template

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type Command struct {
    Node    *TreeNode
    Process bool // true = process this node's value; false = traverse
}

func iterativeTraversal(root *TreeNode, order string) []int {
    if root == nil {
        return nil
    }
    result := []int{}
    stack := []Command{{root, false}}

    for len(stack) > 0 {
        cmd := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if cmd.Node == nil {
            continue
        }

        if cmd.Process {
            result = append(result, cmd.Node.Val)
            continue
        }

        // Push in REVERSE order (stack is LIFO)
        switch order {
        case "preorder": // NLR → push R, L, N(process)
            stack = append(stack, Command{cmd.Node.Right, false})
            stack = append(stack, Command{cmd.Node.Left, false})
            stack = append(stack, Command{cmd.Node, true})
        case "inorder": // LNR → push R, N(process), L
            stack = append(stack, Command{cmd.Node.Right, false})
            stack = append(stack, Command{cmd.Node, true})
            stack = append(stack, Command{cmd.Node.Left, false})
        case "postorder": // LRN → push N(process), R, L
            stack = append(stack, Command{cmd.Node, true})
            stack = append(stack, Command{cmd.Node.Right, false})
            stack = append(stack, Command{cmd.Node.Left, false})
        }
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }

    for _, order := range []string{"preorder", "inorder", "postorder"} {
        fmt.Printf("%-10s: %v\n", order, iterativeTraversal(root, order))
    }
}
```

**Why?** This unified template uses a "command" stack pattern — one approach for all three DFS orders.

---

## Example 4: Traversal to Build String / Reconstruct Tree

```go
package main

import (
    "fmt"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Preorder + Inorder → Unique tree (LeetCode 105)
func buildTree(preorder, inorder []int) *TreeNode {
    if len(preorder) == 0 {
        return nil
    }
    rootVal := preorder[0]
    root := &TreeNode{Val: rootVal}

    mid := 0
    for i, v := range inorder {
        if v == rootVal {
            mid = i
            break
        }
    }

    root.Left = buildTree(preorder[1:1+mid], inorder[:mid])
    root.Right = buildTree(preorder[1+mid:], inorder[mid+1:])
    return root
}

func preorderStr(root *TreeNode) string {
    if root == nil { return "" }
    parts := []string{fmt.Sprint(root.Val)}
    if l := preorderStr(root.Left); l != "" { parts = append(parts, l) }
    if r := preorderStr(root.Right); r != "" { parts = append(parts, r) }
    return strings.Join(parts, ",")
}

func inorderStr(root *TreeNode) string {
    if root == nil { return "" }
    parts := []string{}
    if l := inorderStr(root.Left); l != "" { parts = append(parts, l) }
    parts = append(parts, fmt.Sprint(root.Val))
    if r := inorderStr(root.Right); r != "" { parts = append(parts, r) }
    return strings.Join(parts, ",")
}

func main() {
    pre := []int{3, 9, 20, 15, 7}
    in := []int{9, 3, 15, 20, 7}

    root := buildTree(pre, in)
    fmt.Println("Rebuilt preorder: ", preorderStr(root))
    fmt.Println("Rebuilt inorder:  ", inorderStr(root))
}
```

---

## Example 5: Count Nodes at Each Traversal Step

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func traceTraversal(root *TreeNode, name string) {
    step := 0

    var trace func(node *TreeNode, depth int)
    trace = func(node *TreeNode, depth int) {
        if node == nil {
            return
        }
        indent := ""
        for i := 0; i < depth; i++ {
            indent += "  "
        }

        if name == "preorder" {
            step++
            fmt.Printf("%s%sStep %d: Visit %d\n", indent, "", step, node.Val)
        }
        trace(node.Left, depth+1)
        if name == "inorder" {
            step++
            fmt.Printf("%s%sStep %d: Visit %d\n", indent, "", step, node.Val)
        }
        trace(node.Right, depth+1)
        if name == "postorder" {
            step++
            fmt.Printf("%s%sStep %d: Visit %d\n", indent, "", step, node.Val)
        }
    }

    fmt.Printf("=== %s ===\n", name)
    trace(root, 0)
    fmt.Println()
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }
    traceTraversal(root, "preorder")
    traceTraversal(root, "inorder")
    traceTraversal(root, "postorder")
}
```

---

## Example 6: DFS vs BFS — When to Choose Which

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// DFS: Find path (stack usage = O(h))
func findPathDFS(root *TreeNode, target int, path []int) ([]int, bool) {
    if root == nil {
        return nil, false
    }
    path = append(path, root.Val)
    if root.Val == target {
        return path, true
    }
    if p, ok := findPathDFS(root.Left, target, path); ok {
        return p, true
    }
    return findPathDFS(root.Right, target, path)
}

// BFS: Find shortest level (queue usage = O(w), where w = max width)
func findLevelBFS(root *TreeNode, target int) int {
    if root == nil {
        return -1
    }
    queue := []*TreeNode{root}
    level := 0
    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if node.Val == target {
                return level
            }
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        level++
    }
    return -1
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

    path, _ := findPathDFS(root, 5, nil)
    fmt.Println("DFS path to 5:", path)        // [1 2 5]
    fmt.Println("BFS level of 5:", findLevelBFS(root, 5)) // 2
}
```

---

## Example 7: N-ary Tree Traversals

```go
package main

import "fmt"

type NaryNode struct {
    Val      int
    Children []*NaryNode
}

func preorderNary(root *NaryNode, result *[]int) {
    if root == nil { return }
    *result = append(*result, root.Val)
    for _, child := range root.Children {
        preorderNary(child, result)
    }
}

func postorderNary(root *NaryNode, result *[]int) {
    if root == nil { return }
    for _, child := range root.Children {
        postorderNary(child, result)
    }
    *result = append(*result, root.Val)
}

func levelOrderNary(root *NaryNode) [][]int {
    if root == nil { return nil }
    result := [][]int{}
    queue := []*NaryNode{root}
    for len(queue) > 0 {
        size := len(queue)
        level := []int{}
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            queue = append(queue, node.Children...)
        }
        result = append(result, level)
    }
    return result
}

func main() {
    //       1
    //     / | \
    //    2  3  4
    //   /|     |
    //  5 6     7
    root := &NaryNode{1, []*NaryNode{
        {2, []*NaryNode{{5, nil}, {6, nil}}},
        {3, nil},
        {4, []*NaryNode{{7, nil}}},
    }}

    var pre, post []int
    preorderNary(root, &pre)
    postorderNary(root, &post)
    fmt.Println("Preorder: ", pre)                 // [1 2 5 6 3 4 7]
    fmt.Println("Postorder:", post)                // [5 6 2 3 7 4 1]
    fmt.Println("Levels:   ", levelOrderNary(root))// [[1] [2 3 4] [5 6 7]]
}
```

---

## Example 8: Zigzag (Spiral) Level Order (LeetCode 103)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func zigzagLevelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}
    leftToRight := true

    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, size)

        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]

            idx := i
            if !leftToRight {
                idx = size - 1 - i
            }
            level[idx] = node.Val

            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }

        result = append(result, level)
        leftToRight = !leftToRight
    }
    return result
}

func main() {
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(zigzagLevelOrder(root)) // [[3] [20 9] [15 7]]
}
```

---

## Example 9: Reverse Level Order

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func reverseLevelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        level := []int{}
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        result = append(result, level)
    }

    // Reverse
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }
    return result
}

func main() {
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    for i, level := range reverseLevelOrder(root) {
        fmt.Printf("Level %d: %v\n", i, level)
    }
    // Level 0: [15 7]
    // Level 1: [9 20]
    // Level 2: [3]
}
```

---

## Example 10: Traversal Order Determines Tree Properties

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

// Inorder of BST → sorted!
func isBST(root *TreeNode) bool {
    prev := math.MinInt64
    var check func(node *TreeNode) bool
    check = func(node *TreeNode) bool {
        if node == nil { return true }
        if !check(node.Left) { return false }
        if node.Val <= prev { return false }
        prev = node.Val
        return check(node.Right)
    }
    return check(root)
}

// Preorder determines tree structure (if BST)
func buildBSTFromPreorder(preorder []int) *TreeNode {
    idx := 0
    var build func(lo, hi int) *TreeNode
    build = func(lo, hi int) *TreeNode {
        if idx >= len(preorder) || preorder[idx] < lo || preorder[idx] > hi {
            return nil
        }
        val := preorder[idx]
        idx++
        node := &TreeNode{Val: val}
        node.Left = build(lo, val)
        node.Right = build(val, hi)
        return node
    }
    return build(math.MinInt64, math.MaxInt64)
}

func inorderSlice(root *TreeNode) []int {
    if root == nil { return nil }
    result := inorderSlice(root.Left)
    result = append(result, root.Val)
    result = append(result, inorderSlice(root.Right)...)
    return result
}

func main() {
    // Build BST from preorder
    pre := []int{8, 5, 1, 7, 10, 12}
    bst := buildBSTFromPreorder(pre)
    fmt.Println("Inorder (should be sorted):", inorderSlice(bst))
    fmt.Println("Is BST:", isBST(bst))

    // Non-BST
    notBST := &TreeNode{5,
        &TreeNode{10, nil, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println("Not BST:", isBST(notBST))
}
```

---

## Traversal Cheat Sheet

| Traversal | Order | Use Case | Time | Space |
|-----------|-------|----------|------|-------|
| Preorder | Node→Left→Right | Copy tree, serialize | O(n) | O(h) |
| Inorder | Left→Node→Right | BST sorted order | O(n) | O(h) |
| Postorder | Left→Right→Node | Delete tree, eval expr | O(n) | O(h) |
| Level-order | Level by level | Shortest path, width | O(n) | O(w) |

Where h = height, w = max width.

## Key Takeaways

1. **DFS** uses O(h) space (stack/recursion); **BFS** uses O(w) space (queue)
2. **Inorder on BST** produces sorted output — this is the most important fact
3. **Preorder + Inorder** (or **Postorder + Inorder**) uniquely determine a binary tree
4. Use the **unified command stack** template for all iterative DFS traversals
5. **Zigzag** and **reverse level** are variations of BFS with simple index manipulation
6. Choose DFS when searching deep paths, BFS when searching by proximity/level

> **Next up:** Preorder Traversal (deep dive) →
