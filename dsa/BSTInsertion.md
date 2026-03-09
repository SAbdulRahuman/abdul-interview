# Phase 9: BST — BST Insertion

## Overview

Inserting into a BST maintains the BST property: compare the new value with the current node and recurse left (smaller) or right (larger). New nodes are always inserted as **leaves**.

| Operation | Average | Worst (skewed) |
|-----------|---------|----------------|
| Insert | O(log n) | O(n) |

---

## Example 1: Recursive Insertion

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insertIntoBST(root *TreeNode, val int) *TreeNode {
	if root == nil {
		return &TreeNode{Val: val}
	}
	if val < root.Val {
		root.Left = insertIntoBST(root.Left, val)
	} else {
		root.Right = insertIntoBST(root.Right, val)
	}
	return root
}

func inorder(root *TreeNode) {
	if root == nil { return }
	inorder(root.Left)
	fmt.Printf("%d ", root.Val)
	inorder(root.Right)
}

func main() {
	var root *TreeNode
	for _, v := range []int{5, 3, 7, 1, 4, 6, 8} {
		root = insertIntoBST(root, v)
	}
	inorder(root) // 1 3 4 5 6 7 8
	fmt.Println()
}
```

---

## Example 2: Iterative Insertion

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insertIterative(root *TreeNode, val int) *TreeNode {
	newNode := &TreeNode{Val: val}
	if root == nil { return newNode }

	cur := root
	for {
		if val < cur.Val {
			if cur.Left == nil {
				cur.Left = newNode
				return root
			}
			cur = cur.Left
		} else {
			if cur.Right == nil {
				cur.Right = newNode
				return root
			}
			cur = cur.Right
		}
	}
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	var root *TreeNode
	for _, v := range []int{10, 5, 15, 3, 7} {
		root = insertIterative(root, v)
	}
	inorder(root) // 3 5 7 10 15
	fmt.Println()
}
```

---

## Example 3: Insert with Parent Pointer

```go
package main

import "fmt"

type TreeNode struct {
	Val    int
	Left   *TreeNode
	Right  *TreeNode
	Parent *TreeNode
}

func insertWithParent(root *TreeNode, val int) *TreeNode {
	newNode := &TreeNode{Val: val}
	if root == nil { return newNode }

	var parent *TreeNode
	cur := root
	for cur != nil {
		parent = cur
		if val < cur.Val {
			cur = cur.Left
		} else {
			cur = cur.Right
		}
	}
	newNode.Parent = parent
	if val < parent.Val {
		parent.Left = newNode
	} else {
		parent.Right = newNode
	}
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{8, 3, 10, 1, 6} {
		root = insertWithParent(root, v)
	}
	// Verify parent pointers
	fmt.Println(root.Left.Val, "parent:", root.Left.Parent.Val) // 3 parent: 8
	fmt.Println(root.Left.Left.Val, "parent:", root.Left.Left.Parent.Val) // 1 parent: 3
}
```

---

## Example 4: Insert Multiple Values — Build BST from Array

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func buildBST(vals []int) *TreeNode {
	var root *TreeNode
	for _, v := range vals {
		root = insert(root, v)
	}
	return root
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val {
		root.Left = insert(root.Left, val)
	} else if val > root.Val {
		root.Right = insert(root.Right, val)
	}
	// val == root.Val: skip duplicates
	return root
}

func preorder(r *TreeNode) {
	if r == nil { return }
	fmt.Printf("%d ", r.Val)
	preorder(r.Left)
	preorder(r.Right)
}

func height(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := height(r.Left), height(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func main() {
	// Random order → balanced-ish
	r1 := buildBST([]int{5, 3, 7, 1, 4, 6, 8})
	fmt.Print("Balanced: "); preorder(r1); fmt.Printf(" height=%d\n", height(r1))

	// Sorted order → skewed (worst case)
	r2 := buildBST([]int{1, 2, 3, 4, 5, 6, 7})
	fmt.Print("Skewed:   "); preorder(r2); fmt.Printf(" height=%d\n", height(r2))
}
```

---

## Example 5: Insert and Return Path

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func insertWithPath(root *TreeNode, val int) (*TreeNode, []int) {
	path := []int{}
	if root == nil {
		return &TreeNode{Val: val}, path
	}

	cur := root
	for cur != nil {
		path = append(path, cur.Val)
		if val < cur.Val {
			if cur.Left == nil {
				cur.Left = &TreeNode{Val: val}
				return root, path
			}
			cur = cur.Left
		} else {
			if cur.Right == nil {
				cur.Right = &TreeNode{Val: val}
				return root, path
			}
			cur = cur.Right
		}
	}
	return root, path
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, nil, &TreeNode{20, nil, nil}},
	}
	root, path := insertWithPath(root, 12)
	fmt.Println("Path to insert 12:", path) // [10 15]

	root, path = insertWithPath(root, 4)
	fmt.Println("Path to insert 4:", path) // [10 5 3]
}
```

---

## Example 6: Insert Maintaining Count (Order Statistics)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
	Count int // size of subtree
}

func insertWithCount(root *TreeNode, val int) *TreeNode {
	if root == nil {
		return &TreeNode{Val: val, Count: 1}
	}
	root.Count++
	if val < root.Val {
		root.Left = insertWithCount(root.Left, val)
	} else {
		root.Right = insertWithCount(root.Right, val)
	}
	return root
}

func getCount(node *TreeNode) int {
	if node == nil { return 0 }
	return node.Count
}

// Find kth smallest using counts
func kthSmallest(root *TreeNode, k int) int {
	leftCount := getCount(root.Left)
	if k <= leftCount {
		return kthSmallest(root.Left, k)
	} else if k == leftCount+1 {
		return root.Val
	}
	return kthSmallest(root.Right, k-leftCount-1)
}

func main() {
	var root *TreeNode
	for _, v := range []int{5, 3, 7, 1, 4, 6, 8} {
		root = insertWithCount(root, v)
	}
	fmt.Println("Root count:", root.Count) // 7
	for k := 1; k <= 7; k++ {
		fmt.Printf("k=%d → %d\n", k, kthSmallest(root, k))
	}
}
```

---

## Example 7: BST from Preorder Traversal (LeetCode 1008)

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

func bstFromPreorder(preorder []int) *TreeNode {
	idx := 0
	var build func(lo, hi int) *TreeNode
	build = func(lo, hi int) *TreeNode {
		if idx >= len(preorder) || preorder[idx] < lo || preorder[idx] > hi {
			return nil
		}
		val := preorder[idx]
		idx++
		return &TreeNode{val, build(lo, val), build(val, hi)}
	}
	return build(math.MinInt64, math.MaxInt64)
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := bstFromPreorder([]int{8, 5, 1, 7, 10, 12})
	inorder(root) // 1 5 7 8 10 12
	fmt.Println()
}
```

---

## Example 8: Insert into BST with Duplicates (Multi-set)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
	Freq  int
}

func insertMultiset(root *TreeNode, val int) *TreeNode {
	if root == nil {
		return &TreeNode{Val: val, Freq: 1}
	}
	if val < root.Val {
		root.Left = insertMultiset(root.Left, val)
	} else if val > root.Val {
		root.Right = insertMultiset(root.Right, val)
	} else {
		root.Freq++ // duplicate
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
	for _, v := range []int{5, 3, 7, 3, 5, 5, 1} {
		root = insertMultiset(root, v)
	}
	inorder(root) // 1(x1) 3(x2) 5(x3) 7(x1)
	fmt.Println()
}
```

---

## Example 9: Batch Insert with Balanced Result

```go
package main

import (
	"fmt"
	"sort"
)

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func balancedInsert(values []int) *TreeNode {
	sort.Ints(values)
	// Remove duplicates
	unique := []int{values[0]}
	for i := 1; i < len(values); i++ {
		if values[i] != values[i-1] {
			unique = append(unique, values[i])
		}
	}
	return buildBalanced(unique, 0, len(unique)-1)
}

func buildBalanced(nums []int, lo, hi int) *TreeNode {
	if lo > hi { return nil }
	mid := (lo + hi) / 2
	return &TreeNode{
		Val:   nums[mid],
		Left:  buildBalanced(nums, lo, mid-1),
		Right: buildBalanced(nums, mid+1, hi),
	}
}

func height(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := height(r.Left), height(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func main() {
	root := balancedInsert([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
	fmt.Println("Height:", height(root)) // 4 (balanced for 10 nodes)
}
```

---

## Example 10: Insert and Verify BST Invariant

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

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val {
		root.Left = insert(root.Left, val)
	} else if val > root.Val {
		root.Right = insert(root.Right, val)
	}
	return root
}

func isValidBST(node *TreeNode, min, max int) bool {
	if node == nil { return true }
	if node.Val <= min || node.Val >= max { return false }
	return isValidBST(node.Left, min, node.Val) && isValidBST(node.Right, node.Val, max)
}

func main() {
	var root *TreeNode
	vals := []int{50, 30, 70, 20, 40, 60, 80, 10, 25, 35, 45}
	for _, v := range vals {
		root = insert(root, v)
		valid := isValidBST(root, math.MinInt64, math.MaxInt64)
		fmt.Printf("After insert %d: valid=%v\n", v, valid)
	}
}
```

---

## Key Takeaways

1. New nodes always become **leaves** in standard BST insertion
2. **Recursive** insertion is cleaner; **iterative** avoids stack overflow
3. Insertion order determines tree shape — sorted input creates O(n) skewed trees
4. Use **balanced build** (sort + mid-pick) for guaranteed O(log n) height
5. Track subtree **count** during insertion for O(log n) order statistics

> **Next up:** BST Deletion →
