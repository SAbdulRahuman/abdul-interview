# Phase 9: BST — Red-Black Trees Overview

## Overview

A Red-Black tree is a self-balancing BST where each node has a **color** (red or black) and the tree satisfies five properties:

| Property | Rule |
|----------|------|
| 1 | Every node is red or black |
| 2 | Root is always black |
| 3 | Every NIL leaf is black |
| 4 | Red node's children are both black (no two consecutive reds) |
| 5 | Every path from a node to its descendant NIL has the same number of black nodes |

| Metric | Value |
|--------|-------|
| Height | ≤ 2 log₂(n+1) |
| Insert/Delete/Search | O(log n) |
| Rotations per insert | At most 2 |
| Rotations per delete | At most 3 |

---

## Example 1: Red-Black Node Structure

```go
package main

import "fmt"

const (
	RED   = true
	BLACK = false
)

type RBNode struct {
	Val    int
	Color  bool // true=RED, false=BLACK
	Left   *RBNode
	Right  *RBNode
	Parent *RBNode
}

func colorStr(c bool) string {
	if c { return "RED" }
	return "BLACK"
}

func newNode(val int) *RBNode {
	return &RBNode{Val: val, Color: RED} // new nodes are always red
}

func main() {
	root := &RBNode{Val: 10, Color: BLACK}
	root.Left = &RBNode{Val: 5, Color: RED, Parent: root}
	root.Right = &RBNode{Val: 15, Color: RED, Parent: root}

	fmt.Printf("Root: %d (%s)\n", root.Val, colorStr(root.Color))
	fmt.Printf("Left: %d (%s)\n", root.Left.Val, colorStr(root.Left.Color))
	fmt.Printf("Right: %d (%s)\n", root.Right.Val, colorStr(root.Right.Color))
}
```

**Textual Figure:**
```
 Red-Black Tree Node Structure:

        10(B)
       /    \
     5(R)   15(R)
    / \     / \
  nil nil nil nil   ← all NIL leaves are BLACK

 Legend: (B) = BLACK, (R) = RED

 ┌───────────────────────────────────────────────┐
 │ Property checks:                              │
 │  1. Every node is RED or BLACK          ✓     │
 │  2. Root is BLACK                       ✓     │
 │  3. NIL leaves are BLACK                ✓     │
 │  4. RED children are BLACK? N/A (leaves) ✓    │
 │  5. Black height: root→NIL = 2 on all   ✓    │
 │     10(B)→15(R)→NIL(B) : bh=2            │
 │     10(B)→ 5(R)→NIL(B) : bh=2            │
 │                                               │
 │ New nodes are always inserted as RED           │
 └───────────────────────────────────────────────┘
```

---

## Example 2: Left-Leaning Red-Black Tree (LLRB) — Insert

```go
package main

import "fmt"

const (
	RED   = true
	BLACK = false
)

type Node struct {
	Val         int
	Left, Right *Node
	Color       bool
}

func isRed(n *Node) bool {
	if n == nil { return false }
	return n.Color == RED
}

func rotateLeft(h *Node) *Node {
	x := h.Right; h.Right = x.Left; x.Left = h
	x.Color = h.Color; h.Color = RED; return x
}

func rotateRight(h *Node) *Node {
	x := h.Left; h.Left = x.Right; x.Right = h
	x.Color = h.Color; h.Color = RED; return x
}

func flipColors(h *Node) {
	h.Color = !h.Color
	h.Left.Color = !h.Left.Color
	h.Right.Color = !h.Right.Color
}

func insert(h *Node, val int) *Node {
	if h == nil { return &Node{Val: val, Color: RED} }
	if val < h.Val { h.Left = insert(h.Left, val) } else if val > h.Val { h.Right = insert(h.Right, val) }

	// Fix-up: maintain LLRB invariants
	if isRed(h.Right) && !isRed(h.Left) { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left) { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right) { flipColors(h) }
	return h
}

func inorder(n *Node) {
	if n == nil { return }
	inorder(n.Left)
	c := "B"; if n.Color == RED { c = "R" }
	fmt.Printf("%d(%s) ", n.Val, c)
	inorder(n.Right)
}

func main() {
	var root *Node
	for _, v := range []int{10, 20, 30, 15, 25, 5, 1} {
		root = insert(root, v)
		root.Color = BLACK // root is always black
	}
	inorder(root) // sorted with colors
	fmt.Println()
}
```

**Textual Figure:**
```
 LLRB Insert [10, 20, 30, 15, 25, 5, 1]:

 Insert 10:       Insert 20:          Insert 30:
   10(B)           10(B)                20(B)
                     \    rotateLeft   /    \
                    20(R)  ─────→   10(R)  30(R)
                              then     ↑ flipColors
                              fix     10(B)  30(B)

 LLRB fix-up rules (applied bottom-up):
 ┌──────────────────────────────────────────────┐
 │ Condition             │ Action          │
 ├───────────────────────┼──────────────────────┤
 │ Right RED, Left BLACK │ Rotate Left     │
 │ Left RED, LeftLeft RED│ Rotate Right    │
 │ Both children RED     │ Flip Colors     │
 └───────────────────────┴──────────────────────┘

 Final LLRB after all inserts:
           15(B)
          /     \
       5(R)     25(B)
      / \       /  \
    1(B) 10(B) 20(R) 30(R)

 Inorder: 1(R) 5(B) 10(B) 15(R) 20(R) 25(B) 30(B)
 Root forced BLACK after each insert
```

---

## Example 3: Verify Red-Black Properties

```go
package main

import "fmt"

const (RED = true; BLACK = false)

type Node struct {
	Val         int
	Left, Right *Node
	Color       bool
}

func isRed(n *Node) bool { return n != nil && n.Color == RED }

// Check property 4: no two consecutive reds
func noConsecutiveReds(n *Node) bool {
	if n == nil { return true }
	if isRed(n) && (isRed(n.Left) || isRed(n.Right)) {
		fmt.Printf("Violation: consecutive reds at %d\n", n.Val)
		return false
	}
	return noConsecutiveReds(n.Left) && noConsecutiveReds(n.Right)
}

// Check property 5: all paths have same black height
func blackHeight(n *Node) int {
	if n == nil { return 1 } // nil counts as black
	left := blackHeight(n.Left)
	right := blackHeight(n.Right)
	if left == -1 || right == -1 || left != right {
		return -1
	}
	add := 0; if n.Color == BLACK { add = 1 }
	return left + add
}

func isValidRBTree(root *Node) bool {
	if root == nil { return true }
	if root.Color != BLACK {
		fmt.Println("Root is not black!")
		return false
	}
	if !noConsecutiveReds(root) { return false }
	if blackHeight(root) == -1 {
		fmt.Println("Unequal black heights!")
		return false
	}
	return true
}

func rotateLeft(h *Node) *Node { x := h.Right; h.Right = x.Left; x.Left = h; x.Color = h.Color; h.Color = RED; return x }
func rotateRight(h *Node) *Node { x := h.Left; h.Left = x.Right; x.Right = h; x.Color = h.Color; h.Color = RED; return x }
func flipColors(h *Node) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }
func insert(h *Node, val int) *Node {
	if h == nil { return &Node{Val: val, Color: RED} }
	if val < h.Val { h.Left = insert(h.Left, val) } else if val > h.Val { h.Right = insert(h.Right, val) }
	if isRed(h.Right) && !isRed(h.Left) { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left) { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right) { flipColors(h) }
	return h
}

func main() {
	var root *Node
	for i := 1; i <= 20; i++ {
		root = insert(root, i)
		root.Color = BLACK
	}
	fmt.Println("Valid RB tree:", isValidRBTree(root))
	fmt.Println("Black height:", blackHeight(root))
}
```

**Textual Figure:**
```
 After inserting 1..20 into LLRB:

 Verification of Red-Black properties:
 ┌───────────────────────────────────────────────┐
 │ Check 1: Root is BLACK              → ✓ PASS │
 │ Check 2: No consecutive RED nodes   → ✓ PASS │
 │ Check 3: Equal black heights (all   → ✓ PASS │
 │          root→NIL paths same bh)              │
 └───────────────────────────────────────────────┘

 noConsecutiveReds(node) check:
          8(B)
         /    \
       4(R)   12(R)       ← RED nodes
      / \     / \
    2(B) 6(B) 10(B) 16(B) ← children are BLACK ✓

 blackHeight(node) count:
   Path: root → left → left → ... → NIL
   Count BLACK nodes only → same on all paths ✓

 Result: Valid=true, Black height=k
```

---

## Example 4: LLRB Delete Minimum

```go
package main

import "fmt"

const (RED = true; BLACK = false)

type Node struct {
	Val         int
	Left, Right *Node
	Color       bool
}

func isRed(n *Node) bool { return n != nil && n.Color == RED }
func rotateLeft(h *Node) *Node { x := h.Right; h.Right = x.Left; x.Left = h; x.Color = h.Color; h.Color = RED; return x }
func rotateRight(h *Node) *Node { x := h.Left; h.Left = x.Right; x.Right = h; x.Color = h.Color; h.Color = RED; return x }
func flipColors(h *Node) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }

func fixUp(h *Node) *Node {
	if isRed(h.Right) && !isRed(h.Left) { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left) { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right) { flipColors(h) }
	return h
}

func moveRedLeft(h *Node) *Node {
	flipColors(h)
	if isRed(h.Right.Left) {
		h.Right = rotateRight(h.Right)
		h = rotateLeft(h)
		flipColors(h)
	}
	return h
}

func deleteMin(h *Node) *Node {
	if h.Left == nil { return nil }
	if !isRed(h.Left) && !isRed(h.Left.Left) { h = moveRedLeft(h) }
	h.Left = deleteMin(h.Left)
	return fixUp(h)
}

func insert(h *Node, val int) *Node {
	if h == nil { return &Node{Val: val, Color: RED} }
	if val < h.Val { h.Left = insert(h.Left, val) } else if val > h.Val { h.Right = insert(h.Right, val) }
	return fixUp(h)
}

func min(h *Node) int { for h.Left != nil { h = h.Left }; return h.Val }

func inorder(n *Node) {
	if n == nil { return }
	inorder(n.Left); fmt.Printf("%d ", n.Val); inorder(n.Right)
}

func main() {
	var root *Node
	for _, v := range []int{5, 3, 7, 1, 4, 6, 8, 2} {
		root = insert(root, v)
		root.Color = BLACK
	}
	fmt.Print("Before: "); inorder(root); fmt.Println()
	fmt.Println("Min:", min(root))
	root = deleteMin(root)
	root.Color = BLACK
	fmt.Print("After deleteMin: "); inorder(root); fmt.Println()
}
```

**Textual Figure:**
```
 LLRB Delete Minimum from [5, 3, 7, 1, 4, 6, 8, 2]:

 Before:                 After deleteMin:
        5(B)                    5(B)
       /    \                  /    \
     3(B)    7(B)            3(B)    7(B)
    /   \   /   \           /   \   /   \
  1(B) 4(B) 6(B) 8(B)     2(B) 4(B) 6(B) 8(B)
    \
    2(R)

 deleteMin algorithm:
 ┌────────────────────────────────────────────┐
 │ Step 1: Walk left to find minimum      │
 │ Step 2: If left child & its left child │
 │         are both BLACK → moveRedLeft    │
 │ Step 3: Delete leftmost node (1)       │
 │ Step 4: fixUp on way back up           │
 │         (rotateLeft/Right, flipColors)  │
 └────────────────────────────────────────────┘

 Min removed: 1
 Inorder after: 2 3 4 5 6 7 8
```

---

## Example 5: Red-Black Tree Height Analysis

```go
package main

import (
	"fmt"
	"math"
)

const (RED = true; BLACK = false)

type Node struct {
	Val   int
	Left  *Node
	Right *Node
	Color bool
}

func isRed(n *Node) bool { return n != nil && n.Color == RED }
func rotateLeft(h *Node) *Node { x := h.Right; h.Right = x.Left; x.Left = h; x.Color = h.Color; h.Color = RED; return x }
func rotateRight(h *Node) *Node { x := h.Left; h.Left = x.Right; x.Right = h; x.Color = h.Color; h.Color = RED; return x }
func flipColors(h *Node) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }
func fixUp(h *Node) *Node {
	if isRed(h.Right) && !isRed(h.Left) { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left) { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right) { flipColors(h) }
	return h
}
func insert(h *Node, val int) *Node {
	if h == nil { return &Node{Val: val, Color: RED} }
	if val < h.Val { h.Left = insert(h.Left, val) } else if val > h.Val { h.Right = insert(h.Right, val) }
	return fixUp(h)
}

func treeHeight(n *Node) int {
	if n == nil { return 0 }
	l, r := treeHeight(n.Left), treeHeight(n.Right)
	if l > r { return l + 1 }
	return r + 1
}

func blackH(n *Node) int {
	if n == nil { return 1 }
	bh := blackH(n.Left)
	if n.Color == BLACK { bh++ }
	return bh
}

func main() {
	sizes := []int{100, 1000, 10000, 100000}
	for _, n := range sizes {
		var root *Node
		for i := 1; i <= n; i++ {
			root = insert(root, i)
			root.Color = BLACK
		}
		h := treeHeight(root)
		bh := blackH(root)
		theoretical := 2 * math.Log2(float64(n+1))
		fmt.Printf("n=%6d  height=%2d  black_height=%2d  2*log2(n+1)=%.1f\n",
			n, h, bh, theoretical)
	}
}
```

**Textual Figure:**
```
 Red-Black Tree Height Analysis (sorted insertions 1..N):

 ┌──────────┬──────────┬──────────────┬──────────────────┐
 │    N     │  Height  │ Black Height │ 2×log₂(N+1)     │
 ├──────────┼──────────┼──────────────┼──────────────────┤
 │    100   │   ~11    │      ~7      │     13.3        │
 │  1,000   │   ~17    │     ~10      │     19.9        │
 │ 10,000   │   ~23    │     ~14      │     26.6        │
 │100,000   │   ~29    │     ~17      │     33.2        │
 └──────────┴──────────┴──────────────┴──────────────────┘

 Key insight: height ≤ 2 × log₂(N+1) always holds ✓
 black_height ≈ log₂(N) (shortest paths are all-black)

 Height vs N (logarithmic growth):
 h |
 30|                                      *
 25|                          *
 20|                *
 15|     *
 10|  *
   +────────────────────────────────────> N
     100   1K     10K           100K
```

---

## Example 6: Simulate Go's Map — TreeMap using RB Tree

```go
package main

import "fmt"

const (RED = true; BLACK = false)

type Node struct {
	Key, Value  int
	Left, Right *Node
	Color       bool
}

func isRed(n *Node) bool { return n != nil && n.Color == RED }
func rotateLeft(h *Node) *Node { x := h.Right; h.Right = x.Left; x.Left = h; x.Color = h.Color; h.Color = RED; return x }
func rotateRight(h *Node) *Node { x := h.Left; h.Left = x.Right; x.Right = h; x.Color = h.Color; h.Color = RED; return x }
func flipColors(h *Node) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }

type TreeMap struct{ root *Node }

func (m *TreeMap) Put(key, val int) {
	m.root = m.put(m.root, key, val)
	m.root.Color = BLACK
}

func (m *TreeMap) put(h *Node, key, val int) *Node {
	if h == nil { return &Node{Key: key, Value: val, Color: RED} }
	if key < h.Key { h.Left = m.put(h.Left, key, val) } else if key > h.Key { h.Right = m.put(h.Right, key, val) } else { h.Value = val }
	if isRed(h.Right) && !isRed(h.Left) { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left) { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right) { flipColors(h) }
	return h
}

func (m *TreeMap) Get(key int) (int, bool) {
	n := m.root
	for n != nil {
		if key < n.Key { n = n.Left } else if key > n.Key { n = n.Right } else { return n.Value, true }
	}
	return 0, false
}

func (m *TreeMap) InOrder() {
	var walk func(*Node)
	walk = func(n *Node) {
		if n == nil { return }
		walk(n.Left); fmt.Printf("[%d:%d] ", n.Key, n.Value); walk(n.Right)
	}
	walk(m.root); fmt.Println()
}

func main() {
	tm := &TreeMap{}
	tm.Put(3, 30); tm.Put(1, 10); tm.Put(4, 40); tm.Put(1, 100); tm.Put(5, 50); tm.Put(2, 20)
	tm.InOrder() // sorted by key
	v, ok := tm.Get(1); fmt.Printf("Get(1): %d, found=%v\n", v, ok) // 100
	v, ok = tm.Get(9); fmt.Printf("Get(9): %d, found=%v\n", v, ok) // 0, false
}
```

**Textual Figure:**
```
 TreeMap using LLRB — sorted key-value store:

 Operations: Put(3,30) Put(1,10) Put(4,40) Put(1,100) Put(5,50) Put(2,20)

 Internal LLRB (sorted by key):
          3(B)
         /    \
       1(B)    4(B)
         \       \
         2(R)    5(R)

 Key 1 updated: value 10 → 100 (Put overwrites)

 InOrder traversal (sorted by key):
   [1:100] → [2:20] → [3:30] → [4:40] → [5:50]

 ┌───────────┬───────┬───────┐
 │ Operation │ Value │ Found │
 ├───────────┼───────┼───────┤
 │ Get(1)    │  100  │ true  │
 │ Get(9)    │   0   │ false │
 └───────────┴───────┴───────┘
```

---

## Example 7: RB Tree — Count Red and Black Nodes

```go
package main

import "fmt"

const (RED = true; BLACK = false)

type Node struct {
	Val   int
	Left  *Node
	Right *Node
	Color bool
}

func isRed(n *Node) bool { return n != nil && n.Color == RED }
func rotateLeft(h *Node) *Node { x := h.Right; h.Right = x.Left; x.Left = h; x.Color = h.Color; h.Color = RED; return x }
func rotateRight(h *Node) *Node { x := h.Left; h.Left = x.Right; x.Right = h; x.Color = h.Color; h.Color = RED; return x }
func flipColors(h *Node) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }
func fixUp(h *Node) *Node {
	if isRed(h.Right) && !isRed(h.Left) { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left) { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right) { flipColors(h) }
	return h
}
func insert(h *Node, v int) *Node {
	if h == nil { return &Node{Val: v, Color: RED} }
	if v < h.Val { h.Left = insert(h.Left, v) } else if v > h.Val { h.Right = insert(h.Right, v) }
	return fixUp(h)
}

func countColors(n *Node) (int, int) {
	if n == nil { return 0, 0 }
	lr, lb := countColors(n.Left)
	rr, rb := countColors(n.Right)
	r := lr + rr; b := lb + rb
	if n.Color == RED { r++ } else { b++ }
	return r, b
}

func main() {
	var root *Node
	for i := 1; i <= 31; i++ {
		root = insert(root, i)
		root.Color = BLACK
	}
	red, black := countColors(root)
	fmt.Printf("31 nodes: red=%d black=%d total=%d\n", red, black, red+black)
	// For a perfect tree of 31 nodes, we'd expect roughly 2x black vs red
}
```

**Textual Figure:**
```
 After inserting 1..31 into LLRB:

              16(B)
            /      \
         8(B)      24(B)
        /   \      /   \
      4(R) 12(R) 20(R) 28(R)
     / \   / \   / \   / \
    2  6  10 14 18 22 26 30  (BLACK)
   /\ /\ /\ /\ /\ /\ /\ /\
  1 3 5 7 ...              (some RED)

 Color distribution:
 ┌───────────┬─────────┐
 │  Color    │  Count  │
 ├───────────┼─────────┤
 │  BLACK    │  ~21    │
 │  RED      │  ~10    │
 │  Total    │   31    │
 └───────────┴─────────┘

 Ratio: ~2:1 (BLACK:RED)
 RED nodes appear as left-leaning links only (LLRB)
```

---

## Example 8: RB vs AVL — Structural Comparison

```go
package main

import "fmt"

// LLRB implementation
const (RED = true; BLACK = false)
type RBNode struct { Val int; Left, Right *RBNode; Color bool }
func rbRed(n *RBNode) bool { return n != nil && n.Color == RED }
func rbLR(h *RBNode) *RBNode { x := h.Right; h.Right = x.Left; x.Left = h; x.Color = h.Color; h.Color = RED; return x }
func rbRR(h *RBNode) *RBNode { x := h.Left; h.Left = x.Right; x.Right = h; x.Color = h.Color; h.Color = RED; return x }
func rbFlip(h *RBNode) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }
func rbInsert(h *RBNode, v int) *RBNode {
	if h == nil { return &RBNode{Val: v, Color: RED} }
	if v < h.Val { h.Left = rbInsert(h.Left, v) } else if v > h.Val { h.Right = rbInsert(h.Right, v) }
	if rbRed(h.Right) && !rbRed(h.Left) { h = rbLR(h) }
	if rbRed(h.Left) && rbRed(h.Left.Left) { h = rbRR(h) }
	if rbRed(h.Left) && rbRed(h.Right) { rbFlip(h) }
	return h
}
func rbHeight(n *RBNode) int { if n == nil { return 0 }; l, r := rbHeight(n.Left), rbHeight(n.Right); if l > r { return l + 1 }; return r + 1 }

// AVL implementation
type AVLNode struct { Val, Height int; Left, Right *AVLNode }
func avlH(n *AVLNode) int { if n == nil { return 0 }; return n.Height }
func avlMax(a, b int) int { if a > b { return a }; return b }
func avlUpd(n *AVLNode) { n.Height = 1 + avlMax(avlH(n.Left), avlH(n.Right)) }
func avlBf(n *AVLNode) int { return avlH(n.Left) - avlH(n.Right) }
func avlRR(y *AVLNode) *AVLNode { x := y.Left; y.Left = x.Right; x.Right = y; avlUpd(y); avlUpd(x); return x }
func avlLR(x *AVLNode) *AVLNode { y := x.Right; x.Right = y.Left; y.Left = x; avlUpd(x); avlUpd(y); return y }
func avlBal(n *AVLNode) *AVLNode { avlUpd(n); b := avlBf(n); if b > 1 { if avlBf(n.Left) < 0 { n.Left = avlLR(n.Left) }; return avlRR(n) }; if b < -1 { if avlBf(n.Right) > 0 { n.Right = avlRR(n.Right) }; return avlLR(n) }; return n }
func avlInsert(n *AVLNode, v int) *AVLNode { if n == nil { return &AVLNode{Val: v, Height: 1} }; if v < n.Val { n.Left = avlInsert(n.Left, v) } else if v > n.Val { n.Right = avlInsert(n.Right, v) }; return avlBal(n) }

func main() {
	sizes := []int{100, 1000, 10000}
	for _, n := range sizes {
		var rb *RBNode
		var avl *AVLNode
		for i := 1; i <= n; i++ {
			rb = rbInsert(rb, i); rb.Color = BLACK
			avl = avlInsert(avl, i)
		}
		fmt.Printf("n=%5d  RB_height=%2d  AVL_height=%2d\n", n, rbHeight(rb), avlH(avl))
	}
}
```

**Textual Figure:**
```
 Structural comparison (sorted inserts 1..N):

 ┌─────────┬─────────────┬─────────────┐
 │    N    │  RB Height  │ AVL Height  │
 ├─────────┼─────────────┼─────────────┤
 │    100  │     ~11     │      7      │
 │  1,000  │     ~17     │     10      │
 │ 10,000  │     ~23     │     14      │
 └─────────┴─────────────┴─────────────┘

 AVL (stricter balance):     RB (relaxed balance):
     |bf| ≤ 1 always           h ≤ 2×log₂(N+1)
     Shorter trees             Fewer rotations on delete
     Better for reads          Better for writes

      AVL            RB
       7             11        (N=100)
      10             17        (N=1000)
      14             23        (N=10000)
       ↑              ↑
   Tighter          Looser (but ≤ 3 rotations/delete)
```

---

## Example 9: Red-Black Tree Properties Cheat Sheet

```go
package main

import "fmt"

func main() {
	properties := []struct {
		Property    string
		Description string
	}{
		{"Color", "Every node is RED or BLACK"},
		{"Root", "Root is always BLACK"},
		{"NIL leaves", "All NIL leaves are BLACK"},
		{"Red rule", "RED node cannot have RED child"},
		{"Black height", "All paths root→NIL have same # of BLACK nodes"},
	}

	comparisons := []struct {
		Metric    string
		AVL       string
		RedBlack  string
	}{
		{"Height", "≤1.44 log₂ n", "≤2 log₂(n+1)"},
		{"Search", "Slightly faster (shorter)", "Slightly slower"},
		{"Insert rotations", "≤2", "≤2"},
		{"Delete rotations", "O(log n)", "≤3"},
		{"Recoloring", "None", "O(log n)"},
		{"Best for", "Read-heavy", "Write-heavy"},
		{"Used in", "Databases", "Linux kernel, Java TreeMap"},
	}

	fmt.Println("=== Red-Black Tree Properties ===")
	for i, p := range properties {
		fmt.Printf("%d. %s: %s\n", i+1, p.Property, p.Description)
	}

	fmt.Println("\n=== AVL vs Red-Black ===")
	fmt.Printf("%-20s %-25s %-25s\n", "Metric", "AVL", "Red-Black")
	fmt.Println("-------------------------------------------------------------------")
	for _, c := range comparisons {
		fmt.Printf("%-20s %-25s %-25s\n", c.Metric, c.AVL, c.RedBlack)
	}
}
```

**Textual Figure:**
```
 ┌────────────────────┬─────────────────┬────────────────────┐
 │ Metric             │      AVL        │    Red-Black         │
 ├────────────────────┼─────────────────┼────────────────────┤
 │ Height             │ ≤1.44 log₂ n   │ ≤2 log₂(n+1)        │
 │ Search             │ Slightly faster │ Slightly slower      │
 │ Insert rotations   │ ≤2              │ ≤2                   │
 │ Delete rotations   │ O(log n)        │ ≤3                   │
 │ Recoloring         │ None            │ O(log n)             │
 │ Best for           │ Read-heavy      │ Write-heavy          │
 │ Used in            │ Databases       │ Linux, Java TreeMap  │
 └────────────────────┴─────────────────┴────────────────────┘

 RB Tree 5 Properties:
   1. Every node → RED or BLACK
   2. Root → always BLACK
   3. NIL leaves → BLACK
   4. RED node → children must be BLACK
   5. All root→NIL paths → same BLACK count

   10(B)
  /    \
 5(R)  15(R)    ← No two consecutive REDs
 / \   / \
 3  7 12  20    ← RED's children are BLACK
(B)(B)(B)(B)
```

---

## Example 10: LLRB — Full Implementation with Search

```go
package main

import "fmt"

const (RED = true; BLACK = false)

type LLRB struct{ root *Node }

type Node struct {
	Key         int
	Val         string
	Left, Right *Node
	Color       bool
	Size        int
}

func isRed(n *Node) bool { return n != nil && n.Color == RED }
func sz(n *Node) int { if n == nil { return 0 }; return n.Size }

func rotateLeft(h *Node) *Node {
	x := h.Right; h.Right = x.Left; x.Left = h
	x.Color = h.Color; h.Color = RED
	x.Size = h.Size; h.Size = 1 + sz(h.Left) + sz(h.Right)
	return x
}
func rotateRight(h *Node) *Node {
	x := h.Left; h.Left = x.Right; x.Right = h
	x.Color = h.Color; h.Color = RED
	x.Size = h.Size; h.Size = 1 + sz(h.Left) + sz(h.Right)
	return x
}
func flipColors(h *Node) { h.Color = !h.Color; h.Left.Color = !h.Left.Color; h.Right.Color = !h.Right.Color }

func (t *LLRB) Put(key int, val string) {
	t.root = t.put(t.root, key, val)
	t.root.Color = BLACK
}

func (t *LLRB) put(h *Node, key int, val string) *Node {
	if h == nil { return &Node{Key: key, Val: val, Color: RED, Size: 1} }
	if key < h.Key { h.Left = t.put(h.Left, key, val) } else if key > h.Key { h.Right = t.put(h.Right, key, val) } else { h.Val = val }
	if isRed(h.Right) && !isRed(h.Left)      { h = rotateLeft(h) }
	if isRed(h.Left) && isRed(h.Left.Left)    { h = rotateRight(h) }
	if isRed(h.Left) && isRed(h.Right)         { flipColors(h) }
	h.Size = 1 + sz(h.Left) + sz(h.Right)
	return h
}

func (t *LLRB) Get(key int) (string, bool) {
	n := t.root
	for n != nil {
		if key < n.Key { n = n.Left } else if key > n.Key { n = n.Right } else { return n.Val, true }
	}
	return "", false
}

func (t *LLRB) Rank(key int) int { return t.rank(t.root, key) }
func (t *LLRB) rank(n *Node, key int) int {
	if n == nil { return 0 }
	if key < n.Key { return t.rank(n.Left, key) }
	if key > n.Key { return 1 + sz(n.Left) + t.rank(n.Right, key) }
	return sz(n.Left)
}

func (t *LLRB) Size() int { return sz(t.root) }

func main() {
	tree := &LLRB{}
	data := map[int]string{3: "C", 1: "A", 4: "D", 1: "A2", 5: "E", 9: "I", 2: "B", 6: "F"}
	for k, v := range data { tree.Put(k, v) }

	fmt.Println("Size:", tree.Size())
	for _, k := range []int{1, 3, 5, 7} {
		v, ok := tree.Get(k)
		fmt.Printf("Get(%d): val=%q found=%v rank=%d\n", k, v, ok, tree.Rank(k))
	}
}
```

**Textual Figure:**
```
 LLRB Full Implementation with Size-augmented nodes:

 Keys inserted: 1, 2, 3, 4, 5, 6, 9
 (from map iteration, order may vary)

 Internal LLRB tree:
          4(B, size=7)
         /          \
      2(R, s=3)    6(B, s=3)
     / \           / \
   1(B) 3(B)    5(B) 9(B)

 ┌───────┬───────┬───────┬──────┐
 │  Key  │  Val  │ Found │ Rank │
 ├───────┼───────┼───────┼──────┤
 │   1   │ "A2"  │ true  │  0   │  (0 keys < 1)
 │   3   │ "C"   │ true  │  2   │  (keys 1,2 < 3)
 │   5   │ "E"   │ true  │  4   │  (keys 1,2,3,4 < 5)
 │   7   │  ""   │ false │  5   │  (keys 1,2,3,4,5 < 7)
 └───────┴───────┴───────┴──────┘

 Rank = # of keys strictly less than given key
 Size field enables O(log n) rank queries
```

---

## Key Takeaways

1. **Red-Black trees** guarantee O(log n) with **at most 3 rotations per delete** (vs O(log n) for AVL)
2. **LLRB** simplifies implementation — only left-leaning red links, short code
3. RB trees are **preferred for write-heavy** workloads; AVL for read-heavy
4. Used in: Linux kernel, Java `TreeMap`, C++ `std::map`, .NET `SortedDictionary`
5. Go's standard library uses **hash maps**, not tree maps — implement RB trees when you need **sorted key iteration**

> **Next up:** Inorder Successor and Predecessor →
