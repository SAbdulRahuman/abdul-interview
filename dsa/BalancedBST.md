# Phase 9: BST — Balanced BST

## Overview

A BST is **balanced** when the height difference between left and right subtrees of every node is at most 1 (or bounded by a constant). This guarantees O(log n) operations.

| Tree Type | Balance Guarantee | Height |
|-----------|-------------------|--------|
| Random BST | Expected O(log n) | ~1.39 log₂ n |
| AVL | Strict (|h_L - h_R| ≤ 1) | ~1.44 log₂ n |
| Red-Black | Relaxed | ≤ 2 log₂(n+1) |
| Sorted insert | None (degenerate) | O(n) |

---

## Example 1: Check if BST is Balanced (LeetCode 110)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func isBalanced(root *TreeNode) bool {
	return height(root) != -1
}

func height(node *TreeNode) int {
	if node == nil { return 0 }
	l := height(node.Left)
	if l == -1 { return -1 }
	r := height(node.Right)
	if r == -1 { return -1 }
	if l-r > 1 || r-l > 1 { return -1 }
	if l > r { return l + 1 }
	return r + 1
}

func main() {
	balanced := &TreeNode{5,
		&TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{8, nil, nil}},
	}
	fmt.Println("Balanced:", isBalanced(balanced)) // true

	skewed := &TreeNode{1, nil, &TreeNode{2, nil, &TreeNode{3, nil, nil}}}
	fmt.Println("Skewed:", isBalanced(skewed)) // false
}
```

---

## Example 2: Sorted Array to Balanced BST (LeetCode 108)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func sortedArrayToBST(nums []int) *TreeNode {
	if len(nums) == 0 { return nil }
	mid := len(nums) / 2
	return &TreeNode{
		Val:   nums[mid],
		Left:  sortedArrayToBST(nums[:mid]),
		Right: sortedArrayToBST(nums[mid+1:]),
	}
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func heightOf(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := heightOf(r.Left), heightOf(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func main() {
	nums := []int{1, 2, 3, 4, 5, 6, 7}
	root := sortedArrayToBST(nums)
	inorder(root); fmt.Println()
	fmt.Println("Height:", heightOf(root)) // 3
}
```

---

## Example 3: Sorted Linked List to Balanced BST (LeetCode 109)

```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func sortedListToBST(head *ListNode) *TreeNode {
	// Count list length
	n := 0
	for p := head; p != nil; p = p.Next { n++ }

	cur := head
	var build func(lo, hi int) *TreeNode
	build = func(lo, hi int) *TreeNode {
		if lo > hi { return nil }
		mid := (lo + hi) / 2
		left := build(lo, mid-1)
		node := &TreeNode{Val: cur.Val, Left: left}
		cur = cur.Next
		node.Right = build(mid+1, hi)
		return node
	}
	return build(0, n-1)
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	head := &ListNode{-10, &ListNode{-3, &ListNode{0, &ListNode{5, &ListNode{9, nil}}}}}
	root := sortedListToBST(head)
	inorder(root) // -10 -3 0 5 9
	fmt.Println()
}
```

---

## Example 4: Rebalance an Unbalanced BST (LeetCode 1382)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func balanceBST(root *TreeNode) *TreeNode {
	sorted := []int{}
	var inorder func(node *TreeNode)
	inorder = func(node *TreeNode) {
		if node == nil { return }
		inorder(node.Left)
		sorted = append(sorted, node.Val)
		inorder(node.Right)
	}
	inorder(root)
	return build(sorted, 0, len(sorted)-1)
}

func build(nums []int, lo, hi int) *TreeNode {
	if lo > hi { return nil }
	mid := (lo + hi) / 2
	return &TreeNode{nums[mid], build(nums, lo, mid-1), build(nums, mid+1, hi)}
}

func heightOf(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := heightOf(r.Left), heightOf(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func main() {
	// Skewed BST: 1 → 2 → 3 → 4 → 5
	skewed := &TreeNode{1, nil,
		&TreeNode{2, nil,
			&TreeNode{3, nil,
				&TreeNode{4, nil, &TreeNode{5, nil, nil}}}}}
	fmt.Println("Before height:", heightOf(skewed)) // 5
	balanced := balanceBST(skewed)
	fmt.Println("After height:", heightOf(balanced)) // 3
}
```

---

## Example 5: Day–Stout–Warren Algorithm (In-Place Rebalance)

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

func dsw(root *TreeNode) *TreeNode {
	// Step 1: Tree to right-vine (backbone)
	dummy := &TreeNode{Right: root}
	treeToVine(dummy)
	n := countNodes(dummy.Right)
	// Step 2: Vine to balanced tree
	vineToTree(dummy, n)
	return dummy.Right
}

func treeToVine(root *TreeNode) {
	tail := root
	rest := tail.Right
	for rest != nil {
		if rest.Left == nil {
			tail = rest
			rest = rest.Right
		} else {
			// Right rotate
			temp := rest.Left
			rest.Left = temp.Right
			temp.Right = rest
			rest = temp
			tail.Right = temp
		}
	}
}

func countNodes(root *TreeNode) int {
	n := 0
	for root != nil { n++; root = root.Right }
	return n
}

func vineToTree(root *TreeNode, n int) {
	leaves := n + 1 - int(math.Pow(2, math.Floor(math.Log2(float64(n+1)))))
	compress(root, leaves)
	n -= leaves
	for n > 1 {
		compress(root, n/2)
		n /= 2
	}
}

func compress(root *TreeNode, count int) {
	scanner := root
	for i := 0; i < count; i++ {
		child := scanner.Right
		scanner.Right = child.Right
		scanner = scanner.Right
		child.Right = scanner.Left
		scanner.Left = child
	}
}

func heightOf(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := heightOf(r.Left), heightOf(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	skewed := &TreeNode{1, nil,
		&TreeNode{2, nil,
			&TreeNode{3, nil,
				&TreeNode{4, nil,
					&TreeNode{5, nil,
						&TreeNode{6, nil,
							&TreeNode{7, nil, nil}}}}}}}
	fmt.Println("Before height:", heightOf(skewed)) // 7
	balanced := dsw(skewed)
	fmt.Println("After height:", heightOf(balanced)) // 3
	inorder(balanced); fmt.Println() // 1 2 3 4 5 6 7
}
```

---

## Example 6: Merge Two Balanced BSTs

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func mergeBSTs(r1, r2 *TreeNode) *TreeNode {
	a := toSorted(r1)
	b := toSorted(r2)
	merged := mergeSorted(a, b)
	return buildBalanced(merged, 0, len(merged)-1)
}

func toSorted(root *TreeNode) []int {
	result := []int{}
	var inorder func(*TreeNode)
	inorder = func(n *TreeNode) {
		if n == nil { return }
		inorder(n.Left); result = append(result, n.Val); inorder(n.Right)
	}
	inorder(root)
	return result
}

func mergeSorted(a, b []int) []int {
	result := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		if a[i] <= b[j] { result = append(result, a[i]); i++ } else { result = append(result, b[j]); j++ }
	}
	result = append(result, a[i:]...)
	result = append(result, b[j:]...)
	return result
}

func buildBalanced(nums []int, lo, hi int) *TreeNode {
	if lo > hi { return nil }
	mid := (lo + hi) / 2
	return &TreeNode{nums[mid], buildBalanced(nums, lo, mid-1), buildBalanced(nums, mid+1, hi)}
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left); fmt.Printf("%d ", r.Val); inorder(r.Right)
}

func main() {
	t1 := buildBalanced([]int{1, 3, 5, 7, 9}, 0, 4)
	t2 := buildBalanced([]int{2, 4, 6, 8, 10}, 0, 4)
	merged := mergeBSTs(t1, t2)
	inorder(merged); fmt.Println() // 1 2 3 4 5 6 7 8 9 10
}
```

---

## Example 7: Check if BST Could Be Balanced After One Rotation

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func heightOf(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := heightOf(r.Left), heightOf(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func balanceFactor(r *TreeNode) int {
	if r == nil { return 0 }
	return heightOf(r.Left) - heightOf(r.Right)
}

func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left
	y.Left = x.Right
	x.Right = y
	return x
}

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right
	x.Right = y.Left
	y.Left = x
	return y
}

func fixOneRotation(root *TreeNode) (*TreeNode, bool) {
	bf := balanceFactor(root)
	if bf > 1 {
		if balanceFactor(root.Left) >= 0 {
			return rightRotate(root), true
		}
		root.Left = leftRotate(root.Left)
		return rightRotate(root), true
	}
	if bf < -1 {
		if balanceFactor(root.Right) <= 0 {
			return leftRotate(root), true
		}
		root.Right = rightRotate(root.Right)
		return leftRotate(root), true
	}
	return root, false
}

func main() {
	unbalanced := &TreeNode{30,
		&TreeNode{20, &TreeNode{10, nil, nil}, nil},
		nil,
	}
	fmt.Println("BF before:", balanceFactor(unbalanced)) // 2
	fixed, rotated := fixOneRotation(unbalanced)
	fmt.Printf("Rotated: %v, BF after: %d, root: %d\n", rotated, balanceFactor(fixed), fixed.Val)
}
```

---

## Example 8: Weight-Balanced BST

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
	Size  int
}

func size(n *TreeNode) int {
	if n == nil { return 0 }
	return n.Size
}

func updateSize(n *TreeNode) {
	if n != nil { n.Size = 1 + size(n.Left) + size(n.Right) }
}

func isWeightBalanced(n *TreeNode, alpha float64) bool {
	if n == nil { return true }
	s := float64(n.Size)
	if float64(size(n.Left)) > alpha*s || float64(size(n.Right)) > alpha*s {
		return false
	}
	return isWeightBalanced(n.Left, alpha) && isWeightBalanced(n.Right, alpha)
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val, Size: 1} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	updateSize(root)
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{5, 3, 7, 1, 4, 6, 8} {
		root = insert(root, v)
	}
	alpha := 0.7
	fmt.Printf("Weight-balanced (alpha=%.1f): %v\n", alpha, isWeightBalanced(root, alpha)) // true

	// Skewed
	var skewed *TreeNode
	for _, v := range []int{1, 2, 3, 4, 5} {
		skewed = insert(skewed, v)
	}
	fmt.Printf("Skewed weight-balanced (alpha=%.1f): %v\n", alpha, isWeightBalanced(skewed, alpha)) // false
}
```

---

## Example 9: Convert BST to Balanced Doubly Linked List

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode // prev
	Right *TreeNode // next
}

func bstToBalancedDLL(root *TreeNode) *TreeNode {
	// Collect inorder
	vals := []int{}
	var inorder func(*TreeNode)
	inorder = func(n *TreeNode) {
		if n == nil { return }
		inorder(n.Left); vals = append(vals, n.Val); inorder(n.Right)
	}
	inorder(root)

	if len(vals) == 0 { return nil }

	// Build balanced tree, then flatten
	balanced := buildBalanced(vals, 0, len(vals)-1)

	// Flatten to DLL via inorder
	var head, prev *TreeNode
	var makeDLL func(*TreeNode)
	makeDLL = func(n *TreeNode) {
		if n == nil { return }
		makeDLL(n.Left)
		if prev == nil { head = n } else { prev.Right = n; n.Left = prev }
		prev = n
		makeDLL(n.Right)
	}
	makeDLL(balanced)
	return head
}

func buildBalanced(nums []int, lo, hi int) *TreeNode {
	if lo > hi { return nil }
	mid := (lo + hi) / 2
	return &TreeNode{nums[mid], buildBalanced(nums, lo, mid-1), buildBalanced(nums, mid+1, hi)}
}

func main() {
	root := &TreeNode{1, nil,
		&TreeNode{2, nil,
			&TreeNode{3, nil,
				&TreeNode{4, nil, &TreeNode{5, nil, nil}}}}}

	head := bstToBalancedDLL(root)
	cur := head
	for cur != nil {
		fmt.Printf("%d ", cur.Val)
		cur = cur.Right
	}
	fmt.Println() // 1 2 3 4 5
}
```

---

## Example 10: Balanced BST from Preorder + Inorder

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func buildTree(preorder, inorder []int) *TreeNode {
	if len(preorder) == 0 { return nil }
	rootVal := preorder[0]
	rootIdx := 0
	for i, v := range inorder {
		if v == rootVal { rootIdx = i; break }
	}
	return &TreeNode{
		Val:   rootVal,
		Left:  buildTree(preorder[1:1+rootIdx], inorder[:rootIdx]),
		Right: buildTree(preorder[1+rootIdx:], inorder[rootIdx+1:]),
	}
}

func heightOf(r *TreeNode) int {
	if r == nil { return 0 }
	l, ri := heightOf(r.Left), heightOf(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func isBalanced(root *TreeNode) bool {
	return checkHeight(root) != -1
}

func checkHeight(n *TreeNode) int {
	if n == nil { return 0 }
	l := checkHeight(n.Left); if l == -1 { return -1 }
	r := checkHeight(n.Right); if r == -1 { return -1 }
	if l-r > 1 || r-l > 1 { return -1 }
	if l > r { return l + 1 }
	return r + 1
}

func main() {
	pre := []int{4, 2, 1, 3, 6, 5, 7}
	in := []int{1, 2, 3, 4, 5, 6, 7}
	root := buildTree(pre, in)
	fmt.Println("Height:", heightOf(root))     // 3
	fmt.Println("Balanced:", isBalanced(root))  // true
}
```

---

## Key Takeaways

1. **Balanced BST** guarantees O(log n) height → O(log n) search/insert/delete
2. **Sorted array → BST** always produces a balanced tree (pick middle as root)
3. **Rebalancing** an arbitrary BST: collect inorder → rebuild with mid-pick (O(n))
4. **Day–Stout–Warren** rebalances in-place in O(n) time, O(1) space
5. **Weight-balanced** trees use subtree sizes instead of heights for balance criteria

> **Next up:** Tree Rotations →
