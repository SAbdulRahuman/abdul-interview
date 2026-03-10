# Phase 9: BST — BST Search

## Overview

Searching in a BST leverages the ordering property: at each node, go **left** if the target is smaller, **right** if larger. Average O(log n), worst O(n) for skewed trees.

---

## Example 1: Recursive Search (LeetCode 700)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func searchBST(root *TreeNode, val int) *TreeNode {
	if root == nil || root.Val == val {
		return root
	}
	if val < root.Val {
		return searchBST(root.Left, val)
	}
	return searchBST(root.Right, val)
}

func main() {
	root := &TreeNode{4,
		&TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
		&TreeNode{7, nil, nil},
	}
	node := searchBST(root, 2)
	if node != nil {
		fmt.Println("Found:", node.Val) // Found: 2
	}
	node = searchBST(root, 5)
	fmt.Println("Found:", node) // Found: <nil>
}
```

**Textual Figure:**
```
Recursive BST Search:

       4
      / \
     2   7
    / \
   1   3

  Search for 2:                    Search for 5:
  ┌─────────────────────┐      ┌─────────────────────┐
  │ root=4, 2 < 4       │      │ root=4, 5 > 4       │
  │  → go Left          │      │  → go Right         │
  │ root=2, 2 == 2      │      │ root=7, 5 < 7       │
  │  → FOUND! return 2  │      │  → go Left          │
  │                     │      │ root=nil             │
  │ Output: Found: 2    │      │  → NOT FOUND (nil)   │
  └─────────────────────┘      └─────────────────────┘
```

---

## Example 2: Iterative Search

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func searchIterative(root *TreeNode, val int) *TreeNode {
	for root != nil && root.Val != val {
		if val < root.Val {
			root = root.Left
		} else {
			root = root.Right
		}
	}
	return root
}

func main() {
	root := &TreeNode{10,
		&TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
		&TreeNode{15, &TreeNode{12, nil, nil}, &TreeNode{20, nil, nil}},
	}
	for _, v := range []int{7, 12, 99} {
		node := searchIterative(root, v)
		if node != nil {
			fmt.Printf("Search %d: found\n", v)
		} else {
			fmt.Printf("Search %d: not found\n", v)
		}
	}
}
```

**Textual Figure:**
```
Iterative BST Search:

          10
         / \
        5   15
       / \  / \
      3  7 12  20

  Search 7:              Search 12:            Search 99:
  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
  │ cur=10       │    │ cur=10       │    │ cur=10       │
  │  7<10 → left │    │ 12>10→right │    │ 99>10→right │
  │ cur=5        │    │ cur=15       │    │ cur=15       │
  │  7>5  →right │    │ 12<15→ left │    │ 99>15→right │
  │ cur=7        │    │ cur=12       │    │ cur=20       │
  │  7==7 FOUND  │    │ 12==12 FOUND │    │ 99>20→right │
  └───────────────┘    └───────────────┘    │ cur=nil      │
                                      │  NOT FOUND   │
                                      └───────────────┘
```

---

## Example 3: Search with Path Tracking

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func searchWithPath(root *TreeNode, val int) (bool, []int) {
	path := []int{}
	cur := root
	for cur != nil {
		path = append(path, cur.Val)
		if val == cur.Val {
			return true, path
		}
		if val < cur.Val {
			cur = cur.Left
		} else {
			cur = cur.Right
		}
	}
	return false, path
}

func main() {
	root := &TreeNode{50,
		&TreeNode{30, &TreeNode{20, nil, nil}, &TreeNode{40, nil, nil}},
		&TreeNode{70, &TreeNode{60, nil, nil}, &TreeNode{80, nil, nil}},
	}
	found, path := searchWithPath(root, 40)
	fmt.Println(found, path) // true [50 30 40]

	found, path = searchWithPath(root, 55)
	fmt.Println(found, path) // false [50 70 60]
}
```

**Textual Figure:**
```
Search with Path Tracking:

          50
         / \
        30   70
       / \   / \
      20 40 60  80

  Search 40:                       Search 55:
  ┌────────────────────────┐     ┌────────────────────────┐
  │ cur=50 → path=[50]     │     │ cur=50 → path=[50]     │
  │   40<50 → go left     │     │   55>50 → go right    │
  │ cur=30 → path=[50,30]  │     │ cur=70 → path=[50,70]  │
  │   40>30 → go right    │     │   55<70 → go left     │
  │ cur=40 → path=[50,30,40]│    │ cur=60 → path=[50,70,60]│
  │   40==40 → FOUND!     │     │   55<60 → go left     │
  │                        │     │ cur=nil → NOT FOUND   │
  └────────────────────────┘     └────────────────────────┘
  Output: true [50 30 40]        Output: false [50 70 60]
```

---

## Example 4: Floor and Ceiling in BST

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// Floor: largest value <= target
func floor(root *TreeNode, target int) int {
	result := -1
	cur := root
	for cur != nil {
		if cur.Val == target {
			return cur.Val
		}
		if cur.Val < target {
			result = cur.Val
			cur = cur.Right
		} else {
			cur = cur.Left
		}
	}
	return result
}

// Ceiling: smallest value >= target
func ceiling(root *TreeNode, target int) int {
	result := -1
	cur := root
	for cur != nil {
		if cur.Val == target {
			return cur.Val
		}
		if cur.Val > target {
			result = cur.Val
			cur = cur.Left
		} else {
			cur = cur.Right
		}
	}
	return result
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{10, 5, 15, 3, 7, 12, 20} {
		root = insert(root, v)
	}
	fmt.Println("Floor(6):", floor(root, 6))     // 5
	fmt.Println("Floor(10):", floor(root, 10))    // 10
	fmt.Println("Ceiling(6):", ceiling(root, 6))  // 7
	fmt.Println("Ceiling(13):", ceiling(root, 13))// 15
}
```

**Textual Figure:**
```
Floor & Ceiling Search in BST:

          10
         / \
        5   15
       / \  / \
      3  7 12  20

  ┌── Floor(6): largest value ≤ 6 ───────────────┐
  │  cur=10: 10>6 → go left                    │
  │  cur=5:  5<6  → result=5, go right         │
  │  cur=7:  7>6  → go left                    │
  │  cur=nil → return result = 5                │
  ├── Ceiling(6): smallest value ≥ 6 ──────────┤
  │  cur=10: 10>6 → result=10, go left          │
  │  cur=5:  5<6  → go right                   │
  │  cur=7:  7>6  → result=7, go left           │
  │  cur=nil → return result = 7                │
  ├── Floor(10): exact match ─────────────────┤
  │  cur=10: 10==10 → return 10                 │
  ├── Ceiling(13): ───────────────────────────┤
  │  cur=10: 10<13 → go right                   │
  │  cur=15: 15>13 → result=15, go left         │
  │  cur=12: 12<13 → go right                   │
  │  cur=nil → return result = 15               │
  └─────────────────────────────────────────────┘
```

---

## Example 5: Closest Value in BST (LeetCode 270)

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

func closestValue(root *TreeNode, target float64) int {
	closest := root.Val
	cur := root
	for cur != nil {
		if math.Abs(float64(cur.Val)-target) < math.Abs(float64(closest)-target) {
			closest = cur.Val
		}
		if target < float64(cur.Val) {
			cur = cur.Left
		} else {
			cur = cur.Right
		}
	}
	return closest
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{10, 5, 15, 3, 7, 12, 20} {
		root = insert(root, v)
	}
	fmt.Println("Closest to 6.3:", closestValue(root, 6.3))  // 7
	fmt.Println("Closest to 13.0:", closestValue(root, 13.0)) // 12
	fmt.Println("Closest to 10.0:", closestValue(root, 10.0)) // 10
}
```

**Textual Figure:**
```
Closest Value in BST:

          10
         / \
        5   15
       / \  / \
      3  7 12  20

  Closest to 6.3:                    Closest to 13.0:
  ┌─────────────────────────┐     ┌─────────────────────────┐
  │ cur=10 |10-6.3|=3.7     │     │ cur=10 |10-13|=3      │
  │   closest=10             │     │   closest=10           │
  │   6.3<10 → go left      │     │   13>10 → go right    │
  │ cur=5  |5-6.3|=1.3      │     │ cur=15 |15-13|=2      │
  │   closest=5 (closer)    │     │   closest=15 (closer)  │
  │   6.3>5 → go right     │     │   13<15 → go left     │
  │ cur=7  |7-6.3|=0.7      │     │ cur=12 |12-13|=1      │
  │   closest=7 (closer!)   │     │   closest=12 (closer!) │
  │   6.3<7 → go left      │     │   13>12 → go right   │
  │ cur=nil → done          │     │ cur=nil → done        │
  │ Answer: 7               │     │ Answer: 12             │
  └─────────────────────────┘     └─────────────────────────┘
```

---

## Example 6: Range Search — Find All Values in [lo, hi]

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func rangeSearch(root *TreeNode, lo, hi int) []int {
	result := []int{}
	var dfs func(node *TreeNode)
	dfs = func(node *TreeNode) {
		if node == nil { return }
		if node.Val > lo {
			dfs(node.Left) // might have values in range
		}
		if node.Val >= lo && node.Val <= hi {
			result = append(result, node.Val)
		}
		if node.Val < hi {
			dfs(node.Right)
		}
	}
	dfs(root)
	return result
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{10, 5, 15, 3, 7, 12, 20, 1, 4, 6, 8} {
		root = insert(root, v)
	}
	fmt.Println("Range [4,12]:", rangeSearch(root, 4, 12))
	// [4 5 6 7 8 10 12]
}
```

**Textual Figure:**
```
Range Search [4, 12] — Collect all values in range:

            10
           / \
          5   15
         / \  / \
        3  7 12  20
       /\ /\
      1 4 6 8

  dfs(10):
  ├── 10>4 → search left subtree
  │   dfs(5):
  │   ├── 5>4 → search left
  │   │   dfs(3): 3<4 → skip left, search right
  │   │     dfs(4): 4∈[4,12] → add 4, leaf
  │   ├── 5∈[4,12] → add 5
  │   └── 5<12 → search right
  │       dfs(7): add 7; left→6 add 6; right→8 add 8
  ├── 10∈[4,12] → add 10
  └── 10<12 → search right
      dfs(15): 15>12 → search left only
        dfs(12): 12∈[4,12] → add 12

  Result: [4 5 6 7 8 10 12]  (inorder within range)
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

func countInRange(root *TreeNode, lo, hi int) int {
	if root == nil { return 0 }
	if root.Val < lo {
		return countInRange(root.Right, lo, hi)
	}
	if root.Val > hi {
		return countInRange(root.Left, lo, hi)
	}
	// root.Val in [lo, hi]
	return 1 + countInRange(root.Left, lo, hi) + countInRange(root.Right, lo, hi)
}

func insert(root *TreeNode, val int) *TreeNode {
	if root == nil { return &TreeNode{Val: val} }
	if val < root.Val { root.Left = insert(root.Left, val) } else { root.Right = insert(root.Right, val) }
	return root
}

func main() {
	var root *TreeNode
	for _, v := range []int{10, 5, 15, 3, 7, 12, 20, 1, 4, 6, 8} {
		root = insert(root, v)
	}
	fmt.Println("Count in [4,12]:", countInRange(root, 4, 12)) // 7
	fmt.Println("Count in [1,5]:", countInRange(root, 1, 5))   // 4
}
```

**Textual Figure:**
```
Count Nodes in Range:

            10
           / \
          5   15
         / \  / \
        3  7 12  20
       /\ /\
      1 4 6 8

  countInRange(root, 4, 12):            countInRange(root, 1, 5):
  ┌─────────────────────────┐      ┌─────────────────────────┐
  │ 10∈[4,12]: count 1      │      │ 10>5: prune right,    │
  │  L: 5∈[4,12]: count 1  │      │   search left only    │
  │    L: 3<4: prune left   │      │ 5∈[1,5]: count 1      │
  │      R: 4∈[4,12]: +1   │      │  L: 3∈[1,5]: count 1  │
  │    R: 7∈[4,12]: count 1│      │    L: 1∈[1,5]: +1     │
  │      L: 6∈[4,12]: +1   │      │    R: 4∈[1,5]: +1     │
  │      R: 8∈[4,12]: +1   │      │  R: 7>5: prune right  │
  │  R: 15>12: prune right  │      │                        │
  │    L: 12∈[4,12]: +1    │      │ Total: 4              │
  │ Total: 7               │      │ {1, 3, 4, 5}          │
  └─────────────────────────┘      └─────────────────────────┘
```

---

## Example 8: Kth Smallest Element (LeetCode 230)

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
	count := 0
	for cur != nil || len(stack) > 0 {
		for cur != nil {
			stack = append(stack, cur)
			cur = cur.Left
		}
		cur = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		count++
		if count == k {
			return cur.Val
		}
		cur = cur.Right
	}
	return -1
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
Kth Smallest via Iterative Inorder (stack-based):

         5
        / \
       3   6
      / \
     2   4
    /
   1

  Stack simulation:
  Step  Action              Stack (top→right)  Pop   count
  ─────────────────────────────────────────────────────────
   1    push 5,3,2,1        [5,3,2,1]          ─      ─
   2    pop 1               [5,3,2]            1      1 → k=1? → 1
   3    pop 2               [5,3]              2      2 → k=2? → 2
   4    pop 3, push 4       [5,4]              3      3 → k=3? → 3
   5    pop 4               [5]                4      4 → k=4? → 4
   6    pop 5, push 6       [6]                5      5 → k=5? → 5
   7    pop 6               []                 6      6 → k=6? → 6
```

---

## Example 9: Two Sum in BST (LeetCode 653)

```go
package main

import "fmt"

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func findTarget(root *TreeNode, k int) bool {
	seen := map[int]bool{}
	var dfs func(node *TreeNode) bool
	dfs = func(node *TreeNode) bool {
		if node == nil { return false }
		if seen[k-node.Val] { return true }
		seen[node.Val] = true
		return dfs(node.Left) || dfs(node.Right)
	}
	return dfs(root)
}

func main() {
	root := &TreeNode{5,
		&TreeNode{3, &TreeNode{2, nil, nil}, &TreeNode{4, nil, nil}},
		&TreeNode{6, nil, &TreeNode{7, nil, nil}},
	}
	fmt.Println("Two sum k=9:", findTarget(root, 9))  // true (2+7)
	fmt.Println("Two sum k=28:", findTarget(root, 28)) // false
}
```

**Textual Figure:**
```
Two Sum in BST (DFS + hash set):

        5
       / \
      3   6
     / \   \
    2   4   7

  Target k=9:
  Step  Visit  Need(k-val)  Seen        Found?
  ──────────────────────────────────────────
   1     5      4          {}          no
   2     3      6          {5}         no
   3     2      7          {5,3}       no
   4     4      5          {5,3,2}     5∈seen? YES!

  → 2 + 7 = 9? Actually 4+5=9 ✓
  Output: true

  Target k=28:
  Visit all nodes: {2,3,4,5,6,7}
  No pair sums to 28 → false
```

---

## Example 10: BST Iterator (LeetCode 173)

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
	top := it.stack[len(it.stack)-1]
	it.stack = it.stack[:len(it.stack)-1]
	it.pushLeft(top.Right)
	return top.Val
}

func (it *BSTIterator) HasNext() bool {
	return len(it.stack) > 0
}

func main() {
	root := &TreeNode{7,
		&TreeNode{3, nil, nil},
		&TreeNode{15, &TreeNode{9, nil, nil}, &TreeNode{20, nil, nil}},
	}
	it := NewBSTIterator(root)
	for it.HasNext() {
		fmt.Printf("%d ", it.Next())
	}
	fmt.Println() // 3 7 9 15 20
}
```

**Textual Figure:**
```
BST Iterator (controlled inorder via stack):

       7
      / \
     3   15
        / \
       9   20

  Initialize: pushLeft(7) → stack = [7, 3]

  Call     Stack Before    Pop   pushLeft(right)  Stack After  Output
  ───────────────────────────────────────────────────────────────
  Next()   [7, 3]           3     pushLeft(nil)    [7]           3
  Next()   [7]              7     pushLeft(15)     [15, 9]       7
  Next()   [15, 9]          9     pushLeft(nil)    [15]          9
  Next()   [15]            15     pushLeft(20)     [20]         15
  Next()   [20]            20     pushLeft(nil)    []           20
  HasNext? [] → false

  Output: 3 7 9 15 20 (sorted inorder)
```

---

## Key Takeaways

1. **Iterative search** is preferred — O(1) space, O(h) time
2. **Floor/ceiling** follow the search path while tracking the best candidate
3. **Range search** prunes branches that can't contain in-range values
4. **BST iterator** uses a controlled stack — O(h) space, amortized O(1) per `Next()`
5. Searching is the foundation for all BST operations (insert, delete, rank queries)

> **Next up:** Balanced BST →
