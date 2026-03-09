# Phase 8: Binary Trees — Tree Diameter

## Overview

**Diameter** (also called **width**) of a binary tree = length of the **longest path between any two nodes**. The path may or may not pass through the root.

```
         1
        / \
       2   3       Diameter = 4 (path: 4→2→1→3→6 or 5→2→1→3→6)
      / \   \
     4   5   6
```

**Key insight**: At each node, the longest path *through* that node = `height(left) + height(right)`. The overall diameter = max of this over all nodes.

---

## Example 1: Basic Diameter (LeetCode 543)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func diameterOfBinaryTree(root *TreeNode) int {
    diameter := 0

    var height func(node *TreeNode) int
    height = func(node *TreeNode) int {
        if node == nil { return 0 }
        left := height(node.Left)
        right := height(node.Right)

        // Update diameter (path through this node)
        if left+right > diameter {
            diameter = left + right
        }

        if left > right { return left + 1 }
        return right + 1
    }
    height(root)
    return diameter
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \
    //   4   5
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(diameterOfBinaryTree(root)) // 3 (path: 4→2→1→3 or 5→2→1→3)
}
```

---

## Example 2: Diameter Not Through Root

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func diameterOfBinaryTree(root *TreeNode) int {
    diameter := 0

    var height func(node *TreeNode) int
    height = func(node *TreeNode) int {
        if node == nil { return 0 }
        l := height(node.Left)
        r := height(node.Right)
        if l+r > diameter { diameter = l + r }
        if l > r { return l + 1 }
        return r + 1
    }
    height(root)
    return diameter
}

func main() {
    //         1
    //        /
    //       2
    //      / \
    //     3   4
    //    /     \
    //   5       6
    //  /         \
    // 7           8
    // Diameter = 6 (7→5→3→2→4→6→8), does NOT go through root 1
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{3,
                &TreeNode{5, &TreeNode{7, nil, nil}, nil},
                nil,
            },
            &TreeNode{4, nil,
                &TreeNode{6, nil, &TreeNode{8, nil, nil}},
            },
        },
        nil,
    }
    fmt.Println(diameterOfBinaryTree(root)) // 6
}
```

---

## Example 3: Diameter with Edge Weights

```go
package main

import "fmt"

type TreeNode struct {
    Val        int
    Left       *TreeNode
    Right      *TreeNode
    LeftWeight  int
    RightWeight int
}

func weightedDiameter(root *TreeNode) int {
    maxDia := 0

    var dfs func(node *TreeNode) int // returns max weighted height
    dfs = func(node *TreeNode) int {
        if node == nil { return 0 }
        leftH := 0
        rightH := 0
        if node.Left != nil {
            leftH = dfs(node.Left) + node.LeftWeight
        }
        if node.Right != nil {
            rightH = dfs(node.Right) + node.RightWeight
        }

        if leftH+rightH > maxDia { maxDia = leftH + rightH }

        if leftH > rightH { return leftH }
        return rightH
    }
    dfs(root)
    return maxDia
}

func main() {
    //       1
    //      / \      (edge weights: left=3, right=5)
    //     2   3
    //    / \         (edge weights: left=2, right=4)
    //   4   5
    root := &TreeNode{
        Val: 1,
        Left: &TreeNode{
            Val:  2,
            Left: &TreeNode{Val: 4},
            Right: &TreeNode{Val: 5},
            LeftWeight: 2, RightWeight: 4,
        },
        Right: &TreeNode{Val: 3},
        LeftWeight: 3, RightWeight: 5,
    }
    fmt.Println("Weighted diameter:", weightedDiameter(root))
    // Path 4→2→1→3: 2+3+5=10 or 5→2→1→3: 4+3+5=12
    // Path 4→2→5: 2+4=6
    // Answer: 12
}
```

---

## Example 4: Diameter of N-ary Tree

```go
package main

import "fmt"

type Node struct {
    Val      int
    Children []*Node
}

func diameterNary(root *Node) int {
    maxDia := 0

    var height func(node *Node) int
    height = func(node *Node) int {
        if node == nil { return 0 }

        // Track the two tallest children
        max1, max2 := 0, 0
        for _, child := range node.Children {
            h := height(child) + 1
            if h >= max1 {
                max2 = max1
                max1 = h
            } else if h > max2 {
                max2 = h
            }
        }

        if max1+max2 > maxDia {
            maxDia = max1 + max2
        }
        return max1
    }
    height(root)
    return maxDia
}

func main() {
    //        1
    //      / | \
    //     2  3  4
    //    /|     |
    //   5 6     7
    //   |
    //   8
    root := &Node{1, []*Node{
        {2, []*Node{
            {5, []*Node{{8, nil}}},
            {6, nil},
        }},
        {3, nil},
        {4, []*Node{{7, nil}}},
    }}
    fmt.Println("N-ary diameter:", diameterNary(root)) // 5 (8→5→2→1→4→7)
}
```

---

## Example 5: Longest Path with Same Value

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Similar to diameter but only count edges where parent.Val == child.Val
func longestUnivaluePath(root *TreeNode) int {
    longest := 0

    var dfs func(node *TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil { return 0 }

        leftLen := dfs(node.Left)
        rightLen := dfs(node.Right)

        leftPath, rightPath := 0, 0
        if node.Left != nil && node.Left.Val == node.Val {
            leftPath = leftLen + 1
        }
        if node.Right != nil && node.Right.Val == node.Val {
            rightPath = rightLen + 1
        }

        if leftPath+rightPath > longest {
            longest = leftPath + rightPath
        }

        if leftPath > rightPath { return leftPath }
        return rightPath
    }
    dfs(root)
    return longest
}

func main() {
    //       5
    //      / \
    //     4   5
    //    / \   \
    //   1   1   5
    root := &TreeNode{5,
        &TreeNode{4, &TreeNode{1, nil, nil}, &TreeNode{1, nil, nil}},
        &TreeNode{5, nil, &TreeNode{5, nil, nil}},
    }
    fmt.Println(longestUnivaluePath(root)) // 2 (5→5→5 on right)

    //       1
    //      / \
    //     4   5
    //    / \   \
    //   4   4   5
    root2 := &TreeNode{1,
        &TreeNode{4, &TreeNode{4, nil, nil}, &TreeNode{4, nil, nil}},
        &TreeNode{5, nil, &TreeNode{5, nil, nil}},
    }
    fmt.Println(longestUnivaluePath(root2)) // 2 (4→4→4 on left)
}
```

---

## Example 6: Diameter Endpoints

Find the actual path that forms the diameter.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func diameterPath(root *TreeNode) (int, []int) {
    maxDia := 0
    var bestPath []int

    var dfs func(node *TreeNode) (int, []int)
    dfs = func(node *TreeNode) (int, []int) {
        if node == nil { return 0, nil }

        lh, lpath := dfs(node.Left)
        rh, rpath := dfs(node.Right)

        // Path through this node
        if lh+rh > maxDia {
            maxDia = lh + rh
            // Reverse left path + node + right path
            bestPath = make([]int, 0, lh+rh+1)
            for i := len(lpath) - 1; i >= 0; i-- {
                bestPath = append(bestPath, lpath[i])
            }
            bestPath = append(bestPath, node.Val)
            bestPath = append(bestPath, rpath...)
        }

        // Return taller side
        if lh >= rh {
            return lh + 1, append([]int{node.Val}, lpath...)
        }
        return rh + 1, append([]int{node.Val}, rpath...)
    }
    dfs(root)
    return maxDia, bestPath
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \
    //   4   5
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    dia, path := diameterPath(root)
    fmt.Printf("Diameter: %d, Path: %v\n", dia, path)
    // Diameter: 3, Path: [4 2 1 3] (or [5 2 1 3])
}
```

---

## Example 7: Count Diameter-Length Paths

Count how many distinct diameter-length paths exist.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func countDiameterPaths(root *TreeNode) (int, int) {
    diameter := 0
    count := 0

    var height func(node *TreeNode) int
    height = func(node *TreeNode) int {
        if node == nil { return 0 }
        l := height(node.Left)
        r := height(node.Right)
        pathLen := l + r

        if pathLen > diameter {
            diameter = pathLen
            count = 1
        } else if pathLen == diameter && pathLen > 0 {
            count++
        }

        if l > r { return l + 1 }
        return r + 1
    }
    height(root)
    return diameter, count
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \ / \
    //     4  5 6  7
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
    }
    dia, cnt := countDiameterPaths(root)
    fmt.Printf("Diameter: %d, Nodes with max path through them: %d\n", dia, cnt)
    // Diameter 4, all longest paths pass through root
}
```

---

## Example 8: Diameter Using Two BFS (General Tree)

For a general tree represented as adjacency list.

```go
package main

import "fmt"

func treeDiameter(n int, edges [][]int) (int, []int) {
    if n <= 1 { return 0, []int{0} }

    adj := make([][]int, n)
    for _, e := range edges {
        adj[e[0]] = append(adj[e[0]], e[1])
        adj[e[1]] = append(adj[e[1]], e[0])
    }

    // BFS from any node to find farthest node
    bfs := func(start int) (farthest int, dist []int) {
        dist = make([]int, n)
        for i := range dist { dist[i] = -1 }
        dist[start] = 0
        queue := []int{start}
        farthest = start

        for len(queue) > 0 {
            u := queue[0]
            queue = queue[1:]
            for _, v := range adj[u] {
                if dist[v] == -1 {
                    dist[v] = dist[u] + 1
                    queue = append(queue, v)
                    if dist[v] > dist[farthest] {
                        farthest = v
                    }
                }
            }
        }
        return
    }

    // Two BFS to find diameter
    a, _ := bfs(0)     // Find farthest from 0
    b, dist := bfs(a)  // Find farthest from a → that's b
    diameter := dist[b]

    // Reconstruct path (backtrack from b using distances)
    path := []int{b}
    cur := b
    for cur != a {
        for _, v := range adj[cur] {
            if dist[v] == dist[cur]-1 {
                path = append(path, v)
                cur = v
                break
            }
        }
    }
    return diameter, path
}

func main() {
    //  0 - 1 - 2 - 3 - 4
    //          |
    //          5
    edges := [][]int{{0, 1}, {1, 2}, {2, 3}, {3, 4}, {2, 5}}
    dia, path := treeDiameter(6, edges)
    fmt.Printf("Diameter: %d, Path: %v\n", dia, path)
    // Diameter: 4, Path: [4 3 2 1 0]
}
```

---

## Example 9: Binary Tree Maximum Path Sum (LeetCode 124)

Diameter generalized to sums: longest "weighted" path.

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

        leftGain := max(0, dfs(node.Left))
        rightGain := max(0, dfs(node.Right))

        // "Diameter" in terms of sum
        pathSum := node.Val + leftGain + rightGain
        if pathSum > maxSum { maxSum = pathSum }

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
    root := &TreeNode{-10,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(maxPathSum(root)) // 42 (15→20→7)
}
```

---

## Example 10: Iterative Diameter Calculation

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func diameterIterative(root *TreeNode) int {
    if root == nil { return 0 }

    type Frame struct {
        Node    *TreeNode
        State   int // 0=process left, 1=process right, 2=compute
        LeftH   int
        RightH  int
    }

    stack := []Frame{{Node: root}}
    heightMap := map[*TreeNode]int{}
    maxDia := 0

    for len(stack) > 0 {
        top := &stack[len(stack)-1]

        switch top.State {
        case 0: // push left
            top.State = 1
            if top.Node.Left != nil {
                stack = append(stack, Frame{Node: top.Node.Left})
                continue
            }
        case 1: // left done, push right
            top.State = 2
            if top.Node.Left != nil {
                top.LeftH = heightMap[top.Node.Left]
            }
            if top.Node.Right != nil {
                stack = append(stack, Frame{Node: top.Node.Right})
                continue
            }
        case 2: // both done, compute
            if top.Node.Right != nil {
                top.RightH = heightMap[top.Node.Right]
            }

            dia := top.LeftH + top.RightH
            if dia > maxDia { maxDia = dia }

            h := top.LeftH
            if top.RightH > h { h = top.RightH }
            heightMap[top.Node] = h + 1

            stack = stack[:len(stack)-1]
            continue
        }
    }
    return maxDia
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }
    fmt.Println("Diameter (iterative):", diameterIterative(root)) // 3
}
```

---

## Key Takeaways

| Concept | Detail |
|---------|--------|
| **Diameter formula** | `max over all nodes of height(left) + height(right)` |
| **Time complexity** | O(n) — one DFS pass |
| **Not always through root** | The diameter can be in a subtree |
| **General trees** | Two BFS: find farthest from any node, then farthest from that |
| **Max path sum** | Same pattern — diameter generalized to sums with negative pruning |
| **Variant: univalue** | Only extend path along same-value edges |

> **Next up:** Morris Traversal →
