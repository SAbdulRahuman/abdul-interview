# Phase 9: BST — Inorder Successor and Predecessor

## Overview

| Term | Definition |
|------|-----------|
| **Inorder Successor** | The node with the smallest value **greater than** the given node |
| **Inorder Predecessor** | The node with the largest value **smaller than** the given node |

Two scenarios for finding successor/predecessor:
1. **With parent pointer** — walk up/down the tree
2. **Without parent** — search from root, tracking candidates

---

## Example 1: Inorder Successor — Search from Root

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func inorderSuccessor(root, target *TreeNode) *TreeNode {
	var succ *TreeNode
	cur := root
	for cur != nil {
		if cur.Val > target.Val {
			succ = cur // potential successor
			cur = cur.Left
		} else {
			cur = cur.Right
		}
	}
	return succ
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
	for _, v := range []int{5, 10, 15, 20, 25, 30, 35} {
		node := find(root, v)
		succ := inorderSuccessor(root, node)
		if succ != nil {
			fmt.Printf("Successor of %d: %d\n", v, succ.Val)
		} else {
			fmt.Printf("Successor of %d: none\n", v)
		}
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

 Inorder: 5 → 10 → 15 → 20 → 25 → 30 → 35

 Successor search from root (go left when cur > target, track succ):
 ┌──────┬───────────┬─────────────────────────────────────┐
 │ Node │ Successor │ Trace                               │
 ├──────┼───────────┼─────────────────────────────────────┤
 │   5  │    10     │ 20>5→succ=20,L; 10>5→succ=10,L    │
 │  10  │    15     │ 20>10→succ=20,L; 10→10,R; 15>10    │
 │  15  │    20     │ 20>15→succ=20,L; 10<15→R; 15→15  │
 │  20  │    25     │ 20→20,R; 30>20→succ=30,L; 25>20    │
 │  25  │    30     │ 20<25→R; 30>25→succ=30,L; 25→25  │
 │  30  │    35     │ 20<30→R; 30→30,R; 35>30→succ=35   │
 │  35  │   none    │ 20<35→R; 30<35→R; 35→35,R; nil   │
 └──────┴───────────┴─────────────────────────────────────┘
```

--- — Search from Root

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func inorderPredecessor(root, target *TreeNode) *TreeNode {
	var pred *TreeNode
	cur := root
	for cur != nil {
		if cur.Val < target.Val {
			pred = cur
			cur = cur.Right
		} else {
			cur = cur.Left
		}
	}
	return pred
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
	for _, v := range []int{5, 10, 15, 20, 25, 30, 35} {
		node := find(root, v)
		pred := inorderPredecessor(root, node)
		if pred != nil {
			fmt.Printf("Predecessor of %d: %d\n", v, pred.Val)
		} else {
			fmt.Printf("Predecessor of %d: none\n", v)
		}
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

 Inorder: 5 → 10 → 15 → 20 → 25 → 30 → 35

 Predecessor search from root (go right when cur < target, track pred):
 ┌──────┬─────────────┬───────────────────────────────────┐
 │ Node │ Predecessor │ Trace                             │
 ├──────┼─────────────┼───────────────────────────────────┤
 │   5  │    none     │ 20>5→L; 10>5→L; 5→5,L; nil     │
 │  10  │     5       │ 20>10→L; 10→10,L; 5<10→pred=5  │
 │  15  │    10       │ 20>15→L; 10<15→pred=10,R; 15   │
 │  20  │    15       │ 20→20,L; 10<20→pred=10,R; 15<20 │
 │  25  │    20       │ 20<25→pred=20,R; 30>25→L; 25   │
 │  30  │    25       │ 20<30→pred=20,R; 30→30,L; 25<30 │
 │  35  │    30       │ 20<35→pred=20,R; 30<35→pred=30 │
 └──────┴─────────────┴───────────────────────────────────┘
```

--- (LeetCode 285)

```go
package main

import "fmt"

type TreeNode struct {
	Val    int
	Left   *TreeNode
	Right  *TreeNode
	Parent *TreeNode
}

func inorderSuccessor(node *TreeNode) *TreeNode {
	// Case 1: has right subtree → leftmost in right subtree
	if node.Right != nil {
		cur := node.Right
		for cur.Left != nil { cur = cur.Left }
		return cur
	}
	// Case 2: go up until we come from a left child
	cur := node
	for cur.Parent != nil && cur == cur.Parent.Right {
		cur = cur.Parent
	}
	return cur.Parent
}

func main() {
	//       20
	//      /  \
	//    10    30
	//   / \   / \
	//  5  15 25  35
	root := &TreeNode{Val: 20}
	root.Left = &TreeNode{Val: 10, Parent: root}
	root.Right = &TreeNode{Val: 30, Parent: root}
	root.Left.Left = &TreeNode{Val: 5, Parent: root.Left}
	root.Left.Right = &TreeNode{Val: 15, Parent: root.Left}
	root.Right.Left = &TreeNode{Val: 25, Parent: root.Right}
	root.Right.Right = &TreeNode{Val: 35, Parent: root.Right}

	nodes := []*TreeNode{
		root.Left.Left,   // 5
		root.Left,        // 10
		root.Left.Right,  // 15
		root,             // 20
		root.Right.Left,  // 25
		root.Right,       // 30
		root.Right.Right, // 35
	}
	for _, n := range nodes {
		succ := inorderSuccessor(n)
		if succ != nil {
			fmt.Printf("Successor of %d: %d\n", n.Val, succ.Val)
		} else {
			fmt.Printf("Successor of %d: none\n", n.Val)
		}
	}
}
```

**Textual Figure:**
```
 BST with parent pointers:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 Case 1: Node has right subtree
   → Successor = leftmost in right subtree
   Example: Successor(20) = leftmost of {30,25,35} = 25

 Case 2: No right subtree → go up until coming from left child
   Example: Successor(15):
     15 is RIGHT child of 10 → go up
     10 is LEFT child of 20  → STOP → Successor = 20

 ┌──────┬────────────┬───────────┬─────────────────────┐
 │ Node │ Has Right? │   Case    │ Successor           │
 ├──────┼────────────┼───────────┼─────────────────────┤
 │   5  │    No      │  Up→10(L) │  10 (came from left) │
 │  10  │   Yes(15)  │  leftmost │  15                   │
 │  15  │    No      │  Up→10(R) │  20 (10→up, left of   │
 │      │            │  Up→20(L) │  20)                  │
 │  20  │   Yes(30)  │  leftmost │  25                   │
 │  25  │    No      │  Up→30(L) │  30                   │
 │  30  │   Yes(35)  │  leftmost │  35                   │
 │  35  │    No      │  Up→30(R) │  none (root reached)  │
 │      │            │  Up→nil   │                       │
 └──────┴────────────┴───────────┴─────────────────────┘
```

---

```go
package main

import "fmt"

type TreeNode struct {
	Val    int
	Left   *TreeNode
	Right  *TreeNode
	Parent *TreeNode
}

func inorderPredecessor(node *TreeNode) *TreeNode {
	// Case 1: has left subtree → rightmost in left subtree
	if node.Left != nil {
		cur := node.Left
		for cur.Right != nil { cur = cur.Right }
		return cur
	}
	// Case 2: go up until we come from a right child
	cur := node
	for cur.Parent != nil && cur == cur.Parent.Left {
		cur = cur.Parent
	}
	return cur.Parent
}

func main() {
	root := &TreeNode{Val: 20}
	root.Left = &TreeNode{Val: 10, Parent: root}
	root.Right = &TreeNode{Val: 30, Parent: root}
	root.Left.Left = &TreeNode{Val: 5, Parent: root.Left}
	root.Left.Right = &TreeNode{Val: 15, Parent: root.Left}

	node := root.Left.Right // 15
	pred := inorderPredecessor(node)
	fmt.Printf("Predecessor of %d: %d\n", node.Val, pred.Val) // 10

	node = root // 20
	pred = inorderPredecessor(node)
	fmt.Printf("Predecessor of %d: %d\n", node.Val, pred.Val) // 15

	node = root.Left.Left // 5
	pred = inorderPredecessor(node)
	fmt.Printf("Predecessor of %d: %v\n", node.Val, pred) // <nil>
}
```

**Textual Figure:**
```
 BST with parent pointers:
          20
         /  \
       10    30
      /  \
     5    15

 Case 1: Node has left subtree
   → Predecessor = rightmost in left subtree
   Example: Predecessor(20) = rightmost of {10,5,15} = 15

 Case 2: No left subtree → go up until coming from right child
   Example: Predecessor(5):
     5 is LEFT child of 10 → go up
     10 is LEFT child of 20 → go up
     20 has no parent → Predecessor = nil

 ┌──────┬────────────┬─────────────────┬─────────────┐
 │ Node │ Has Left?  │     Case          │ Predecessor │
 ├──────┼────────────┼─────────────────┼─────────────┤
 │  15  │    No      │ Up→10 (R child)   │     10      │
 │  20  │  Yes(10)   │ rightmost of left │     15      │
 │   5  │    No      │ Up→10(L)→20(L)→nil│    none     │
 └──────┴────────────┴─────────────────┴─────────────┘
```

--- of a Value

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func findSuccPred(root *TreeNode, key int) (succ, pred *TreeNode) {
	cur := root
	for cur != nil {
		if cur.Val > key {
			succ = cur
			cur = cur.Left
		} else if cur.Val < key {
			pred = cur
			cur = cur.Right
		} else {
			// Found exact match
			// Predecessor: rightmost of left subtree
			if cur.Left != nil {
				p := cur.Left
				for p.Right != nil { p = p.Right }
				pred = p
			}
			// Successor: leftmost of right subtree
			if cur.Right != nil {
				s := cur.Right
				for s.Left != nil { s = s.Left }
				succ = s
			}
			return
		}
	}
	return
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}

	tests := []int{10, 20, 12, 35, 3}
	for _, key := range tests {
		succ, pred := findSuccPred(root, key)
		succVal, predVal := "nil", "nil"
		if succ != nil { succVal = fmt.Sprintf("%d", succ.Val) }
		if pred != nil { predVal = fmt.Sprintf("%d", pred.Val) }
		fmt.Printf("Key=%d: pred=%s succ=%s\n", key, predVal, succVal)
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

 findSuccPred(key) — finds both successor and predecessor:
 ┌─────┬───────────────┬──────┬──────┬─────────────────────────────────┐
 │ Key │ In tree?      │ Pred │ Succ │ How                             │
 ├─────┼───────────────┼──────┼──────┼─────────────────────────────────┤
 │ 10  │ Yes           │   5  │  15  │ rightmost(left)=5, leftmost(R)=15│
 │ 20  │ Yes           │  15  │  25  │ rightmost(left)=15,leftmost(R)=25│
 │ 12  │ No            │  10  │  15  │ tracked during search: 10<12,15>12│
 │ 35  │ Yes           │  30  │ nil  │ rightmost(left)=30, no right     │
 │  3  │ No            │ nil  │   5  │ no val<3 found, 5>3→succ        │
 └─────┴───────────────┴──────┴──────┴─────────────────────────────────┘

 When key is IN tree: use subtree (rightmost of left, leftmost of right)
 When key is NOT in tree: tracked during search path
```

---

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
		// Node found
		if root.Left == nil { return root.Right }
		if root.Right == nil { return root.Left }

		// Find inorder successor (min of right subtree)
		succ := root.Right
		for succ.Left != nil { succ = succ.Left }
		fmt.Printf("  Replacing %d with successor %d\n", root.Val, succ.Val)
		root.Val = succ.Val
		root.Right = deleteNode(root.Right, succ.Val)
	}
	return root
}

func inorder(r *TreeNode) {
	if r == nil { return }
	inorder(r.Left); fmt.Printf("%d ", r.Val); inorder(r.Right)
}

func main() {
	root := &TreeNode{50,
		&TreeNode{30,
			&TreeNode{20, nil, nil},
			&TreeNode{40, nil, nil}},
		&TreeNode{70,
			&TreeNode{60, nil, nil},
			&TreeNode{80, nil, nil}},
	}
	fmt.Print("Before: "); inorder(root); fmt.Println()
	root = deleteNode(root, 50)
	fmt.Print("After:  "); inorder(root); fmt.Println()
}
```

**Textual Figure:**
```
 BST Delete using inorder successor:

 Before:               After deleting 50:
       50                     60
      /  \                   /  \
    30    70               30    70
   / \   / \              / \     \
  20 40 60  80           20  40    80

 Delete 50 algorithm:
 ┌─────────────────────────────────────────────┐
 │ Step 1: Find node 50 (has both children) │
 │ Step 2: Find successor = leftmost in     │
 │         right subtree = 60               │
 │ Step 3: Replace 50's value with 60       │
 │ Step 4: Delete 60 from right subtree     │
 │         (60 has no left child → easy)    │
 └─────────────────────────────────────────────┘

 Inorder before: 20 30 40 50 60 70 80
 Inorder after:  20 30 40 60 70 80
```

---

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func allSuccessors(root *TreeNode) map[int]int {
	result := map[int]int{}
	var prev *TreeNode

	// Reverse inorder (right → root → left) to find successors
	var reverseInorder func(node *TreeNode)
	reverseInorder = func(node *TreeNode) {
		if node == nil { return }
		reverseInorder(node.Right)
		if prev != nil {
			result[node.Val] = prev.Val
		}
		prev = node
		reverseInorder(node.Left)
	}
	reverseInorder(root)
	return result
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}
	succs := allSuccessors(root)
	for _, v := range []int{5, 10, 15, 20, 25, 30, 35} {
		if s, ok := succs[v]; ok {
			fmt.Printf("%d → %d\n", v, s)
		} else {
			fmt.Printf("%d → none\n", v)
		}
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

 Reverse inorder traversal (right → root → left):
   35 → 30 → 25 → 20 → 15 → 10 → 5

 Building successor map (prev tracks last visited):
   Visit 35: prev=nil  → no entry
   Visit 30: prev=35   → succ[30] = 35
   Visit 25: prev=30   → succ[25] = 30
   Visit 20: prev=25   → succ[20] = 25
   Visit 15: prev=20   → succ[15] = 20
   Visit 10: prev=15   → succ[10] = 15
   Visit  5: prev=10   → succ[5]  = 10

 Result:
   5→10  10→15  15→20  20→25  25→30  30→35  35→none
```

---

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func allPredecessors(root *TreeNode) map[int]int {
	result := map[int]int{}
	var prev *TreeNode

	var inorder func(node *TreeNode)
	inorder = func(node *TreeNode) {
		if node == nil { return }
		inorder(node.Left)
		if prev != nil {
			result[node.Val] = prev.Val
		}
		prev = node
		inorder(node.Right)
	}
	inorder(root)
	return result
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}
	preds := allPredecessors(root)
	for _, v := range []int{5, 10, 15, 20, 25, 30, 35} {
		if p, ok := preds[v]; ok {
			fmt.Printf("%d ← %d\n", v, p)
		} else {
			fmt.Printf("%d ← none\n", v)
		}
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

 Inorder traversal (left → root → right):
   5 → 10 → 15 → 20 → 25 → 30 → 35

 Building predecessor map (prev tracks last visited):
   Visit  5: prev=nil  → no entry
   Visit 10: prev=5    → pred[10] = 5
   Visit 15: prev=10   → pred[15] = 10
   Visit 20: prev=15   → pred[20] = 15
   Visit 25: prev=20   → pred[25] = 20
   Visit 30: prev=25   → pred[30] = 25
   Visit 35: prev=30   → pred[35] = 30

 Result:
   5←none  10←5  15←10  20←15  25←20  30←25  35←30
```

---

```go
package main

import "fmt"

type ThreadedNode struct {
	Val         int
	Left, Right *ThreadedNode
	RightThread bool // true if Right points to inorder successor
}

func createThreaded(root *TreeNode) *ThreadedNode {
	vals := []int{}
	var inorder func(*TreeNode)
	inorder = func(n *TreeNode) {
		if n == nil { return }
		inorder(n.Left); vals = append(vals, n.Val); inorder(n.Right)
	}
	inorder(root)

	// Build threaded tree
	nodes := make(map[int]*ThreadedNode)
	for _, v := range vals { nodes[v] = &ThreadedNode{Val: v} }

	// Set threads: each node's Right points to its successor
	for i := 0; i < len(vals)-1; i++ {
		n := nodes[vals[i]]
		n.Right = nodes[vals[i+1]]
		n.RightThread = true
	}
	return nodes[vals[0]]
}

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	head := createThreaded(root)

	// Traverse using threads
	cur := head
	for cur != nil {
		fmt.Printf("%d ", cur.Val)
		if cur.RightThread {
			cur = cur.Right
		} else {
			cur = nil
		}
	}
	fmt.Println() // 5 10 15 20 25 30 35
}
```

**Textual Figure:**
```
 Original BST:
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 Threaded Binary Tree (right threads point to successor):

   5 ──thread─→ 10 ──thread─→ 15 ──thread─→ 20

   20 ──thread─→ 25 ──thread─→ 30 ──thread─→ 35 ─→ nil

 RightThread = true:
   Right pointer → inorder successor (not a real child)

 Traversal without stack or recursion:
 ┌───────────────────────────────────────┐
 │ cur = 5  → print 5                  │
 │ cur.RightThread=true → cur = 10     │
 │ cur = 10 → print 10                 │
 │ cur.RightThread=true → cur = 15     │
 │ ...continues until cur = nil        │
 └───────────────────────────────────────┘

 Output: 5 10 15 20 25 30 35
```

---

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

type BSTIterator struct {
	stack []*TreeNode
}

func NewBSTIterator(root *TreeNode) *BSTIterator {
	it := &BSTIterator{}
	it.pushLeft(root)
	return it
}

func (it *BSTIterator) pushLeft(node *TreeNode) {
	for node != nil {
		it.stack = append(it.stack, node)
		node = node.Left
	}
}

func (it *BSTIterator) Next() int {
	// Pops current smallest, finds its successor
	top := it.stack[len(it.stack)-1]
	it.stack = it.stack[:len(it.stack)-1]
	// If right child exists, successor is leftmost of right subtree
	it.pushLeft(top.Right)
	return top.Val
}

func (it *BSTIterator) HasNext() bool { return len(it.stack) > 0 }

// Predecessor iterator (reverse)
type BSTReverseIterator struct {
	stack []*TreeNode
}

func NewBSTReverseIterator(root *TreeNode) *BSTReverseIterator {
	it := &BSTReverseIterator{}
	it.pushRight(root)
	return it
}

func (it *BSTReverseIterator) pushRight(node *TreeNode) {
	for node != nil {
		it.stack = append(it.stack, node)
		node = node.Right
	}
}

func (it *BSTReverseIterator) Next() int {
	top := it.stack[len(it.stack)-1]
	it.stack = it.stack[:len(it.stack)-1]
	it.pushRight(top.Left)
	return top.Val
}

func (it *BSTReverseIterator) HasNext() bool { return len(it.stack) > 0 }

func main() {
	root := &TreeNode{7,
		&TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{5, nil, nil}},
		&TreeNode{11, &TreeNode{9, nil, nil}, &TreeNode{13, nil, nil}},
	}

	fmt.Print("Forward (successors):   ")
	fwd := NewBSTIterator(root)
	for fwd.HasNext() { fmt.Printf("%d ", fwd.Next()) }
	fmt.Println()

	fmt.Print("Backward (predecessors): ")
	rev := NewBSTReverseIterator(root)
	for rev.HasNext() { fmt.Printf("%d ", rev.Next()) }
	fmt.Println()
}
```

**Textual Figure:**
```
 BST:
          7
         / \
        3   11
       / \  / \
      1  5 9  13

 Forward Iterator (successor logic via stack):
 ┌──────────────────────────────────────────────┐
 │ pushLeft(7): stack = [7, 3, 1]              │
 │ Next(): pop 1, pushLeft(nil)  → return 1    │
 │ Next(): pop 3, pushLeft(5)    → return 3    │
 │   stack = [7, 5]                             │
 │ Next(): pop 5, pushLeft(nil)  → return 5    │
 │ Next(): pop 7, pushLeft(11,9) → return 7    │
 │   stack = [11, 9]                            │
 │ Next(): pop 9, pushLeft(nil)  → return 9    │
 │ Next(): pop 11, pushLeft(13)  → return 11   │
 │ Next(): pop 13                → return 13   │
 └──────────────────────────────────────────────┘

 Forward:   1 → 3 → 5 → 7 → 9 → 11 → 13
 Backward: 13 → 11 → 9 → 7 → 5 → 3 → 1
   (Reverse iterator uses pushRight instead)

 O(h) space, O(1) amortized per Next() call
```

---

## Key Takeaways

1. **Successor** = leftmost node in right subtree, or first ancestor where node is in left subtree
2. **Predecessor** = rightmost node in left subtree, or first ancestor where node is in right subtree
3. Without parent pointers: search from root keeping track of best candidate — O(h)
4. **Reverse inorder** traversal naturally visits predecessors → successors relationship
5. **BST Iterator** implements successor logic with O(h) space using a stack

> **Next up:** Self-Balancing BST Trade-offs →
