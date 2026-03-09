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
