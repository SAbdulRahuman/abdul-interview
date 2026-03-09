# Phase 9: BST — Tree Rotations

## Overview

Rotations are the fundamental rebalancing operation for self-balancing BSTs. They rearrange nodes while preserving the BST property.

| Rotation | When Used |
|----------|-----------|
| **Left rotation** | Right subtree too heavy |
| **Right rotation** | Left subtree too heavy |
| **Left-Right** | Left child is right-heavy (double rotation) |
| **Right-Left** | Right child is left-heavy (double rotation) |

Rotations run in O(1) — only pointer changes.

---

## Example 1: Left Rotation

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

//     x              y
//      \            / \
//       y    →     x   c
//      / \          \
//     b   c          b
func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right
	x.Right = y.Left
	y.Left = x
	return y
}

func printTree(r *TreeNode, prefix string, isLeft bool) {
	if r == nil { return }
	marker := "└── "
	if isLeft { marker = "├── " }
	fmt.Printf("%s%s%d\n", prefix, marker, r.Val)
	newPrefix := prefix
	if isLeft { newPrefix += "│   " } else { newPrefix += "    " }
	printTree(r.Left, newPrefix, true)
	printTree(r.Right, newPrefix, false)
}

func main() {
	root := &TreeNode{10, nil,
		&TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{30, nil, nil}}}
	fmt.Println("Before left rotation:")
	printTree(root, "", false)

	root = leftRotate(root)
	fmt.Println("After left rotation:")
	printTree(root, "", false)
}
```

---

## Example 2: Right Rotation

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

//       y            x
//      /            / \
//     x      →     a   y
//    / \               /
//   a   b             b
func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left
	y.Left = x.Right
	x.Right = y
	return x
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d ", r.Val)
	inorder(r.Right)
}

func main() {
	root := &TreeNode{30,
		&TreeNode{20, &TreeNode{10, nil, nil}, &TreeNode{25, nil, nil}},
		nil,
	}
	fmt.Print("Before: "); inorder(root); fmt.Println() // 10 20 25 30
	root = rightRotate(root)
	fmt.Printf("New root: %d\n", root.Val) // 20
	fmt.Print("After:  "); inorder(root); fmt.Println() // 10 20 25 30 (same inorder!)
}
```

---

## Example 3: Left-Right (Double) Rotation

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}

func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

// Left-Right rotation: left-rotate left child, then right-rotate root
//      z              z             x
//     /              /             / \
//    y       →      x      →     y   z
//     \            /
//      x          y
func leftRightRotate(z *TreeNode) *TreeNode {
	z.Left = leftRotate(z.Left)
	return rightRotate(z)
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left); fmt.Printf("%d ", r.Val); inorder(r.Right)
}

func main() {
	root := &TreeNode{30,
		&TreeNode{10, nil, &TreeNode{20, nil, nil}},
		nil,
	}
	fmt.Print("Before LR: "); inorder(root); fmt.Printf(" root=%d\n", root.Val)
	root = leftRightRotate(root)
	fmt.Print("After LR:  "); inorder(root); fmt.Printf(" root=%d\n", root.Val)
	// Root changed from 30 to 20, order preserved
}
```

---

## Example 4: Right-Left (Double) Rotation

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}
func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

// Right-Left: right-rotate right child, then left-rotate root
//    x              x              y
//     \              \            / \
//      z      →       y    →    x   z
//     /                \
//    y                  z
func rightLeftRotate(x *TreeNode) *TreeNode {
	x.Right = rightRotate(x.Right)
	return leftRotate(x)
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left); fmt.Printf("%d ", r.Val); inorder(r.Right)
}

func main() {
	root := &TreeNode{10, nil,
		&TreeNode{30, &TreeNode{20, nil, nil}, nil}}
	fmt.Print("Before RL: "); inorder(root); fmt.Printf(" root=%d\n", root.Val)
	root = rightLeftRotate(root)
	fmt.Print("After RL:  "); inorder(root); fmt.Printf(" root=%d\n", root.Val)
}
```

---

## Example 5: Rotation Preserves Inorder (Proof by Example)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}
func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

func collectInorder(r *TreeNode) []int {
	if r == nil { return nil }
	result := collectInorder(r.Left)
	result = append(result, r.Val)
	result = append(result, collectInorder(r.Right)...)
	return result
}

func equal(a, b []int) bool {
	if len(a) != len(b) { return false }
	for i := range a {
		if a[i] != b[i] { return false }
	}
	return true
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	before := collectInorder(root)

	// Left rotate at root
	rotated := leftRotate(root)
	after := collectInorder(rotated)

	fmt.Println("Before:", before)
	fmt.Println("After: ", after)
	fmt.Println("Inorder preserved:", equal(before, after)) // true
}
```

---

## Example 6: Splay Rotation (Zig-Zig, Zig-Zag)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}
func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

// Splay: bring key to root
func splay(root *TreeNode, key int) *TreeNode {
	if root == nil || root.Val == key { return root }

	if key < root.Val {
		if root.Left == nil { return root }
		if key < root.Left.Val {
			// Zig-Zig (left-left)
			root.Left.Left = splay(root.Left.Left, key)
			root = rightRotate(root)
		} else if key > root.Left.Val {
			// Zig-Zag (left-right)
			root.Left.Right = splay(root.Left.Right, key)
			if root.Left.Right != nil {
				root.Left = leftRotate(root.Left)
			}
		}
		if root.Left == nil { return root }
		return rightRotate(root)
	} else {
		if root.Right == nil { return root }
		if key > root.Right.Val {
			// Zig-Zig (right-right)
			root.Right.Right = splay(root.Right.Right, key)
			root = leftRotate(root)
		} else if key < root.Right.Val {
			// Zig-Zag (right-left)
			root.Right.Left = splay(root.Right.Left, key)
			if root.Right.Left != nil {
				root.Right = rightRotate(root.Right)
			}
		}
		if root.Right == nil { return root }
		return leftRotate(root)
	}
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, &TreeNode{1, nil, nil}, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, nil, nil},
	}
	fmt.Println("Root before splay:", root.Val)       // 10
	root = splay(root, 1)
	fmt.Println("Root after splay(1):", root.Val)      // 1
}
```

---

## Example 7: Count Rotations to Balance a BST

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

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}
func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

func avlBalance(root *TreeNode) (*TreeNode, int) {
	if root == nil { return nil, 0 }
	leftRot, rightRot := 0, 0
	root.Left, leftRot = avlBalance(root.Left)
	root.Right, rightRot = avlBalance(root.Right)
	rots := leftRot + rightRot
	bf := balanceFactor(root)
	if bf > 1 {
		if balanceFactor(root.Left) < 0 {
			root.Left = leftRotate(root.Left)
			rots++
		}
		root = rightRotate(root)
		rots++
	} else if bf < -1 {
		if balanceFactor(root.Right) > 0 {
			root.Right = rightRotate(root.Right)
			rots++
		}
		root = leftRotate(root)
		rots++
	}
	return root, rots
}

func main() {
	// Build skewed tree
	root := &TreeNode{1, nil,
		&TreeNode{2, nil,
			&TreeNode{3, nil,
				&TreeNode{4, nil,
					&TreeNode{5, nil, nil}}}}}
	balanced, rotations := avlBalance(root)
	fmt.Printf("Rotations needed: %d, new root: %d, height: %d\n",
		rotations, balanced.Val, heightOf(balanced))
}
```

---

## Example 8: Rotation with Height Tracking (AVL-Style)

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

func updateHeight(n *AVLNode) {
	l, r := height(n.Left), height(n.Right)
	if l > r { n.Height = l + 1 } else { n.Height = r + 1 }
}

func bf(n *AVLNode) int {
	return height(n.Left) - height(n.Right)
}

func leftRotate(x *AVLNode) *AVLNode {
	y := x.Right; x.Right = y.Left; y.Left = x
	updateHeight(x); updateHeight(y)
	return y
}

func rightRotate(y *AVLNode) *AVLNode {
	x := y.Left; y.Left = x.Right; x.Right = y
	updateHeight(y); updateHeight(x)
	return x
}

func insert(root *AVLNode, val int) *AVLNode {
	if root == nil { return &AVLNode{Val: val, Height: 1} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	updateHeight(root)

	b := bf(root)
	if b > 1 && val < root.Left.Val { return rightRotate(root) }
	if b < -1 && val > root.Right.Val { return leftRotate(root) }
	if b > 1 && val > root.Left.Val { root.Left = leftRotate(root.Left); return rightRotate(root) }
	if b < -1 && val < root.Right.Val { root.Right = rightRotate(root.Right); return leftRotate(root) }
	return root
}

func main() {
	var root *AVLNode
	for _, v := range []int{1, 2, 3, 4, 5, 6, 7} {
		root = insert(root, v)
		fmt.Printf("Insert %d: root=%d height=%d\n", v, root.Val, root.Height)
	}
}
```

---

## Example 9: Visualize Rotation Steps

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func printLevel(nodes []*TreeNode) {
	next := []*TreeNode{}
	for _, n := range nodes {
		if n == nil {
			fmt.Print("_ ")
			next = append(next, nil, nil)
		} else {
			fmt.Printf("%d ", n.Val)
			next = append(next, n.Left, n.Right)
		}
	}
	fmt.Println()
	hasNodes := false
	for _, n := range next {
		if n != nil { hasNodes = true; break }
	}
	if hasNodes { printLevel(next) }
}

func leftRotate(x *TreeNode) *TreeNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}
func rightRotate(y *TreeNode) *TreeNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{8, &TreeNode{7, nil, nil}, &TreeNode{10, nil, nil}},
	}
	fmt.Println("Original:")
	printLevel([]*TreeNode{root})

	fmt.Println("\nAfter left rotation at root:")
	r1 := leftRotate(root)
	printLevel([]*TreeNode{r1})

	// Reset
	root = &TreeNode{5,
		&TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{8, &TreeNode{7, nil, nil}, &TreeNode{10, nil, nil}},
	}
	fmt.Println("\nAfter right rotation at root:")
	r2 := rightRotate(root)
	printLevel([]*TreeNode{r2})
}
```

---

## Example 10: Treap — Rotation-Based Priority Maintenance

```go
package main

import (
	"fmt"
	"math/rand"
)

type TreapNode struct {
	Val      int
	Priority int
	Left     *TreapNode
	Right    *TreapNode
}

func leftRotate(x *TreapNode) *TreapNode {
	y := x.Right; x.Right = y.Left; y.Left = x; return y
}
func rightRotate(y *TreapNode) *TreapNode {
	x := y.Left; y.Left = x.Right; x.Right = y; return x
}

func insert(root *TreapNode, val int) *TreapNode {
	if root == nil {
		return &TreapNode{Val: val, Priority: rand.Intn(1000)}
	}
	if val < root.Val {
		root.Left = insert(root.Left, val)
		if root.Left.Priority > root.Priority {
			root = rightRotate(root)
		}
	} else {
		root.Right = insert(root.Right, val)
		if root.Right.Priority > root.Priority {
			root = leftRotate(root)
		}
	}
	return root
}

func inorder(r *TreapNode) {
	if r == nil { return }
	inorder(r.Left)
	fmt.Printf("%d(p=%d) ", r.Val, r.Priority)
	inorder(r.Right)
}

func height(r *TreapNode) int {
	if r == nil { return 0 }
	l, ri := height(r.Left), height(r.Right)
	if l > ri { return l + 1 }
	return ri + 1
}

func main() {
	var root *TreapNode
	for _, v := range []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10} {
		root = insert(root, v)
	}
	fmt.Println("Treap root:", root.Val)
	fmt.Println("Treap height:", height(root))
	fmt.Print("Inorder: "); inorder(root); fmt.Println()
}
```

---

## Key Takeaways

1. **Single rotations**: left and right — O(1) pointer changes
2. **Double rotations**: left-right and right-left — needed when child is heavy in opposite direction
3. **Rotations always preserve inorder** (BST property maintained)
4. AVL uses rotations after every insert/delete; Red-Black uses rotations + recoloring
5. **Treap** uses rotations to maintain heap order on random priorities → expected O(log n) height

> **Next up:** AVL Trees Overview →
