# Phase 8: Binary Trees — Inorder Traversal

## Overview

**Inorder** visits nodes in **Left → Node → Right** order. For a BST, inorder produces elements in **sorted ascending** order — this is the single most important property to remember.

```
         4
        / \
       2   6
      / \ / \
     1  3 5  7

Inorder: 1, 2, 3, 4, 5, 6, 7  ← sorted!
```

---

## Example 1: Recursive Inorder

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func inorderTraversal(root *TreeNode) []int {
    result := []int{}
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil { return }
        dfs(node.Left)
        result = append(result, node.Val)
        dfs(node.Right)
    }
    dfs(root)
    return result
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(inorderTraversal(root)) // [1 2 3 4 5 6 7]
}
```

---

## Example 2: Iterative Inorder with Stack

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func inorderIterative(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    cur := root

    for cur != nil || len(stack) > 0 {
        // Go as far left as possible
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        // Pop and process
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, cur.Val)
        // Move to right subtree
        cur = cur.Right
    }
    return result
}

func main() {
    root := &TreeNode{1,
        nil,
        &TreeNode{2, &TreeNode{3, nil, nil}, nil},
    }
    fmt.Println(inorderIterative(root)) // [1 3 2]
}
```

---

## Example 3: Validate BST Using Inorder (LeetCode 98)

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

func isValidBST(root *TreeNode) bool {
    prev := math.MinInt64
    stack := []*TreeNode{}
    cur := root

    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if cur.Val <= prev {
            return false
        }
        prev = cur.Val
        cur = cur.Right
    }
    return true
}

func main() {
    //   Valid BST:    5          Invalid:    5
    //               / \                    / \
    //              1   7                  1   4
    //                 / \                    / \
    //                6   8                  3   6
    valid := &TreeNode{5,
        &TreeNode{1, nil, nil},
        &TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{8, nil, nil}},
    }
    invalid := &TreeNode{5,
        &TreeNode{1, nil, nil},
        &TreeNode{4, &TreeNode{3, nil, nil}, &TreeNode{6, nil, nil}},
    }

    fmt.Println("Valid BST?", isValidBST(valid))    // true
    fmt.Println("Invalid BST?", isValidBST(invalid)) // false
}
```

---

## Example 4: Kth Smallest Element in BST (LeetCode 230)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func kthSmallest(root *TreeNode, k int) int {
    stack := []*TreeNode{}
    cur := root
    count := 0

    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        count++
        if count == k {
            return cur.Val
        }
        cur = cur.Right
    }
    return -1
}

func main() {
    //       5
    //      / \
    //     3   6
    //    / \
    //   2   4
    //  /
    // 1
    root := &TreeNode{5,
        &TreeNode{3,
            &TreeNode{2, &TreeNode{1, nil, nil}, nil},
            &TreeNode{4, nil, nil},
        },
        &TreeNode{6, nil, nil},
    }

    for k := 1; k <= 6; k++ {
        fmt.Printf("k=%d → %d\n", k, kthSmallest(root, k))
    }
    // 1,2,3,4,5,6
}
```

---

## Example 5: BST Iterator (LeetCode 173)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type BSTIterator struct {
    stack []*TreeNode
}

func NewBSTIterator(root *TreeNode) *BSTIterator {
    it := &BSTIterator{}
    it.pushLeft(root)
    return it
}

func (it *BSTIterator) pushLeft(node *TreeNode) {
    for node != nil {
        it.stack = append(it.stack, node)
        node = node.Left
    }
}

func (it *BSTIterator) Next() int {
    node := it.stack[len(it.stack)-1]
    it.stack = it.stack[:len(it.stack)-1]
    it.pushLeft(node.Right)
    return node.Val
}

func (it *BSTIterator) HasNext() bool {
    return len(it.stack) > 0
}

func main() {
    root := &TreeNode{7,
        &TreeNode{3, nil, nil},
        &TreeNode{15, &TreeNode{9, nil, nil}, &TreeNode{20, nil, nil}},
    }

    it := NewBSTIterator(root)
    fmt.Print("BST Iterator: ")
    for it.HasNext() {
        fmt.Printf("%d ", it.Next())
    }
    fmt.Println() // 3 7 9 15 20
}
```

---

## Example 6: Convert BST to Sorted Doubly Linked List

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode // prev in DLL
    Right *TreeNode // next in DLL
}

func treeToDoublyList(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }

    var first, prev *TreeNode

    var inorder func(node *TreeNode)
    inorder = func(node *TreeNode) {
        if node == nil {
            return
        }
        inorder(node.Left)

        if prev == nil {
            first = node // leftmost = first
        } else {
            prev.Right = node
            node.Left = prev
        }
        prev = node

        inorder(node.Right)
    }

    inorder(root)

    // Make circular
    first.Left = prev
    prev.Right = first

    return first
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{5, nil, nil},
    }

    head := treeToDoublyList(root)

    // Print forward
    fmt.Print("Forward:  ")
    cur := head
    for i := 0; i < 5; i++ {
        fmt.Printf("%d ", cur.Val)
        cur = cur.Right
    }
    fmt.Println() // 1 2 3 4 5

    // Print backward
    fmt.Print("Backward: ")
    cur = head.Left // last node
    for i := 0; i < 5; i++ {
        fmt.Printf("%d ", cur.Val)
        cur = cur.Left
    }
    fmt.Println() // 5 4 3 2 1
}
```

---

## Example 7: Morris Inorder Traversal (O(1) Space)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func morrisInorder(root *TreeNode) []int {
    result := []int{}
    cur := root

    for cur != nil {
        if cur.Left == nil {
            // No left subtree: visit and go right
            result = append(result, cur.Val)
            cur = cur.Right
        } else {
            // Find inorder predecessor
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                // Create thread
                pred.Right = cur
                cur = cur.Left
            } else {
                // Thread already exists: visit, remove thread
                pred.Right = nil
                result = append(result, cur.Val)
                cur = cur.Right
            }
        }
    }
    return result
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(morrisInorder(root)) // [1 2 3 4 5 6 7]
}
```

---

## Example 8: Recover BST — Two Swapped Nodes (LeetCode 99)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func recoverTree(root *TreeNode) {
    var first, second, prev *TreeNode

    var inorder func(node *TreeNode)
    inorder = func(node *TreeNode) {
        if node == nil { return }
        inorder(node.Left)

        if prev != nil && prev.Val > node.Val {
            if first == nil {
                first = prev
            }
            second = node
        }
        prev = node

        inorder(node.Right)
    }

    inorder(root)
    first.Val, second.Val = second.Val, first.Val
}

func inorderSlice(root *TreeNode) []int {
    if root == nil { return nil }
    r := inorderSlice(root.Left)
    r = append(r, root.Val)
    r = append(r, inorderSlice(root.Right)...)
    return r
}

func main() {
    //  Broken BST (3 and 1 swapped):
    //       3
    //      / \
    //     3   2   ← wrong: should be 1,3 swapped
    // Actually:
    //       1         →       3
    //      / \               / \
    //     3   2             1   2  (swap 1 and 3)
    root := &TreeNode{3,
        &TreeNode{1, nil, nil},
        &TreeNode{2, nil, nil},
    }
    // Swap 3 and 2 to break BST property
    // inorder should be [1,2,3] but we have broken version
    // Let's make a clear example:
    root2 := &TreeNode{1,
        &TreeNode{3, nil, nil},
        &TreeNode{2, nil, nil},
    }

    fmt.Println("Before:", inorderSlice(root2)) // [3 1 2]
    recoverTree(root2)
    fmt.Println("After: ", inorderSlice(root2)) // [1 2 3]
}
```

---

## Example 9: Inorder Successor in BST (LeetCode 285)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// O(h) solution
func inorderSuccessor(root, p *TreeNode) *TreeNode {
    var successor *TreeNode
    cur := root

    for cur != nil {
        if cur.Val > p.Val {
            successor = cur // potential successor
            cur = cur.Left
        } else {
            cur = cur.Right
        }
    }
    return successor
}

func inorderPredecessor(root, p *TreeNode) *TreeNode {
    var predecessor *TreeNode
    cur := root

    for cur != nil {
        if cur.Val < p.Val {
            predecessor = cur
            cur = cur.Right
        } else {
            cur = cur.Left
        }
    }
    return predecessor
}

func main() {
    //       20
    //      /  \
    //    10    30
    //   / \   / \
    //  5  15 25  35

    root := &TreeNode{20,
        &TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
        &TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
    }

    targets := []*TreeNode{
        root.Left,              // 10
        root.Left.Right,        // 15
        root.Right.Left,        // 25
        root.Right.Right,       // 35
    }

    for _, t := range targets {
        succ := inorderSuccessor(root, t)
        pred := inorderPredecessor(root, t)

        succVal, predVal := "nil", "nil"
        if succ != nil { succVal = fmt.Sprint(succ.Val) }
        if pred != nil { predVal = fmt.Sprint(pred.Val) }

        fmt.Printf("Node %d: predecessor=%s, successor=%s\n", t.Val, predVal, succVal)
    }
}
```

---

## Example 10: Minimum Distance Between BST Nodes (LeetCode 783)

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

func minDiffInBST(root *TreeNode) int {
    minDiff := math.MaxInt64
    prev := -1

    var inorder func(node *TreeNode)
    inorder = func(node *TreeNode) {
        if node == nil { return }
        inorder(node.Left)

        if prev >= 0 {
            diff := node.Val - prev
            if diff < minDiff {
                minDiff = diff
            }
        }
        prev = node.Val

        inorder(node.Right)
    }
    inorder(root)
    return minDiff
}

func main() {
    //     4
    //    / \
    //   2   6
    //  / \
    // 1   3
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, nil, nil},
    }
    fmt.Println("Min difference:", minDiffInBST(root)) // 1

    // Tree: [1, nil, 3, nil, nil, 2]  → 1, 2, 3 → min diff = 1
    root2 := &TreeNode{1, nil, &TreeNode{3, &TreeNode{2, nil, nil}, nil}}
    fmt.Println("Min difference:", minDiffInBST(root2)) // 1
}
```

---

## Inorder Patterns

| Problem | Key Insight |
|---------|------------|
| Validate BST | Inorder must be strictly increasing |
| Kth smallest | Stop at kth node in inorder |
| Recover BST | Find two swapped elements in inorder |
| BST to sorted list | Inorder connects nodes in order |
| BST Iterator | Controlled inorder with stack |
| Min distance | Compare consecutive inorder values |
| Successor/Predecessor | Binary search guided by BST property |

## Key Takeaways

1. **Inorder of BST = sorted** — the most fundamental BST property
2. **Iterative pattern**: push all left children, pop & process, go right
3. **Morris traversal** achieves O(1) space by creating temporary right-thread links
4. **BST Iterator** is just a "paused" iterative inorder traversal
5. **Recover BST**: find exactly 2 violations in the sorted order during inorder
6. **Successor in BST**: smallest value greater than target → go right then leftmost, or walk up

> **Next up:** Postorder Traversal →
