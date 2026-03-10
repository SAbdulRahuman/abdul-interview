# Phase 9: BST — Lowest Common Ancestor in BST

## Overview

The **Lowest Common Ancestor (LCA)** of two nodes `p` and `q` in a BST is the deepest node that is an ancestor of both.

| Property | Detail |
|----------|--------|
| BST LCA | Exploits sorted property — O(h) time |
| General tree LCA | Requires postorder — O(n) time |
| Key insight | If both p,q < node → LCA is in left; if both > node → LCA is in right; otherwise node is LCA |

---

## Example 1: LCA in BST — Iterative (LeetCode 235)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	cur := root
	for cur != nil {
		if p.Val < cur.Val && q.Val < cur.Val {
			cur = cur.Left
		} else if p.Val > cur.Val && q.Val > cur.Val {
			cur = cur.Right
		} else {
			return cur // split point = LCA
		}
	}
	return nil
}

func find(root *TreeNode, val int) *TreeNode {
	for root != nil {
		if val == root.Val { return root }
		if val < root.Val { root = root.Left } else { root = root.Right }
	}
	return nil
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	tests := [][2]int{{5, 15}, {5, 35}, {25, 35}, {10, 15}, {5, 10}}
	for _, t := range tests {
		p, q := find(root, t[0]), find(root, t[1])
		lca := lowestCommonAncestor(root, p, q)
		fmt.Printf("LCA(%d, %d) = %d\n", t[0], t[1], lca.Val)
	}
}
```

**Textual Figure:**
```
 BST:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 LCA algorithm: walk down from root until p and q split

 ┌──────────────┬─────┬────────────────────────────────────┐
 │   Query      │ LCA │ Trace                              │
 ├──────────────┼─────┼────────────────────────────────────┤
 │ LCA(5, 15)   │ 10  │ 20: both<20→L; 10: 5<10,15>10→ret │
 │ LCA(5, 35)   │ 20  │ 20: 5<20,35>20 → split → ret 20   │
 │ LCA(25, 35)  │ 30  │ 20: both>20→R; 30: 25<30,35>30→ret│
 │ LCA(10, 15)  │ 10  │ 20: both<20→L; 10: 10=cur → ret   │
 │ LCA(5, 10)   │ 10  │ 20: both<20→L; 10: 5<10,10=cur→ret│
 └──────────────┴─────┴────────────────────────────────────┘

 Key insight: LCA is the SPLIT POINT where p and q
 diverge to different subtrees (or one equals current)
```

---

## Example 2: LCA in BST — Recursive

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func lcaRecursive(root, p, q *TreeNode) *TreeNode {
	if p.Val < root.Val && q.Val < root.Val {
		return lcaRecursive(root.Left, p, q)
	}
	if p.Val > root.Val && q.Val > root.Val {
		return lcaRecursive(root.Right, p, q)
	}
	return root
}

func main() {
	root := &TreeNode{6,
		&TreeNode{2,
			&TreeNode{0, nil, nil},
			&TreeNode{4, &TreeNode{3, nil, nil}, &TreeNode{5, nil, nil}}},
		&TreeNode{8, &TreeNode{7, nil, nil}, &TreeNode{9, nil, nil}},
	}
	p := &TreeNode{Val: 2}
	q := &TreeNode{Val: 8}
	fmt.Println("LCA(2,8):", lcaRecursive(root, p, q).Val) // 6

	p = &TreeNode{Val: 2}
	q = &TreeNode{Val: 4}
	fmt.Println("LCA(2,4):", lcaRecursive(root, p, q).Val) // 2
}
```

**Textual Figure:**
```
 BST:
          6
         / \
        2   8
       / \ / \
      0  4 7  9
        / \
       3   5

 LCA(2, 8) — recursive trace:
   lcaRecursive(6, 2, 8)
   │ 2 < 6 and 8 > 6 → split point → return 6 ✓

 LCA(2, 4) — recursive trace:
   lcaRecursive(6, 2, 4)
   │ both < 6 → go left
   └─ lcaRecursive(2, 2, 4)
      │ 2 = cur.Val → not both < or both > → return 2 ✓

 Call stack visualization:
 ┌──────────────────────────────────────┐
 │ lcaRecursive(6, p=2, q=4)           │
 │   both < 6 → recurse left           │
 │ ┌──────────────────────────────────┐ │
 │ │ lcaRecursive(2, p=2, q=4)       │ │
 │ │   2≤2, 4>2 → split → return 2   │ │
 │ └──────────────────────────────────┘ │
 └──────────────────────────────────────┘
```

---

## Example 3: LCA in General Binary Tree (LeetCode 236)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	if root == nil || root == p || root == q {
		return root
	}
	left := lowestCommonAncestor(root.Left, p, q)
	right := lowestCommonAncestor(root.Right, p, q)
	if left != nil && right != nil {
		return root // both found → root is LCA
	}
	if left != nil { return left }
	return right
}

func main() {
	n5 := &TreeNode{Val: 5}
	n1 := &TreeNode{Val: 1}
	n4 := &TreeNode{Val: 4}
	root := &TreeNode{3,
		&TreeNode{5,
			&TreeNode{6, nil, nil},
			&TreeNode{2, &TreeNode{7, nil, nil}, n4}},
		&TreeNode{1, &TreeNode{0, nil, nil}, &TreeNode{8, nil, nil}},
	}
	// Fix references
	root.Left = n5
	n5.Left = &TreeNode{6, nil, nil}
	n5.Right = &TreeNode{2, &TreeNode{7, nil, nil}, n4}
	root.Right = n1
	n1.Left = &TreeNode{0, nil, nil}
	n1.Right = &TreeNode{8, nil, nil}

	fmt.Println("LCA(5,1):", lowestCommonAncestor(root, n5, n1).Val) // 3
	fmt.Println("LCA(5,4):", lowestCommonAncestor(root, n5, n4).Val) // 5
}
```

**Textual Figure:**
```
 Binary Tree (NOT a BST):
           3
          / \
         5   1
        / \ / \
       6  2 0  8
         / \
        7   4

 LCA(5, 1) — postorder search:
 ┌─────────────────────────────────────────┐
 │ lca(3)                                  │
 │  ├─ left  = lca(5) → found 5 → ret 5   │
 │  ├─ right = lca(1) → found 1 → ret 1   │
 │  └─ both non-nil → return 3 ← LCA      │
 └─────────────────────────────────────────┘

 LCA(5, 4) — postorder search:
 ┌─────────────────────────────────────────┐
 │ lca(3)                                  │
 │  ├─ left = lca(5)                       │
 │  │   5 == p → return 5                  │
 │  │   (4 is in subtree of 5)             │
 │  ├─ right = lca(1) → nil               │
 │  └─ only left non-nil → return 5 ← LCA │
 └─────────────────────────────────────────┘

 BST vs General Tree LCA:
 ┌───────────────┬────────────┬─────────────┐
 │               │   BST LCA  │ General LCA │
 ├───────────────┼────────────┼─────────────┤
 │ Strategy      │ Split point│ Postorder   │
 │ Time          │   O(h)     │   O(n)      │
 │ Space         │   O(1)     │   O(h)      │
 └───────────────┴────────────┴─────────────┘
```

---

## Example 4: LCA with Path Recording

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func findPath(root *TreeNode, target int) []int {
	path := []int{}
	cur := root
	for cur != nil {
		path = append(path, cur.Val)
		if target == cur.Val {
			return path
		} else if target < cur.Val {
			cur = cur.Left
		} else {
			cur = cur.Right
		}
	}
	return nil // not found
}

func lcaWithPath(root *TreeNode, p, q int) (int, []int, []int) {
	pathP := findPath(root, p)
	pathQ := findPath(root, q)
	lca := root.Val
	for i := 0; i < len(pathP) && i < len(pathQ); i++ {
		if pathP[i] == pathQ[i] {
			lca = pathP[i]
		} else {
			break
		}
	}
	return lca, pathP, pathQ
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	lca, p1, p2 := lcaWithPath(root, 5, 25)
	fmt.Printf("LCA(%d,%d) = %d\n", 5, 25, lca)
	fmt.Println("Path to 5: ", p1)
	fmt.Println("Path to 25:", p2)

	lca, p1, p2 = lcaWithPath(root, 5, 15)
	fmt.Printf("LCA(%d,%d) = %d\n", 5, 15, lca)
	fmt.Println("Path to 5: ", p1)
	fmt.Println("Path to 15:", p2)
}
```

**Textual Figure:**
```
 BST:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 LCA(5, 25) via path comparison:
   Path to 5:  [20, 10, 5]
   Path to 25: [20, 30, 25]

   Compare:
     i=0: 20 == 20 → lca=20
     i=1: 10 != 30 → STOP
   LCA = 20 ✓

 LCA(5, 15) via path comparison:
   Path to 5:  [20, 10, 5]
   Path to 15: [20, 10, 15]

   Compare:
     i=0: 20 == 20 → lca=20
     i=1: 10 == 10 → lca=10
     i=2: 5  != 15 → STOP
   LCA = 10 ✓

 Paths diverge at LCA:
       20           20
      ↙   ↘        ↙
    10      30    10
   ↙       ↙     ↙  ↘
  5      25     5    15
```

---

## Example 5: Distance Between Two BST Nodes

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func lca(root *TreeNode, p, q int) *TreeNode {
	cur := root
	for cur != nil {
		if p < cur.Val && q < cur.Val {
			cur = cur.Left
		} else if p > cur.Val && q > cur.Val {
			cur = cur.Right
		} else {
			return cur
		}
	}
	return nil
}

func depth(root *TreeNode, val int) int {
	d := 0
	cur := root
	for cur != nil {
		if val == cur.Val { return d }
		if val < cur.Val { cur = cur.Left } else { cur = cur.Right }
		d++
	}
	return -1
}

func distance(root *TreeNode, p, q int) int {
	ancestor := lca(root, p, q)
	return depth(ancestor, p) + depth(ancestor, q)
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	pairs := [][2]int{{5, 15}, {5, 35}, {10, 30}, {25, 35}}
	for _, pair := range pairs {
		d := distance(root, pair[0], pair[1])
		fmt.Printf("Distance(%d, %d) = %d\n", pair[0], pair[1], d)
	}
}
```

**Textual Figure:**
```
 BST:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 Distance(p, q) = depth(LCA→p) + depth(LCA→q)

 ┌──────────────┬─────┬────────────┬────────────┬──────────┐
 │    Query     │ LCA │ depth(→p)  │ depth(→q)  │ Distance │
 ├──────────────┼─────┼────────────┼────────────┼──────────┤
 │ dist(5, 15)  │ 10  │ 10→5 = 1  │ 10→15 = 1  │    2     │
 │ dist(5, 35)  │ 20  │ 20→10→5=2 │ 20→30→35=2 │    4     │
 │ dist(10, 30) │ 20  │ 20→10 = 1 │ 20→30 = 1  │    2     │
 │ dist(25, 35) │ 30  │ 30→25 = 1 │ 30→35 = 1  │    2     │
 └──────────────┴─────┴────────────┴────────────┴──────────┘

 Example: dist(5, 35):
          20   ← LCA
         ↙  ↘
       10    30
      ↙        ↘
     5          35
   depth=2    depth=2   → total = 4
```

---

## Example 6: LCA Queries in Batch

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func lca(root *TreeNode, p, q int) int {
	cur := root
	for cur != nil {
		if p < cur.Val && q < cur.Val {
			cur = cur.Left
		} else if p > cur.Val && q > cur.Val {
			cur = cur.Right
		} else {
			return cur.Val
		}
	}
	return -1
}

func main() {
	var root *TreeNode
	for _, v := range []int{50, 30, 70, 20, 40, 60, 80, 10, 25, 35, 45} {
		root = insert(root, v)
	}

	queries := [][2]int{
		{10, 25}, {10, 45}, {35, 80}, {60, 80}, {20, 40}, {10, 80},
	}
	for _, q := range queries {
		fmt.Printf("LCA(%d, %d) = %d\n", q[0], q[1], lca(root, q[0], q[1]))
	}
}
```

**Textual Figure:**
```
 BST:
              50
            /    \
          30      70
         / \     /  \
       20   40  60   80
      / \  / \
    10 25 35 45

 Batch LCA queries:
 ┌──────────────┬─────┬──────────────────────────────────┐
 │    Query     │ LCA │ Trace                            │
 ├──────────────┼─────┼──────────────────────────────────┤
 │ LCA(10, 25)  │  20 │ 50→L; 30→L; 20: 10<20,25>20→ret │
 │ LCA(10, 45)  │  30 │ 50→L; 30: 10<30,45>30 → ret     │
 │ LCA(35, 80)  │  50 │ 50: 35<50,80>50 → ret            │
 │ LCA(60, 80)  │  70 │ 50→R; 70: 60<70,80>70 → ret     │
 │ LCA(20, 40)  │  30 │ 50→L; 30: 20<30,40>30 → ret     │
 │ LCA(10, 80)  │  50 │ 50: 10<50,80>50 → ret            │
 └──────────────┴─────┴──────────────────────────────────┘

 Pattern: LCA is always at the SPLIT POINT
   where p goes left and q goes right (or vice versa)
```

---

## Example 7: LCA — Check if Node is Ancestor

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func isAncestor(root *TreeNode, ancestor, node int) bool {
	cur := root
	// First find ancestor
	for cur != nil && cur.Val != ancestor {
		if ancestor < cur.Val { cur = cur.Left } else { cur = cur.Right }
	}
	if cur == nil { return false }
	// Now check if node is in subtree of ancestor
	return search(cur, node)
}

func search(root *TreeNode, val int) bool {
	for root != nil {
		if val == root.Val { return true }
		if val < root.Val { root = root.Left } else { root = root.Right }
	}
	return false
}

func lca(root *TreeNode, p, q int) int {
	cur := root
	for cur != nil {
		if p < cur.Val && q < cur.Val { cur = cur.Left } else if p > cur.Val && q > cur.Val { cur = cur.Right } else { return cur.Val }
	}
	return -1
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	fmt.Println("Is 20 ancestor of 5?", isAncestor(root, 20, 5))  // true
	fmt.Println("Is 10 ancestor of 25?", isAncestor(root, 10, 25)) // false
	fmt.Println("Is 30 ancestor of 35?", isAncestor(root, 30, 35)) // true
	fmt.Println("LCA(5,15):", lca(root, 5, 15))                     // 10
}
```

**Textual Figure:**
```
 BST:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 Ancestor checks:
 ┌─────────────────────────┬────────┬─────────────────────────┐
 │ Query                   │ Result │ Reasoning               │
 ├─────────────────────────┼────────┼─────────────────────────┤
 │ Is 20 ancestor of 5?    │  true  │ 20→10→5 found in subtree│
 │ Is 10 ancestor of 25?   │ false  │ 10's subtree: {5,10,15} │
 │                         │        │ 25 not in subtree       │
 │ Is 30 ancestor of 35?   │  true  │ 30→35 found in subtree  │
 └─────────────────────────┴────────┴─────────────────────────┘

 Subtree view:
       20 ← ancestor of all
      /  \
    10    30 ← ancestor of {25,35}
   / \   / \
  5  15 25  35

 LCA(5,15) = 10:
   10 is the deepest node that is ancestor of BOTH 5 and 15
```

---

## Example 8: LCA with Path Printing

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func pathFromRoot(root *TreeNode, val int) []int {
	path := []int{}
	cur := root
	for cur != nil {
		path = append(path, cur.Val)
		if val == cur.Val { break }
		if val < cur.Val { cur = cur.Left } else { cur = cur.Right }
	}
	return path
}

func lcaAndPath(root *TreeNode, p, q int) {
	pathP := pathFromRoot(root, p)
	pathQ := pathFromRoot(root, q)

	// Find LCA
	lca := root.Val
	for i := 0; i < len(pathP) && i < len(pathQ); i++ {
		if pathP[i] == pathQ[i] { lca = pathP[i] } else { break }
	}

	// Path from p to q goes through LCA
	fmt.Printf("Path %d → %d: ", p, q)
	// p to LCA (reverse the part after LCA)
	lcaIdx := -1
	for i, v := range pathP { if v == lca { lcaIdx = i; break } }
	for i := len(pathP) - 1; i > lcaIdx; i-- { fmt.Printf("%d → ", pathP[i]) }
	fmt.Printf("%d", lca)
	for i := lcaIdx + 1; i < len(pathQ); i++ { fmt.Printf(" → %d", pathQ[i]) }
	fmt.Println()
}

func main() {
	var root *TreeNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}
	lcaAndPath(root, 5, 35)
	lcaAndPath(root, 5, 15)
	lcaAndPath(root, 25, 35)
}
```

**Textual Figure:**
```
 BST:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 Path from p to q through LCA:

 Path 5 → 35:  LCA = 20
   5 → 10 → [20] → 30 → 35
        ↑     ↑          ↑
       up    LCA       down

 Path 5 → 15:  LCA = 10
   5 → [10] → 15
        ↑
       LCA

 Path 25 → 35:  LCA = 30
   25 → [30] → 35
          ↑
         LCA

 Visual path for 5 → 35:
          20  ← LCA (turn point)
         ↗  ↘
       10    30
      ↗        ↘
     5          35
   (start)    (end)
```

---

## Example 9: LCA — Count Nodes Between Two Values

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func countBetween(root *TreeNode, lo, hi int) int {
	if root == nil { return 0 }
	if root.Val < lo { return countBetween(root.Right, lo, hi) }
	if root.Val > hi { return countBetween(root.Left, lo, hi) }
	return 1 + countBetween(root.Left, lo, hi) + countBetween(root.Right, lo, hi)
}

func lca(root *TreeNode, p, q int) int {
	cur := root
	for cur != nil {
		if p < cur.Val && q < cur.Val { cur = cur.Left } else if p > cur.Val && q > cur.Val { cur = cur.Right } else { return cur.Val }
	}
	return -1
}

func main() {
	var root *TreeNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35, 3, 7, 12, 18} {
		root = insert(root, v)
	}
	fmt.Println("LCA(3, 18):", lca(root, 3, 18))    // 10
	fmt.Println("Count [3,18]:", countBetween(root, 3, 18)) // 7
	fmt.Println("LCA(5, 35):", lca(root, 5, 35))    // 20
	fmt.Println("Count [5,35]:", countBetween(root, 5, 35)) // 9
}
```

**Textual Figure:**
```
 BST:
              20
            /    \
          10      30
         / \     /  \
        5   15  25   35
       / \  / \
      3  7 12 18

 LCA(3, 18) = 10
   Nodes in range [3, 18]:
   3, 5, 7, 10, 12, 15, 18 → count = 7

              20
            /
         [10]  ← LCA
         / \
       [5] [15]
       / \  / \
     [3][7][12][18]     ← all in range [3,18]

 LCA(5, 35) = 20
   Nodes in range [5, 35]:
   5, 7, 10, 12, 15, 18, 20, 25, 30, 35 → count = 10

 countBetween prunes branches outside [lo, hi]:
   If node < lo → only search right
   If node > hi → only search left
   Otherwise → count node + search both
```

---

## Example 10: LCA — Iterative vs Recursive Performance

```go
package main

import (
	"fmt"
	"time"
)

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func lcaIterative(root *TreeNode, p, q int) int {
	cur := root
	for cur != nil {
		if p < cur.Val && q < cur.Val { cur = cur.Left } else if p > cur.Val && q > cur.Val { cur = cur.Right } else { return cur.Val }
	}
	return -1
}

func lcaRecursive(root *TreeNode, p, q int) int {
	if p < root.Val && q < root.Val { return lcaRecursive(root.Left, p, q) }
	if p > root.Val && q > root.Val { return lcaRecursive(root.Right, p, q) }
	return root.Val
}

func main() {
	var root *TreeNode
	n := 100000
	for i := 0; i < n; i++ {
		root = insert(root, i*2+1) // odd numbers, will be unbalanced
	}

	p, q := 1, 2*n-1
	iterations := 1000

	start := time.Now()
	for i := 0; i < iterations; i++ { lcaIterative(root, p, q) }
	iterTime := time.Since(start)

	start = time.Now()
	for i := 0; i < iterations; i++ { lcaRecursive(root, p, q) }
	recurTime := time.Since(start)

	fmt.Printf("Iterative: %v\n", iterTime)
	fmt.Printf("Recursive: %v\n", recurTime)
	fmt.Printf("LCA(%d, %d) = %d\n", p, q, lcaIterative(root, p, q))
}
```

**Textual Figure:**
```
 Performance comparison: Iterative vs Recursive LCA

 ┌─────────────────┬────────────┬────────────────────────┐
 │ Approach        │ Space      │ Notes                  │
 ├─────────────────┼────────────┼────────────────────────┤
 │ Iterative       │ O(1)       │ No stack overhead      │
 │ Recursive       │ O(h)       │ Call stack = tree height│
 └─────────────────┴────────────┴────────────────────────┘

 For BST with 100,000 nodes (unbalanced):
   Height ≈ 100,000 (degenerate case)

 Iterative:                 Recursive:
  cur = root                 lcaRecursive(root)
  while cur != nil             ↓
    compare & move           lcaRecursive(child)
  return cur                   ↓
                             lcaRecursive(child)
  Space: O(1)                  ↓ ... O(h) stack frames

 LCA(1, 199999) = 1  (smallest node, split at root)

 Recommendation:
   → Use ITERATIVE for BST LCA (O(1) space)
   → Use RECURSIVE for general tree LCA (need postorder)
```

---

## Key Takeaways

1. **BST LCA** exploits sorted property — walk to split point in O(h) with O(1) space
2. **General tree LCA** requires postorder traversal — O(n) time, O(h) space
3. LCA is the **split point** where p and q diverge to different subtrees
4. **Distance** between two nodes = depth(LCA→p) + depth(LCA→q)
5. Prefer **iterative** BST LCA to avoid stack overflow on deep trees

> **Next up:** Kth Smallest in BST →
