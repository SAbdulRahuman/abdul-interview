# Phase 15: Dynamic Programming — DP on Trees

## Overview

**Tree DP** computes optimal values bottom-up from leaves to root using post-order traversal. Each node's DP state depends on its children's computed values.

| Pattern | Description |
|---------|-------------|
| **Subtree DP** | dp[node] computed from dp[children] |
| **Include/Exclude** | dp[node][0] = exclude node, dp[node][1] = include node |
| **Re-rooting** | Compute answer rooted at every node efficiently in O(n) |
| **Path DP** | Track best path through subtree |

---

## Example 1: Max Path Sum (Root to Leaf)

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

func maxPathRootToLeaf(root *TreeNode) int {
	if root == nil { return math.MinInt64 }
	if root.Left == nil && root.Right == nil { return root.Val }

	left := maxPathRootToLeaf(root.Left)
	right := maxPathRootToLeaf(root.Right)

	best := left
	if right > best { best = right }
	return root.Val + best
}

func main() {
	//       10
	//      /  \
	//     5    -3
	//    / \     \
	//   3   2     11
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{2, nil, nil}},
		&TreeNode{-3, nil, &TreeNode{11, nil, nil}},
	}
	fmt.Println("Max root-to-leaf:", maxPathRootToLeaf(root)) // 18
}
```

---

## Example 2: House Robber III (LeetCode 337)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// Returns (robThis, skipThis)
func rob(root *TreeNode) (int, int) {
	if root == nil { return 0, 0 }

	leftRob, leftSkip := rob(root.Left)
	rightRob, rightSkip := rob(root.Right)

	// If we rob this node, children must be skipped
	robThis := root.Val + leftSkip + rightSkip
	// If we skip this node, children can be robbed or not
	skipThis := max(leftRob, leftSkip) + max(rightRob, rightSkip)

	return robThis, skipThis
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	//     3
	//    / \
	//   2   3
	//    \   \
	//     3   1
	root := &TreeNode{3,
		&TreeNode{2, nil, &TreeNode{3, nil, nil}},
		&TreeNode{3, nil, &TreeNode{1, nil, nil}},
	}
	r, s := rob(root)
	fmt.Println("Max robbery:", max(r, s)) // 7
}
```

---

## Example 3: Tree Diameter via DP

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

var diameter int

func height(root *TreeNode) int {
	if root == nil { return 0 }
	l := height(root.Left)
	r := height(root.Right)
	// Diameter through this node = left height + right height
	if l+r > diameter { diameter = l + r }
	if l > r { return l + 1 }
	return r + 1
}

func treeDiameter(root *TreeNode) int {
	diameter = 0
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
	fmt.Println("Diameter:", treeDiameter(root)) // 3
}
```

---

## Example 4: Binary Tree Maximum Path Sum (LeetCode 124)

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

var maxSum int

func maxGain(node *TreeNode) int {
	if node == nil { return 0 }

	leftGain := max(maxGain(node.Left), 0)
	rightGain := max(maxGain(node.Right), 0)

	// Path through this node
	pathSum := node.Val + leftGain + rightGain
	if pathSum > maxSum { maxSum = pathSum }

	// Return max gain to parent (can only go one direction)
	if leftGain > rightGain {
		return node.Val + leftGain
	}
	return node.Val + rightGain
}

func max(a, b int) int { if a > b { return a }; return b }

func maxPathSum(root *TreeNode) int {
	maxSum = math.MinInt64
	maxGain(root)
	return maxSum
}

func main() {
	//    -10
	//    /  \
	//   9    20
	//       /  \
	//      15   7
	root := &TreeNode{-10,
		&TreeNode{9, nil, nil},
		&TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
	}
	fmt.Println("Max path sum:", maxPathSum(root)) // 42
}
```

---

## Example 5: Subtree Sizes (N-ary Tree DP)

```go
package main

import "fmt"

func computeSubtreeSize(adj [][]int, root int) []int {
	n := len(adj)
	size := make([]int, n)
	visited := make([]bool, n)

	var dfs func(node int)
	dfs = func(node int) {
		visited[node] = true
		size[node] = 1
		for _, child := range adj[node] {
			if !visited[child] {
				dfs(child)
				size[node] += size[child]
			}
		}
	}

	dfs(root)
	return size
}

func main() {
	//     0
	//    /|\
	//   1 2 3
	//  / \
	// 4   5
	adj := [][]int{
		{1, 2, 3},
		{0, 4, 5},
		{0},
		{0},
		{1},
		{1},
	}
	sizes := computeSubtreeSize(adj, 0)
	for i, s := range sizes {
		fmt.Printf("Node %d: subtree size = %d\n", i, s)
	}
}
```

---

## Example 6: Re-rooting Technique — Sum of Distances (LeetCode 834)

```go
package main

import "fmt"

func sumOfDistancesInTree(n int, edges [][]int) []int {
	adj := make([][]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
	}

	count := make([]int, n) // subtree size
	dist := make([]int, n)  // sum of distances

	// DFS 1: compute count[] and dist[] rooted at 0
	var dfs1 func(node, parent int)
	dfs1 = func(node, parent int) {
		count[node] = 1
		for _, child := range adj[node] {
			if child != parent {
				dfs1(child, node)
				count[node] += count[child]
				dist[node] += dist[child] + count[child]
			}
		}
	}

	// DFS 2: re-root to compute dist[] for all nodes
	var dfs2 func(node, parent int)
	dfs2 = func(node, parent int) {
		for _, child := range adj[node] {
			if child != parent {
				// Moving root from node to child:
				// child's subtree gets 1 closer (count[child] nodes)
				// rest gets 1 farther (n - count[child] nodes)
				dist[child] = dist[node] - count[child] + (n - count[child])
				dfs2(child, node)
			}
		}
	}

	dfs1(0, -1)
	dfs2(0, -1)
	return dist
}

func main() {
	edges := [][]int{{0, 1}, {0, 2}, {2, 3}, {2, 4}, {2, 5}}
	result := sumOfDistancesInTree(6, edges)
	fmt.Println(result) // [8 12 6 10 10 10]
}
```

---

## Example 7: Minimum Height Trees (LeetCode 310)

```go
package main

import "fmt"

func findMinHeightTrees(n int, edges [][]int) []int {
	if n == 1 { return []int{0} }

	adj := make([][]int, n)
	degree := make([]int, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], e[1])
		adj[e[1]] = append(adj[e[1]], e[0])
		degree[e[0]]++
		degree[e[1]]++
	}

	// Start with leaves
	queue := []int{}
	for i := 0; i < n; i++ {
		if degree[i] == 1 { queue = append(queue, i) }
	}

	remaining := n
	for remaining > 2 {
		remaining -= len(queue)
		next := []int{}
		for _, leaf := range queue {
			for _, neighbor := range adj[leaf] {
				degree[neighbor]--
				if degree[neighbor] == 1 {
					next = append(next, neighbor)
				}
			}
		}
		queue = next
	}
	return queue
}

func main() {
	fmt.Println(findMinHeightTrees(4, [][]int{{1, 0}, {1, 2}, {1, 3}})) // [1]
	fmt.Println(findMinHeightTrees(6, [][]int{{3, 0}, {3, 1}, {3, 2}, {3, 4}, {5, 4}})) // [3 4]
}
```

---

## Example 8: Longest Path in Tree (Weighted)

```go
package main

import "fmt"

type Edge struct {
	to, weight int
}

func longestPath(n int, edges [][]int) int {
	adj := make([][]Edge, n)
	for _, e := range edges {
		adj[e[0]] = append(adj[e[0]], Edge{e[1], e[2]})
		adj[e[1]] = append(adj[e[1]], Edge{e[0], e[2]})
	}

	best := 0

	var dfs func(node, parent int) int
	dfs = func(node, parent int) int {
		max1, max2 := 0, 0 // two longest paths from this node downward

		for _, e := range adj[node] {
			if e.to != parent {
				childDist := dfs(e.to, node) + e.weight
				if childDist > max1 {
					max2 = max1
					max1 = childDist
				} else if childDist > max2 {
					max2 = childDist
				}
			}
		}

		pathThrough := max1 + max2
		if pathThrough > best { best = pathThrough }
		return max1
	}

	dfs(0, -1)
	return best
}

func main() {
	// 0 --5-- 1 --3-- 2
	//         |
	//         4
	//         |
	//         3
	edges := [][]int{{0, 1, 5}, {1, 2, 3}, {1, 3, 4}}
	fmt.Println("Longest path:", longestPath(4, edges)) // 9
}
```

---

## Example 9: Count Good Nodes (DP on Tree)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// A node is "good" if no node on path from root has value greater than it
func goodNodes(root *TreeNode) int {
	var dfs func(node *TreeNode, maxSoFar int) int
	dfs = func(node *TreeNode, maxSoFar int) int {
		if node == nil { return 0 }
		count := 0
		if node.Val >= maxSoFar {
			count = 1
			maxSoFar = node.Val
		}
		return count + dfs(node.Left, maxSoFar) + dfs(node.Right, maxSoFar)
	}
	return dfs(root, root.Val)
}

func main() {
	//       3
	//      / \
	//     1   4
	//    /   / \
	//   3   1   5
	root := &TreeNode{3,
		&TreeNode{1, &TreeNode{3, nil, nil}, nil},
		&TreeNode{4, &TreeNode{1, nil, nil}, &TreeNode{5, nil, nil}},
	}
	fmt.Println("Good nodes:", goodNodes(root)) // 4 (3, 3, 4, 5)
}
```

---

## Example 10: Tree DP — Maximum Independent Set

```go
package main

import "fmt"

// Maximum weight independent set on a tree (no two adjacent nodes selected)
func maxIndependentSet(adj [][]int, weights []int, root int) int {
	n := len(adj)
	// dp[node][0] = max weight NOT including node
	// dp[node][1] = max weight including node
	dp := make([][2]int, n)
	visited := make([]bool, n)

	var dfs func(node int)
	dfs = func(node int) {
		visited[node] = true
		dp[node][1] = weights[node]

		for _, child := range adj[node] {
			if !visited[child] {
				dfs(child)
				dp[node][0] += max(dp[child][0], dp[child][1])
				dp[node][1] += dp[child][0] // can't include adjacent
			}
		}
	}

	dfs(root)
	return max(dp[root][0], dp[root][1])
}

func max(a, b int) int { if a > b { return a }; return b }

func main() {
	//     0(1)
	//    / \
	//   1(5) 2(3)
	//   |
	//   3(8)
	adj := [][]int{
		{1, 2},
		{0, 3},
		{0},
		{1},
	}
	weights := []int{1, 5, 3, 8}
	fmt.Println("Max independent set:", maxIndependentSet(adj, weights, 0)) // 11 (3 + 8)
}
```

---

## Key Takeaways

1. Tree DP = post-order DFS: compute children first, then parent
2. Include/exclude pattern: dp[node][0] and dp[node][1] for binary choices
3. Re-rooting: two DFS passes — first computes one root, second re-roots in O(n)
4. Path problems: track two longest branches at each node
5. Tree DP naturally fits recursive solutions — iterative topological order also works

> **Next up:** Bitmask DP →
