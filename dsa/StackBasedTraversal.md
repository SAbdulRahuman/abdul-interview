# Phase 6: Stack — Stack-Based Traversal

## Overview

Stacks replace **recursion** for tree/graph traversal. The call stack is simulated explicitly, giving control over traversal order and avoiding stack overflow.

| Traversal | Recursive | Iterative (Stack) |
|-----------|-----------|-------------------|
| DFS Preorder | Natural | Push right first, then left |
| DFS Inorder | Natural | Go left, process, go right |
| DFS Postorder | Natural | Two stacks or modified approach |
| Graph DFS | Natural | Explicit stack |

---

## Example 1: Iterative Preorder Traversal

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
    //     1
    //    / \
    //   2   3
    //  / \   \
    // 4   5   6
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 4},
            Right: &TreeNode{Val: 5},
        },
        Right: &TreeNode{
            Val:   3,
            Right: &TreeNode{Val: 6},
        },
    }

    fmt.Println("Preorder:", preorderIterative(root))
    // [1, 2, 4, 5, 3, 6]
}
```

**Textual Figure: Iterative Preorder Traversal**

```
        1
       / \
      2   3
     / \   \
    4   5   6

Stack trace (push right first, then left):

  Step  Pop   Visit  Push (R then L)    Stack
  ────  ───   ─────  ──────────────    ─────
  init                                  [1]
  1     1     1      push 3, push 2     [3, 2]
  2     2     2      push 5, push 4     [3, 5, 4]
  3     4     4      (no children)      [3, 5]
  4     5     5      (no children)      [3]
  5     3     3      push 6             [6]
  6     6     6      (no children)      []

  Output: [1, 2, 4, 5, 3, 6]  (root → left subtree → right subtree)

  Key: push RIGHT before LEFT so LEFT is popped first (LIFO).
```

---

## Example 2: Iterative Inorder Traversal

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
    curr := root

    for curr != nil || len(stack) > 0 {
        // Go as far left as possible
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }

        // Process node
        curr = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, curr.Val)

        // Move to right subtree
        curr = curr.Right
    }

    return result
}

func main() {
    //     4
    //    / \
    //   2   6
    //  / \ / \
    // 1  3 5  7
    root := &TreeNode{
        Val: 4,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 1},
            Right: &TreeNode{Val: 3},
        },
        Right: &TreeNode{
            Val:   6,
            Left:  &TreeNode{Val: 5},
            Right: &TreeNode{Val: 7},
        },
    }

    fmt.Println("Inorder:", inorderIterative(root))
    // [1, 2, 3, 4, 5, 6, 7]
}
```

**Textual Figure: Iterative Inorder Traversal**

```
        4
       / \
      2   6
     / \ / \
    1  3 5  7

Stack trace (go left until nil, pop & process, go right):

  Step curr   Action              Stack         Output
  ──── ────   ──────              ─────         ──────
  1    4      push, go left       [4]           []
  2    2      push, go left       [4,2]         []
  3    1      push, go left       [4,2,1]       []
  4    nil    pop 1, process      [4,2]         [1]
              curr = 1.Right=nil
  5    nil    pop 2, process      [4]           [1,2]
              curr = 2.Right=3
  6    3      push, go left       [4,3]         [1,2]
  7    nil    pop 3, process      [4]           [1,2,3]
              curr = 3.Right=nil
  8    nil    pop 4, process      []            [1,2,3,4]
              curr = 4.Right=6
  9    6      push, go left       [6]           [1,2,3,4]
  10   5      push, go left       [6,5]         [1,2,3,4]
  11   nil    pop 5, process      [6]           [1,2,3,4,5]
  12   nil    pop 6, process      []            [1,2,3,4,5,6]
              curr = 6.Right=7
  13   7      push                [7]           [1,2,3,4,5,6]
  14   nil    pop 7, process      []            [1,2,3,4,5,6,7]

  Output: [1, 2, 3, 4, 5, 6, 7]  (sorted for BST!)
```

---

## Example 3: Iterative Postorder Traversal (Two Stacks)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func postorderIterative(root *TreeNode) []int {
    if root == nil {
        return nil
    }

    stack1 := []*TreeNode{root}
    stack2 := []*TreeNode{}

    for len(stack1) > 0 {
        node := stack1[len(stack1)-1]
        stack1 = stack1[:len(stack1)-1]
        stack2 = append(stack2, node)

        if node.Left != nil {
            stack1 = append(stack1, node.Left)
        }
        if node.Right != nil {
            stack1 = append(stack1, node.Right)
        }
    }

    result := make([]int, len(stack2))
    for i := len(stack2) - 1; i >= 0; i-- {
        result[len(stack2)-1-i] = stack2[i].Val
    }
    return result
}

func main() {
    //     1
    //    / \
    //   2   3
    //  / \
    // 4   5
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 4},
            Right: &TreeNode{Val: 5},
        },
        Right: &TreeNode{Val: 3},
    }

    fmt.Println("Postorder:", postorderIterative(root))
    // [4, 5, 2, 3, 1]
}
```

**Textual Figure: Iterative Postorder (Two Stacks)**

```
        1
       / \
      2   3
     / \
    4   5

Two-stack approach:
  Stack1: process order    Stack2: collect in reverse

  Step  Pop S1   Push to S2   Push L,R to S1    S1         S2
  ────  ──────  ──────────  ──────────────  ──────     ────────
  init                                          [1]        []
  1     1        [1]          push 2, push 3    [2,3]      [1]
  2     3        [1,3]        (no children)     [2]        [1,3]
  3     2        [1,3,2]      push 4, push 5    [4,5]      [1,3,2]
  4     5        [1,3,2,5]    (no children)     [4]        [1,3,2,5]
  5     4        [1,3,2,5,4]  (no children)     []         [1,3,2,5,4]

  Read S2 from top: [4, 5, 2, 3, 1]  ← postorder!

  Trick: S2 collects reverse-preorder (root→right→left)
         Reversed = postorder (left→right→root)
```

---

## Example 4: Postorder with Single Stack

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func postorderSingleStack(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    var lastVisited *TreeNode
    curr := root

    for curr != nil || len(stack) > 0 {
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }

        peek := stack[len(stack)-1]

        // If right child exists and hasn't been visited
        if peek.Right != nil && peek.Right != lastVisited {
            curr = peek.Right
        } else {
            stack = stack[:len(stack)-1]
            result = append(result, peek.Val)
            lastVisited = peek
        }
    }

    return result
}

func main() {
    //     1
    //    / \
    //   2   3
    //  / \
    // 4   5
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 4},
            Right: &TreeNode{Val: 5},
        },
        Right: &TreeNode{Val: 3},
    }

    fmt.Println("Postorder:", postorderSingleStack(root))
    // [4, 5, 2, 3, 1]
}
```

**Textual Figure: Postorder with Single Stack**

```
        1
       / \
      2   3
     / \
    4   5

Use lastVisited to know when right subtree is done:

  Step  Stack(top→)  Peek  lastVisited  Action
  ────  ────────   ────  ───────────  ──────
  1     [1,2,4]     4     nil          4.R=nil, pop & visit 4
  2     [1,2]       2     4            2.R=5≠4, go right
  3     [1,2,5]     5     4            5.R=nil, pop & visit 5
  4     [1,2]       2     5            2.R=5=last! pop & visit 2
  5     [1]         1     2            1.R=3≠2, go right
  6     [1,3]       3     2            3.R=nil, pop & visit 3
  7     [1]         1     3            1.R=3=last! pop & visit 1

  Output: [4, 5, 2, 3, 1]

  Key: peek at top, if right child exists and ≠ lastVisited,
       go right. Otherwise, pop and process.
```

---

## Example 5: Iterative DFS on Graph

```go
package main

import "fmt"

func dfsIterative(graph map[int][]int, start int) []int {
    visited := map[int]bool{}
    stack := []int{start}
    order := []int{}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if visited[node] {
            continue
        }
        visited[node] = true
        order = append(order, node)

        // Push neighbors in reverse for left-to-right ordering
        neighbors := graph[node]
        for i := len(neighbors) - 1; i >= 0; i-- {
            if !visited[neighbors[i]] {
                stack = append(stack, neighbors[i])
            }
        }
    }

    return order
}

func main() {
    graph := map[int][]int{
        0: {1, 2},
        1: {3, 4},
        2: {5},
        3: {},
        4: {},
        5: {},
    }

    fmt.Println("DFS:", dfsIterative(graph, 0))
    // [0, 1, 3, 4, 2, 5]
}
```

**Textual Figure: Iterative DFS on Graph**

```
Graph (adjacency list):
  0 → [1, 2]
  1 → [3, 4]
  2 → [5]
  3 → []
  4 → []
  5 → []

      0
     / \
    1   2
   / \   \
  3   4   5

Stack trace (push neighbors in reverse):

  Step  Pop  Visited?  Push (reverse)   Stack       Order
  ────  ───  ────────  ────────────   ─────       ─────
  init                                  [0]         []
  1     0    new       push 2, push 1   [2, 1]      [0]
  2     1    new       push 4, push 3   [2, 4, 3]   [0,1]
  3     3    new       (no neighbors)   [2, 4]      [0,1,3]
  4     4    new       (no neighbors)   [2]         [0,1,3,4]
  5     2    new       push 5           [5]         [0,1,3,4,2]
  6     5    new       (no neighbors)   []          [0,1,3,4,2,5]

  DFS Order: [0, 1, 3, 4, 2, 5]
  Uses visited set to avoid revisiting nodes.
```

---

## Example 6: Flatten Binary Tree to Linked List (LeetCode 114)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func flatten(root *TreeNode) {
    if root == nil {
        return
    }

    stack := []*TreeNode{root}
    var prev *TreeNode

    for len(stack) > 0 {
        curr := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if curr.Right != nil {
            stack = append(stack, curr.Right)
        }
        if curr.Left != nil {
            stack = append(stack, curr.Left)
        }

        if prev != nil {
            prev.Left = nil
            prev.Right = curr
        }
        prev = curr
    }
}

func printFlattened(root *TreeNode) {
    for curr := root; curr != nil; curr = curr.Right {
        fmt.Printf("%d → ", curr.Val)
    }
    fmt.Println("nil")
}

func main() {
    //     1
    //    / \
    //   2   5
    //  / \   \
    // 3   4   6
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 3},
            Right: &TreeNode{Val: 4},
        },
        Right: &TreeNode{
            Val:   5,
            Right: &TreeNode{Val: 6},
        },
    }

    flatten(root)
    printFlattened(root)
    // 1 → 2 → 3 → 4 → 5 → 6 → nil
}
```

**Textual Figure: Flatten Binary Tree to Linked List**

```
Before:                    After (right-only linked list):
      1                    1 → 2 → 3 → 4 → 5 → 6 → nil
     / \
    2   5
   / \   \
  3   4   6

Stack-based preorder flattening:

  Step  Pop  prev   Push (R, L)    Stack       Linked list
  ────  ───  ────   ──────────    ─────       ───────────
  1     1    nil    push 5, 2      [5,2]       1
  2     2    1      push 4, 3      [5,4,3]     1→2
  3     3    2      (leaves)       [5,4]       1→2→3
  4     4    3      (leaves)       [5]         1→2→3→4
  5     5    4      push 6         [6]         1→2→3→4→5
  6     6    5      (leaves)       []          1→2→3→4→5→6

  Each node: prev.Left=nil, prev.Right=curr
  Result: preorder sequence as right-only list.
```

---

## Example 7: Binary Tree Zigzag Level Order (BFS + Stack)

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

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }

        result = append(result, level)
        leftToRight = !leftToRight
    }

    return result
}

func main() {
    //     3
    //    / \
    //   9  20
    //     /  \
    //    15   7
    root := &TreeNode{
        Val:  3,
        Left: &TreeNode{Val: 9},
        Right: &TreeNode{
            Val:   20,
            Left:  &TreeNode{Val: 15},
            Right: &TreeNode{Val: 7},
        },
    }

    fmt.Println(zigzagLevelOrder(root))
    // [[3], [20, 9], [15, 7]]
}
```

**Textual Figure: Binary Tree Zigzag Level Order**

```
        3
       / \
      9  20
        /  \
       15   7

Level-by-level with alternating direction:

  Level 0 (L→R):  [3]
  Level 1 (R→L):  [20, 9]     (reversed!)
  Level 2 (L→R):  [15, 7]

  ┌─────────────────────┐
  │ Level 0:  3            │  →→→
  │ Level 1:  20  9        │  ←←←
  │ Level 2:  15  7        │  →→→
  └─────────────────────┘

  BFS with queue, toggle leftToRight flag each level.
  On even levels: fill left-to-right.
  On odd levels:  fill right-to-left (idx = size-1-i).
```

---

## Example 8: BST Iterator (LeetCode 173) — Controlled Inorder

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
    //     7
    //    / \
    //   3  15
    //     /  \
    //    9   20
    root := &TreeNode{
        Val:  7,
        Left: &TreeNode{Val: 3},
        Right: &TreeNode{
            Val:   15,
            Left:  &TreeNode{Val: 9},
            Right: &TreeNode{Val: 20},
        },
    }

    it := NewBSTIterator(root)
    for it.HasNext() {
        fmt.Printf("%d ", it.Next())
    }
    fmt.Println() // 3 7 9 15 20
}
```

**Textual Figure: BST Iterator (Controlled Inorder)**

```
        7
       / \
      3  15
        /  \
       9   20

Initial pushLeft(7): push 7, then push 3 (leftmost path)

  Stack: [7, 3]    (3 on top)

  Call     Pop   pushLeft(right)  Stack     Output
  ────     ───   ─────────────  ─────     ──────
  Next()   3     3.R=nil          [7]       3
  Next()   7     pushLeft(15)     [15,9]    7
                  →push 15, push 9
  Next()   9     9.R=nil          [15]      9
  Next()   15    pushLeft(20)     [20]      15
  Next()   20    20.R=nil         []        20

  Output: 3, 7, 9, 15, 20  (sorted inorder!)

  ┌─────────────────────────────┐
  │ pushLeft: follow left chain  │
  │ Next: pop, pushLeft(right)   │
  │ O(h) space, amortized O(1)   │
  └─────────────────────────────┘
```

---

## Example 9: Path Sum II — Iterative DFS with Path Tracking

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
    Path []int
    Sum  int
}

func pathSum(root *TreeNode, targetSum int) [][]int {
    if root == nil {
        return nil
    }

    result := [][]int{}
    stack := []StackItem{{Node: root, Path: []int{root.Val}, Sum: root.Val}}

    for len(stack) > 0 {
        item := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        // Leaf node check
        if item.Node.Left == nil && item.Node.Right == nil {
            if item.Sum == targetSum {
                pathCopy := make([]int, len(item.Path))
                copy(pathCopy, item.Path)
                result = append(result, pathCopy)
            }
            continue
        }

        if item.Node.Right != nil {
            newPath := make([]int, len(item.Path)+1)
            copy(newPath, item.Path)
            newPath[len(item.Path)] = item.Node.Right.Val
            stack = append(stack, StackItem{
                Node: item.Node.Right,
                Path: newPath,
                Sum:  item.Sum + item.Node.Right.Val,
            })
        }
        if item.Node.Left != nil {
            newPath := make([]int, len(item.Path)+1)
            copy(newPath, item.Path)
            newPath[len(item.Path)] = item.Node.Left.Val
            stack = append(stack, StackItem{
                Node: item.Node.Left,
                Path: newPath,
                Sum:  item.Sum + item.Node.Left.Val,
            })
        }
    }

    return result
}

func main() {
    //       5
    //      / \
    //     4   8
    //    /   / \
    //   11  13  4
    //  / \     / \
    // 7   2   5   1
    root := &TreeNode{
        Val: 5,
        Left: &TreeNode{
            Val: 4,
            Left: &TreeNode{
                Val:   11,
                Left:  &TreeNode{Val: 7},
                Right: &TreeNode{Val: 2},
            },
        },
        Right: &TreeNode{
            Val:  8,
            Left: &TreeNode{Val: 13},
            Right: &TreeNode{
                Val:   4,
                Left:  &TreeNode{Val: 5},
                Right: &TreeNode{Val: 1},
            },
        },
    }

    paths := pathSum(root, 22)
    fmt.Println("Paths summing to 22:")
    for _, p := range paths {
        fmt.Println(p)
    }
    // [5, 4, 11, 2]
    // [5, 8, 4, 5]
}
```

**Textual Figure: Path Sum II — Iterative DFS**

```
         5           target = 22
        / \
       4   8
      /   / \
     11  13  4
    / \     / \
   7   2   5   1

Stack items carry (node, path, sum):

  Pop Item                    Leaf?  Sum=22?
  ────────                    ─────  ──────
  (5, [5], 5)                 no
    push (4,[5,4],9)
    push (8,[5,8],13)
  (4, [5,4], 9)               no
    push (11,[5,4,11],20)
  (11, [5,4,11], 20)          no
    push (7,[5,4,11,7],27)
    push (2,[5,4,11,2],22)
  (2, [5,4,11,2], 22)         yes    22=22 ✓ FOUND!
  (7, [5,4,11,7], 27)         yes    27≠22 ✘
  (8, [5,8], 13)              no
    push (13,[5,8,13],21)
    push (4,[5,8,4],17)
  (4, [5,8,4], 17)            no
    push (5,[5,8,4,5],22)
    push (1,[5,8,4,1],18)
  (1, [5,8,4,1], 18)          yes    18≠22 ✘
  (5, [5,8,4,5], 22)          yes    22=22 ✓ FOUND!
  (13, [5,8,13], 21)          yes    21≠22 ✘

  Paths: [5,4,11,2] and [5,8,4,5]
```

---

## Example 10: N-ary Tree Preorder (Iterative)

```go
package main

import "fmt"

type Node struct {
    Val      int
    Children []*Node
}

func preorder(root *Node) []int {
    if root == nil {
        return nil
    }

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
    root := &Node{
        Val: 1,
        Children: []*Node{
            {
                Val: 3,
                Children: []*Node{
                    {Val: 5},
                    {Val: 6},
                },
            },
            {Val: 2},
            {Val: 4},
        },
    }

    fmt.Println("Preorder:", preorder(root))
    // [1, 3, 5, 6, 2, 4]
}
```

**Textual Figure: N-ary Tree Preorder (Iterative)**

```
        1
      / | \
     3  2  4
    / \
   5   6

Stack trace (push children in reverse order):

  Step  Pop  Visit  Push children (reverse)  Stack
  ────  ───  ─────  ───────────────────  ─────
  init                                       [1]
  1     1    1      push 4, 2, 3             [4, 2, 3]
  2     3    3      push 6, 5               [4, 2, 6, 5]
  3     5    5      (no children)            [4, 2, 6]
  4     6    6      (no children)            [4, 2]
  5     2    2      (no children)            [4]
  6     4    4      (no children)            []

  Output: [1, 3, 5, 6, 2, 4]

  Same idea as binary tree preorder:
  push children in reverse so leftmost is popped first.
```

---

## Example 11: Iterative Deepening DFS (IDDFS)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// DFS with depth limit — uses stack
func depthLimitedDFS(root *TreeNode, target, limit int) bool {
    type Item struct {
        Node  *TreeNode
        Depth int
    }

    stack := []Item{{root, 0}}

    for len(stack) > 0 {
        item := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if item.Node == nil {
            continue
        }

        fmt.Printf("  Visiting %d at depth %d\n", item.Node.Val, item.Depth)

        if item.Node.Val == target {
            return true
        }

        if item.Depth < limit {
            if item.Node.Right != nil {
                stack = append(stack, Item{item.Node.Right, item.Depth + 1})
            }
            if item.Node.Left != nil {
                stack = append(stack, Item{item.Node.Left, item.Depth + 1})
            }
        }
    }

    return false
}

// IDDFS: repeat DFS with increasing depth limits
func iddfs(root *TreeNode, target int) int {
    for depth := 0; depth < 100; depth++ {
        fmt.Printf("--- Depth limit: %d ---\n", depth)
        if depthLimitedDFS(root, target, depth) {
            return depth
        }
    }
    return -1
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \
    //   4   5
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:   2,
            Left:  &TreeNode{Val: 4},
            Right: &TreeNode{Val: 5},
        },
        Right: &TreeNode{Val: 3},
    }

    foundDepth := iddfs(root, 5)
    fmt.Printf("\nFound at depth: %d\n", foundDepth)
}
```

**Textual Figure: Iterative Deepening DFS (IDDFS)**

```
        1
       / \
      2   3
     / \
    4   5   ← target

IDDFS: repeat depth-limited DFS with increasing limits:

  Depth limit 0:  Visit 1 only
    Stack: [(1,d=0)] → visit 1 → not target, d=0=limit, stop

  Depth limit 1:  Visit 1, 2, 3
    Stack: [(1,0)] → visit 1 → push (3,1),(2,1)
    Pop (2,1) → visit 2 → d=1=limit, stop
    Pop (3,1) → visit 3 → d=1=limit, stop

  Depth limit 2:  Visit 1, 2, 4, 5 ← FOUND!
    Stack: [(1,0)] → visit 1 → push (3,1),(2,1)
    Pop (2,1) → visit 2 → push (5,2),(4,2)
    Pop (4,2) → visit 4 → not target
    Pop (5,2) → visit 5 → FOUND! return depth=2

  ┌──────────────────────────────────────────┐
  │ IDDFS = DFS space O(d) + BFS optimality │
  │ Finds shallowest target first            │
  │ Overhead: re-visiting upper levels        │
  └──────────────────────────────────────────┘
```

---

## Traversal Comparison

| Traversal | Stack State | Process When |
|-----------|-------------|-------------|
| Preorder | Push right, then left | On pop |
| Inorder | Push all left first | After exhausting left |
| Postorder | Two stacks or last-visited | After both children done |
| Level order (BFS) | Queue, not stack | Level by level |

## Key Takeaways

1. **Preorder** is simplest iteratively: pop → process → push right then left
2. **Inorder** uses a `curr` pointer: go left until nil, pop and process, go right
3. **Postorder** is hardest: two-stack approach or track `lastVisited` node
4. **Graph DFS**: same as tree but with a `visited` set to avoid cycles
5. **BST Iterator**: controlled inorder using `pushLeft` helper — O(h) space
6. **IDDFS**: combines DFS space efficiency with BFS completeness guarantee
7. Converting recursion to iteration: store function parameters in a stack struct

> **Next up:** Call Stack Simulation →
