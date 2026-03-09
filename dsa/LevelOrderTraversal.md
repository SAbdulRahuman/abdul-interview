# Phase 8: Binary Trees — Level Order Traversal

## Overview

**Level-order traversal** (BFS on trees) visits nodes level by level, left to right. Uses a **queue**. Space is O(w) where w is the maximum width of the tree.

```
         1         Level 0
        / \
       2   3       Level 1
      / \   \
     4   5   6     Level 2

Level-order: [[1], [2,3], [4,5,6]]
```

---

## Example 1: Basic Level Order (LeetCode 102)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func levelOrder(root *TreeNode) [][]int {
    if root == nil { return nil }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, 0, size)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        result = append(result, level)
    }
    return result
}

func main() {
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    for i, level := range levelOrder(root) {
        fmt.Printf("Level %d: %v\n", i, level)
    }
}
```

---

## Example 2: Level Order Bottom Up (LeetCode 107)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func levelOrderBottom(root *TreeNode) [][]int {
    if root == nil { return nil }
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
    fmt.Println(levelOrderBottom(root)) // [[15,7],[9,20],[3]]
}
```

---

## Example 3: Zigzag Level Order (LeetCode 103)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func zigzagLevelOrder(root *TreeNode) [][]int {
    if root == nil { return nil }
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
            if !leftToRight { idx = size - 1 - i }
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
    //       3
    //      / \
    //     9  20
    //       /  \
    //      15   7
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(zigzagLevelOrder(root)) // [[3],[20,9],[15,7]]
}
```

---

## Example 4: Right Side View (LeetCode 199)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func rightSideView(root *TreeNode) []int {
    if root == nil { return nil }
    result := []int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if i == size-1 {
                result = append(result, node.Val) // last node in level
            }
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
    }
    return result
}

func leftSideView(root *TreeNode) []int {
    if root == nil { return nil }
    result := []int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if i == 0 {
                result = append(result, node.Val) // first node in level
            }
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
    //      \   \
    //       5   4
    root := &TreeNode{1,
        &TreeNode{2, nil, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{4, nil, nil}},
    }
    fmt.Println("Right view:", rightSideView(root)) // [1,3,4]
    fmt.Println("Left view:", leftSideView(root))   // [1,2,5]
}
```

---

## Example 5: Maximum Width of Binary Tree (LeetCode 662)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func widthOfBinaryTree(root *TreeNode) int {
    if root == nil { return 0 }

    type Pair struct {
        Node *TreeNode
        Idx  int
    }

    maxWidth := 1
    queue := []Pair{{root, 0}}

    for len(queue) > 0 {
        size := len(queue)
        firstIdx := queue[0].Idx
        lastIdx := queue[size-1].Idx

        width := lastIdx - firstIdx + 1
        if width > maxWidth {
            maxWidth = width
        }

        for i := 0; i < size; i++ {
            p := queue[0]
            queue = queue[1:]
            idx := p.Idx - firstIdx // normalize to prevent overflow

            if p.Node.Left != nil {
                queue = append(queue, Pair{p.Node.Left, 2*idx + 1})
            }
            if p.Node.Right != nil {
                queue = append(queue, Pair{p.Node.Right, 2*idx + 2})
            }
        }
    }
    return maxWidth
}

func main() {
    //       1
    //      / \
    //     3   2
    //    / \   \
    //   5   3   9
    root := &TreeNode{1,
        &TreeNode{3, &TreeNode{5, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{2, nil, &TreeNode{9, nil, nil}},
    }
    fmt.Println("Max width:", widthOfBinaryTree(root)) // 4 (5,3,null,9)
}
```

---

## Example 6: Average of Levels (LeetCode 637)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func averageOfLevels(root *TreeNode) []float64 {
    if root == nil { return nil }
    result := []float64{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        sum := 0.0
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            sum += float64(node.Val)
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        result = append(result, sum/float64(size))
    }
    return result
}

func main() {
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(averageOfLevels(root)) // [3.0 14.5 11.0]
}
```

---

## Example 7: Level Order Using DFS (Recursive)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func levelOrderDFS(root *TreeNode) [][]int {
    result := [][]int{}
    var dfs func(node *TreeNode, level int)
    dfs = func(node *TreeNode, level int) {
        if node == nil { return }
        // Expand result if new level
        if level >= len(result) {
            result = append(result, []int{})
        }
        result[level] = append(result[level], node.Val)
        dfs(node.Left, level+1)
        dfs(node.Right, level+1)
    }
    dfs(root, 0)
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, &TreeNode{6, nil, nil}},
    }
    for i, level := range levelOrderDFS(root) {
        fmt.Printf("Level %d: %v\n", i, level)
    }
    // Level 0: [1]
    // Level 1: [2 3]
    // Level 2: [4 5 6]
}
```

**Why?** DFS-based level order uses O(h) stack space vs O(w) queue space. Better for very wide but shallow trees.

---

## Example 8: Largest Value in Each Row (LeetCode 515)

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

func largestValues(root *TreeNode) []int {
    if root == nil { return nil }
    result := []int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        maxVal := math.MinInt64
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            if node.Val > maxVal { maxVal = node.Val }
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        result = append(result, maxVal)
    }
    return result
}

func main() {
    root := &TreeNode{1,
        &TreeNode{3, &TreeNode{5, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{2, nil, &TreeNode{9, nil, nil}},
    }
    fmt.Println(largestValues(root)) // [1 3 9]
}
```

---

## Example 9: Connect Next Right Pointers (LeetCode 116)

```go
package main

import "fmt"

type Node struct {
    Val   int
    Left  *Node
    Right *Node
    Next  *Node
}

func connect(root *Node) *Node {
    if root == nil { return nil }
    queue := []*Node{root}

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]

            if i < size-1 {
                node.Next = queue[0]
            } // else Next stays nil

            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
    }
    return root
}

// O(1) space solution for perfect binary trees
func connectOptimal(root *Node) *Node {
    if root == nil { return nil }
    leftmost := root

    for leftmost.Left != nil {
        cur := leftmost
        for cur != nil {
            cur.Left.Next = cur.Right
            if cur.Next != nil {
                cur.Right.Next = cur.Next.Left
            }
            cur = cur.Next
        }
        leftmost = leftmost.Left
    }
    return root
}

func printNextPointers(root *Node) {
    level := root
    for level != nil {
        cur := level
        for cur != nil {
            next := "nil"
            if cur.Next != nil { next = fmt.Sprint(cur.Next.Val) }
            fmt.Printf("%d→%s  ", cur.Val, next)
            cur = cur.Next
        }
        fmt.Println()
        level = level.Left
    }
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \ / \
    //   4  5 6  7
    root := &Node{1,
        &Node{2, &Node{4, nil, nil, nil}, &Node{5, nil, nil, nil}, nil},
        &Node{3, &Node{6, nil, nil, nil}, &Node{7, nil, nil, nil}, nil},
        nil,
    }

    connectOptimal(root)
    printNextPointers(root)
    // 1→nil
    // 2→3  3→nil
    // 4→5  5→6  6→7  7→nil
}
```

---

## Example 10: Minimum Depth of Binary Tree (LeetCode 111)

BFS finds minimum depth faster — stops at the first leaf.

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
    queue := []*TreeNode{root}
    depth := 1

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]

            // First leaf found = minimum depth
            if node.Left == nil && node.Right == nil {
                return depth
            }
            if node.Left != nil { queue = append(queue, node.Left) }
            if node.Right != nil { queue = append(queue, node.Right) }
        }
        depth++
    }
    return depth
}

func maxDepth(root *TreeNode) int {
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
    //       1
    //      / \
    //     2   3
    //    /
    //   4
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println("Min depth:", minDepth(root)) // 2 (1→3)
    fmt.Println("Max depth:", maxDepth(root)) // 3 (1→2→4)
}
```

---

## Key Takeaways

1. Level-order = BFS with a queue; capture `size := len(queue)` to process level-by-level
2. **Zigzag**: flip insertion index each level
3. **Right/Left side view**: last/first element in each level
4. **Width**: use heap-style indexing (2i+1, 2i+2) and normalize per level
5. BFS finds **minimum depth** optimally — stops at first leaf
6. **Connect next pointers**: O(1) space possible for perfect trees using established Next links
7. DFS can also produce level-order results by passing `level` parameter

> **Next up:** Depth First Search (Trees) →
