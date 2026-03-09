# Phase 8: Binary Trees — Breadth First Search (BFS)

## Overview

**BFS on trees** explores nodes level by level using a queue. While Level Order Traversal focuses on collecting levels, this file covers **BFS problem-solving patterns**: multi-source BFS, BFS for shortest path in trees, BFS state tracking, and BFS vs DFS trade-offs in tree contexts.

| Property | BFS | DFS |
|----------|-----|-----|
| Data structure | Queue | Stack / recursion |
| Space | O(w) width | O(h) height |
| Best for | Shortest path, level ops | Path enumeration, subtree ops |
| Find min depth | Optimal — stops early | Must explore all paths |

---

## Example 1: BFS — Cousins in Binary Tree (LeetCode 993)

Two nodes are cousins if same depth but different parents.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isCousins(root *TreeNode, x, y int) bool {
    if root == nil { return false }

    type Info struct {
        Node   *TreeNode
        Parent *TreeNode
    }

    queue := []Info{{root, nil}}
    for len(queue) > 0 {
        size := len(queue)
        var px, py *TreeNode
        for i := 0; i < size; i++ {
            info := queue[0]
            queue = queue[1:]
            node := info.Node

            if node.Val == x { px = info.Parent }
            if node.Val == y { py = info.Parent }

            if node.Left != nil { queue = append(queue, Info{node.Left, node}) }
            if node.Right != nil { queue = append(queue, Info{node.Right, node}) }
        }
        // Both found at same level, different parents
        if px != nil && py != nil { return px != py }
        // Only one found at this level → not cousins
        if px != nil || py != nil { return false }
    }
    return false
}

func main() {
    //       1
    //      / \
    //     2   3
    //    /     \
    //   4       5
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, &TreeNode{5, nil, nil}},
    }
    fmt.Println(isCousins(root, 4, 5)) // true  (same depth, diff parents)
    fmt.Println(isCousins(root, 2, 3)) // false (siblings, not cousins by some definitions - actually they're siblings)

    root2 := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(isCousins(root2, 4, 3)) // false (different depths)
}
```

---

## Example 2: BFS — Complete Binary Tree Check (LeetCode 958)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isCompleteTree(root *TreeNode) bool {
    if root == nil { return true }
    queue := []*TreeNode{root}
    foundNil := false

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if node == nil {
            foundNil = true
            continue
        }
        // If we've seen nil before and now see a non-nil, not complete
        if foundNil { return false }

        queue = append(queue, node.Left)
        queue = append(queue, node.Right)
    }
    return true
}

func main() {
    //     1
    //    / \
    //   2   3
    //  / \  /
    // 4  5 6
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, nil},
    }
    fmt.Println(isCompleteTree(root)) // true

    //     1
    //    / \
    //   2   3
    //  / \   \
    // 4  5    7
    root2 := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{7, nil, nil}},
    }
    fmt.Println(isCompleteTree(root2)) // false
}
```

---

## Example 3: Multi-Source BFS — Closest Leaf (LeetCode 742)

Find the closest leaf to target node k.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func findClosestLeaf(root *TreeNode, k int) int {
    // Step 1: Build parent map and find target node
    parent := map[*TreeNode]*TreeNode{}
    var target *TreeNode

    var buildParent func(node, par *TreeNode)
    buildParent = func(node, par *TreeNode) {
        if node == nil { return }
        parent[node] = par
        if node.Val == k { target = node }
        buildParent(node.Left, node)
        buildParent(node.Right, node)
    }
    buildParent(root, nil)

    // Step 2: BFS from target through children AND parent
    visited := map[*TreeNode]bool{target: true}
    queue := []*TreeNode{target}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        // Leaf check
        if node.Left == nil && node.Right == nil {
            return node.Val
        }

        neighbors := []*TreeNode{node.Left, node.Right, parent[node]}
        for _, nb := range neighbors {
            if nb != nil && !visited[nb] {
                visited[nb] = true
                queue = append(queue, nb)
            }
        }
    }
    return -1
}

func main() {
    //       1
    //      / \
    //     3   2
    //    /
    //   5
    //  / \
    // 6   7
    root := &TreeNode{1,
        &TreeNode{3,
            &TreeNode{5, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
            nil,
        },
        &TreeNode{2, nil, nil},
    }
    fmt.Println(findClosestLeaf(root, 3)) // 6 or 7 (distance 2)
    fmt.Println(findClosestLeaf(root, 1)) // 2 (distance 1)
}
```

---

## Example 4: BFS — Even-Odd Tree (LeetCode 1609)

Level 0: all odd, strictly increasing. Level 1: all even, strictly decreasing. Alternating.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func isEvenOddTree(root *TreeNode) bool {
    if root == nil { return true }
    queue := []*TreeNode{root}
    level := 0

    for len(queue) > 0 {
        size := len(queue)
        prev := 0
        if level%2 == 0 {
            prev = 0 // for even levels, values must be > prev (start at 0, all odd)
        } else {
            prev = 1<<31 - 1 // for odd levels, values must be < prev
        }

        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]

            if level%2 == 0 {
                // Even level: values must be odd and strictly increasing
                if node.Val%2 == 0 || node.Val <= prev { return false }
            } else {
                // Odd level: values must be even and strictly decreasing
                if node.Val%2 != 0 || node.Val >= prev { return false }
            }
            prev = node.Val

            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        level++
    }
    return true
}

func main() {
    //        1
    //       / \
    //      10   4
    //     /  \
    //    3    7
    root := &TreeNode{1,
        &TreeNode{10, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
        &TreeNode{4, nil, nil},
    }
    fmt.Println(isEvenOddTree(root)) // true
}
```

---

## Example 5: BFS — All Nodes Distance K (LeetCode 863)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func distanceK(root *TreeNode, target *TreeNode, k int) []int {
    // Build parent map
    parent := map[*TreeNode]*TreeNode{}
    var buildParent func(node, par *TreeNode)
    buildParent = func(node, par *TreeNode) {
        if node == nil { return }
        parent[node] = par
        buildParent(node.Left, node)
        buildParent(node.Right, node)
    }
    buildParent(root, nil)

    // BFS from target
    visited := map[*TreeNode]bool{target: true}
    queue := []*TreeNode{target}
    dist := 0

    for len(queue) > 0 {
        if dist == k {
            result := []int{}
            for _, node := range queue {
                result = append(result, node.Val)
            }
            return result
        }
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            for _, nb := range []*TreeNode{node.Left, node.Right, parent[node]} {
                if nb != nil && !visited[nb] {
                    visited[nb] = true
                    queue = append(queue, nb)
                }
            }
        }
        dist++
    }
    return nil
}

func main() {
    //       3
    //      / \
    //     5   1
    //    / \ / \
    //   6  2 0  8
    //     / \
    //    7   4
    n4 := &TreeNode{4, nil, nil}
    n7 := &TreeNode{7, nil, nil}
    n2 := &TreeNode{2, n7, n4}
    n6 := &TreeNode{6, nil, nil}
    n5 := &TreeNode{5, n6, n2}
    n0 := &TreeNode{0, nil, nil}
    n8 := &TreeNode{8, nil, nil}
    n1 := &TreeNode{1, n0, n8}
    root := &TreeNode{3, n5, n1}

    fmt.Println(distanceK(root, n5, 2)) // [7 4 1]
}
```

---

## Example 6: BFS — Deepest Leaves Sum (LeetCode 1302)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func deepestLeavesSum(root *TreeNode) int {
    if root == nil { return 0 }
    queue := []*TreeNode{root}
    sum := 0

    for len(queue) > 0 {
        size := len(queue)
        sum = 0 // reset every level; last level's sum is the answer
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            sum += node.Val
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
    }
    return sum
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \   \
    //     4   5   6
    //    /         \
    //   7           8
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, &TreeNode{7, nil, nil}, nil},
            &TreeNode{5, nil, nil},
        },
        &TreeNode{3, nil, &TreeNode{6, nil, &TreeNode{8, nil, nil}}},
    }
    fmt.Println(deepestLeavesSum(root)) // 15 (7+8)
}
```

---

## Example 7: BFS with State — Add One Row to Tree (LeetCode 623)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func addOneRow(root *TreeNode, val int, depth int) *TreeNode {
    if depth == 1 {
        return &TreeNode{val, root, nil}
    }

    queue := []*TreeNode{root}
    d := 1

    for len(queue) > 0 {
        if d == depth-1 {
            // Insert new row here
            for _, node := range queue {
                newLeft := &TreeNode{val, node.Left, nil}
                newRight := &TreeNode{val, nil, node.Right}
                node.Left = newLeft
                node.Right = newRight
            }
            return root
        }
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        d++
    }
    return root
}

func printLevelOrder(root *TreeNode) {
    if root == nil { return }
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            fmt.Printf("%d ", node.Val)
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        fmt.Println()
    }
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{3, nil, nil}, &TreeNode{1, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, nil},
    }
    fmt.Println("Before:")
    printLevelOrder(root)
    addOneRow(root, 1, 2)
    fmt.Println("After adding row of 1s at depth 2:")
    printLevelOrder(root)
}
```

---

## Example 8: BFS — Check If Tree is Complete (Numbered Approach)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// A complete binary tree with n nodes has all indices in [0, n-1]
// when numbered level-order. Max index == n-1.
func isCompleteTree(root *TreeNode) bool {
    if root == nil { return true }

    type Pair struct {
        Node *TreeNode
        Idx  int
    }

    count := 0
    maxIdx := 0
    queue := []Pair{{root, 0}}

    for len(queue) > 0 {
        p := queue[0]
        queue = queue[1:]
        count++
        if p.Idx > maxIdx { maxIdx = p.Idx }

        if p.Node.Left != nil {
            queue = append(queue, Pair{p.Node.Left, 2*p.Idx + 1})
        }
        if p.Node.Right != nil {
            queue = append(queue, Pair{p.Node.Right, 2*p.Idx + 2})
        }
    }
    return maxIdx == count-1
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, nil},
    }
    fmt.Println(isCompleteTree(root)) // true

    root2 := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, &TreeNode{7, nil, nil}},
    }
    fmt.Println(isCompleteTree(root2)) // false (gap at index 4)
}
```

---

## Example 9: BFS — Serialize/Deserialize Level Order

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
    if root == nil { return "" }
    parts := []string{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        if node == nil {
            parts = append(parts, "null")
        } else {
            parts = append(parts, strconv.Itoa(node.Val))
            queue = append(queue, node.Left)
            queue = append(queue, node.Right)
        }
    }
    // Trim trailing nulls
    for len(parts) > 0 && parts[len(parts)-1] == "null" {
        parts = parts[:len(parts)-1]
    }
    return strings.Join(parts, ",")
}

func deserialize(data string) *TreeNode {
    if data == "" { return nil }
    parts := strings.Split(data, ",")
    val, _ := strconv.Atoi(parts[0])
    root := &TreeNode{Val: val}
    queue := []*TreeNode{root}
    i := 1

    for len(queue) > 0 && i < len(parts) {
        node := queue[0]
        queue = queue[1:]

        if i < len(parts) && parts[i] != "null" {
            v, _ := strconv.Atoi(parts[i])
            node.Left = &TreeNode{Val: v}
            queue = append(queue, node.Left)
        }
        i++

        if i < len(parts) && parts[i] != "null" {
            v, _ := strconv.Atoi(parts[i])
            node.Right = &TreeNode{Val: v}
            queue = append(queue, node.Right)
        }
        i++
    }
    return root
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
    }
    s := serialize(root)
    fmt.Println("Serialized:", s) // 1,2,3,null,null,4,5

    tree := deserialize(s)
    fmt.Println("Re-serialized:", serialize(tree)) // 1,2,3,null,null,4,5
}
```

---

## Example 10: BFS — Find Bottom Left Value (LeetCode 513)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Trick: BFS right-to-left, last node visited is bottom-left
func findBottomLeftValue(root *TreeNode) int {
    queue := []*TreeNode{root}
    var last *TreeNode

    for len(queue) > 0 {
        last = queue[0]
        queue = queue[1:]
        // Enqueue right before left
        if last.Right != nil { queue = append(queue, last.Right) }
        if last.Left != nil { queue = append(queue, last.Left) }
    }
    return last.Val
}

// Standard approach
func findBottomLeftStandard(root *TreeNode) int {
    queue := []*TreeNode{root}
    result := root.Val

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if i == 0 { result = node.Val } // first node in each level
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
    }
    return result
}

func main() {
    //       1
    //      / \
    //     2   3
    //    /   / \
    //   4   5   6
    //      /
    //     7
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3,
            &TreeNode{5, &TreeNode{7, nil, nil}, nil},
            &TreeNode{6, nil, nil},
        },
    }
    fmt.Println(findBottomLeftValue(root))   // 7
    fmt.Println(findBottomLeftStandard(root)) // 7
}
```

---

## Key Takeaways

1. **BFS finds min depth optimally** — stops at first leaf (DFS must check all paths)
2. **Parent map + BFS** turns tree into a general graph for distance queries (LeetCode 863, 742)
3. **Right-to-left BFS** trick: last visited node is the bottom-left value
4. **Level-based BFS** pattern: capture `size := len(queue)` to process exactly one level
5. BFS space is O(w) — can be O(n) for perfectly balanced trees (last level has n/2 nodes)
6. Use **numbered indexing** (2i+1, 2i+2) for width and completeness checks

> **Next up:** Tree Height →
