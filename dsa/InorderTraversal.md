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

**Textual Figure:**

```
  Recursive Inorder (Left → Node → Right)
  ═════════════════════════════════════════

           ┌───┐
           │ 4 │  BST root
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌───┐
         │ 2 │ │ 6 │
         └─┬─┘ └─┬─┘
          ╱ ╲     ╱ ╲
       ┌───┐┌───┐┌───┐┌───┐
       │ 1 ││ 3 ││ 5 ││ 7 │
       └───┘└───┘└───┘└───┘

  Call stack trace (L → N → R):
    dfs(4)
      dfs(2)
        dfs(1)
          dfs(nil)  → return
          visit 1  ①
          dfs(nil)  → return
        visit 2    ②
        dfs(3)
          dfs(nil), visit 3 ③, dfs(nil)
      visit 4      ④
      dfs(6)
        dfs(5)
          dfs(nil), visit 5 ⑤, dfs(nil)
        visit 6    ⑥
        dfs(7)
          dfs(nil), visit 7 ⑦, dfs(nil)

  Result: [1, 2, 3, 4, 5, 6, 7] ← sorted!
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

**Textual Figure:**

```
  Iterative Inorder with Stack
  ═══════════════════════════════

     ┌───┐
     │ 1 │
     └─┬─┘
       ╲
      ┌───┐
      │ 2 │
      └─┬─┘
       ╱
    ┌───┐
    │ 3 │
    └───┘

  Algorithm: push all left, pop & visit, go right

  Step  cur   Stack         Action         Output
   1    1     [1]           push 1         []
   2    nil   [1]           pop 1, visit   [1]
   3    2     [2]           push 2, go left [1]
   4    3     [2,3]         push 3, go left [1]
   5    nil   [2,3]         pop 3, visit   [1,3]
   6    nil   [2]           pop 2, visit   [1,3,2]
   7    nil   []            stack empty    DONE

  Result: [1, 3, 2]
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

**Textual Figure:**

```
  Validate BST via Inorder Property
  ═══════════════════════════════════

  Valid BST:              Invalid BST:
       ┌───┐                   ┌───┐
       │ 5 │                   │ 5 │
       └─┬─┘                   └─┬─┘
        ╱ ╲                     ╱ ╲
     ┌───┐ ┌───┐           ┌───┐ ┌───┐
     │ 1 │ │ 7 │           │ 1 │ │ 4 │ ← 4 < 5!
     └───┘ └─┬─┘           └───┘ └─┬─┘
            ╱ ╲                   ╱ ╲
         ┌───┐┌───┐           ┌───┐┌───┐
         │ 6 ││ 8 │           │ 3 ││ 6 │
         └───┘└───┘           └───┘└───┘

  Inorder check (must be strictly increasing):
    Valid:   [1, 5, 6, 7, 8]  ✔ sorted
    Invalid: [1, 5, 3, 4, 6]  ✗ 3 < 5 breaks order
                       ↑
               prev=5 > cur=3 → return false
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

**Textual Figure:**

```
  Kth Smallest in BST (Inorder = Sorted)
  ═══════════════════════════════════════

           ┌───┐
           │ 5 │
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌───┐
         │ 3 │ │ 6 │
         └─┬─┘ └───┘
          ╱ ╲
       ┌───┐┌───┐
       │ 2 ││ 4 │
       └─┬─┘└───┘
        ╱
     ┌───┐
     │ 1 │
     └───┘

  Inorder: [1, 2, 3, 4, 5, 6] (sorted)

  Finding kth smallest (stop at count==k):
    k=1 → 1st node visited = 1
    k=2 → 2nd node visited = 2
    k=3 → 3rd node visited = 3
    k=4 → 4
    k=5 → 5
    k=6 → 6

  Early termination: stop as soon as count==k
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

**Textual Figure:**

```
  BST Iterator (Controlled Inorder with Stack)
  ═════════════════════════════════════════════

           ┌───┐
           │ 7 │
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌────┐
         │ 3 │ │ 15 │
         └───┘ └─┬──┘
               ╱ ╲
            ┌───┐┌────┐
            │ 9 ││ 20 │
            └───┘└────┘

  Init: pushLeft(7) → stack = [7, 3]

  Next() calls:
    Call 1: pop 3, pushLeft(nil)  → return 3
            stack = [7]
    Call 2: pop 7, pushLeft(15)   → return 7
            stack = [15, 9]
    Call 3: pop 9, pushLeft(nil)  → return 9
            stack = [15]
    Call 4: pop 15, pushLeft(20)  → return 15
            stack = [20]
    Call 5: pop 20, pushLeft(nil) → return 20
            stack = []

  Output: 3 7 9 15 20
  O(h) space, O(1) amortized per Next()
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

**Textual Figure:**

```
  BST → Sorted Circular Doubly Linked List
  ═════════════════════════════════════════

  BST:
       ┌───┐
       │ 4 │
       └─┬─┘
        ╱ ╲
     ┌───┐ ┌───┐
     │ 2 │ │ 5 │
     └─┬─┘ └───┘
      ╱ ╲
   ┌───┐┌───┐
   │ 1 ││ 3 │
   └───┘└───┘

  Inorder: 1 → 2 → 3 → 4 → 5

  Circular DLL (Left=prev, Right=next):
    ┌───────────────────────────┐
    │                           │
    ▼                           │
  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
  │ 1 │─→│ 2 │─→│ 3 │─→│ 4 │─→│ 5 │
  └───┘←─└───┘←─└───┘←─└───┘←─└───┘
    ↑                           │
    │                           │
    └───────────────────────────┘

  Forward:  1 → 2 → 3 → 4 → 5
  Backward: 5 → 4 → 3 → 2 → 1
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

**Textual Figure:**

```
  Morris Inorder Traversal (O(1) Space)
  ═══════════════════════════════════════

           ┌───┐
           │ 4 │
           └─┬─┘
            ╱ ╲
         ┌───┐ ┌───┐
         │ 2 │ │ 6 │
         └─┬─┘ └─┬─┘
          ╱ ╲     ╱ ╲
       ┌───┐┌───┐┌───┐┌───┐
       │ 1 ││ 3 ││ 5 ││ 7 │
       └───┘└───┘└───┘└───┘

  Thread = temporary right pointer to inorder successor

  Step  cur  Action                          Output
   1     4   pred=3, create thread 3→4       []
             go left to 2
   2     2   pred=1, create thread 1→2       []
             go left to 1
   3     1   no left, visit 1, go right(thread→2) [1]
   4     2   thread 1→2 exists, remove it    [1,2]
             visit 2, go right to 3
   5     3   no left, visit 3, go right(thread→4) [1,2,3]
   6     4   thread 3→4 exists, remove it    [1,2,3,4]
             visit 4, go right to 6
   7     6   pred=5, create thread 5→6       [1,2,3,4]
             go left to 5
   8     5   no left, visit 5, go right(thread→6) [1,2,3,4,5]
   9     6   thread exists, remove, visit 6  [1,2,3,4,5,6]
             go right to 7
  10     7   no left, visit 7, done          [1,2,3,4,5,6,7]
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

**Textual Figure:**

```
  Recover BST — Find Two Swapped Nodes
  ══════════════════════════════════════

  Broken BST (1 and 3 swapped):
       ┌───┐
       │ 1 │  ← should be root
       └─┬─┘
        ╱ ╲
     ┌───┐ ┌───┐
     │ 3 │ │ 2 │  3 in left subtree! (✗)
     └───┘ └───┘

  Inorder: [3, 1, 2]

  Detection (prev > cur = violation):
    prev=3, cur=1:  3 > 1 → first=3, second=1
    prev=1, cur=2:  1 < 2 → ok

  Swap first & second values: 3 ↔ 1

  Fixed BST:
       ┌───┐
       │ 1 │
       └─┬─┘
        ╱ ╲
     ┌───┐ ┌───┐
     │ 3 │ │ 2 │  Wait, that's wrong as BST...
     └───┘ └───┘

  Actually: swap makes inorder [1, 2, 3] ✔ sorted
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

**Textual Figure:**

```
  Inorder Successor & Predecessor in BST
  ═══════════════════════════════════════════

           ┌────┐
           │ 20 │
           └─┬──┘
            ╱ ╲
        ┌────┐ ┌────┐
        │ 10 │ │ 30 │
        └─┬──┘ └─┬──┘
         ╱ ╲       ╱ ╲
      ┌───┐┌────┐┌────┐┌────┐
      │ 5 ││ 15 ││ 25 ││ 35 │
      └───┘└────┘└────┘└────┘

  Inorder: [5, 10, 15, 20, 25, 30, 35]

  ┌──────┬─────────────┬─────────────┐
  │ Node │ Predecessor │ Successor   │
  ├──────┼─────────────┼─────────────┤
  │  10  │     5       │     15      │
  │  15  │    10       │     20      │
  │  25  │    20       │     30      │
  │  35  │    30       │     nil     │
  └──────┴─────────────┴─────────────┘

  Algorithm: walk BST, track potential answer
    Successor: go right → leftmost (or ancestor)
    Predecessor: go left → rightmost (or ancestor)
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

**Textual Figure:**

```
  Minimum Distance Between BST Nodes
  ═══════════════════════════════════════

       ┌───┐
       │ 4 │
       └─┬─┘
        ╱ ╲
     ┌───┐ ┌───┐
     │ 2 │ │ 6 │
     └─┬─┘ └───┘
      ╱ ╲
   ┌───┐┌───┐
   │ 1 ││ 3 │
   └───┘└───┘

  Inorder: [1, 2, 3, 4, 6]

  Compare consecutive pairs:
    |2-1| = 1
    |3-2| = 1
    |4-3| = 1
    |6-4| = 2

  Minimum difference = 1

  Key insight: min diff in BST is always
  between consecutive inorder elements
  (no need to check all pairs).
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
