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

**Textual Figure:**
```
Check if BST is Balanced:

  Balanced tree:                 Skewed tree:
       5                           1
      / \                           \
     3   7                           2
    / \ / \                           \
   1  4 6  8                           3

  height(5):                     height(1):
  ├─ L: height(3)                ├─ L: height(nil) = 0
  │  ├─ L: height(1)=1          └─ R: height(2)
  │  └─ R: height(4)=1             ├─ L: height(nil) = 0
  │  |1-1|=0 ≤ 1 ✓ return 2       └─ R: height(3) = 1
  ├─ R: height(7)                   |0-1|=1 ≤ 1 ✓ return 2
  │  ├─ L: height(6)=1          height(1):
  │  └─ R: height(8)=1            |0-2|=2 > 1 → return -1
  │  |1-1|=0 ≤ 1 ✓ return 2       ✗ NOT BALANCED!
  |2-2|=0 ≤ 1 ✓ return 3
  ✓ BALANCED!

  Output: Balanced: true         Output: Skewed: false
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

**Textual Figure:**
```
Sorted Array → Balanced BST:

  Input: [1, 2, 3, 4, 5, 6, 7]
                  ↑ mid=3 → root=4

  Recursive mid-picking:
  build([1,2,3,4,5,6,7]):
    mid=3 → root=4
    ├─ left = build([1,2,3]) → mid=1 → node=2
    │   ├─ left = build([1]) → node=1
    │   └─ right= build([3]) → node=3
    └─ right= build([5,6,7]) → mid=1 → node=6
        ├─ left = build([5]) → node=5
        └─ right= build([7]) → node=7

  Result:
         4
        / \
       2   6
      / \ / \
     1  3 5  7

  Inorder: 1 2 3 4 5 6 7 (sorted ✓)
  Height: 3 (perfectly balanced with 7 nodes)
  2^3 - 1 = 7 ✓ (complete binary tree)
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

**Textual Figure:**
```
Sorted Linked List → Balanced BST:

  Input: -10 → -3 → 0 → 5 → 9  (n=5)

  build(0, 4):  mid=2
  ├─ build(0,1): mid=0
  │   ├─ build(0,-1) = nil
  │   │  node(-10), cur advances to -3
  │   └─ build(1,1): mid=1
  │      ├─ build(1,0) = nil
  │      │  node(-3), cur advances to 0
  │      └─ build(2,1) = nil
  │  (subtree: -10 → left=nil, right=-3)
  ├─ node(0), cur advances to 5      ← mid node
  └─ build(3,4): mid=3
     ├─ build(3,2) = nil
     │  node(5), cur advances to 9
     └─ build(4,4): mid=4
        node(9)

  Result:
         0
        / \
      -10   5
        \    \
        -3    9

  Inorder: -10 -3 0 5 9 ✓
  Key: uses inorder simulation — no random access needed!
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

**Textual Figure:**
```
Rebalance an Unbalanced BST:

  Before (skewed):          After rebalancing:
  1                              3
   \                            / \
    2                          1   4
     \                          \   \
      3                          2   5
       \
        4
         \
          5
  height=5                  height=3

  Steps:
  ┌─────────────────────────────────────────┐
  │ 1. Inorder traversal → [1, 2, 3, 4, 5]    │
  │ 2. Build balanced from sorted array:      │
  │    mid=2 → root=3                         │
  │    left=[1,2] → mid=0 → 1, right=2       │
  │    right=[4,5] → mid=0 → 4, right=5      │
  │ 3. Height: 5 → 3                          │
  └─────────────────────────────────────────┘
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

**Textual Figure:**
```
Day–Stout–Warren Algorithm (in-place rebalance):

  Step 1: Tree → Right Vine (backbone):
  1                       1
   \                       \
    2         Right         2
     \       rotations       \
      3      ──────→           3
       \                       \
        4                       4
         \                       \
          5                       5
           \                       \
            6                       6
             \                       \
              7                       7
  (already a vine!)        n=7 nodes

  Step 2: Vine → Balanced Tree (left rotations + compress):
  leaves = 7+1 - 2^(⌊log₂(8)⌋) = 8-8 = 0
  compress(0), then compress(3), compress(1):

          4
        /   \
       2     6
      / \   / \
     1   3 5   7

  height: 7 → 3   O(n) time, O(1) space!
  Inorder: 1 2 3 4 5 6 7 (preserved ✓)
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

**Textual Figure:**
```
Merge Two Balanced BSTs:

  Tree 1:          Tree 2:
     3                4
    / \              / \
   1   5            2   8
    \   \          /   / \
     3?  9        2?  6  10
  Actually:       Actually:
     5               6
    / \             / \
   1   7           2   8
    \   \         /   / \
     3   9       2?  4  10

  (Built from [1,3,5,7,9] and [2,4,6,8,10])

  Steps:
  ┌─────────────────────────────────────────────┐
  │ 1. Inorder(T1) → [1, 3, 5, 7, 9]             │
  │ 2. Inorder(T2) → [2, 4, 6, 8, 10]            │
  │ 3. Merge sorted: [1,2,3,4,5,6,7,8,9,10]      │
  │ 4. Build balanced BST:                        │
  │              5                                │
  │            /   \                              │
  │           2     8                             │
  │          / \   / \                            │
  │         1   3  6   9                          │
  │              \  \   \                         │
  │               4  7  10                        │
  └─────────────────────────────────────────────┘
  Inorder: 1 2 3 4 5 6 7 8 9 10 ✓
  Time: O(n+m),  Space: O(n+m)
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

**Textual Figure:**
```
Fix Imbalance with One Rotation:

  Before (left-heavy, BF=2):     After right rotation:
       30                            20
      /                             / \
    20                             10  30
   /
  10

  Right Rotate(30):
  ┌─────────────────────────────────────────┐
  │   y=30, x=y.Left=20                │
  │   y.Left = x.Right (nil)           │
  │   x.Right = y (30)                 │
  │   return x (20) as new root        │
  └─────────────────────────────────────────┘

  Four rotation cases:
  LL (left-left):   right rotate
  RR (right-right): left rotate
  LR (left-right):  left rotate child, then right rotate
  RL (right-left):  right rotate child, then left rotate

  BF before: 2,  BF after: 0,  root: 20
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

**Textual Figure:**
```
Weight-Balanced BST (balance by subtree sizes):

  Balanced tree (size annotations):    Skewed tree:
           5 (7)                        1 (5)
          / \                            \
     (3) 3   7 (3)                      2 (4)
        / \ / \                            \
   (1) 1 4 6  8 (1)                       3 (3)
       (1)(1)                                \
                                            4 (2)
  alpha = 0.7                                  \
                                              5 (1)
  Check: size(child) ≤ alpha * size(parent)?

  ┌── Balanced tree: ────────────────────────┐
  │ Node 5: left=3, right=3                │
  │   3 ≤ 0.7*7=4.9 ✓  3 ≤ 4.9 ✓           │
  │ Node 3: left=1, right=1, both ≤0.7*3 ✓ │
  │ Weight-balanced: true                  │
  ├── Skewed tree: ─────────────────────────┤
  │ Node 1: left=0, right=4                │
  │   4 ≤ 0.7*5=3.5? NO! 4 > 3.5 ✗         │
  │ Weight-balanced: false                 │
  └────────────────────────────────────────┘
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

**Textual Figure:**
```
BST → Balanced Doubly Linked List:

  Input (skewed BST):    Step 1: Collect inorder    Step 2: Build balanced
  1                      [1, 2, 3, 4, 5]                3
   \                                                   / \
    2                                                  1   4
     \                                                  \   \
      3                                                  2   5
       \
        4                Step 3: Flatten to DLL
         \
          5

  Doubly Linked List (using Left=prev, Right=next):

  nil ← 1 ↔ 2 ↔ 3 ↔ 4 ↔ 5 → nil
        ↑                   ↑
       head               tail

  ┌─────────────────────────────────────┐
  │ Node 1: Left=nil,  Right=2          │
  │ Node 2: Left=1,    Right=3          │
  │ Node 3: Left=2,    Right=4          │
  │ Node 4: Left=3,    Right=5          │
  │ Node 5: Left=4,    Right=nil        │
  └─────────────────────────────────────┘

  Output: 1 2 3 4 5
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

**Textual Figure:**
```
Balanced BST from Preorder + Inorder:

  Preorder: [4, 2, 1, 3, 6, 5, 7]
  Inorder:  [1, 2, 3, 4, 5, 6, 7]

  Build process:
  ┌──────────────────────────────────────────────┐
  │ pre[0]=4 → root                                │
  │ Find 4 in inorder at index 3                   │
  │   left inorder:  [1,2,3]   right: [5,6,7]      │
  │   left preorder: [2,1,3]   right: [6,5,7]      │
  │                                                 │
  │ Left:  pre[0]=2 → node                          │
  │   inorder idx=1 → L:[1] R:[3]                   │
  │   node(1), node(3)                              │
  │                                                 │
  │ Right: pre[0]=6 → node                          │
  │   inorder idx=1 → L:[5] R:[7]                   │
  │   node(5), node(7)                              │
  └──────────────────────────────────────────────┘

  Result:
         4
        / \
       2   6
      / \ / \
     1  3 5  7

  Height: 3   Balanced: true ✓
  (Perfect binary tree with 2^3 - 1 = 7 nodes)
```

---

## Key Takeaways

1. **Balanced BST** guarantees O(log n) height → O(log n) search/insert/delete
2. **Sorted array → BST** always produces a balanced tree (pick middle as root)
3. **Rebalancing** an arbitrary BST: collect inorder → rebuild with mid-pick (O(n))
4. **Day–Stout–Warren** rebalances in-place in O(n) time, O(1) space
5. **Weight-balanced** trees use subtree sizes instead of heights for balance criteria

> **Next up:** Tree Rotations →
