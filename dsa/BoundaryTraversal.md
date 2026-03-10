# Phase 8: Binary Trees — Boundary Traversal

## Overview

**Boundary traversal** visits nodes on the boundary of a binary tree in anti-clockwise order:
1. **Root**
2. **Left boundary** (top to bottom, excluding leaves)
3. **Leaves** (left to right)
4. **Right boundary** (bottom to top, excluding leaves)

```
         1
        / \
       2   3
      / \   \
     4   5   6
        / \
       7   8

Boundary: [1, 2, 4, 7, 8, 6, 3]
```

---

## Example 1: Basic Boundary Traversal

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func boundaryOfBinaryTree(root *TreeNode) []int {
    if root == nil { return nil }
    result := []int{root.Val}
    if isLeaf(root) { return result }

    // Left boundary (top to bottom, excluding root and leaves)
    addLeftBoundary(root.Left, &result)
    // All leaves (left to right)
    addLeaves(root, &result)
    // Right boundary (bottom to top, excluding root and leaves)
    addRightBoundary(root.Right, &result)

    return result
}

func isLeaf(node *TreeNode) bool {
    return node != nil && node.Left == nil && node.Right == nil
}

func addLeftBoundary(node *TreeNode, result *[]int) {
    for node != nil {
        if !isLeaf(node) {
            *result = append(*result, node.Val)
        }
        if node.Left != nil {
            node = node.Left
        } else {
            node = node.Right
        }
    }
}

func addLeaves(node *TreeNode, result *[]int) {
    if node == nil { return }
    if isLeaf(node) {
        *result = append(*result, node.Val)
        return
    }
    addLeaves(node.Left, result)
    addLeaves(node.Right, result)
}

func addRightBoundary(node *TreeNode, result *[]int) {
    // Collect then reverse (we need bottom to top)
    temp := []int{}
    for node != nil {
        if !isLeaf(node) {
            temp = append(temp, node.Val)
        }
        if node.Right != nil {
            node = node.Right
        } else {
            node = node.Left
        }
    }
    // Add in reverse
    for i := len(temp) - 1; i >= 0; i-- {
        *result = append(*result, temp[i])
    }
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \   \
    //     4   5   6
    //        / \
    //       7   8
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, nil, nil},
            &TreeNode{5, &TreeNode{7, nil, nil}, &TreeNode{8, nil, nil}},
        },
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }
    fmt.Println(boundaryOfBinaryTree(root))
    // [1 2 4 7 8 6 3]
}
```

**Textual Figure – Example 1:**
```
 Boundary Traversal (Anti-clockwise):
 ────────────────────────
         1
        / \
       2   3
      / \   \
     4   5   6
        / \
       7   8

 Three passes:
 ┌────────────────────────────────────────────┐
 │ 1. Root:            [1]                          │
 │ 2. Left boundary:   [2] (top→bottom, skip leaves) │
 │ 3. Leaves (L→R):    [4, 7, 8, 6]                 │
 │ 4. Right boundary:  [3] (bottom→top, skip leaves) │
 └────────────────────────────────────────────┘

 Traversal path around the boundary:
       → 1 →                   Anti-clockwise boundary:
      ↓       ↑                 1 → 2 → 4 → 7 → 8
      2        3                     ↓
      ↓         ↑               → 6 → 3 (bottom to top)
      4  5──6  ↑
         |  |                   Result: [1, 2, 4, 7, 8, 6, 3]
         7  8 →
```

---

## Example 2: Left-Only Tree

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool { return n != nil && n.Left == nil && n.Right == nil }

func boundary(root *TreeNode) []int {
    if root == nil { return nil }
    res := []int{root.Val}
    if isLeaf(root) { return res }

    // Left boundary
    node := root.Left
    for node != nil {
        if !isLeaf(node) { res = append(res, node.Val) }
        if node.Left != nil { node = node.Left } else { node = node.Right }
    }
    // Leaves
    var leaves func(n *TreeNode)
    leaves = func(n *TreeNode) {
        if n == nil { return }
        if isLeaf(n) { res = append(res, n.Val); return }
        leaves(n.Left); leaves(n.Right)
    }
    leaves(root)
    // Right boundary
    temp := []int{}
    node = root.Right
    for node != nil {
        if !isLeaf(node) { temp = append(temp, node.Val) }
        if node.Right != nil { node = node.Right } else { node = node.Left }
    }
    for i := len(temp) - 1; i >= 0; i-- { res = append(res, temp[i]) }
    return res
}

func main() {
    //   1
    //  /
    // 2
    //  \
    //   3
    //  /
    // 4
    root := &TreeNode{1,
        &TreeNode{2, nil,
            &TreeNode{3, &TreeNode{4, nil, nil}, nil},
        }, nil,
    }
    fmt.Println(boundary(root)) // [1 2 3 4]
    // Left boundary: [2, 3] (following left/right)
    // Leaves: [4]
    // Right boundary: none (root.Right is nil)
    // But root is included once → [1 2 3 4]
}
```

**Textual Figure – Example 2:**
```
 Left-Only Tree Boundary:
 ────────────────────
   1                   Left boundary path:
  /                    1 → 2 (has right, no left)
 2                       → 3 (has left, no right → go left)
  \                      → stop (4 is a leaf)
   3
  /                    Left boundary: [2, 3]
 4                     Leaves: [4]
                       Right boundary: none

 ┌───────────────┬───────────────────────┐
 │ Component     │ Nodes                 │
 ├───────────────┼───────────────────────┤
 │ Root          │ [1]                   │
 │ Left boundary │ [2, 3] (skip leaf 4)  │
 │ Leaves        │ [4]                   │
 │ Right boundary│ [] (root.Right=nil)   │
 └───────────────┴───────────────────────┘
 Result: [1, 2, 3, 4]
```

---

## Example 3: Right-Only Tree

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool { return n != nil && n.Left == nil && n.Right == nil }

func boundary(root *TreeNode) []int {
    if root == nil { return nil }
    res := []int{root.Val}
    if isLeaf(root) { return res }

    node := root.Left
    for node != nil {
        if !isLeaf(node) { res = append(res, node.Val) }
        if node.Left != nil { node = node.Left } else { node = node.Right }
    }
    var leaves func(n *TreeNode)
    leaves = func(n *TreeNode) {
        if n == nil { return }
        if isLeaf(n) { res = append(res, n.Val); return }
        leaves(n.Left); leaves(n.Right)
    }
    leaves(root)
    temp := []int{}
    node = root.Right
    for node != nil {
        if !isLeaf(node) { temp = append(temp, node.Val) }
        if node.Right != nil { node = node.Right } else { node = node.Left }
    }
    for i := len(temp) - 1; i >= 0; i-- { res = append(res, temp[i]) }
    return res
}

func main() {
    //   1
    //    \
    //     2
    //    /
    //   3
    //    \
    //     4
    root := &TreeNode{1, nil,
        &TreeNode{2,
            &TreeNode{3, nil, &TreeNode{4, nil, nil}},
            nil,
        },
    }
    fmt.Println(boundary(root)) // [1 4 3 2]
    // Left boundary: none (root.Left nil)
    // Leaves: [4]
    // Right boundary (bottom-up): [3 2]
}
```

**Textual Figure – Example 3:**
```
 Right-Only Tree Boundary:
 ────────────────────
   1                    Right boundary path (top→bottom):
    \                   1 → 2 (has left, not right)
     2                    → 3 (has right)
    /                     → 4 (leaf, stop)
   3
    \                   Right boundary: [2, 3]
     4                  Reversed: [3, 2]

 ┌───────────────┬───────────────────────┐
 │ Component     │ Nodes                 │
 ├───────────────┼───────────────────────┤
 │ Root          │ [1]                   │
 │ Left boundary │ [] (root.Left=nil)    │
 │ Leaves        │ [4]                   │
 │ Right (rev)   │ [3, 2]                │
 └───────────────┴───────────────────────┘
 Result: [1, 4, 3, 2]
```

---

## Example 4: Single Node & Two-Node Trees

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool { return n != nil && n.Left == nil && n.Right == nil }

func boundary(root *TreeNode) []int {
    if root == nil { return nil }
    res := []int{root.Val}
    if isLeaf(root) { return res }

    node := root.Left
    for node != nil {
        if !isLeaf(node) { res = append(res, node.Val) }
        if node.Left != nil { node = node.Left } else { node = node.Right }
    }
    var leaves func(n *TreeNode)
    leaves = func(n *TreeNode) {
        if n == nil { return }
        if isLeaf(n) { res = append(res, n.Val); return }
        leaves(n.Left); leaves(n.Right)
    }
    leaves(root)
    temp := []int{}
    node = root.Right
    for node != nil {
        if !isLeaf(node) { temp = append(temp, node.Val) }
        if node.Right != nil { node = node.Right } else { node = node.Left }
    }
    for i := len(temp) - 1; i >= 0; i-- { res = append(res, temp[i]) }
    return res
}

func main() {
    // Single node
    single := &TreeNode{Val: 42}
    fmt.Println("Single:", boundary(single)) // [42]

    // Root with left only
    twoL := &TreeNode{1, &TreeNode{2, nil, nil}, nil}
    fmt.Println("Root+Left:", boundary(twoL)) // [1 2]

    // Root with right only
    twoR := &TreeNode{1, nil, &TreeNode{3, nil, nil}}
    fmt.Println("Root+Right:", boundary(twoR)) // [1 3]

    // Root with both children (both leaves)
    both := &TreeNode{1, &TreeNode{2, nil, nil}, &TreeNode{3, nil, nil}}
    fmt.Println("Root+Both:", boundary(both)) // [1 2 3]
}
```

**Textual Figure – Example 4:**
```
 Edge Cases — Single Node & Two-Node Trees:
 ────────────────────────────────────

 Case 1: Single node    Case 2: Root+Left
   [42]                    1
   isLeaf → return [42]   /
                         2 (leaf)

 Case 3: Root+Right     Case 4: Root+Both
   1                       1
    \                     / \
     3 (leaf)            2   3 (both leaves)

 ┌────────────┬─────┬────────┬─────────┬───────────┐
 │ Case       │ R   │ LBound │ Leaves  │ RBound    │
 ├────────────┼─────┼────────┼─────────┼───────────┤
 │ Single     │[42]│ n/a    │ n/a     │ n/a       │
 │ Root+Left  │ [1]│ []     │ [2]     │ []        │
 │ Root+Right │ [1]│ []     │ [3]     │ []        │
 │ Root+Both  │ [1]│ []     │ [2,3]   │ []        │
 └────────────┴─────┴────────┴─────────┴───────────┘

 Results: [42], [1,2], [1,3], [1,2,3]
```

---

## Example 5: Boundary Using Single DFS with Flags

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func boundaryDFS(root *TreeNode) []int {
    if root == nil { return nil }

    left := []int{}
    leaves := []int{}
    right := []int{}

    // flag: 0=root, 1=left boundary, 2=right boundary, 3=internal
    var dfs func(node *TreeNode, flag int)
    dfs = func(node *TreeNode, flag int) {
        if node == nil { return }

        isLeafNode := node.Left == nil && node.Right == nil

        if isLeafNode {
            leaves = append(leaves, node.Val)
        } else if flag == 0 { // root
            left = append(left, node.Val)
        } else if flag == 1 { // left boundary
            left = append(left, node.Val)
        } else if flag == 2 { // right boundary
            right = append(right, node.Val)
        }

        // Determine child flags
        leftChildFlag := 3  // internal by default
        rightChildFlag := 3

        if flag == 0 || flag == 1 {
            leftChildFlag = 1
            if node.Left != nil {
                rightChildFlag = 3 // if left exists, right child is internal
            } else {
                rightChildFlag = 1 // if no left, right becomes left boundary
            }
        }
        if flag == 0 || flag == 2 {
            rightChildFlag = 2
            if node.Right != nil {
                leftChildFlag = 3
            } else {
                leftChildFlag = 2
            }
        }
        // Root: left child is left boundary, right child is right boundary
        if flag == 0 {
            leftChildFlag = 1
            rightChildFlag = 2
        }

        dfs(node.Left, leftChildFlag)
        dfs(node.Right, rightChildFlag)
    }
    dfs(root, 0)

    // Combine: left + leaves + reversed right
    result := append(left, leaves...)
    for i := len(right) - 1; i >= 0; i-- {
        result = append(result, right[i])
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, nil, nil},
            &TreeNode{5, &TreeNode{7, nil, nil}, &TreeNode{8, nil, nil}},
        },
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }
    fmt.Println(boundaryDFS(root)) // [1 2 4 7 8 6 3]
}
```

**Textual Figure – Example 5:**
```
 Boundary Using Single DFS with Flags:
 ───────────────────────────────
         1 (flag=0, root)
        / \
       2   3              Flags:
      / \   \               0 = root
     4   5   6              1 = left boundary
        / \                 2 = right boundary
       7   8                3 = internal

 DFS assigns flags to children:
 ┌──────┬───────┬──────┬─────────────────────┐
 │ Node │ Flag  │ Leaf │ Collected in        │
 ├──────┼───────┼──────┼─────────────────────┤
 │  1   │  0    │  No  │ left[]              │
 │  2   │  1    │  No  │ left[]              │
 │  4   │  1    │  Yes │ leaves[]            │
 │  5   │  3    │  No  │ (internal)          │
 │  7   │  3    │  Yes │ leaves[]            │
 │  8   │  3    │  Yes │ leaves[]            │
 │  3   │  2    │  No  │ right[]             │
 │  6   │  2    │  Yes │ leaves[]            │
 └──────┴───────┴──────┴─────────────────────┘

 Combine: left[1,2] + leaves[4,7,8,6] + rev(right[3])
 Result: [1, 2, 4, 7, 8, 6, 3]
```

---

## Example 6: Outer Boundary Anti-Clockwise (LeetCode 545)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool { return n.Left == nil && n.Right == nil }

func boundaryOfBinaryTree(root *TreeNode) []int {
    if root == nil { return nil }
    res := []int{}

    if !isLeaf(root) { res = append(res, root.Val) }

    // Left boundary
    t := root.Left
    for t != nil {
        if !isLeaf(t) { res = append(res, t.Val) }
        if t.Left != nil { t = t.Left } else { t = t.Right }
    }

    // Leaves
    var addLeaves func(n *TreeNode)
    addLeaves = func(n *TreeNode) {
        if n == nil { return }
        if isLeaf(n) { res = append(res, n.Val); return }
        addLeaves(n.Left)
        addLeaves(n.Right)
    }
    addLeaves(root)

    // Right boundary (reverse)
    stack := []int{}
    t = root.Right
    for t != nil {
        if !isLeaf(t) { stack = append(stack, t.Val) }
        if t.Right != nil { t = t.Right } else { t = t.Left }
    }
    for i := len(stack) - 1; i >= 0; i-- {
        res = append(res, stack[i])
    }
    return res
}

func main() {
    //         1
    //          \
    //           2
    //          / \
    //         3   4
    //        /   / \
    //       5   6   7
    //          / \
    //         8   9
    root := &TreeNode{1, nil,
        &TreeNode{2,
            &TreeNode{3, &TreeNode{5, nil, nil}, nil},
            &TreeNode{4,
                &TreeNode{6, &TreeNode{8, nil, nil}, &TreeNode{9, nil, nil}},
                &TreeNode{7, nil, nil},
            },
        },
    }
    fmt.Println(boundaryOfBinaryTree(root))
    // [1, 5, 8, 9, 7, 4, 2]
}
```

**Textual Figure – Example 6:**
```
 Outer Boundary (LeetCode 545):
 ────────────────────────
         1
          \
           2
          / \
         3   4
        /   / \
       5   6   7
          / \
         8   9

 ┌───────────────┬────────────────────────────┐
 │ Component     │ Nodes                      │
 ├───────────────┼────────────────────────────┤
 │ Root (not lf) │ [1]                        │
 │ Left boundary │ [] (root.Left = nil)       │
 │ Leaves (L→R)  │ [5, 8, 9, 7]               │
 │ Right (rev)   │ stack=[2,4] → rev=[4, 2]   │
 └───────────────┴────────────────────────────┘

 Right boundary walk: 2→4→(7 is leaf, stop)
 Stack: [2, 4], reversed: [4, 2]

 Result: [1] + [] + [5,8,9,7] + [4,2]
       = [1, 5, 8, 9, 7, 4, 2]
```

---

## Example 7: Clockwise Boundary Traversal

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool { return n.Left == nil && n.Right == nil }

// Clockwise: root → right boundary (top→bottom) → leaves (right→left) → left boundary (bottom→top)
func clockwiseBoundary(root *TreeNode) []int {
    if root == nil { return nil }
    res := []int{root.Val}
    if isLeaf(root) { return res }

    // Right boundary (top to bottom)
    node := root.Right
    for node != nil {
        if !isLeaf(node) { res = append(res, node.Val) }
        if node.Right != nil { node = node.Right } else { node = node.Left }
    }

    // Leaves (right to left) — reverse DFS
    var leavesRL func(n *TreeNode)
    leavesRL = func(n *TreeNode) {
        if n == nil { return }
        if isLeaf(n) { res = append(res, n.Val); return }
        leavesRL(n.Right) // right first for right-to-left
        leavesRL(n.Left)
    }
    leavesRL(root)

    // Left boundary (bottom to top)
    temp := []int{}
    node = root.Left
    for node != nil {
        if !isLeaf(node) { temp = append(temp, node.Val) }
        if node.Left != nil { node = node.Left } else { node = node.Right }
    }
    for i := len(temp) - 1; i >= 0; i-- {
        res = append(res, temp[i])
    }
    return res
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println("Clockwise:", clockwiseBoundary(root))
    // [1 3 7 6 5 4 2]
}
```

**Textual Figure – Example 7:**
```
 Clockwise Boundary Traversal (reversed order):
 ───────────────────────────────────────
       1
      / \             Anti-clockwise: root → L → leaves → R↑
     2   3            Clockwise:     root → R → leaves← → L↑
    / \ / \
   4  5 6  7         ┌───────────────┬──────────────────┐
                     │ Component     │ Nodes            │
                     ├───────────────┼──────────────────┤
                     │ Root          │ [1]              │
                     │ Right bound.  │ [3] (top→bottom) │
                     │ Leaves (R→L)  │ [7, 6, 5, 4]     │
                     │ Left (rev)    │ [2]              │
                     └───────────────┴──────────────────┘

     1 →                  Clockwise:
    ↑     ↓                1 → 3 → 7 → 6 → 5 → 4 → 2
    2      3
    ↑      ↓
    4─5  6─7
      ←──┘

 Result: [1, 3, 7, 6, 5, 4, 2]
```

---

## Example 8: Leaves of Binary Tree (Left to Right)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func getLeaves(root *TreeNode) []int {
    result := []int{}
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil { return }
        if node.Left == nil && node.Right == nil {
            result = append(result, node.Val)
            return
        }
        dfs(node.Left)
        dfs(node.Right)
    }
    dfs(root)
    return result
}

// Check if two trees have same leaf sequence (LeetCode 872)
func leafSimilar(root1, root2 *TreeNode) bool {
    leaves1 := getLeaves(root1)
    leaves2 := getLeaves(root2)

    if len(leaves1) != len(leaves2) { return false }
    for i := range leaves1 {
        if leaves1[i] != leaves2[i] { return false }
    }
    return true
}

func main() {
    //     3           5
    //    / \         / \
    //   5   1       6   2
    //  / \ / \     / \
    // 6  2 9  8   7   4
    //   / \
    //  7   4
    root1 := &TreeNode{3,
        &TreeNode{5,
            &TreeNode{6, nil, nil},
            &TreeNode{2, &TreeNode{7, nil, nil}, &TreeNode{4, nil, nil}},
        },
        &TreeNode{1, &TreeNode{9, nil, nil}, &TreeNode{8, nil, nil}},
    }
    root2 := &TreeNode{5,
        &TreeNode{6, &TreeNode{7, nil, nil}, nil},
        &TreeNode{2, nil, &TreeNode{4, nil, nil}},
    }

    fmt.Println("Tree1 leaves:", getLeaves(root1)) // [6 7 4 9 8]
    fmt.Println("Tree2 leaves:", getLeaves(root2)) // [7 4]
    fmt.Println("Leaf similar:", leafSimilar(root1, root2)) // false
}
```

**Textual Figure – Example 8:**
```
 Leaf Sequence Comparison (LeetCode 872):
 ────────────────────────────────────
  Tree 1:              Tree 2:
      3                    5
     / \                  / \
    5   1                6   2
   / \ / \              / \
  6  2 9  8            7   4
    / \
   7   4

 Leaf extraction (DFS left → right):
 ┌────────┬─────────────────────┐
 │ Tree   │ Leaves              │
 ├────────┼─────────────────────┤
 │ Tree1  │ [6, 7, 4, 9, 8]     │
 │ Tree2  │ [7, 4]              │
 └────────┴─────────────────────┘

 lengths differ (5 ≠ 2) → NOT leaf-similar

 Leaf-similar trees have identical leaf sequences
 when read left-to-right, regardless of structure.
```

---

## Example 9: Sum of All Boundary Nodes

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isLeaf(n *TreeNode) bool { return n.Left == nil && n.Right == nil }

func boundarySum(root *TreeNode) int {
    if root == nil { return 0 }
    visited := map[*TreeNode]bool{}
    sum := root.Val
    visited[root] = true

    // Left boundary
    node := root.Left
    for node != nil {
        if !visited[node] {
            sum += node.Val
            visited[node] = true
        }
        if node.Left != nil { node = node.Left } else { node = node.Right }
    }

    // Leaves
    var addLeaves func(n *TreeNode)
    addLeaves = func(n *TreeNode) {
        if n == nil { return }
        if isLeaf(n) && !visited[n] {
            sum += n.Val
            visited[n] = true
        }
        addLeaves(n.Left)
        addLeaves(n.Right)
    }
    addLeaves(root)

    // Right boundary
    node = root.Right
    for node != nil {
        if !visited[node] {
            sum += node.Val
            visited[node] = true
        }
        if node.Right != nil { node = node.Right } else { node = node.Left }
    }

    return sum
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println("Boundary sum:", boundarySum(root)) // 1+2+4+5+6+7+3 = 28
}
```

**Textual Figure – Example 9:**
```
 Sum of All Boundary Nodes:
 ──────────────────────
       1
      / \              All nodes are boundary nodes
     2   3             in this tree!
    / \ / \
   4  5 6  7

 Boundary nodes (marked with *):
      *1*               Root: 1
      / \               Left boundary: 2
    *2* *3*             Leaves: 4, 5, 6, 7
    / \ / \             Right boundary: 3
  *4**5**6**7*

 Uses visited map to avoid double-counting:
 ┌───────────────┬───────┬─────────────┐
 │ Pass          │ Added │ Running sum │
 ├───────────────┼───────┼─────────────┤
 │ Root          │  1    │  1          │
 │ Left bound.   │  2    │  3          │
 │ Leaves        │ 4,5,  │  3+4+5+6+7  │
 │               │ 6,7   │  = 25       │
 │ Right bound.  │  3    │  25+3 = 28  │
 └───────────────┴───────┴─────────────┘
 Result: 28
```

---

## Example 10: Boundary Traversal of N-ary Tree

```go
package main

import "fmt"

type Node struct {
    Val      int
    Children []*Node
}

func isLeafN(n *Node) bool { return len(n.Children) == 0 }

func boundaryNary(root *Node) []int {
    if root == nil { return nil }
    result := []int{root.Val}
    if isLeafN(root) { return result }

    // Left boundary: always take first child until leaf
    addLeftBoundaryN(root.Children[0], &result)

    // All leaves left to right
    var addLeavesN func(node *Node)
    addLeavesN = func(node *Node) {
        if node == nil { return }
        if isLeafN(node) {
            result = append(result, node.Val)
            return
        }
        for _, child := range node.Children {
            addLeavesN(child)
        }
    }
    addLeavesN(root)

    // Right boundary: always take last child, then reverse
    temp := []int{}
    addRightBoundaryN(root.Children[len(root.Children)-1], &temp)
    for i := len(temp) - 1; i >= 0; i-- {
        result = append(result, temp[i])
    }
    return result
}

func addLeftBoundaryN(node *Node, result *[]int) {
    if node == nil || isLeafN(node) { return }
    *result = append(*result, node.Val)
    addLeftBoundaryN(node.Children[0], result) // leftmost child
}

func addRightBoundaryN(node *Node, result *[]int) {
    if node == nil || isLeafN(node) { return }
    *result = append(*result, node.Val)
    addRightBoundaryN(node.Children[len(node.Children)-1], result) // rightmost child
}

func main() {
    //         1
    //       / | \
    //      2  3  4
    //     / \     \
    //    5   6     7
    //   /         / \
    //  8         9  10
    root := &Node{1, []*Node{
        {2, []*Node{
            {5, []*Node{{8, nil}}},
            {6, nil},
        }},
        {3, nil},
        {4, []*Node{
            {7, []*Node{{9, nil}, {10, nil}}},
        }},
    }}
    fmt.Println("N-ary boundary:", boundaryNary(root))
    // [1 2 5 8 6 3 9 10 7 4]
}
```

**Textual Figure – Example 10:**
```
 N-ary Tree Boundary Traversal:
 ────────────────────────────
         1
       / | \
      2  3  4
     / \     \
    5   6     7
   /         / \
  8         9  10

 N-ary boundary rules:
   Left boundary:  always follow first child
   Right boundary: always follow last child
   Leaves: depth-first left-to-right

 ┌───────────────┬────────────────────────┐
 │ Component     │ Trace                  │
 ├───────────────┼────────────────────────┤
 │ Root          │ [1]                    │
 │ Left bound.   │ 2→5 (1st child chain)  │
 │               │ stop at 8 (leaf)       │
 │               │ → [2, 5]               │
 │ Leaves (L→R)  │ [8, 6, 3, 9, 10]       │
 │ Right bound.  │ 4→7 (last child chain)  │
 │               │ stop at 9/10 (leaves)  │
 │               │ rev → [7, 4]           │
 └───────────────┴────────────────────────┘

 Result: [1] + [2,5] + [8,6,3,9,10] + [7,4]
       = [1, 2, 5, 8, 6, 3, 9, 10, 7, 4]
```

---

## Key Takeaways

| Component | Logic |
|-----------|-------|
| **Left boundary** | Go left; if no left, go right. Skip leaves. |
| **Right boundary** | Go right; if no right, go left. Skip leaves. Reverse at end. |
| **Leaves** | DFS left-to-right, collect nodes with no children. |
| **Root** | Added once at the start (unless it's a leaf). |
| **No duplicates** | Leaves shared with boundaries are only counted once. |

1. Three separate passes (left boundary, leaves, right boundary) is cleanest
2. Handle edge cases: single node, left-only, right-only, all-left chain
3. For clockwise traversal, flip the order: right boundary → reversed leaves → left boundary reversed

> **Phase 8 Complete! Next up: Phase 9 — Binary Search Trees →**
