# Phase 9: BST — Kth Smallest Element in BST

## Overview

Finding the **kth smallest** element leverages the BST's **inorder traversal** property: nodes visited in sorted order.

| Approach | Time | Space | Notes |
|----------|------|-------|-------|
| Inorder traversal | O(n) | O(h) | Visit all nodes up to kth |
| Morris traversal | O(n) | O(1) | No extra space |
| Augmented BST (size) | O(h) | O(1) | Requires size field |
| Iterative stack | O(h+k) | O(h) | Stops early |

---

## Example 1: Kth Smallest — Recursive Inorder (LeetCode 230)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func kthSmallest(root *TreeNode, k int) int {
	count := 0
	result := 0
	var inorder func(*TreeNode)
	inorder = func(node *TreeNode) {
		if node == nil { return }
		inorder(node.Left)
		count++
		if count == k {
			result = node.Val
			return
		}
		inorder(node.Right)
	}
	inorder(root)
	return result
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{2, &TreeNode{1, nil, nil}, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{6, nil, nil},
	}
	for k := 1; k <= 6; k++ {
		fmt.Printf("kth(%d) = %d\n", k, kthSmallest(root, k))
	}
}
```

**Textual Figure:**
```
 BST:
        5
       / \
      3   6
     / \
    2   4
   /
  1

 Inorder traversal: 1 → 2 → 3 → 4 → 5 → 6

 Finding kth smallest (stop counting at k):
 ┌───┬───────┬─────────────────────────────────────────┐
 │ k │ Value │ Inorder path                            │
 ├───┼───────┼─────────────────────────────────────────┤
 │ 1 │   1   │ →L→L→L→[1] count=1 ✓ STOP              │
 │ 2 │   2   │ →L→L→L→1→[2] count=2 ✓ STOP            │
 │ 3 │   3   │ →L→L→L→1→2→[3] count=3 ✓ STOP          │
 │ 4 │   4   │ →L→L→L→1→2→3→[4] count=4 ✓ STOP        │
 │ 5 │   5   │ →L→L→L→1→2→3→4→[5] count=5 ✓ STOP      │
 │ 6 │   6   │ →L→L→L→1→2→3→4→5→[6] count=6 ✓ STOP    │
 └───┴───────┴─────────────────────────────────────────┘

 Time: O(h + k) — traverse left spine + k nodes
```

---

## Example 2: Kth Smallest — Iterative Stack

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func kthSmallest(root *TreeNode, k int) int {
	stack := []*TreeNode{}
	cur := root
	for cur != nil || len(stack) > 0 {
		for cur != nil {
			stack = append(stack, cur)
			cur = cur.Left
		}
		cur = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		k--
		if k == 0 { return cur.Val }
		cur = cur.Right
	}
	return -1
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	for k := 1; k <= 7; k++ {
		fmt.Printf("kth(%d) = %d\n", k, kthSmallest(root, k))
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

 Iterative stack-based inorder (k=3, find 3rd smallest):

 ┌──────┬─────────────────┬─────────┬────────────────┐
 │ Step │ Stack           │ Pop/Val │ k remaining    │
 ├──────┼─────────────────┼─────────┼────────────────┤
 │  1   │ [20, 10, 5]     │   —     │ pushLeft(20)   │
 │  2   │ [20, 10]        │  pop 5  │ k=3→2          │
 │  3   │ [20, 10]        │   —     │ 5.Right=nil    │
 │  4   │ [20]            │ pop 10  │ k=2→1          │
 │  5   │ [20, 15]        │   —     │ pushLeft(15)   │
 │  6   │ [20]            │ pop 15  │ k=1→0 ✓ STOP   │
 └──────┴─────────────────┴─────────┴────────────────┘

 Result: kth(3) = 15

 All results:
   kth(1)=5, kth(2)=10, kth(3)=15, kth(4)=20,
   kth(5)=25, kth(6)=30, kth(7)=35
```

---

## Example 3: Kth Smallest — Morris Traversal (O(1) Space)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func kthSmallestMorris(root *TreeNode, k int) int {
	cur := root
	count := 0
	for cur != nil {
		if cur.Left == nil {
			count++
			if count == k { return cur.Val }
			cur = cur.Right
		} else {
			pred := cur.Left
			for pred.Right != nil && pred.Right != cur {
				pred = pred.Right
			}
			if pred.Right == nil {
				pred.Right = cur // create thread
				cur = cur.Left
			} else {
				pred.Right = nil // remove thread
				count++
				if count == k { return cur.Val }
				cur = cur.Right
			}
		}
	}
	return -1
}

func main() {
	root := &TreeNode{4,
		&TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
		&TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
	}
	for k := 1; k <= 7; k++ {
		fmt.Printf("Morris kth(%d) = %d\n", k, kthSmallestMorris(root, k))
	}
}
```

**Textual Figure:**
```
 BST:
        4
       / \
      2   6
     / \ / \
    1  3 5  7

 Morris traversal — threading & unthreading:

 Step 1: cur=4, has left → find pred of 4 in left subtree
         pred=3 (rightmost of left subtree)
         3.Right=nil → create thread: 3→4
         cur = 2

 Step 2: cur=2, has left → pred=1
         1.Right=nil → create thread: 1→2
         cur = 1

 Step 3: cur=1, no left → count=1, k=1→found! (if k=1)
         cur = 1.Right = 2 (via thread)

 Step 4: cur=2, has left → pred=1
         1.Right=2 (thread exists!) → remove thread
         count=2, cur = 2.Right = 3

 Step 5: cur=3, no left → count=3
         cur = 3.Right = 4 (via thread)

 Step 6: cur=4, has left → pred=3
         3.Right=4 (thread!) → remove, count=4
         cur = 4.Right = 6

 ...continues visiting 5, 6, 7

 ┌────────────────────────────────────────────┐
 │ Morris: O(n) time, O(1) space             │
 │ Each edge traversed at most 3 times       │
 │ Creates temporary threads, then removes   │
 └────────────────────────────────────────────┘
```

---

## Example 4: Kth Smallest with Augmented BST (Size Field)

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

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val, Size: 1} }
	if val < root.Val {
		root.Left = insert(root.Left, val)
	} else {
		root.Right = insert(root.Right, val)
	}
	root.Size = 1 + size(root.Left) + size(root.Right)
	return root
}

func kthSmallest(root *TreeNode, k int) int {
	leftSize := size(root.Left)
	if k <= leftSize {
		return kthSmallest(root.Left, k)
	} else if k == leftSize+1 {
		return root.Val
	}
	return kthSmallest(root.Right, k-leftSize-1)
}

func main() {
	var root *TreeNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}
	for k := 1; k <= 7; k++ {
		fmt.Printf("kth(%d) = %d\n", k, kthSmallest(root, k))
	}
}
```

**Textual Figure:**
```
 Augmented BST with Size field:
            20 (size=7)
           /  \
     10 (s=3)   30 (s=3)
     /  \       /  \
   5(1) 15(1) 25(1) 35(1)

 kthSmallest algorithm using size:
 ┌───┬───────┬─────────────────────────────────────┐
 │ k │ Value │ Decision path                       │
 ├───┼───────┼─────────────────────────────────────┤
 │ 1 │   5   │ ls=3, k≤3→L; ls=0, k≤0? no,       │
 │   │       │ k=1=0+1→return 5                    │
 │ 2 │  10   │ ls=3, k≤3→L; ls=1, k=2=1+1→ret 10 │
 │ 3 │  15   │ ls=3, k≤3→L; ls=1, k>2→R(k=1);    │
 │   │       │ ls=0, k=1=0+1→return 15             │
 │ 4 │  20   │ ls=3, k=4=3+1→return 20             │
 │ 5 │  25   │ ls=3, k>4→R(k=1); ls=1, k=1≤1→L;  │
 │   │       │ ls=0, k=1=0+1→return 25             │
 │ 6 │  30   │ ls=3, k>4→R(k=2); ls=1, k=2=1+1   │
 │   │       │ →return 30                           │
 │ 7 │  35   │ ls=3, k>4→R(k=3); ls=1, k>2→R(k=1)│
 │   │       │ ls=0, k=1=0+1→return 35             │
 └───┴───────┴─────────────────────────────────────┘

 Time: O(h) per query — no traversal needed!
```

---

## Example 5: Kth Largest Element

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func kthLargest(root *TreeNode, k int) int {
	count := 0
	result := 0
	var reverseInorder func(*TreeNode)
	reverseInorder = func(node *TreeNode) {
		if node == nil || count >= k { return }
		reverseInorder(node.Right)
		count++
		if count == k {
			result = node.Val
			return
		}
		reverseInorder(node.Left)
	}
	reverseInorder(root)
	return result
}

func main() {
	root := &TreeNode{20,
		&TreeNode{10, &TreeNode{5, nil, nil}, &TreeNode{15, nil, nil}},
		&TreeNode{30, &TreeNode{25, nil, nil}, &TreeNode{35, nil, nil}},
	}
	for k := 1; k <= 7; k++ {
		fmt.Printf("kth largest(%d) = %d\n", k, kthLargest(root, k))
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

 Reverse inorder: 35 → 30 → 25 → 20 → 15 → 10 → 5
                   1st   2nd  3rd  4th  5th  6th  7th largest

 ┌───┬───────────────┬──────────────────────────────────┐
 │ k │ kth largest   │ Trace (reverse inorder)          │
 ├───┼───────────────┼──────────────────────────────────┤
 │ 1 │     35        │ →R→R→[35] count=1 ✓ STOP         │
 │ 2 │     30        │ →R→R→35→[30] count=2 ✓ STOP      │
 │ 3 │     25        │ ···→30→[25] count=3 ✓ STOP       │
 │ 4 │     20        │ ···→25→[20] count=4 ✓ STOP       │
 │ 5 │     15        │ ···→20→[15] count=5 ✓ STOP       │
 │ 6 │     10        │ ···→15→[10] count=6 ✓ STOP       │
 │ 7 │      5        │ ···→10→[5]  count=7 ✓ STOP       │
 └───┴───────────────┴──────────────────────────────────┘

 kth largest = (n - k + 1)th smallest
 Reverse inorder: Right → Root → Left
```

---

## Example 6: Two Sum in BST Using Kth Smallest Iterator

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

type BSTIterator struct{ stack []*TreeNode }

func NewForward(root *TreeNode) *BSTIterator {
	it := &BSTIterator{}
	for root != nil { it.stack = append(it.stack, root); root = root.Left }
	return it
}
func (it *BSTIterator) Next() int {
	top := it.stack[len(it.stack)-1]; it.stack = it.stack[:len(it.stack)-1]
	cur := top.Right; for cur != nil { it.stack = append(it.stack, cur); cur = cur.Left }
	return top.Val
}
func (it *BSTIterator) HasNext() bool { return len(it.stack) > 0 }

type BSTRevIterator struct{ stack []*TreeNode }

func NewReverse(root *TreeNode) *BSTRevIterator {
	it := &BSTRevIterator{}
	for root != nil { it.stack = append(it.stack, root); root = root.Right }
	return it
}
func (it *BSTRevIterator) Next() int {
	top := it.stack[len(it.stack)-1]; it.stack = it.stack[:len(it.stack)-1]
	cur := top.Left; for cur != nil { it.stack = append(it.stack, cur); cur = cur.Right }
	return top.Val
}
func (it *BSTRevIterator) HasNext() bool { return len(it.stack) > 0 }

func twoSumBST(root *TreeNode, target int) (int, int, bool) {
	fwd := NewForward(root)
	rev := NewReverse(root)
	lo := fwd.Next()
	hi := rev.Next()
	for lo < hi {
		sum := lo + hi
		if sum == target { return lo, hi, true }
		if sum < target { lo = fwd.Next() } else { hi = rev.Next() }
	}
	return 0, 0, false
}

func main() {
	root := &TreeNode{7,
		&TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{5, nil, nil}},
		&TreeNode{11, &TreeNode{9, nil, nil}, &TreeNode{13, nil, nil}},
	}
	targets := []int{14, 8, 20, 12}
	for _, t := range targets {
		a, b, ok := twoSumBST(root, t)
		if ok {
			fmt.Printf("Two sum = %d: %d + %d\n", t, a, b)
		} else {
			fmt.Printf("Two sum = %d: not found\n", t)
		}
	}
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

 Inorder: 1, 3, 5, 7, 9, 11, 13

 Two-pointer approach using iterators:
   Forward  iterator → produces: 1, 3, 5, 7, ...
   Reverse iterator → produces: 13, 11, 9, 7, ...

 twoSumBST(target=14):
 ┌──────┬────┬────┬─────┬──────────────┐
 │ Step │ lo │ hi │ sum │ Action       │
 ├──────┼────┼────┼─────┼──────────────┤
 │  1   │  1 │ 13 │  14 │ 14=14 ✓ FOUND│
 └──────┴────┴────┴─────┴──────────────┘

 twoSumBST(target=8):
 ┌──────┬────┬────┬─────┬──────────────┐
 │ Step │ lo │ hi │ sum │ Action       │
 ├──────┼────┼────┼─────┼──────────────┤
 │  1   │  1 │ 13 │  14 │ 14>8 → hi-- │
 │  2   │  1 │ 11 │  12 │ 12>8 → hi-- │
 │  3   │  1 │  9 │  10 │ 10>8 → hi-- │
 │  4   │  1 │  7 │   8 │ 8=8 ✓ FOUND │
 └──────┴────┴────┴─────┴──────────────┘

 O(n) time, O(h) space — two stacks of height h
```

---

## Example 7: Kth Smallest with Follow-up (Frequent Modifications)

```go
package main

import "fmt"

type AugNode struct {
	Val         int
	Left, Right *AugNode
	Size        int
}

func sz(n *AugNode) int { if n == nil { return 0 }; return n.Size }

func insert(root *AugNode, val int) *AugNode {
	if root == nil { return &AugNode{Val: val, Size: 1} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	root.Size = 1 + sz(root.Left) + sz(root.Right)
	return root
}

func delete(root *AugNode, val int) *AugNode {
	if root == nil { return nil }
	if val < root.Val {
		root.Left = delete(root.Left, val)
	} else if val > root.Val {
		root.Right = delete(root.Right, val)
	} else {
		if root.Left == nil { return root.Right }
		if root.Right == nil { return root.Left }
		succ := root.Right
		for succ.Left != nil { succ = succ.Left }
		root.Val = succ.Val
		root.Right = delete(root.Right, succ.Val)
	}
	root.Size = 1 + sz(root.Left) + sz(root.Right)
	return root
}

func kthSmallest(root *AugNode, k int) int {
	ls := sz(root.Left)
	if k <= ls { return kthSmallest(root.Left, k) }
	if k == ls+1 { return root.Val }
	return kthSmallest(root.Right, k-ls-1)
}

func main() {
	var root *AugNode
	for _, v := range []int{20, 10, 30, 5, 15, 25, 35} {
		root = insert(root, v)
	}
	fmt.Println("Before deletion:")
	for k := 1; k <= 7; k++ { fmt.Printf("  kth(%d) = %d\n", k, kthSmallest(root, k)) }

	root = delete(root, 15)
	root = delete(root, 30)
	fmt.Println("After deleting 15, 30:")
	for k := 1; k <= 5; k++ { fmt.Printf("  kth(%d) = %d\n", k, kthSmallest(root, k)) }

	root = insert(root, 12)
	root = insert(root, 28)
	fmt.Println("After inserting 12, 28:")
	for k := 1; k <= 7; k++ { fmt.Printf("  kth(%d) = %d\n", k, kthSmallest(root, k)) }
}
```

**Textual Figure:**
```
 Augmented BST with Size — supports insert/delete + kth:

 Before:                     After delete 15, 30:
      20 (s=7)                    20 (s=5)
     /  \                        /  \
  10(3)  30(3)                10(2)  35(2)
  / \    / \                 /       /
 5  15  25  35              5      25

 After insert 12, 28:
        20 (s=7)
       /  \
    10(3)  35(3)
    / \    /
   5  12  25
          \
          28

 Size field updated on every insert/delete:
 ┌───────────┬──────────────────────────────────┐
 │ Operation │ kth results                      │
 ├───────────┼──────────────────────────────────┤
 │ Initial   │ 5,10,15,20,25,30,35              │
 │ Del 15,30 │ 5,10,20,25,35                    │
 │ Ins 12,28 │ 5,10,12,20,25,28,35              │
 └───────────┴──────────────────────────────────┘

 O(h) per insert, delete, AND kthSmallest query
```

---

## Example 8: Kth Smallest in Two BSTs (Merge Problem)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

type Iterator struct{ stack []*TreeNode }

func NewIter(root *TreeNode) *Iterator {
	it := &Iterator{}
	for root != nil { it.stack = append(it.stack, root); root = root.Left }
	return it
}
func (it *Iterator) Peek() int { return it.stack[len(it.stack)-1].Val }
func (it *Iterator) Next() int {
	top := it.stack[len(it.stack)-1]; it.stack = it.stack[:len(it.stack)-1]
	cur := top.Right; for cur != nil { it.stack = append(it.stack, cur); cur = cur.Left }
	return top.Val
}
func (it *Iterator) HasNext() bool { return len(it.stack) > 0 }

func kthMerged(root1, root2 *TreeNode, k int) int {
	it1 := NewIter(root1)
	it2 := NewIter(root2)
	for k > 0 {
		k--
		if it1.HasNext() && it2.HasNext() {
			if it1.Peek() <= it2.Peek() {
				val := it1.Next()
				if k == 0 { return val }
			} else {
				val := it2.Next()
				if k == 0 { return val }
			}
		} else if it1.HasNext() {
			val := it1.Next()
			if k == 0 { return val }
		} else {
			val := it2.Next()
			if k == 0 { return val }
		}
	}
	return -1
}

func main() {
	tree1 := &TreeNode{5, &TreeNode{2, nil, nil}, &TreeNode{8, nil, nil}}
	tree2 := &TreeNode{4, &TreeNode{1, nil, nil}, &TreeNode{7, &TreeNode{6, nil, nil}, nil}}
	// Tree1 inorder: 2, 5, 8
	// Tree2 inorder: 1, 4, 6, 7
	// Merged sorted: 1, 2, 4, 5, 6, 7, 8
	for k := 1; k <= 7; k++ {
		fmt.Printf("kth(%d) in merged = %d\n", k, kthMerged(tree1, tree2, k))
	}
}
```

**Textual Figure:**
```
 Tree 1:     Tree 2:
    5           4
   / \         / \
  2   8       1   7
                 /
                6

 Inorder 1: 2, 5, 8
 Inorder 2: 1, 4, 6, 7

 Merge using two iterators (like merge step in merge sort):
 ┌──────┬─────────┬─────────┬─────────────────────┐
 │ Step │  it1    │  it2    │ Pick (smaller)       │
 ├──────┼─────────┼─────────┼─────────────────────┤
 │  1   │ peek=2  │ peek=1  │ pick 1 from it2      │
 │  2   │ peek=2  │ peek=4  │ pick 2 from it1      │
 │  3   │ peek=5  │ peek=4  │ pick 4 from it2      │
 │  4   │ peek=5  │ peek=6  │ pick 5 from it1      │
 │  5   │ peek=8  │ peek=6  │ pick 6 from it2      │
 │  6   │ peek=8  │ peek=7  │ pick 7 from it2      │
 │  7   │ peek=8  │ (empty) │ pick 8 from it1      │
 └──────┴─────────┴─────────┴─────────────────────┘

 Merged: 1, 2, 4, 5, 6, 7, 8
 O(k) time, O(h1 + h2) space
```

---

## Example 9: Median of BST

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func countNodes(root *TreeNode) int {
	if root == nil { return 0 }
	return 1 + countNodes(root.Left) + countNodes(root.Right)
}

func kthSmallest(root *TreeNode, k int) int {
	stack := []*TreeNode{}
	cur := root
	for cur != nil || len(stack) > 0 {
		for cur != nil { stack = append(stack, cur); cur = cur.Left }
		cur = stack[len(stack)-1]; stack = stack[:len(stack)-1]
		k--
		if k == 0 { return cur.Val }
		cur = cur.Right
	}
	return -1
}

func median(root *TreeNode) float64 {
	n := countNodes(root)
	if n%2 == 1 {
		return float64(kthSmallest(root, n/2+1))
	}
	a := kthSmallest(root, n/2)
	b := kthSmallest(root, n/2+1)
	return float64(a+b) / 2.0
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
	fmt.Printf("n=%d, median=%.1f\n", countNodes(root), median(root))

	root = insert(root, 22)
	fmt.Printf("n=%d, median=%.1f\n", countNodes(root), median(root))
}
```

**Textual Figure:**
```
 BST (7 nodes):
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35

 Inorder: 5, 10, 15, [20], 25, 30, 35
 n=7 (odd) → median = kth(4) = 20

 After insert 22 (8 nodes):
          20
         /  \
       10    30
      /  \   / \
     5   15 25  35
            /
           22

 Inorder: 5, 10, 15, [20, 22], 25, 30, 35
 n=8 (even) → median = (kth(4) + kth(5)) / 2
            = (20 + 22) / 2 = 21.0

 ┌─────────┬──────────┬───────────────────────┐
 │    n    │  Median  │ Formula               │
 ├─────────┼──────────┼───────────────────────┤
 │ 7 (odd) │  20.0    │ kth(4)                │
 │ 8 (even)│  21.0    │ (kth(4)+kth(5))/2     │
 └─────────┴──────────┴───────────────────────┘
```

---

## Example 10: Kth Smallest — Approach Comparison

```go
package main

import (
	"fmt"
	"time"
)

type TreeNode struct {
	Val, Size   int
	Left, Right *TreeNode
}

func sz(n *TreeNode) int { if n == nil { return 0 }; return n.Size }

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val, Size: 1} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	root.Size = 1 + sz(root.Left) + sz(root.Right)
	return root
}

// Approach 1: Recursive inorder
func kthInorder(root *TreeNode, k int) int {
	count := 0; result := 0
	var f func(*TreeNode)
	f = func(n *TreeNode) {
		if n == nil || count >= k { return }
		f(n.Left); count++; if count == k { result = n.Val; return }; f(n.Right)
	}
	f(root); return result
}

// Approach 2: Augmented size
func kthAugmented(root *TreeNode, k int) int {
	ls := sz(root.Left)
	if k <= ls { return kthAugmented(root.Left, k) }
	if k == ls+1 { return root.Val }
	return kthAugmented(root.Right, k-ls-1)
}

func main() {
	var root *TreeNode
	n := 10000
	for i := 1; i <= n; i++ { root = insert(root, i) }

	iters := 1000
	k := n / 2

	start := time.Now()
	for i := 0; i < iters; i++ { kthInorder(root, k) }
	t1 := time.Since(start)

	start = time.Now()
	for i := 0; i < iters; i++ { kthAugmented(root, k) }
	t2 := time.Since(start)

	fmt.Printf("Inorder kth(%d): %v\n", k, t1)
	fmt.Printf("Augmented kth(%d): %v\n", k, t2)
	fmt.Println("Result:", kthAugmented(root, k))
}
```

**Textual Figure:**
```
 Approach comparison for kth smallest:

 ┌───────────────────┬──────────┬──────────┬────────────────────┐
 │ Approach          │  Time    │  Space   │ Notes              │
 ├───────────────────┼──────────┼──────────┼────────────────────┤
 │ Recursive inorder │ O(h + k) │  O(h)   │ Simple, no augment │
 │ Iterative stack   │ O(h + k) │  O(h)   │ Can stop early     │
 │ Morris traversal  │ O(n)     │  O(1)   │ No extra space     │
 │ Augmented (size)  │ O(h)     │  O(1)*  │ Fastest per query  │
 └───────────────────┴──────────┴──────────┴────────────────────┘
 * O(1) additional per query; O(n) for size field storage

 For n=10,000 (degenerate BST), k=5000:
   Inorder:   must visit ~5000 nodes to reach kth
   Augmented: follows single path using size → O(h) steps

 Decision guide:
 ┌──────────────────────────┬──────────────────────────┐
 │ Use case                 │ Best approach            │
 ├──────────────────────────┼──────────────────────────┤
 │ One-time query           │ Iterative stack          │
 │ Frequent queries, static │ Augmented BST            │
 │ Frequent modifications   │ Augmented BST (maintain) │
 │ Memory constrained       │ Morris traversal         │
 └──────────────────────────┴──────────────────────────┘
```

---

## Key Takeaways

1. **Inorder traversal** of BST yields sorted order — kth smallest is kth element visited
2. **Iterative stack** is preferred over recursive for early termination
3. **Augmented BST** with `Size` field enables O(h) kth queries
4. **Morris traversal** achieves O(1) space but modifies tree temporarily
5. For **kth largest**, use reverse inorder (right → root → left)
6. **Two-pointer** technique (forward + reverse iterator) solves Two Sum in BST

> **Next up:** BST — Self-Balancing Trade-offs →
