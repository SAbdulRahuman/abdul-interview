# Phase 8: Binary Trees — Preorder Traversal

## Overview

**Preorder** visits nodes in **Node → Left → Right** order. The root is always the first element. Useful for creating copies of trees, serialization, and prefix expressions.

```
         1
        / \
       2   3
      / \
     4   5

Preorder: 1, 2, 4, 5, 3
```

---

## Example 1: Recursive Preorder

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func preorderTraversal(root *TreeNode) []int {
    result := []int{}
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil {
            return
        }
        result = append(result, node.Val) // Visit
        dfs(node.Left)                    // Left
        dfs(node.Right)                   // Right
    }
    dfs(root)
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(preorderTraversal(root)) // [1 2 4 5 3]
}
```

---

## Example 2: Iterative Preorder with Stack

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func preorderIterative(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := []int{}
    stack := []*TreeNode{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)

        // Push right first so left is processed first
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(preorderIterative(root)) // [1 2 4 5 3 6 7]
}
```

---

## Example 3: Morris Preorder (O(1) Space)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func morrisPreorder(root *TreeNode) []int {
    result := []int{}
    cur := root

    for cur != nil {
        if cur.Left == nil {
            result = append(result, cur.Val)
            cur = cur.Right
        } else {
            // Find inorder predecessor
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                // First visit: add to result, create thread
                result = append(result, cur.Val)
                pred.Right = cur
                cur = cur.Left
            } else {
                // Thread exists: remove it, move right
                pred.Right = nil
                cur = cur.Right
            }
        }
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(morrisPreorder(root)) // [1 2 4 5 3]
}
```

---

## Example 4: Flatten Binary Tree to Linked List (LeetCode 114)

Preorder traversal determines the flattened order.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func flatten(root *TreeNode) {
    cur := root
    for cur != nil {
        if cur.Left != nil {
            // Find rightmost node of left subtree
            rightmost := cur.Left
            for rightmost.Right != nil {
                rightmost = rightmost.Right
            }
            // Rewire
            rightmost.Right = cur.Right
            cur.Right = cur.Left
            cur.Left = nil
        }
        cur = cur.Right
    }
}

func printList(root *TreeNode) {
    for root != nil {
        fmt.Printf("%d → ", root.Val)
        root = root.Right
    }
    fmt.Println("nil")
}

func main() {
    //       1
    //      / \
    //     2   5
    //    / \   \
    //   3   4   6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{3, nil, nil}, &TreeNode{4, nil, nil}},
        &TreeNode{5, nil, &TreeNode{6, nil, nil}},
    }

    flatten(root)
    printList(root) // 1 → 2 → 3 → 4 → 5 → 6 → nil
}
```

---

## Example 5: Construct BST from Preorder (LeetCode 1008)

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

func bstFromPreorder(preorder []int) *TreeNode {
    idx := 0
    var build func(bound int) *TreeNode
    build = func(bound int) *TreeNode {
        if idx >= len(preorder) || preorder[idx] > bound {
            return nil
        }
        val := preorder[idx]
        idx++
        node := &TreeNode{Val: val}
        node.Left = build(val)
        node.Right = build(bound)
        return node
    }
    return build(math.MaxInt64)
}

func preorder(root *TreeNode) []int {
    if root == nil { return nil }
    result := []int{root.Val}
    result = append(result, preorder(root.Left)...)
    result = append(result, preorder(root.Right)...)
    return result
}

func main() {
    pre := []int{8, 5, 1, 7, 10, 12}
    root := bstFromPreorder(pre)
    fmt.Println("Reconstructed:", preorder(root)) // [8 5 1 7 10 12]
}
```

---

## Example 6: Clone a Binary Tree (Preorder Copy)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func cloneTree(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    // Preorder: copy root first, then left, then right
    clone := &TreeNode{Val: root.Val}
    clone.Left = cloneTree(root.Left)
    clone.Right = cloneTree(root.Right)
    return clone
}

func isSameTree(p, q *TreeNode) bool {
    if p == nil && q == nil { return true }
    if p == nil || q == nil { return false }
    return p.Val == q.Val && isSameTree(p.Left, q.Left) && isSameTree(p.Right, q.Right)
}

func main() {
    original := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }

    clone := cloneTree(original)

    fmt.Println("Same structure?", isSameTree(original, clone)) // true
    fmt.Println("Same object?", original == clone)                // false

    // Modify clone doesn't affect original
    clone.Left.Val = 99
    fmt.Println("Original left:", original.Left.Val) // 2
    fmt.Println("Clone left:", clone.Left.Val)       // 99
}
```

---

## Example 7: Construct Binary Tree from Preorder and Inorder (LeetCode 105)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func buildTree(preorder []int, inorder []int) *TreeNode {
    inMap := map[int]int{}
    for i, v := range inorder {
        inMap[v] = i
    }

    preIdx := 0
    var build func(left, right int) *TreeNode
    build = func(left, right int) *TreeNode {
        if left > right {
            return nil
        }
        rootVal := preorder[preIdx]
        preIdx++

        node := &TreeNode{Val: rootVal}
        mid := inMap[rootVal]
        node.Left = build(left, mid-1)
        node.Right = build(mid+1, right)
        return node
    }
    return build(0, len(inorder)-1)
}

func printPreorder(root *TreeNode) {
    if root == nil { return }
    fmt.Printf("%d ", root.Val)
    printPreorder(root.Left)
    printPreorder(root.Right)
}

func main() {
    pre := []int{3, 9, 20, 15, 7}
    in := []int{9, 3, 15, 20, 7}

    root := buildTree(pre, in)
    fmt.Print("Rebuilt preorder: ")
    printPreorder(root) // 3 9 20 15 7
    fmt.Println()
}
```

---

## Example 8: Serialize Tree Using Preorder

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

func serialize(root *TreeNode) string {
    parts := []string{}
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil {
            parts = append(parts, "#")
            return
        }
        parts = append(parts, strconv.Itoa(node.Val))
        dfs(node.Left)
        dfs(node.Right)
    }
    dfs(root)
    return strings.Join(parts, ",")
}

func deserialize(data string) *TreeNode {
    parts := strings.Split(data, ",")
    idx := 0
    var dfs func() *TreeNode
    dfs = func() *TreeNode {
        if idx >= len(parts) || parts[idx] == "#" {
            idx++
            return nil
        }
        val, _ := strconv.Atoi(parts[idx])
        idx++
        node := &TreeNode{Val: val}
        node.Left = dfs()
        node.Right = dfs()
        return node
    }
    return dfs()
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
    }

    encoded := serialize(root)
    fmt.Println("Serialized:", encoded) // 1,2,#,#,3,4,#,#,5,#,#

    decoded := deserialize(encoded)
    fmt.Println("Reserialized:", serialize(decoded)) // same
}
```

---

## Example 9: N-ary Tree Preorder (LeetCode 589)

```go
package main

import "fmt"

type Node struct {
    Val      int
    Children []*Node
}

// Recursive
func preorderRecursive(root *Node) []int {
    if root == nil { return nil }
    result := []int{root.Val}
    for _, child := range root.Children {
        result = append(result, preorderRecursive(child)...)
    }
    return result
}

// Iterative
func preorderIterative(root *Node) []int {
    if root == nil { return nil }
    result := []int{}
    stack := []*Node{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)

        // Push children in reverse order
        for i := len(node.Children) - 1; i >= 0; i-- {
            stack = append(stack, node.Children[i])
        }
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

    fmt.Println("Recursive:", preorderRecursive(root))  // [1 3 5 6 2 4]
    fmt.Println("Iterative:", preorderIterative(root))  // [1 3 5 6 2 4]
}
```

---

## Example 10: Preorder in Expression Trees

```go
package main

import "fmt"

type ExprNode struct {
    Val   string
    Left  *ExprNode
    Right *ExprNode
}

// Preorder of expression tree → prefix notation
func toPrefix(root *ExprNode) string {
    if root == nil { return "" }
    if root.Left == nil && root.Right == nil {
        return root.Val // operand
    }
    return root.Val + " " + toPrefix(root.Left) + " " + toPrefix(root.Right)
}

// Inorder → infix
func toInfix(root *ExprNode) string {
    if root == nil { return "" }
    if root.Left == nil { return root.Val }
    return "(" + toInfix(root.Left) + " " + root.Val + " " + toInfix(root.Right) + ")"
}

func main() {
    //       +
    //      / \
    //     *   3
    //    / \
    //   1   2
    // Expression: (1 * 2) + 3
    root := &ExprNode{"+",
        &ExprNode{"*",
            &ExprNode{"1", nil, nil},
            &ExprNode{"2", nil, nil},
        },
        &ExprNode{"3", nil, nil},
    }

    fmt.Println("Prefix:", toPrefix(root))  // + * 1 2 3
    fmt.Println("Infix:", toInfix(root))    // ((1 * 2) + 3)
}
```

---

## Key Takeaways

1. **Preorder** visits root FIRST → first element is always the root
2. **Iterative**: push right child before left → left pops first (LIFO)
3. **Morris preorder**: O(1) space by creating temporary thread links
4. **Serialization**: preorder with null markers uniquely encodes any binary tree
5. **Flatten to linked list**: preorder order becomes the right-pointer chain
6. **BST from preorder**: use upper bound to partition left/right subtrees
7. **Expression tree**: preorder → prefix notation; postorder → postfix

> **Next up:** Inorder Traversal →
