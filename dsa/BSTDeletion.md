# Phase 9: BST — BST Deletion

## Overview

Deleting from a BST has three cases:

| Case | Action |
|------|--------|
| **Leaf node** (no children) | Simply remove it |
| **One child** | Replace node with its child |
| **Two children** | Replace with inorder successor (or predecessor), then delete the successor |

Time: O(h), where h = tree height.

---

## Example 1: Basic Recursive Deletion (LeetCode 450)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func deleteNode(root *TreeNode, key int) *TreeNode {
	if root == nil { return nil }
	if key < root.Val {
		root.Left = deleteNode(root.Left, key)
	} else if key > root.Val {
		root.Right = deleteNode(root.Right, key)
	} else {
		// Found node to delete
		if root.Left == nil { return root.Right }
		if root.Right == nil { return root.Left }
		// Two children: find inorder successor
		succ := root.Right
		for succ.Left != nil { succ = succ.Left }
		root.Val = succ.Val
		root.Right = deleteNode(root.Right, succ.Val)
	}
	return root
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{2, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{6, nil, &TreeNode{7, nil, nil}},
	}
	fmt.Print("Before: "); inorder(root); fmt.Println()
	root = deleteNode(root, 3)
	fmt.Print("After delete 3: "); inorder(root); fmt.Println()
	root = deleteNode(root, 5)
	fmt.Print("After delete 5: "); inorder(root); fmt.Println()
}
```

**Textual Figure:**
```
BST Deletion — Three Cases:

  Initial tree:        Delete 3 (two children):   Delete 5 (two children):
       5                    5                          6
      / \                  / \                        / \
     3   6                4   6                      2   6  ← was 7
    / \   \              /     \                    /     \
   2   4   7            2       7                  4       7

  Delete 3:
  ┌──────────────────────────────────────────┐
  │ Node 3 has TWO children (2 and 4)         │
  │ Find inorder successor: 4 (min of right)  │
  │ Copy 4’s value to node 3’s position       │
  │ Delete successor 4 from right subtree     │
  └──────────────────────────────────────────┘
  Before: 2 3 4 5 6 7
  After delete 3: 2 4 5 6 7
  After delete 5: 2 4 6 7
```

---

## Example 2: Iterative Deletion

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func findMin(node *TreeNode) *TreeNode {
	for node.Left != nil { node = node.Left }
	return node
}

func deleteIterative(root *TreeNode, key int) *TreeNode {
	dummy := &TreeNode{Right: root}
	parent, cur, isLeft := dummy, root, false

	// Find the node
	for cur != nil && cur.Val != key {
		parent = cur
		if key < cur.Val {
			cur = cur.Left
			isLeft = true
		} else {
			cur = cur.Right
			isLeft = false
		}
	}
	if cur == nil { return dummy.Right } // not found

	// Case: two children
	if cur.Left != nil && cur.Right != nil {
		succParent := cur
		succ := cur.Right
		for succ.Left != nil {
			succParent = succ
			succ = succ.Left
		}
		cur.Val = succ.Val
		// Now delete successor (0 or 1 child)
		if succParent == cur {
			succParent.Right = succ.Right
		} else {
			succParent.Left = succ.Right
		}
		return dummy.Right
	}

	// Case: 0 or 1 child
	var child *TreeNode
	if cur.Left != nil { child = cur.Left } else { child = cur.Right }
	if isLeft || parent == dummy {
		if parent == dummy { parent.Right = child } else { parent.Left = child }
	} else {
		parent.Right = child
	}
	return dummy.Right
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{2, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{8, nil, nil}},
	}
	root = deleteIterative(root, 5)
	inorder(root) // 2 3 4 6 7 8
	fmt.Println()
}
```

**Textual Figure:**
```
Iterative Deletion — Delete 5 (root, two children):

  Before:              After:
       5                   6
      / \                 / \
     3   7               3   7
    / \ / \             / \   \
   2  4 6  8           2   4   8

  Steps:
  ┌────────────────────────────────────────────┐
  │ 1. Find node 5 (it’s the root)              │
  │ 2. Two children → find inorder successor    │
  │    succParent=5, succ=7                     │
  │    succ.Left=6 → succParent=7, succ=6       │
  │    succ.Left=nil → succ=6 is the successor  │
  │ 3. Copy: node.Val = 6                       │
  │ 4. Remove succ: succParent(7).Left = nil     │
  └────────────────────────────────────────────┘

  Inorder: 2 3 4 6 7 8 (sorted, BST valid ✓)
```

---

## Example 3: Delete Using Inorder Predecessor

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func deleteWithPredecessor(root *TreeNode, key int) *TreeNode {
	if root == nil { return nil }
	if key < root.Val {
		root.Left = deleteWithPredecessor(root.Left, key)
	} else if key > root.Val {
		root.Right = deleteWithPredecessor(root.Right, key)
	} else {
		if root.Left == nil { return root.Right }
		if root.Right == nil { return root.Left }
		// Use predecessor (max in left subtree)
		pred := root.Left
		for pred.Right != nil { pred = pred.Right }
		root.Val = pred.Val
		root.Left = deleteWithPredecessor(root.Left, pred.Val)
	}
	return root
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, nil, nil},
	}
	root = deleteWithPredecessor(root, 10)
	fmt.Print("After delete 10 (predecessor): ")
	inorder(root) // 3 5 7 15
	fmt.Println()
}
```

**Textual Figure:**
```
Delete Using Inorder Predecessor — Delete 10:

  Before:               After:
       10                   7
      / \                  / \
     5   15               5   15
    / \                  / \
   3   7                3   (removed)

  Steps:
  ┌───────────────────────────────────────────────┐
  │ 1. Find node 10 (root)                       │
  │ 2. Two children → find predecessor            │
  │    pred = max of left subtree                 │
  │    Start at 5, go right: 7 (no right child)  │
  │    Predecessor = 7                            │
  │ 3. Copy: node.Val = 7                         │
  │ 4. Recursively delete 7 from left subtree     │
  │    7 is a leaf → simply remove                │
  └───────────────────────────────────────────────┘

  Successor approach:  replace with min of right (next larger)
  Predecessor approach: replace with max of left (next smaller)

  Inorder: 3 5 7 15 (sorted, BST valid ✓)
```

---

## Example 4: Delete All Occurrences (Multiset BST)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
	Freq  int
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val, Freq: 1} }
	if val < root.Val {
		root.Left = insert(root.Left, val)
	} else if val > root.Val {
		root.Right = insert(root.Right, val)
	} else {
		root.Freq++
	}
	return root
}

func deleteOne(root *TreeNode, val int) *TreeNode {
	if root == nil { return nil }
	if val < root.Val {
		root.Left = deleteOne(root.Left, val)
	} else if val > root.Val {
		root.Right = deleteOne(root.Right, val)
	} else {
		if root.Freq > 1 {
			root.Freq--
			return root
		}
		// Freq == 1, actually remove
		if root.Left == nil { return root.Right }
		if root.Right == nil { return root.Left }
		succ := root.Right
		for succ.Left != nil { succ = succ.Left }
		root.Val = succ.Val
		root.Freq = succ.Freq
		root.Right = deleteOne(root.Right, succ.Val)
	}
	return root
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d(x%d) ", r.Val, r.Freq)
	inorder(r.Right)
}

func main() {
	var root *TreeNode
	for _, v := range []int{5, 3, 7, 3, 5, 1} {
		root = insert(root, v)
	}
	fmt.Print("Before: "); inorder(root); fmt.Println()
	root = deleteOne(root, 5)
	fmt.Print("After delete one 5: "); inorder(root); fmt.Println()
	root = deleteOne(root, 5)
	fmt.Print("After delete another 5: "); inorder(root); fmt.Println()
}
```

**Textual Figure:**
```
Multiset BST — Delete with Frequency Tracking:

  Insert [5, 3, 7, 3, 5, 1]:

       5(x2)
      / \
    3(x2) 7(x1)
    /
  1(x1)

  Delete one 5:            Delete another 5:
     5(x1)  ← freq--        Actual removal!
    / \                   Find successor: 7
  3(x2) 7(x1)             Copy val+freq, delete 7
  /
 1(x1)                       7(x1)
                             /
                           3(x2)
                           /
                         1(x1)

  ┌────────────────────────────────────────┐
  │ Before:            1(x1) 3(x2) 5(x2) 7(x1)│
  │ After delete one 5: 1(x1) 3(x2) 5(x1) 7(x1)│
  │ After delete two 5: 1(x1) 3(x2) 7(x1)      │
  └────────────────────────────────────────┘
```

---

## Example 5: Delete Leaf Nodes

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func deleteLeaves(root *TreeNode) *TreeNode {
	if root == nil { return nil }
	if root.Left == nil && root.Right == nil {
		return nil // leaf → remove
	}
	root.Left = deleteLeaves(root.Left)
	root.Right = deleteLeaves(root.Right)
	return root
}

func printTree(r *TreeNode, prefix string) {
	if r == nil { return }
	fmt.Printf("%s%d\n", prefix, r.Val)
	printTree(r.Left, prefix+"  L:")
	printTree(r.Right, prefix+"  R:")
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{2, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{8, nil, nil}},
	}
	fmt.Println("Before:")
	printTree(root, "")
	root = deleteLeaves(root)
	fmt.Println("After removing all leaves:")
	printTree(root, "")
}
```

**Textual Figure:**
```
Delete All Leaf Nodes:

  Before:                After:
       5                    5
      / \                  / \
     3   7                3   7
    / \ / \
   2  4 6  8         (all leaves removed)

  Process (post-order):
  ┌─────────────────────────────────────┐
  │ Visit 2: leaf → return nil (remove) │
  │ Visit 4: leaf → return nil (remove) │
  │ Visit 3: not leaf → keep            │
  │ Visit 6: leaf → return nil (remove) │
  │ Visit 8: leaf → return nil (remove) │
  │ Visit 7: not leaf → keep            │
  │ Visit 5: not leaf → keep            │
  └─────────────────────────────────────┘

  printTree output:
  Before: 5, L:3, L:2, R:4, R:7, L:6, R:8
  After:  5, L:3, R:7
```

---

## Example 6: Delete Nodes Outside Range (Trim BST — LeetCode 669)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func trimBST(root *TreeNode, low, high int) *TreeNode {
	if root == nil { return nil }
	if root.Val < low {
		return trimBST(root.Right, low, high) // discard root and left
	}
	if root.Val > high {
		return trimBST(root.Left, low, high) // discard root and right
	}
	root.Left = trimBST(root.Left, low, high)
	root.Right = trimBST(root.Right, low, high)
	return root
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, &TreeNode{12, nil, nil}, &TreeNode{20, nil, nil}},
	}
	inorder(root); fmt.Println()       // 3 5 7 10 12 15 20
	root = trimBST(root, 5, 15)
	inorder(root); fmt.Println()       // 5 7 10 12 15
}
```

**Textual Figure:**
```
Trim BST — Remove nodes outside range [5, 15]:

  Before:                 After trimming [5,15]:
         10                      10
        / \                     / \
       5   15                  5   15
      / \  / \                  \  /
     3  7 12  20                7 12

  Process:
  ┌─────────────────────────────────────────────┐
  │ trim(10, 5, 15):                             │
  │   10∈[5,15] → keep                           │
  │   Left:  trim(5, 5, 15)                      │
  │     5∈[5,15] → keep                          │
  │     Left: trim(3, 5, 15)                     │
  │       3 < 5 → discard 3 & its left           │
  │       return trim(nil) = nil                  │
  │     Right: trim(7) → 7∈[5,15], keep (leaf)   │
  │   Right: trim(15, 5, 15)                     │
  │     15∈[5,15] → keep                         │
  │     Left: trim(12) → 12∈[5,15], keep         │
  │     Right: trim(20)                          │
  │       20 > 15 → discard 20 & its right       │
  └─────────────────────────────────────────────┘

  Inorder before: 3 5 7 10 12 15 20
  Inorder after:  5 7 10 12 15
```

---

## Example 7: Delete Nodes and Return Forest (LeetCode 1110)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func delNodes(root *TreeNode, toDelete []int) []*TreeNode {
	delSet := map[int]bool{}
	for _, v := range toDelete { delSet[v] = true }
	forest := []*TreeNode{}

	var dfs func(node *TreeNode, isRoot bool) *TreeNode
	dfs = func(node *TreeNode, isRoot bool) *TreeNode {
		if node == nil { return nil }
		deleted := delSet[node.Val]
		if isRoot && !deleted {
			forest = append(forest, node)
		}
		node.Left = dfs(node.Left, deleted)
		node.Right = dfs(node.Right, deleted)
		if deleted { return nil }
		return node
	}
	dfs(root, true)
	return forest
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{1,
		&TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
		&TreeNode{3, &TreeNode{6, nil, nil}, &TreeNode{7, nil, nil}},
	}
	forest := delNodes(root, []int{3, 5})
	fmt.Println("Forest trees:")
	for _, t := range forest {
		inorder(t); fmt.Println()
	}
	// Tree rooted at 1: 4 2 1 6 7
	// Tree rooted at 6: 6
	// Tree rooted at 7: 7
}
```

**Textual Figure:**
```
Delete Nodes and Return Forest — delete [3, 5]:

  Before:                 After deleting {3, 5}:
       1                       1
      / \                     /
     2   3  ← delete         2
    / \ / \                 /
   4  5 6  7              4     6    7
      ↑ delete
                      (3 separate trees in forest)

  Process (DFS post-order):
  ┌───────────────────────────────────────────────┐
  │ dfs(1, isRoot=true):                           │
  │   1 not in delSet → add to forest              │
  │   dfs(2, deleted=false):                       │
  │     dfs(4, false) → keep                       │
  │     dfs(5, false) → 5 in delSet!               │
  │       children are roots → but 5 is leaf        │
  │       return nil                                │
  │   dfs(3, deleted=false):                       │
  │     3 in delSet! → children become roots        │
  │     dfs(6, isRoot=true) → add 6 to forest      │
  │     dfs(7, isRoot=true) → add 7 to forest      │
  └───────────────────────────────────────────────┘

  Forest: [Tree(1: 4,2), Tree(6), Tree(7)]
```

---

## Example 8: Delete and Rebalance (Rebuild)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func deleteNode(root *TreeNode, key int) *TreeNode {
	if root == nil { return nil }
	if key < root.Val { root.Left = deleteNode(root.Left, key) } else if key > root.Val { root.Right = deleteNode(root.Right, key) } else {
		if root.Left == nil { return root.Right }
		if root.Right == nil { return root.Left }
		succ := root.Right; for succ.Left != nil { succ = succ.Left }
		root.Val = succ.Val; root.Right = deleteNode(root.Right, succ.Val)
	}
	return root
}

func collectInorder(r *TreeNode, vals *[]int) {
	if r == nil { return }
	collectInorder(r.Left, vals)
	*vals = append(*vals, r.Val)
	collectInorder(r.Right, vals)
}

func buildBalanced(vals []int, lo, hi int) *TreeNode {
	if lo > hi { return nil }
	mid := (lo + hi) / 2
	return &TreeNode{vals[mid], buildBalanced(vals, lo, mid-1), buildBalanced(vals, mid+1, hi)}
}

func height(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := height(r.Left), height(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func main() {
	// Build skewed tree
	var root *TreeNode
	for _, v := range []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10} {
		root = &TreeNode{v, nil, root}
	}
	fmt.Println("Before rebalance, height:", height(root)) // 10

	// Delete 5 then rebalance
	root = deleteNode(root, 5)
	vals := []int{}
	collectInorder(root, &vals)
	root = buildBalanced(vals, 0, len(vals)-1)
	fmt.Println("After delete 5 + rebalance, height:", height(root)) // 4
}
```

**Textual Figure:**
```
Delete and Rebalance (Rebuild from sorted values):

  Before (skewed):       After delete 5:        After rebalance:
  10                     10                          6
   \                      \                        /   \
    9                      9                      3     9
     \                      \                    / \   / \
      8                      8                  2   4  7  10
       \                      \                /       \
        7                      7              1         8
         \                      \
          ...                    6
           \                      \
            1                      ...
                                    1
  height=10               height=9            height=4

  Process:
  ┌───────────────────────────────────────────┐
  │ 1. Delete node 5 from skewed tree         │
  │ 2. Collect inorder: [1,2,3,4,6,7,8,9,10]  │
  │ 3. Build balanced: pick mid recursively    │
  │ 4. Height drops from 9 → 4                │
  └───────────────────────────────────────────┘
```

---

## Example 9: Lazy Deletion (Mark as Deleted)

```go
package main

import "fmt"

type TreeNode struct {
	Val     int
	Left    *TreeNode
	Right   *TreeNode
	Deleted bool
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else if val > root.Val { root.Right = insert(root.Right, val) } else { root.Deleted = false }
	return root
}

func lazyDelete(root *TreeNode, val int) {
	cur := root
	for cur != nil {
		if val < cur.Val { cur = cur.Left } else if val > cur.Val { cur = cur.Right } else {
			cur.Deleted = true
			return
		}
	}
}

func search(root *TreeNode, val int) bool {
	cur := root
	for cur != nil {
		if val < cur.Val { cur = cur.Left } else if val > cur.Val { cur = cur.Right } else {
			return !cur.Deleted
		}
	}
	return false
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	if !r.Deleted { fmt.Printf("%d ", r.Val) }
	inorder(r.Right)
}

func main() {
	var root *TreeNode
	for _, v := range []int{5, 3, 7, 1, 4, 6, 8} {
		root = insert(root, v)
	}
	fmt.Print("All: "); inorder(root); fmt.Println()
	lazyDelete(root, 3)
	lazyDelete(root, 7)
	fmt.Print("After lazy-deleting 3,7: "); inorder(root); fmt.Println()
	fmt.Println("Search 3:", search(root, 3)) // false
	fmt.Println("Search 5:", search(root, 5)) // true
}
```

**Textual Figure:**
```
Lazy Deletion (mark nodes instead of removing):

  Initial BST:           After lazy-delete 3, 7:
       5                      5
      / \                    / \
     3   7                 [3]  [7]  ← marked Deleted=true
    / \ / \               / \  / \
   1  4 6  8             1  4  6  8

  Tree structure unchanged! Nodes 3,7 still exist but
  are skipped during traversal and search.

  ┌───────────────────────────────────┐
  │ lazyDelete(3):                  │
  │   Walk: 5→3 (found)             │
  │   Set Deleted=true              │
  │                                 │
  │ search(3): walk to 3,           │
  │   found but Deleted=true        │
  │   → return false                │
  │                                 │
  │ search(5): walk to 5,           │
  │   Deleted=false → return true   │
  └───────────────────────────────────┘

  Inorder (active only): 1 4 5 6 8
  Advantage: O(h) delete, no restructuring
```

---

## Example 10: Delete and Merge Subtrees

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// Merge two BSTs where all values in left < all values in right
func merge(left, right *TreeNode) *TreeNode {
	if left == nil { return right }
	if right == nil { return left }
	// Attach right as rightmost of left
	cur := left
	for cur.Right != nil { cur = cur.Right }
	cur.Right = right
	return left
}

func deleteWithMerge(root *TreeNode, key int) *TreeNode {
	if root == nil { return nil }
	if key < root.Val {
		root.Left = deleteWithMerge(root.Left, key)
		return root
	}
	if key > root.Val {
		root.Right = deleteWithMerge(root.Right, key)
		return root
	}
	// Found: merge its left and right subtrees
	return merge(root.Left, root.Right)
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{2, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{8, nil, nil}},
	}
	fmt.Print("Before: "); inorder(root); fmt.Println() // 2 3 4 5 6 7 8
	root = deleteWithMerge(root, 5)
	fmt.Print("After delete 5 (merge): "); inorder(root); fmt.Println() // 2 3 4 6 7 8
}
```

**Textual Figure:**
```
Delete and Merge Subtrees — Delete 5:

  Before:              Merge left & right:     Result:
       5                  3                      3
      / \                / \                    / \
     3   7              2   4                  2   4
    / \ / \                  \                      \
   2  4 6  8                  7                      7
                             / \                    / \
                            6   8                  6   8

  Steps:
  ┌─────────────────────────────────────────────┐
  │ 1. Found node 5                              │
  │ 2. merge(left=3, right=7):                   │
  │    Walk to rightmost of left subtree          │
  │    3 → 4 (rightmost, cur.Right==nil)          │
  │    Attach right subtree (7) as 4.Right       │
  │ 3. Return left subtree root (3) as result    │
  └─────────────────────────────────────────────┘

  Inorder: 2 3 4 6 7 8 (sorted, BST valid ✓)
  Note: tree may become unbalanced with this approach
```

---

## Key Takeaways

1. **Three cases**: leaf, one child, two children — handle each separately
2. **Inorder successor** (or predecessor) replaces the deleted node with two children
3. **Trim BST** is a specialized delete: removes all nodes outside a range in O(n)
4. **Lazy deletion** marks nodes instead of restructuring — amortize cost
5. **Merge-based deletion** avoids finding successor by combining subtrees

> **Next up:** BST Search →
