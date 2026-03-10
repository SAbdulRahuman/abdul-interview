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

**Textual Figure:**

```
Bottom-up DFS — dp[node] = max root-to-leaf sum through node:

           ┌────┐
           │ 10 │  dp = 10 + max(8, 8) = 18
           └──┬─┘
          ┌───┴────┐
       ┌──┴─┐   ┌──┴──┐
       │  5 │   │ -3  │
       └──┬─┘   └──┬──┘
       dp=8      dp=8
     ┌──┴──┐      └──┐
  ┌──┴┐  ┌─┴─┐  ┌───┴─┐
  │ 3 │  │ 2 │  │  11 │  ← leaves return own value
  └───┘  └───┘  └─────┘
  dp=3   dp=2   dp=11

  Step-by-step:
    Node 3  (leaf) → dp = 3
    Node 2  (leaf) → dp = 2
    Node 11 (leaf) → dp = 11
    Node 5  → best child = max(3,2) = 3 → dp = 5+3 = 8
    Node -3 → only child = 11          → dp = -3+11 = 8
    Node 10 → best child = max(8,8) = 8 → dp = 10+8 = 18

  Result: 18  (path: 10 → 5 → 3)
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

**Textual Figure:**

```
Include/Exclude DP — (rob, skip) per node, post-order:

          ┌───┐  rob = 3 + skip(2) + skip(3R) = 3+3+1 = 7
          │ 3 │  skip = max(2,3) + max(3,1) = 3+3 = 6
          └─┬─┘
         ┌──┴──┐
      ┌──┴─┐ ┌─┴──┐
      │  2 │ │  3R │
      └──┬─┘ └──┬─┘
      rob=2   rob=3
      skip=3  skip=1
         └┐      └┐
       ┌──┴─┐  ┌──┴─┐
       │  3 │  │  1  │  ← leaves: (rob=val, skip=0)
       └────┘  └────┘
       (3, 0)  (1, 0)

  Post-order computation:
    Leaf 3:  rob=3, skip=0
    Node 2:  rob = 2+skip(3) = 2+0 = 2
             skip = max(3,0) = 3
    Leaf 1:  rob=1, skip=0
    Node 3R: rob = 3+skip(1) = 3+0 = 3
             skip = max(1,0) = 1
    Root 3:  rob = 3+3+1 = 7   ← root + grandchildren
             skip = 3+3 = 6

  Result: max(7, 6) = 7
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

**Textual Figure:**

```
Tree Diameter — track height(h) and update diameter at each node:

          ┌───┐  h=3, l+r = 2+1 = 3 ← diameter!
          │ 1 │
          └─┬─┘
        ┌───┴───┐
     ┌──┴─┐  ┌──┴─┐
     │  2 │  │  3 │  h=1, l+r=0
     └──┬─┘  └────┘
     h=2, l+r = 1+1 = 2
    ┌───┴───┐
 ┌──┴─┐ ┌──┴─┐
 │  4 │ │  5 │  h=1 (leaves)
 └────┘ └────┘

  Post-order height computation:
    Node 4: l=0, r=0 → diam 0,  h=1
    Node 5: l=0, r=0 → diam 0,  h=1
    Node 2: l=1, r=1 → diam=max(0,2)=2, h=2
    Node 3: l=0, r=0 → diam=max(2,0)=2, h=1
    Node 1: l=2, r=1 → diam=max(2,3)=3, h=3

  Diameter path: 4 → 2 → 1 → 3  (3 edges)
  Result: 3
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

**Textual Figure:**

```
Max Path Sum — maxGain returns best single-branch gain to parent:

         ┌─────┐  pathSum = -10+9+35 = 34
         │ -10 │  gain = -10 + max(9,35) = 25
         └──┬──┘
        ┌───┴────┐
     ┌──┴─┐   ┌──┴──┐  pathSum = 20+15+7 = 42 ★
     │  9 │   │  20 │  gain = 20+15 = 35
     └────┘   └──┬──┘
     gain=9   ┌──┴───┐
           ┌──┴──┐ ┌─┴──┐
           │  15 │ │  7 │  (leaves, gain=own value)
           └─────┘ └────┘
           gain=15  gain=7

  Post-order trace:
    Node 9:   lG=0, rG=0 → path=9,   maxSum=9,   return 9
    Node 15:  lG=0, rG=0 → path=15,  maxSum=15,  return 15
    Node 7:   lG=0, rG=0 → path=7,   maxSum=15,  return 7
    Node 20:  lG=15,rG=7 → path=42,  maxSum=42,  return 35
    Node -10: lG=9, rG=35→ path=34,  maxSum=42,  return 25

  Result: 42  (path: 15 → 20 → 7)
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

**Textual Figure:**

```
Subtree Size — DFS post-order accumulation (N-ary tree):

          ┌───┐  size = 1+3+1+1 = 6
          │ 0 │
          └─┬─┘
       ┌────┼────┐
    ┌──┴─┐┌─┴──┐┌┴──┐
    │  1 ││  2 ││ 3 │  size=1
    └──┬─┘└────┘└───┘
    size=3 size=1
   ┌──┴──┐
┌──┴─┐┌──┴─┐
│  4 ││  5 │  size=1 (leaves)
└────┘└────┘

  DFS order and computation:
    Node 4 → size[4] = 1
    Node 5 → size[5] = 1
    Node 1 → size[1] = 1 + 1 + 1 = 3
    Node 2 → size[2] = 1
    Node 3 → size[3] = 1
    Node 0 → size[0] = 1 + 3 + 1 + 1 = 6

  Result: [6, 3, 1, 1, 1, 1]
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

**Textual Figure:**

```
Re-rooting — two DFS passes, O(n) total:

  Tree: 0─1, 0─2, 2─3, 2─4, 2─5
        ┌───┐
        │ 0 │
        └─┬─┘
      ┌──┴──┐
   ┌──┴┐ ┌─┴─┐
   │ 1 │ │ 2 │
   └──┘ └─┬─┘
         ┌─┼──┐
       ┌─┴┐┌┴┐┌┴─┐
       │ 3││ 4││ 5 │
       └─┘└─┘└──┘

  DFS 1 (root=0): compute count[] and dist[]
    count: [6, 1, 4, 1, 1, 1]
    dist[0] = 8  (sum of distances from node 0 to all others)

  DFS 2 (re-root): dist[child] = dist[parent] - count[child] + (n - count[child])
    ┌──────┬───────┬───────────────────────────────┐
    │ node │ count │ dist[] computation                │
    ├──────┼───────┼───────────────────────────────┤
    │  0   │   6   │ dist[0] = 8  (from DFS 1)        │
    │  1   │   1   │ 8 - 1 + (6-1)  = 12             │
    │  2   │   4   │ 8 - 4 + (6-4)  = 6              │
    │  3   │   1   │ 6 - 1 + (6-1)  = 10             │
    │  4   │   1   │ 6 - 1 + (6-1)  = 10             │
    │  5   │   1   │ 6 - 1 + (6-1)  = 10             │
    └──────┴───────┴───────────────────────────────┘

  Result: [8, 12, 6, 10, 10, 10]
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

**Textual Figure:**

```
Leaf-pruning — peel leaves layer by layer until ≤2 nodes remain:

Test 1: n=4, edges: 1─0, 1─2, 1─3

   0    2    3       Round 1: leaves = {0, 2, 3}
    \   |   /        remaining = 4 - 3 = 1 (≤ 2) → stop
     ┌─┴─┐
     │ 1 │ ← center    Result: [1]
     └───┘

Test 2: n=6, edges: 3─0, 3─1, 3─2, 3─4, 5─4

  0  1  2             Round 1: leaves = {0,1,2,5} (degree 1)
   \ | /              remaining = 6 - 4 = 2 (≤ 2) → stop
  ┌─┴─┐ ┌──┐ ┌──┐
  │ 3 │─│ 4 │─│ 5 │   Result: [3, 4]
  └───┘ └──┘ └──┘
         ↑
       centers
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

**Textual Figure:**

```
Longest weighted path — track two longest branches per node:

  0 ──5── 1 ──3── 2
           │
           4  (weight 4)
           │
           3

  DFS post-order from node 0:
    Node 2: max1=0           → path=0,   return 0
    Node 3: max1=0           → path=0,   return 0
    Node 1: child dists: 2→0+3=3, 3→0+4=4
             max1=4, max2=3  → path=4+3=7, best=7
             return 4
    Node 0: child dists: 1→4+5=9
             max1=9, max2=0  → path=9+0=9, best=max(7,9)=9
             return 9

  Longest path: 0 ──5── 1 ──4── 3  (total weight = 9)
  Result: 9
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

**Textual Figure:**

```
Good nodes — node.Val ≥ maxSoFar on path from root:

          ┌───┐  maxSoFar=3, 3≥3 ✓ GOOD
          │ 3 │
          └─┬─┘
        ┌──┴────┐
     ┌──┴─┐   ┌──┴─┐
     │  1 │   │  4 │  maxSoFar=3, 4≥3 ✓ GOOD
     └──┬─┘   └──┬─┘
     1<3 ✗  ┌───┴───┐
     ┌─┘  ┌─┴──┐ ┌──┴─┐
  ┌──┴─┐ │  1 │ │  5 │  maxSoFar=4, 5≥4 ✓ GOOD
  │  3 │ └────┘ └────┘
  └────┘ 1<4 ✗
  3≥3 ✓ GOOD

  DFS trace (node, maxSoFar → good?):
    (3, 3) → ✓  maxSoFar=3
    (1, 3) → ✗  1 < 3
    (3, 3) → ✓  3 ≥ 3
    (4, 3) → ✓  maxSoFar=4
    (1, 4) → ✗  1 < 4
    (5, 4) → ✓  5 ≥ 4

  Good nodes: {3, 3, 4, 5} → count = 4
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

**Textual Figure:**

```
Max Independent Set — dp[node][0]=exclude, dp[node][1]=include:

      ┌────────┐  dp[0][0] = max(8,5) + max(0,3) = 8+3 = 11
      │ 0 (w=1)│  dp[0][1] = 1 + dp[1][0] + dp[2][0] = 1+8+0 = 9
      └───┬────┘
     ┌────┴────┐
  ┌──┴────┐ ┌──┴────┐
  │ 1(w=5) │ │ 2(w=3) │  dp[2]=[0, 3]
  └───┬───┘ └───────┘
  dp[1]=[8, 5]
      │
  ┌───┴───┐
  │ 3(w=8) │  dp[3]=[0, 8]  (leaf)
  └───────┘

  Post-order computation:
    Node 3: dp[3] = [0, 8]
    Node 1: dp[1][0] = max(0,8) = 8  (skip 1, can take 3)
            dp[1][1] = 5 + 0 = 5    (take 1, must skip 3)
    Node 2: dp[2] = [0, 3]          (leaf)
    Node 0: dp[0][0] = max(8,5) + max(0,3) = 11
            dp[0][1] = 1 + 8 + 0 = 9

  Result: max(11, 9) = 11  (select nodes 3, 2 → 8+3)
```

---

## Key Takeaways

1. Tree DP = post-order DFS: compute children first, then parent
2. Include/exclude pattern: dp[node][0] and dp[node][1] for binary choices
3. Re-rooting: two DFS passes — first computes one root, second re-roots in O(n)
4. Path problems: track two longest branches at each node
5. Tree DP naturally fits recursive solutions — iterative topological order also works

> **Next up:** Bitmask DP →
