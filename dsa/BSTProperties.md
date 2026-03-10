# Phase 9: Binary Search Trees — BST Properties

## Overview

A **Binary Search Tree** (BST) is a binary tree where for every node:
- All values in the **left subtree** are **less than** the node's value
- All values in the **right subtree** are **greater than** the node's value
- Both subtrees are also BSTs

```
        8          ✓ BST
       / \
      3   10
     / \    \
    1   6   14
       / \  /
      4  7 13
```

**Key property**: Inorder traversal of a BST yields a **sorted sequence**.

---

## Example 1: BST Node Definition and Validation (LeetCode 98)

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

func isValidBST(root *TreeNode) bool {
	return validate(root, math.MinInt64, math.MaxInt64)
}

func validate(node *TreeNode, min, max int) bool {
	if node == nil { return true }
	if node.Val <= min || node.Val >= max {
		return false
	}
	return validate(node.Left, min, node.Val) &&
		validate(node.Right, node.Val, max)
}

func main() {
	// Valid BST
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{9, nil, nil}},
	}
	fmt.Println("Valid BST:", isValidBST(root)) // true

	// Invalid BST (4 in right subtree of 5 is wrong)
	bad := &TreeNode{5,
		&TreeNode{1, nil, nil},
		&TreeNode{4, &TreeNode{3, nil, nil}, &TreeNode{6, nil, nil}},
	}
	fmt.Println("Valid BST:", isValidBST(bad)) // false
}
```

**Textual Figure:**
```
Validate BST Using Min/Max Bounds:

  ┌─── Valid BST ───┐          ┌─── Invalid BST ──┐
  │       5         │          │       5          │
  │      / \        │          │      / \         │
  │     3   7       │          │     1   4 ← 4<5! │
  │    / \ / \      │          │        / \       │
  │   1  4 6  9     │          │       3   6      │
  └─────────────────┘          └──────────────────┘

  validate(5, -∞, +∞)
  ├── 5 ∈ (-∞,+∞) ✓
  ├── Left:  validate(3, -∞, 5)
  │   ├── 3 ∈ (-∞,5) ✓
  │   ├── validate(1, -∞, 3) ✓       validate(5, -∞, +∞)
  │   └── validate(4, 3, 5)  ✓       ├── validate(1, -∞, 5) ✓
  └── Right: validate(7, 5, +∞)      └── validate(4, 5, +∞)
      ├── 7 ∈ (5,+∞) ✓                   4 ≤ 5 → ✗ INVALID!
      ├── validate(6, 5, 7) ✓
      └── validate(9, 7, +∞) ✓       Output: false

  Output: true
```

---

## Example 2: Validate BST Using Inorder Traversal

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

func isValidBSTInorder(root *TreeNode) bool {
	prev := math.MinInt64
	valid := true

	var inorder func(node *TreeNode)
	inorder = func(node *TreeNode) {
		if node == nil || !valid { return }
		inorder(node.Left)
		if node.Val <= prev {
			valid = false
			return
		}
		prev = node.Val
		inorder(node.Right)
	}
	inorder(root)
	return valid
}

func main() {
	root := &TreeNode{2,
		&TreeNode{1, nil, nil},
		&TreeNode{3, nil, nil},
	}
	fmt.Println(isValidBSTInorder(root)) // true

	root2 := &TreeNode{2,
		&TreeNode{2, nil, nil},
		&TreeNode{2, nil, nil},
	}
	fmt.Println(isValidBSTInorder(root2)) // false (equal values)
}
```

**Textual Figure:**
```
Validate BST via Inorder Traversal (must be strictly increasing):

  Tree 1 (Valid):              Tree 2 (Invalid):
       2                            2
      / \                          / \
     1   3                        2   2

  Inorder walk:                Inorder walk:
  Step 1: visit 1             Step 1: visit 2
          prev = -∞                   prev = -∞
          1 > -∞ ✓                    2 > -∞ ✓
          prev ← 1                   prev ← 2

  Step 2: visit 2             Step 2: visit 2
          2 > 1 ✓                    2 > 2? ✗ NOT STRICTLY INCREASING
          prev ← 2                   → return false

  Step 3: visit 3
          3 > 2 ✓
          prev ← 3

  Sorted: [1, 2, 3] ✓         [2, 2, 2] has equal values ✗
  Output: true                 Output: false
```

---

## Example 3: BST Property — LCA is Simpler (LeetCode 235)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// In BST, LCA is found by comparing values — no need for full DFS
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	for root != nil {
		if p.Val < root.Val && q.Val < root.Val {
			root = root.Left
		} else if p.Val > root.Val && q.Val > root.Val {
			root = root.Right
		} else {
			return root // split point
		}
	}
	return nil
}

func main() {
	//       6
	//      / \
	//     2   8
	//    / \ / \
	//   0  4 7  9
	//     / \
	//    3   5
	n3 := &TreeNode{Val: 3}
	n5 := &TreeNode{Val: 5}
	n4 := &TreeNode{4, n3, n5}
	n0 := &TreeNode{Val: 0}
	n2 := &TreeNode{2, n0, n4}
	n7 := &TreeNode{Val: 7}
	n9 := &TreeNode{Val: 9}
	n8 := &TreeNode{8, n7, n9}
	root := &TreeNode{6, n2, n8}

	fmt.Println(lowestCommonAncestor(root, n2, n8).Val) // 6
	fmt.Println(lowestCommonAncestor(root, n2, n4).Val) // 2
	fmt.Println(lowestCommonAncestor(root, n3, n5).Val) // 4
}
```

**Textual Figure:**
```
LCA in BST — Follow Values (no full DFS needed):

         6
        / \
       2   8
      / \ / \
     0  4 7  9
       / \
      3   5

  ┌──────────────────────────────────────────────────────┐
  │ LCA(2, 8):                                          │
  │   root=6 → p=2 < 6 AND q=8 > 6 → split! return 6   │
  │   Answer: 6                                         │
  ├──────────────────────────────────────────────────────┤
  │ LCA(2, 4):                                          │
  │   root=6 → both 2,4 < 6 → go left                  │
  │   root=2 → p=2 == 2 → split point! return 2        │
  │   Answer: 2                                         │
  ├──────────────────────────────────────────────────────┤
  │ LCA(3, 5):                                          │
  │   root=6 → both 3,5 < 6 → go left                  │
  │   root=2 → both 3,5 > 2 → go right                 │
  │   root=4 → p=3 < 4 AND q=5 > 4 → split! return 4   │
  │   Answer: 4                                         │
  └──────────────────────────────────────────────────────┘
```

---

## Example 4: BST Property — Sorted Array to BST (LeetCode 108)

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

func inorder(root *TreeNode) {
	if root == nil { return }
	inorder(root.Left)
	fmt.Printf("%d ", root.Val)
	inorder(root.Right)
}

func height(root *TreeNode) int {
	if root == nil { return 0 }
	l, r := height(root.Left), height(root.Right)
	if l > r { return l + 1 }
	return r + 1
}

func main() {
	nums := []int{-10, -3, 0, 5, 9}
	root := sortedArrayToBST(nums)
	fmt.Print("Inorder: "); inorder(root); fmt.Println() // -10 -3 0 5 9
	fmt.Println("Height:", height(root))                   // 3 (balanced)
}
```

**Textual Figure:**
```
Sorted Array → Balanced BST (pick middle as root):

  Input: [-10, -3, 0, 5, 9]
                  ↑ mid=2 → root

  Step 1: root = 0
          left half  = [-10, -3]  → mid=0 → -10, right=-3
          right half = [5, 9]     → mid=0 → 5, right=9

  Resulting Balanced BST:
             0
            / \
          -3    5
          /      \
        -10       9

  Inorder: -10 -3 0 5 9  (sorted ✓)
  Height: 3 (balanced ✓)

  BST Property at every node:
    0:  left(-3,-10) < 0 < right(5,9) ✓
   -3:  left(-10) < -3               ✓
    5:  5 < right(9)                 ✓
```

---

## Example 5: BST Property — Range Sum Query (LeetCode 938)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func rangeSumBST(root *TreeNode, low, high int) int {
	if root == nil { return 0 }
	if root.Val < low {
		return rangeSumBST(root.Right, low, high) // prune left
	}
	if root.Val > high {
		return rangeSumBST(root.Left, low, high) // prune right
	}
	// root.Val in [low, high]
	return root.Val +
		rangeSumBST(root.Left, low, high) +
		rangeSumBST(root.Right, low, high)
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, nil, &TreeNode{18, nil, nil}},
	}
	fmt.Println(rangeSumBST(root, 7, 15)) // 32 (7 + 10 + 15)
}
```

**Textual Figure:**
```
Range Sum BST: sum of all nodes in [7, 15]

         10          Range: [7, 15]
        / \
       5   15
      / \    \
     3   7   18

  rangeSumBST(10, 7, 15):
  ├── 10 ∈ [7,15] → include 10
  ├── Left:  rangeSumBST(5, 7, 15)
  │   └── 5 < 7 → prune left subtree (skip 3)
  │       └── Right: rangeSumBST(7, 7, 15)
  │           └── 7 ∈ [7,15] → include 7
  │               (no children)
  └── Right: rangeSumBST(15, 7, 15)
      └── 15 ∈ [7,15] → include 15
          └── Right: rangeSumBST(18, 7, 15)
              └── 18 > 15 → prune right subtree

  Sum = 7 + 10 + 15 = 32
  Pruned nodes: 3, 5 (too small), 18 (too large)
```

---

## Example 6: BST Floor and Ceiling

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// Floor: largest value ≤ target
func floor(root *TreeNode, target int) int {
	result := -1
	for root != nil {
		if root.Val == target { return target }
		if root.Val < target {
			result = root.Val
			root = root.Right
		} else {
			root = root.Left
		}
	}
	return result
}

// Ceiling: smallest value ≥ target
func ceiling(root *TreeNode, target int) int {
	result := -1
	for root != nil {
		if root.Val == target { return target }
		if root.Val > target {
			result = root.Val
			root = root.Left
		} else {
			root = root.Right
		}
	}
	return result
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, &TreeNode{12, nil, nil}, &TreeNode{20, nil, nil}},
	}
	fmt.Println("Floor(6):", floor(root, 6))     // 5
	fmt.Println("Floor(10):", floor(root, 10))    // 10
	fmt.Println("Ceiling(6):", ceiling(root, 6))  // 7
	fmt.Println("Ceiling(13):", ceiling(root, 13)) // 15
}
```

**Textual Figure:**
```
Floor & Ceiling in BST:

         10
        / \
       5   15
      / \  / \
     3  7 12  20

  ┌──────────────────────────────────────────────┐
  │ Floor(6): largest value ≤ 6              │
  │   10 → 6<10 → go left                   │
  │   5  → 5<6  → result=5, go right         │
  │   7  → 7>6  → go left                    │
  │   nil → done                              │
  │   Answer: 5                               │
  ├──────────────────────────────────────────────┤
  │ Ceiling(6): smallest value ≥ 6           │
  │   10 → 10>6 → result=10, go left          │
  │   5  → 5<6  → go right                   │
  │   7  → 7>6  → result=7, go left           │
  │   nil → done                              │
  │   Answer: 7                               │
  ├──────────────────────────────────────────────┤
  │ Ceiling(13):                              │
  │   10 → 10<13 → go right                   │
  │   15 → 15>13 → result=15, go left         │
  │   12 → 12<13 → go right                   │
  │   nil → done                              │
  │   Answer: 15                              │
  └──────────────────────────────────────────────┘
```

---

## Example 7: Count Nodes in Range

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func countInRange(root *TreeNode, low, high int) int {
	if root == nil { return 0 }
	if root.Val < low {
		return countInRange(root.Right, low, high)
	}
	if root.Val > high {
		return countInRange(root.Left, low, high)
	}
	return 1 + countInRange(root.Left, low, high) + countInRange(root.Right, low, high)
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, &TreeNode{12, nil, nil}, &TreeNode{20, nil, nil}},
	}
	fmt.Println("Count [5,15]:", countInRange(root, 5, 15))   // 5
	fmt.Println("Count [7,12]:", countInRange(root, 7, 12))   // 3
}
```

**Textual Figure:**
```
Count Nodes in Range [5, 15]:

         10
        / \
       5   15
      / \  / \
     3  7 12  20

  countInRange(10, 5, 15):
  ├── 10 ∈ [5,15] → count 1
  ├── Left: countInRange(5, 5, 15)
  │   ├── 5 ∈ [5,15] → count 1
  │   ├── Left: 3 < 5 → prune left, go right only
  │   │   └── countInRange(nil) = 0
  │   └── Right: countInRange(7, 5, 15)
  │       └── 7 ∈ [5,15] → count 1 (leaf)
  └── Right: countInRange(15, 5, 15)
      ├── 15 ∈ [5,15] → count 1
      ├── Left: countInRange(12, 5, 15)
      │   └── 12 ∈ [5,15] → count 1 (leaf)
      └── Right: 20 > 15 → prune right

  Total for [5,15]: 5 nodes {5, 7, 10, 12, 15}
  Total for [7,12]: 3 nodes {7, 10, 12}
```

---

## Example 8: BST Property — Kth Smallest (LeetCode 230)

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

	var inorder func(node *TreeNode)
	inorder = func(node *TreeNode) {
		if node == nil || count >= k { return }
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
		fmt.Printf("k=%d → %d\n", k, kthSmallest(root, k))
	}
}
```

**Textual Figure:**
```
Kth Smallest in BST via Inorder Traversal:

         5
        / \
       3   6
      / \
     2   4
    /
   1

  Inorder traversal visits nodes in sorted order:

  Step  Node  count  k?    Result
  ────────────────────────────────
   1     1     1    k=1? →  1
   2     2     2    k=2? →  2
   3     3     3    k=3? →  3
   4     4     4    k=4? →  4
   5     5     5    k=5? →  5
   6     6     6    k=6? →  6

  Output:
  k=1 → 1,  k=2 → 2,  k=3 → 3
  k=4 → 4,  k=5 → 5,  k=6 → 6
```

---

## Example 9: BST to Greater Sum Tree (LeetCode 1038)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// Reverse inorder: right → node → left accumulates greater sums
func bstToGst(root *TreeNode) *TreeNode {
	sum := 0
	var reverseInorder func(node *TreeNode)
	reverseInorder = func(node *TreeNode) {
		if node == nil { return }
		reverseInorder(node.Right)
		sum += node.Val
		node.Val = sum
		reverseInorder(node.Left)
	}
	reverseInorder(root)
	return root
}

func inorder(root *TreeNode) {
	if root == nil { return }
	inorder(root.Left)
	fmt.Printf("%d ", root.Val)
	inorder(root.Right)
}

func main() {
	// BST: [1,2,3,4,5,6,7]
	root := &TreeNode{4,
		&TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
		&TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
	}
	bstToGst(root)
	inorder(root) // 28 27 25 22 18 13 7
	fmt.Println()
}
```

**Textual Figure:**
```
BST to Greater Sum Tree (reverse inorder accumulation):

  Before:               After (Greater Sum Tree):
       4                      22
      / \                    /  \
     2   6                 27    13
    / \ / \               / \   / \
   1  3 5  7             28 25 18  7

  Reverse inorder: 7 → 6 → 5 → 4 → 3 → 2 → 1

  Step  Visit  sum    New Val
  ────────────────────────────
   1     7    0+7=7     7
   2     6    7+6=13   13
   3     5   13+5=18   18
   4     4   18+4=22   22
   5     3   22+3=25   25
   6     2   25+2=27   27
   7     1   27+1=28   28

  Inorder output: 28 27 25 22 18 13 7
```

---

## Example 10: BST Properties Comparison Table

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

// Compute multiple BST properties in one pass
type BSTInfo struct {
	IsBST     bool
	Size      int
	Min, Max  int
	Height    int
}

func analyzeBST(node *TreeNode) BSTInfo {
	if node == nil {
		return BSTInfo{true, 0, math.MaxInt64, math.MinInt64, 0}
	}

	left := analyzeBST(node.Left)
	right := analyzeBST(node.Right)

	info := BSTInfo{}
	info.IsBST = left.IsBST && right.IsBST &&
		node.Val > left.Max && node.Val < right.Min
	info.Size = left.Size + right.Size + 1

	info.Min = left.Min
	if node.Val < info.Min { info.Min = node.Val }

	info.Max = right.Max
	if node.Val > info.Max { info.Max = node.Val }

	info.Height = left.Height
	if right.Height > info.Height { info.Height = right.Height }
	info.Height++

	return info
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{1, nil, nil}, &TreeNode{8, nil, nil}},
		&TreeNode{15, &TreeNode{12, nil, nil}, &TreeNode{20, nil, nil}},
	}
	info := analyzeBST(root)
	fmt.Printf("IsBST: %v\n", info.IsBST)   // true
	fmt.Printf("Size: %d\n", info.Size)       // 7
	fmt.Printf("Min: %d\n", info.Min)         // 1
	fmt.Printf("Max: %d\n", info.Max)         // 20
	fmt.Printf("Height: %d\n", info.Height)   // 3
}
```

**Textual Figure:**
```
Analyze All BST Properties in One Pass:

          10
         / \
        5   15
       / \  / \
      1  8 12  20

  analyzeBST(10):
  ├── Left subtree (rooted at 5):
  │   ├── Left(1):  IsBST=✓ Size=1 Min=1  Max=1  Height=1
  │   ├── Right(8): IsBST=✓ Size=1 Min=8  Max=8  Height=1
  │   └── Node(5):  5>1(leftMax) ✓, 5<8(rightMin) ✓
  │       IsBST=✓ Size=3 Min=1 Max=8 Height=2
  ├── Right subtree (rooted at 15):
  │   ├── Left(12): IsBST=✓ Size=1 Min=12 Max=12 Height=1
  │   ├── Right(20):IsBST=✓ Size=1 Min=20 Max=20 Height=1
  │   └── Node(15): 15>12 ✓, 15<20 ✓
  │       IsBST=✓ Size=3 Min=12 Max=20 Height=2
  └── Root(10): 10>8(leftMax) ✓, 10<12(rightMin) ✓
      IsBST=true  Size=7  Min=1  Max=20  Height=3
```

---

## Key Takeaways

| Property | Benefit |
|----------|---------|
| **Inorder = sorted** | Kth smallest, validate, convert to sorted list |
| **Search = O(h)** | Binary search on tree structure |
| **LCA = O(h)** | Just follow values, no full DFS needed |
| **Range queries** | Prune entire subtrees outside range |
| **Floor/Ceiling** | Track best candidate while traversing |

> **Next up:** BST Insertion →
