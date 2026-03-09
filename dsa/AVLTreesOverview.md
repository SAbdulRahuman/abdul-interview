# Phase 9: BST — AVL Trees Overview

## Overview

AVL tree (Adelson-Velsky and Landis) is a **self-balancing BST** where the balance factor of every node is in {-1, 0, 1}.

| Property | Value |
|----------|-------|
| Balance factor | height(left) - height(right) ∈ {-1, 0, 1} |
| Max height | ~1.44 log₂(n) |
| Insert/Delete/Search | O(log n) guaranteed |
| Rotations per insert | At most 2 |
| Rotations per delete | O(log n) |

---

## Example 1: AVL Node with Height

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

func height(n *AVLNode) int {
	if n == nil { return 0 }
	return n.Height
}

func max(a, b int) int {
	if a > b { return a }
	return b
}

func updateHeight(n *AVLNode) {
	n.Height = 1 + max(height(n.Left), height(n.Right))
}

func balanceFactor(n *AVLNode) int {
	if n == nil { return 0 }
	return height(n.Left) - height(n.Right)
}

func main() {
	root := &AVLNode{Val: 10, Height: 3,
		Left: &AVLNode{Val: 5, Height: 2,
			Left:  &AVLNode{Val: 3, Height: 1},
			Right: &AVLNode{Val: 7, Height: 1}},
		Right: &AVLNode{Val: 15, Height: 1},
	}
	fmt.Printf("Root BF: %d\n", balanceFactor(root))       // 1
	fmt.Printf("Left BF: %d\n", balanceFactor(root.Left))  // 0
	fmt.Printf("Right BF: %d\n", balanceFactor(root.Right)) // 0
}
```

---

## Example 2: AVL Insertion with Rotations

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func max(a, b int) int { if a > b { return a }; return b }
func updateHeight(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)) }
func bf(n *AVLNode) int { if n == nil { return 0 }; return height(n.Left) - height(n.Right) }

func rightRotate(y *AVLNode) *AVLNode {
	x := y.Left; y.Left = x.Right; x.Right = y
	updateHeight(y); updateHeight(x); return x
}

func leftRotate(x *AVLNode) *AVLNode {
	y := x.Right; x.Right = y.Left; y.Left = x
	updateHeight(x); updateHeight(y); return y
}

func insert(node *AVLNode, val int) *AVLNode {
	if node == nil { return &AVLNode{Val: val, Height: 1} }
	if val < node.Val {
		node.Left = insert(node.Left, val)
	} else if val > node.Val {
		node.Right = insert(node.Right, val)
	} else {
		return node // no duplicates
	}
	updateHeight(node)
	b := bf(node)

	// Left Left
	if b > 1 && val < node.Left.Val { return rightRotate(node) }
	// Right Right
	if b < -1 && val > node.Right.Val { return leftRotate(node) }
	// Left Right
	if b > 1 && val > node.Left.Val { node.Left = leftRotate(node.Left); return rightRotate(node) }
	// Right Left
	if b < -1 && val < node.Right.Val { node.Right = rightRotate(node.Right); return leftRotate(node) }

	return node
}

func inorder(n *AVLNode) {
	if n == nil { return }
	inorder(n.Left); fmt.Printf("%d ", n.Val); inorder(n.Right)
}

func main() {
	var root *AVLNode
	values := []int{10, 20, 30, 40, 50, 25}
	for _, v := range values {
		root = insert(root, v)
		fmt.Printf("Insert %d → root=%d height=%d\n", v, root.Val, root.Height)
	}
	fmt.Print("Inorder: "); inorder(root); fmt.Println()
}
```

---

## Example 3: AVL Deletion

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func max(a, b int) int { if a > b { return a }; return b }
func updateHeight(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)) }
func bf(n *AVLNode) int { if n == nil { return 0 }; return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; updateHeight(y); updateHeight(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; updateHeight(x); updateHeight(y); return y }

func rebalance(node *AVLNode) *AVLNode {
	updateHeight(node)
	b := bf(node)
	if b > 1 {
		if bf(node.Left) < 0 { node.Left = leftRotate(node.Left) }
		return rightRotate(node)
	}
	if b < -1 {
		if bf(node.Right) > 0 { node.Right = rightRotate(node.Right) }
		return leftRotate(node)
	}
	return node
}

func insert(node *AVLNode, val int) *AVLNode {
	if node == nil { return &AVLNode{Val: val, Height: 1} }
	if val < node.Val { node.Left = insert(node.Left, val) } else if val > node.Val { node.Right = insert(node.Right, val) } else { return node }
	return rebalance(node)
}

func minNode(n *AVLNode) *AVLNode { for n.Left != nil { n = n.Left }; return n }

func delete(node *AVLNode, val int) *AVLNode {
	if node == nil { return nil }
	if val < node.Val {
		node.Left = delete(node.Left, val)
	} else if val > node.Val {
		node.Right = delete(node.Right, val)
	} else {
		if node.Left == nil { return node.Right }
		if node.Right == nil { return node.Left }
		succ := minNode(node.Right)
		node.Val = succ.Val
		node.Right = delete(node.Right, succ.Val)
	}
	return rebalance(node)
}

func inorder(n *AVLNode) {
	if n == nil { return }
	inorder(n.Left); fmt.Printf("%d ", n.Val); inorder(n.Right)
}

func main() {
	var root *AVLNode
	for _, v := range []int{9, 5, 10, 0, 6, 11, -1, 1, 2} {
		root = insert(root, v)
	}
	fmt.Print("Before: "); inorder(root); fmt.Println()
	root = delete(root, 10)
	fmt.Print("After delete 10: "); inorder(root); fmt.Println()
	fmt.Printf("Root: %d, Height: %d\n", root.Val, root.Height)
}
```

---

## Example 4: AVL Tree — Verify Balance at Every Node

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func max(a, b int) int { if a > b { return a }; return b }
func updateHeight(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)) }
func bf(n *AVLNode) int { if n == nil { return 0 }; return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; updateHeight(y); updateHeight(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; updateHeight(x); updateHeight(y); return y }
func rebalance(n *AVLNode) *AVLNode { updateHeight(n); b := bf(n); if b > 1 { if bf(n.Left) < 0 { n.Left = leftRotate(n.Left) }; return rightRotate(n) }; if b < -1 { if bf(n.Right) > 0 { n.Right = rightRotate(n.Right) }; return leftRotate(n) }; return n }
func insert(n *AVLNode, val int) *AVLNode { if n == nil { return &AVLNode{Val: val, Height: 1} }; if val < n.Val { n.Left = insert(n.Left, val) } else if val > n.Val { n.Right = insert(n.Right, val) }; return rebalance(n) }

func verifyAVL(node *AVLNode) bool {
	if node == nil { return true }
	b := bf(node)
	if b < -1 || b > 1 {
		fmt.Printf("VIOLATION at %d: bf=%d\n", node.Val, b)
		return false
	}
	return verifyAVL(node.Left) && verifyAVL(node.Right)
}

func main() {
	var root *AVLNode
	for i := 1; i <= 100; i++ {
		root = insert(root, i)
	}
	fmt.Printf("100 sequential inserts: height=%d (log2(100)≈7)\n", root.Height)
	fmt.Println("All nodes balanced:", verifyAVL(root))
}
```

---

## Example 5: AVL — Rank Queries (Order Statistics)

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
	Size   int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func size(n *AVLNode) int { if n == nil { return 0 }; return n.Size }
func max(a, b int) int { if a > b { return a }; return b }
func update(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)); n.Size = 1 + size(n.Left) + size(n.Right) }
func bf(n *AVLNode) int { return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; update(y); update(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; update(x); update(y); return y }
func rebalance(n *AVLNode) *AVLNode { update(n); b := bf(n); if b > 1 { if bf(n.Left) < 0 { n.Left = leftRotate(n.Left) }; return rightRotate(n) }; if b < -1 { if bf(n.Right) > 0 { n.Right = rightRotate(n.Right) }; return leftRotate(n) }; return n }
func insert(n *AVLNode, val int) *AVLNode { if n == nil { return &AVLNode{Val: val, Height: 1, Size: 1} }; if val < n.Val { n.Left = insert(n.Left, val) } else if val > n.Val { n.Right = insert(n.Right, val) }; return rebalance(n) }

func kthSmallest(n *AVLNode, k int) int {
	leftSize := size(n.Left)
	if k <= leftSize { return kthSmallest(n.Left, k) }
	if k == leftSize+1 { return n.Val }
	return kthSmallest(n.Right, k-leftSize-1)
}

func rank(n *AVLNode, val int) int {
	if n == nil { return 0 }
	if val < n.Val { return rank(n.Left, val) }
	if val > n.Val { return 1 + size(n.Left) + rank(n.Right, val) }
	return size(n.Left) + 1
}

func main() {
	var root *AVLNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}
	for k := 1; k <= 7; k++ {
		fmt.Printf("kth(%d) = %d\n", k, kthSmallest(root, k))
	}
	fmt.Println("Rank of 25:", rank(root, 25)) // 5
}
```

---

## Example 6: AVL — Range Count

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
	Size   int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func size(n *AVLNode) int { if n == nil { return 0 }; return n.Size }
func max(a, b int) int { if a > b { return a }; return b }
func update(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)); n.Size = 1 + size(n.Left) + size(n.Right) }
func bf(n *AVLNode) int { return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; update(y); update(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; update(x); update(y); return y }
func rebalance(n *AVLNode) *AVLNode { update(n); b := bf(n); if b > 1 { if bf(n.Left) < 0 { n.Left = leftRotate(n.Left) }; return rightRotate(n) }; if b < -1 { if bf(n.Right) > 0 { n.Right = rightRotate(n.Right) }; return leftRotate(n) }; return n }
func insert(n *AVLNode, val int) *AVLNode { if n == nil { return &AVLNode{Val: val, Height: 1, Size: 1} }; if val < n.Val { n.Left = insert(n.Left, val) } else if val > n.Val { n.Right = insert(n.Right, val) }; return rebalance(n) }

// countLess returns count of nodes with val < target
func countLess(n *AVLNode, target int) int {
	if n == nil { return 0 }
	if target <= n.Val { return countLess(n.Left, target) }
	return 1 + size(n.Left) + countLess(n.Right, target)
}

func rangeCount(root *AVLNode, lo, hi int) int {
	return countLess(root, hi+1) - countLess(root, lo)
}

func main() {
	var root *AVLNode
	for _, v := range []int{10, 5, 15, 3, 7, 12, 20, 1, 4, 6, 8} {
		root = insert(root, v)
	}
	fmt.Println("Count in [4,12]:", rangeCount(root, 4, 12)) // 7
	fmt.Println("Count in [1,5]:", rangeCount(root, 1, 5))   // 4
}
```

---

## Example 7: AVL — Count Inversions

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
	Size   int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func size(n *AVLNode) int { if n == nil { return 0 }; return n.Size }
func max(a, b int) int { if a > b { return a }; return b }
func update(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)); n.Size = 1 + size(n.Left) + size(n.Right) }
func bf(n *AVLNode) int { return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; update(y); update(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; update(x); update(y); return y }
func rebalance(n *AVLNode) *AVLNode { update(n); b := bf(n); if b > 1 { if bf(n.Left) < 0 { n.Left = leftRotate(n.Left) }; return rightRotate(n) }; if b < -1 { if bf(n.Right) > 0 { n.Right = rightRotate(n.Right) }; return leftRotate(n) }; return n }

func insertCount(n *AVLNode, val int, count *int) *AVLNode {
	if n == nil { return &AVLNode{Val: val, Height: 1, Size: 1} }
	if val < n.Val {
		*count += 1 + size(n.Right) // current node + all in right subtree are greater
		n.Left = insertCount(n.Left, val, count)
	} else {
		n.Right = insertCount(n.Right, val, count)
	}
	return rebalance(n)
}

func countInversions(arr []int) int {
	var root *AVLNode
	total := 0
	for _, v := range arr {
		root = insertCount(root, v, &total)
	}
	return total
}

func main() {
	arr := []int{8, 4, 2, 1}
	fmt.Println("Inversions:", countInversions(arr)) // 6

	arr2 := []int{1, 2, 3, 4}
	fmt.Println("Inversions (sorted):", countInversions(arr2)) // 0
}
```

---

## Example 8: AVL Tree Visualization

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func max(a, b int) int { if a > b { return a }; return b }
func updateHeight(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)) }
func bf(n *AVLNode) int { return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; updateHeight(y); updateHeight(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; updateHeight(x); updateHeight(y); return y }
func rebalance(n *AVLNode) *AVLNode { updateHeight(n); b := bf(n); if b > 1 { if bf(n.Left) < 0 { n.Left = leftRotate(n.Left) }; return rightRotate(n) }; if b < -1 { if bf(n.Right) > 0 { n.Right = rightRotate(n.Right) }; return leftRotate(n) }; return n }
func insert(n *AVLNode, val int) *AVLNode { if n == nil { return &AVLNode{Val: val, Height: 1} }; if val < n.Val { n.Left = insert(n.Left, val) } else if val > n.Val { n.Right = insert(n.Right, val) }; return rebalance(n) }

func printTree(node *AVLNode, prefix string, isLeft bool) {
	if node == nil { return }
	connector := "└── "
	if isLeft { connector = "├── " }
	fmt.Printf("%s%s%d (h=%d, bf=%d)\n", prefix, connector, node.Val, node.Height, bf(node))
	ext := "    "
	if isLeft { ext = "│   " }
	printTree(node.Left, prefix+ext, true)
	printTree(node.Right, prefix+ext, false)
}

func main() {
	var root *AVLNode
	for _, v := range []int{10, 20, 30, 40, 50, 25} {
		root = insert(root, v)
	}
	printTree(root, "", false)
}
```

---

## Example 9: AVL — Lower Bound and Upper Bound

```go
package main

import "fmt"

type AVLNode struct {
	Val    int
	Left   *AVLNode
	Right  *AVLNode
	Height int
}

func height(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func max(a, b int) int { if a > b { return a }; return b }
func updateHeight(n *AVLNode) { n.Height = 1 + max(height(n.Left), height(n.Right)) }
func bf(n *AVLNode) int { return height(n.Left) - height(n.Right) }
func rightRotate(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; updateHeight(y); updateHeight(x); return x }
func leftRotate(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; updateHeight(x); updateHeight(y); return y }
func rebalance(n *AVLNode) *AVLNode { updateHeight(n); b := bf(n); if b > 1 { if bf(n.Left) < 0 { n.Left = leftRotate(n.Left) }; return rightRotate(n) }; if b < -1 { if bf(n.Right) > 0 { n.Right = rightRotate(n.Right) }; return leftRotate(n) }; return n }
func insert(n *AVLNode, val int) *AVLNode { if n == nil { return &AVLNode{Val: val, Height: 1} }; if val < n.Val { n.Left = insert(n.Left, val) } else if val > n.Val { n.Right = insert(n.Right, val) }; return rebalance(n) }

// lowerBound: smallest val >= target
func lowerBound(n *AVLNode, target int) (int, bool) {
	result, found := 0, false
	for n != nil {
		if n.Val >= target {
			result = n.Val; found = true
			n = n.Left
		} else {
			n = n.Right
		}
	}
	return result, found
}

// upperBound: smallest val > target
func upperBound(n *AVLNode, target int) (int, bool) {
	result, found := 0, false
	for n != nil {
		if n.Val > target {
			result = n.Val; found = true
			n = n.Left
		} else {
			n = n.Right
		}
	}
	return result, found
}

func main() {
	var root *AVLNode
	for _, v := range []int{10, 20, 30, 40, 50} {
		root = insert(root, v)
	}
	v, ok := lowerBound(root, 25); fmt.Printf("LB(25): %d found=%v\n", v, ok) // 30
	v, ok = lowerBound(root, 30); fmt.Printf("LB(30): %d found=%v\n", v, ok) // 30
	v, ok = upperBound(root, 30); fmt.Printf("UB(30): %d found=%v\n", v, ok) // 40
	v, ok = upperBound(root, 50); fmt.Printf("UB(50): %d found=%v\n", v, ok) // not found
}
```

---

## Example 10: AVL vs Unbalanced BST — Performance Comparison

```go
package main

import (
	"fmt"
	"time"
)

type BST struct{ Val int; Left, Right *BST }
type AVL struct{ Val, Height int; Left, Right *AVL }

func bstInsert(n *BST, v int) *BST {
	if n == nil { return &BST{Val: v} }
	if v < n.Val { n.Left = bstInsert(n.Left, v) } else { n.Right = bstInsert(n.Right, v) }
	return n
}
func bstSearch(n *BST, v int) bool {
	for n != nil { if v == n.Val { return true }; if v < n.Val { n = n.Left } else { n = n.Right } }; return false
}

func avlH(n *AVL) int { if n == nil { return 0 }; return n.Height }
func avlMax(a, b int) int { if a > b { return a }; return b }
func avlUpd(n *AVL) { n.Height = 1 + avlMax(avlH(n.Left), avlH(n.Right)) }
func avlBf(n *AVL) int { return avlH(n.Left) - avlH(n.Right) }
func avlRR(y *AVL) *AVL { x := y.Left; y.Left = x.Right; x.Right = y; avlUpd(y); avlUpd(x); return x }
func avlLR(x *AVL) *AVL { y := x.Right; x.Right = y.Left; y.Left = x; avlUpd(x); avlUpd(y); return y }
func avlBal(n *AVL) *AVL { avlUpd(n); b := avlBf(n); if b > 1 { if avlBf(n.Left) < 0 { n.Left = avlLR(n.Left) }; return avlRR(n) }; if b < -1 { if avlBf(n.Right) > 0 { n.Right = avlRR(n.Right) }; return avlLR(n) }; return n }
func avlInsert(n *AVL, v int) *AVL { if n == nil { return &AVL{Val: v, Height: 1} }; if v < n.Val { n.Left = avlInsert(n.Left, v) } else if v > n.Val { n.Right = avlInsert(n.Right, v) }; return avlBal(n) }
func avlSearch(n *AVL, v int) bool {
	for n != nil { if v == n.Val { return true }; if v < n.Val { n = n.Left } else { n = n.Right } }; return false
}

func main() {
	n := 50000
	// Worst case: sorted insert
	var bstRoot *BST
	start := time.Now()
	for i := 0; i < n; i++ { bstRoot = bstInsert(bstRoot, i) }
	bstInsertTime := time.Since(start)

	var avlRoot *AVL
	start = time.Now()
	for i := 0; i < n; i++ { avlRoot = avlInsert(avlRoot, i) }
	avlInsertTime := time.Since(start)

	// Search for last element
	start = time.Now()
	bstSearch(bstRoot, n-1)
	bstSearchTime := time.Since(start)

	start = time.Now()
	avlSearch(avlRoot, n-1)
	avlSearchTime := time.Since(start)

	fmt.Printf("BST insert %d sorted: %v\n", n, bstInsertTime)
	fmt.Printf("AVL insert %d sorted: %v\n", n, avlInsertTime)
	fmt.Printf("BST search last: %v\n", bstSearchTime)
	fmt.Printf("AVL search last: %v\n", avlSearchTime)
	fmt.Printf("AVL height: %d (vs BST height: %d)\n", avlRoot.Height, n)
}
```

---

## Key Takeaways

1. AVL maintains **strict balance** (|bf| ≤ 1) → height ≤ 1.44 log₂(n)
2. At most **2 rotations** per insertion, up to **O(log n)** per deletion
3. Better for **read-heavy** workloads than Red-Black trees (stricter balance)
4. Augment with `Size` field for O(log n) order statistics and range queries
5. Always update height **bottom-up** after structural changes

> **Next up:** Red-Black Trees Overview →
