# Phase 8: Binary Trees — Morris Traversal

## Overview

**Morris traversal** achieves inorder/preorder tree traversal in **O(1) extra space** (no stack, no recursion) by temporarily modifying the tree using **threaded binary tree** technique.

**Core idea**: For each node, find its **inorder predecessor** (rightmost node in left subtree). Make predecessor's right pointer point back to current node (thread). Use threads to return to ancestor without a stack.

| Traversal | Time | Space | Modifies Tree? |
|-----------|------|-------|----------------|
| Recursive | O(n) | O(h) | No |
| Iterative (stack) | O(n) | O(h) | No |
| **Morris** | O(n) | O(1) | Temporarily (restored) |

---

## Example 1: Morris Inorder Traversal

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func morrisInorder(root *TreeNode) []int {
    result := []int{}
    cur := root

    for cur != nil {
        if cur.Left == nil {
            // No left subtree: visit and go right
            result = append(result, cur.Val)
            cur = cur.Right
        } else {
            // Find inorder predecessor
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                // Thread doesn't exist: create it
                pred.Right = cur
                cur = cur.Left
            } else {
                // Thread exists: we've returned, restore and visit
                pred.Right = nil
                result = append(result, cur.Val)
                cur = cur.Right
            }
        }
    }
    return result
}

func main() {
    //       4
    //      / \
    //     2   6
    //    / \ / \
    //   1  3 5  7
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(morrisInorder(root)) // [1 2 3 4 5 6 7]
}
```

### Step-by-Step Trace

```
cur=4: left exists → pred=3 → thread 3.Right=4 → go left
cur=2: left exists → pred=1 → thread 1.Right=2 → go left
cur=1: no left → visit 1 → go right (thread to 2)
cur=2: left exists → pred=1 → 1.Right==2 (thread!) → restore, visit 2 → go right
cur=3: no left → visit 3 → go right (thread to 4)
cur=4: left exists → pred=3 → 3.Right==4 (thread!) → restore, visit 4 → go right
cur=6: same pattern...
```

**Textual Figure – Example 1:**
```
 Morris Inorder — Threading & Visiting:
 ───────────────────────────────────
       4                     Step 1: Thread 3→4
      / \                           4
     2   6                         / \
    / \ / \                       2   6
   1  3 5  7                     / \
                                1   3──┐
 Step 2: Thread 1→2                  │ thread
       4                            ↓
      / \                    Step 3: Visit 1, follow
     2   6                   thread to 2
    / \
   1   3                 Step 4: Remove thread 1→2
   └─┐ └─┐               Visit 2, go right to 3
     ↓   ↓
  thread thread          Step 5: Visit 3, follow
                         thread to 4
 Result: [1, 2, 3, 4, 5, 6, 7]
 ───────────────────────────────────
 Threads created:  3→R→4, 1→R→2, 5→R→6
 Threads removed:  same (tree restored)
 Visit order:      1 → 2 → 3 → 4 → 5 → 6 → 7
```

---

## Example 2: Morris Preorder Traversal

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func morrisPreorder(root *TreeNode) []int {
    result := []int{}
    cur := root

    for cur != nil {
        if cur.Left == nil {
            result = append(result, cur.Val)
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                // First visit: print BEFORE going left (preorder)
                result = append(result, cur.Val)
                pred.Right = cur
                cur = cur.Left
            } else {
                // Returning: just restore and go right
                pred.Right = nil
                cur = cur.Right
            }
        }
    }
    return result
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(morrisPreorder(root)) // [4 2 1 3 6 5 7]
}
```

**Textual Figure – Example 2:**
```
 Morris Preorder vs Inorder — When to visit:
 ────────────────────────────────────
       4                  Preorder: visit BEFORE going left
      / \                 Inorder:  visit AFTER returning
     2   6
    / \ / \            Step-by-step:
   1  3 5  7           cur=4: create thread 3→4, VISIT 4, go left
                       cur=2: create thread 1→2, VISIT 2, go left
 ┌───────────────────────────────────┐
 │ Inorder:  visit at thread removal  │
 │ → [1, 2, 3, 4, 5, 6, 7]           │
 │                                    │
 │ Preorder: visit at thread creation  │
 │ → [4, 2, 1, 3, 6, 5, 7]           │
 └───────────────────────────────────┘
            4 (visit 1st)
           / \
    (2nd) 2   6 (5th)
         / \ / \
   (3rd)1  3 5  7 (4th,6th,7th)
```

**Key difference from inorder**: Visit the node when *creating* the thread (first time), not when *removing* it.

---

## Example 3: Morris Postorder Traversal

Postorder with Morris is trickier — requires reverse printing of the "right boundary" of left subtrees.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func morrisPostorder(root *TreeNode) []int {
    result := []int{}
    dummy := &TreeNode{0, root, nil}
    cur := dummy

    for cur != nil {
        if cur.Left == nil {
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                // Reverse and collect nodes from cur.Left to pred
                addReverse(cur.Left, pred, &result)
                pred.Right = nil
                cur = cur.Right
            }
        }
    }
    return result
}

func addReverse(from, to *TreeNode, result *[]int) {
    reverse(from, to)
    node := to
    for {
        *result = append(*result, node.Val)
        if node == from { break }
        node = node.Right
    }
    reverse(to, from)
}

func reverse(from, to *TreeNode) {
    if from == to { return }
    x := from
    y := from.Right
    for {
        z := y.Right
        y.Right = x
        x = y
        y = z
        if x == to { break }
    }
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }
    fmt.Println(morrisPostorder(root)) // [1 3 2 5 7 6 4]
}
```

**Textual Figure – Example 3:**
```
 Morris Postorder — Reverse right boundary technique:
 ──────────────────────────────────────
  dummy(0)               Uses a dummy node pointing
     \                   to root as its left child.
      4
     / \                 When thread is found (2nd visit):
    2   6                  → reverse-print path from
   / \ / \                   cur.Left to pred
  1  3 5  7

 Step-by-step:
 ┌─────┬─────────────────────────────────────┐
 │ cur │ Action                                │
 ├─────┼─────────────────────────────────────┤
 │  D  │ thread 3→D, go left to 4             │
 │  4  │ thread 3→4... go left to 2            │
 │  2  │ thread 1→2, go left to 1              │
 │  1  │ no left → go right(thread→2)          │
 │  2  │ thread found! reverse print 1 → [1]   │
 │  3  │ no left → go right(thread→4)          │
 │  4  │ thread found! reverse print 2→3 →[3,2]│
 │  6  │ thread 5→6, go left to 5              │
 │  5  │ no left → go right(thread→6)          │
 │  6  │ thread found! reverse print 5 → [5]   │
 │  7  │ no left → go right(thread→D)          │
 │  D  │ thread found! reverse 4→6→7 →[7,6,4]│
 └─────┴─────────────────────────────────────┘
 Result: [1] + [3,2] + [5] + [7,6,4] = [1,3,2,5,7,6,4]
```

---

## Example 4: Validate BST Using Morris (LeetCode 98)

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
    cur := root
    prev := math.MinInt64

    for cur != nil {
        if cur.Left == nil {
            if cur.Val <= prev { return false }
            prev = cur.Val
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                if cur.Val <= prev { return false }
                prev = cur.Val
                cur = cur.Right
            }
        }
    }
    return true
}

func main() {
    // Valid BST
    root := &TreeNode{2,
        &TreeNode{1, nil, nil},
        &TreeNode{3, nil, nil},
    }
    fmt.Println(isValidBST(root)) // true

    // Invalid BST
    root2 := &TreeNode{5,
        &TreeNode{1, nil, nil},
        &TreeNode{4, &TreeNode{3, nil, nil}, &TreeNode{6, nil, nil}},
    }
    fmt.Println(isValidBST(root2)) // false (4 < 5 but right child)
}
```

**Textual Figure – Example 4:**
```
 Morris BST Validation — Inorder must be strictly increasing:
 ────────────────────────────────────────────
 Valid BST:          Invalid BST:
    2                   5
   / \                 / \
  1   3               1   4     ← 4 < 5 but right child!
                         / \
                        3   6

 Morris inorder on valid BST:
  prev=-∞ → visit 1 (1>-∞ ✓) → visit 2 (2>1 ✓) → visit 3 (3>2 ✓)
  Result: true

 Morris inorder on invalid BST:
  prev=-∞ → visit 1 (1>-∞ ✓) → visit 3 (3>1 ✓)
         → visit 4 (4>3 ✓) → visit 5 (5>4 ✓)
         → visit 6 (6>5 ✓)
  Wait — inorder = [1,3,4,5,6] looks sorted?
  Actually inorder = [1,3,5,4,6]
                          ↑ ↑
                          5>4 is NOT sorted → false!
 ┌────────────────────────────────────┐
 │ cur.Val(4) <= prev.Val(5) → false │
 └────────────────────────────────────┘
```

---

## Example 5: Kth Smallest in BST Using Morris (LeetCode 230)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func kthSmallest(root *TreeNode, k int) int {
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
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                count++
                if count == k { return cur.Val }
                cur = cur.Right
            }
        }
    }
    return -1
}

func main() {
    //       5
    //      / \
    //     3   6
    //    / \
    //   2   4
    //  /
    // 1
    root := &TreeNode{5,
        &TreeNode{3,
            &TreeNode{2, &TreeNode{1, nil, nil}, nil},
            &TreeNode{4, nil, nil},
        },
        &TreeNode{6, nil, nil},
    }
    for k := 1; k <= 6; k++ {
        fmt.Printf("k=%d → %d\n", k, kthSmallest(root, k))
    }
}
```

**Textual Figure – Example 5:**
```
 Morris Kth Smallest — Count during inorder:
 ───────────────────────────────────
       5
      / \
     3   6                 Inorder: 1, 2, 3, 4, 5, 6
    / \
   2   4                   ┌────┬────────┬─────────┐
  /                        │  k │ count  │ kth val │
 1                         ├────┼────────┼─────────┤
                           │  1 │  1✓    │    1    │
 Morris inorder visits:    │  2 │  2✓    │    2    │
  1(cnt=1) → 2(cnt=2)      │  3 │  3✓    │    3    │
  → 3(cnt=3) → 4(cnt=4)   │  4 │  4✓    │    4    │
  → 5(cnt=5) → 6(cnt=6)   │  5 │  5✓    │    5    │
                           │  6 │  6✓    │    6    │
 Stop when count==k         └────┴────────┴─────────┘
 O(1) space, O(k) average time to reach kth element
```

---

## Example 6: Recover BST Using Morris (LeetCode 99)

Two nodes are swapped in a BST. Fix it using O(1) space.

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

func recoverTree(root *TreeNode) {
    var first, second, prev *TreeNode
    prev = &TreeNode{Val: math.MinInt64}
    cur := root

    for cur != nil {
        if cur.Left == nil {
            // Visit
            if cur.Val < prev.Val {
                if first == nil { first = prev }
                second = cur
            }
            prev = cur
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                // Visit
                if cur.Val < prev.Val {
                    if first == nil { first = prev }
                    second = cur
                }
                prev = cur
                cur = cur.Right
            }
        }
    }

    first.Val, second.Val = second.Val, first.Val
}

func inorder(root *TreeNode) {
    if root == nil { return }
    inorder(root.Left)
    fmt.Printf("%d ", root.Val)
    inorder(root.Right)
}

func main() {
    // BST: [1,2,3,4,5] with 2 and 4 swapped → [1,4,3,2,5]
    root := &TreeNode{3,
        &TreeNode{4, &TreeNode{1, nil, nil}, nil}, // 4 should be 2
        &TreeNode{2, nil, &TreeNode{5, nil, nil}},  // 2 should be 4
    }
    fmt.Print("Before: "); inorder(root); fmt.Println()
    recoverTree(root)
    fmt.Print("After:  "); inorder(root); fmt.Println()
}
```

**Textual Figure – Example 6:**
```
 Morris Recover BST — Find two swapped nodes:
 ───────────────────────────────────────
      3                Correct BST inorder:  [1,2,3,4,5]
     / \               Swapped BST inorder:  [1,4,3,2,5]
    4   2                                       ↑   ↑
   /     \             nodes 2 and 4 are swapped
  1       5

 Morris inorder traversal detects violations:
 ┌──────┬──────┬────────────────────────────┐
 │ prev │ cur  │ prev > cur?                  │
 ├──────┼──────┼────────────────────────────┤
 │  -∞  │  1   │ No                           │
 │  1   │  4   │ No                           │
 │  4   │  3   │ YES → first=4, second=3      │
 │  3   │  2   │ YES → second=2 (update)      │
 │  2   │  5   │ No                           │
 └──────┴──────┴────────────────────────────┘
 Swap first(4) ↔ second(2) → tree restored
 After: [1, 2, 3, 4, 5] ✓
```

---

## Example 7: Count BST Nodes in Range Using Morris

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func rangeSumBST(root *TreeNode, low, high int) int {
    sum := 0
    cur := root

    for cur != nil {
        if cur.Left == nil {
            if cur.Val >= low && cur.Val <= high {
                sum += cur.Val
            }
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                if cur.Val >= low && cur.Val <= high {
                    sum += cur.Val
                }
                cur = cur.Right
            }
        }
    }
    return sum
}

func main() {
    root := &TreeNode{10,
        &TreeNode{5, &TreeNode{3, nil, nil}, &TreeNode{7, nil, nil}},
        &TreeNode{15, nil, &TreeNode{18, nil, nil}},
    }
    fmt.Println(rangeSumBST(root, 7, 15)) // 32 (7 + 10 + 15)
}
```

**Textual Figure – Example 7:**
```
 Morris Range Sum — Sum values in [low, high]:
 ─────────────────────────────────────
       10
      /  \            low=7, high=15
     5   15
    / \    \
   3   7   18

 Morris inorder: 3, 5, 7, 10, 15, 18
 ┌───────┬───────────┬───────────┐
 │ Visit │ In range? │ sum       │
 ├───────┼───────────┼───────────┤
 │   3   │  3<7  No  │  0        │
 │   5   │  5<7  No  │  0        │
 │   7   │  7≥7 Yes  │  0+7=7    │
 │  10   │ 10≤15 Yes │  7+10=17  │
 │  15   │ 15≤15 Yes │  17+15=32 │
 │  18   │ 18>15 No  │  32       │
 └───────┴───────────┴───────────┘
 Result: 32 (7 + 10 + 15)
```

---

## Example 8: Flatten BST to Sorted List Using Morris

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Convert BST to sorted linked list (right pointers) using O(1) space
func flattenBSTToList(root *TreeNode) *TreeNode {
    dummy := &TreeNode{}
    tail := dummy
    cur := root

    for cur != nil {
        if cur.Left == nil {
            tail.Right = cur
            tail = cur
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }

            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                tail.Right = cur
                tail = cur
                next := cur.Right
                cur.Left = nil
                cur = next
            }
        }
    }
    tail.Right = nil
    return dummy.Right
}

func main() {
    root := &TreeNode{4,
        &TreeNode{2, &TreeNode{1, nil, nil}, &TreeNode{3, nil, nil}},
        &TreeNode{6, &TreeNode{5, nil, nil}, &TreeNode{7, nil, nil}},
    }

    head := flattenBSTToList(root)
    for head != nil {
        fmt.Printf("%d ", head.Val)
        head = head.Right
    }
    fmt.Println() // 1 2 3 4 5 6 7
}
```

**Textual Figure – Example 8:**
```
 Morris Flatten BST to Sorted Linked List:
 ─────────────────────────────────────
  Input BST:              Output linked list:
       4                  (via right pointers)
      / \
     2   6               dummy → 1 → 2 → 3 → 4 → 5 → 6 → 7 → nil
    / \ / \
   1  3 5  7              Each node in inorder
                          gets appended to tail.
 Process during Morris inorder:
 ┌───────┬──────────────────────────────┐
 │ Visit │ tail.Right = cur, tail = cur │
 ├───────┼──────────────────────────────┤
 │   1   │ dummy→R=1, tail=1             │
 │   2   │ 1→R=2, tail=2, Left=nil       │
 │   3   │ 2→R=3, tail=3                 │
 │   4   │ 3→R=4, tail=4                 │
 │   5   │ 4→R=5, tail=5                 │
 │   6   │ 5→R=6, tail=6                 │
 │   7   │ 6→R=7, tail=7, 7→R=nil       │
 └───────┴──────────────────────────────┘
 Result: 1 → 2 → 3 → 4 → 5 → 6 → 7
```

---

## Example 9: Morris Traversal to Find Mode in BST (LeetCode 501)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func findMode(root *TreeNode) []int {
    var modes []int
    maxCount, curCount, curVal := 0, 0, 0
    first := true

    process := func(val int) {
        if first || val != curVal {
            curVal = val
            curCount = 1
            first = false
        } else {
            curCount++
        }

        if curCount > maxCount {
            maxCount = curCount
            modes = []int{curVal}
        } else if curCount == maxCount {
            modes = append(modes, curVal)
        }
    }

    // Morris inorder
    cur := root
    for cur != nil {
        if cur.Left == nil {
            process(cur.Val)
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }
            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                process(cur.Val)
                cur = cur.Right
            }
        }
    }
    return modes
}

func main() {
    //   1
    //    \
    //     2
    //    /
    //   2
    root := &TreeNode{1, nil, &TreeNode{2, &TreeNode{2, nil, nil}, nil}}
    fmt.Println(findMode(root)) // [2]
}
```

**Textual Figure – Example 9:**
```
 Morris Find Mode in BST:
 ──────────────────────
   1                   Morris inorder: [1, 2, 2]
    \
     2                 ┌───────┬────────┬──────────┬──────┐
    /                  │ visit │ curCnt │ maxCount │ modes│
   2                   ├───────┼────────┼──────────┼──────┤
                       │   1   │   1    │    1     │ [1]  │
                       │   2   │   1    │    1     │[1,2] │
                       │   2   │   2    │    2     │ [2]  │
                       └───────┴────────┴──────────┴──────┘

 curCount tracks consecutive equal values.
 When curCount > maxCount → new mode found, reset list.
 When curCount == maxCount → add to modes list.
 Result: [2] (appears 2 times, most frequent)
```

---

## Example 10: Morris vs Stack vs Recursive – Benchmark

```go
package main

import (
    "fmt"
    "time"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Build a balanced BST of n nodes
func buildBST(low, high int) *TreeNode {
    if low > high { return nil }
    mid := (low + high) / 2
    return &TreeNode{mid, buildBST(low, mid-1), buildBST(mid+1, high)}
}

func morrisInorder(root *TreeNode) int {
    count := 0
    cur := root
    for cur != nil {
        if cur.Left == nil {
            count++
            cur = cur.Right
        } else {
            pred := cur.Left
            for pred.Right != nil && pred.Right != cur {
                pred = pred.Right
            }
            if pred.Right == nil {
                pred.Right = cur
                cur = cur.Left
            } else {
                pred.Right = nil
                count++
                cur = cur.Right
            }
        }
    }
    return count
}

func stackInorder(root *TreeNode) int {
    count := 0
    stack := []*TreeNode{}
    cur := root
    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        count++
        cur = cur.Right
    }
    return count
}

func recursiveInorder(root *TreeNode, count *int) {
    if root == nil { return }
    recursiveInorder(root.Left, count)
    *count++
    recursiveInorder(root.Right, count)
}

func main() {
    n := 1000000
    root := buildBST(1, n)

    start := time.Now()
    c1 := morrisInorder(root)
    t1 := time.Since(start)

    start = time.Now()
    c2 := stackInorder(root)
    t2 := time.Since(start)

    start = time.Now()
    c3 := 0
    recursiveInorder(root, &c3)
    t3 := time.Since(start)

    fmt.Printf("Morris:    %d nodes, %v\n", c1, t1)
    fmt.Printf("Stack:     %d nodes, %v\n", c2, t2)
    fmt.Printf("Recursive: %d nodes, %v\n", c3, t3)
}
```

**Textual Figure – Example 10:**
```
 Morris vs Stack vs Recursive — Comparison:
 ──────────────────────────────────────────

 ┌────────────┬────────┬────────┬──────────────┐
 │ Method     │ Time   │ Space  │ Modifies Tree │
 ├────────────┼────────┼────────┼──────────────┤
 │ Morris     │ O(n)   │ O(1)   │ Yes (temp)    │
 │ Stack      │ O(n)   │ O(h)   │ No            │
 │ Recursive  │ O(n)   │ O(h)   │ No            │
 └────────────┴────────┴────────┴──────────────┘

 For n=1,000,000 balanced BST (h ≈ 20):

 Memory usage:
  Morris:    █ (constant)
  Stack:     █████ (~20 pointers on stack)
  Recursive: █████ (~20 frames on call stack)

 Morris wins in space-constrained environments.
 All three: O(n) time, same visit count.

 Key trade-off:
  ┌───────────────────────────────────┐
  │ O(1) space ↔ temporary tree mods │
  └───────────────────────────────────┘
```

---

## Morris Traversal Algorithm Summary

```
MORRIS-INORDER(root):
    cur = root
    while cur != nil:
        if cur.Left == nil:
            VISIT(cur)
            cur = cur.Right
        else:
            pred = inorderPredecessor(cur)
            if pred.Right == nil:         // First visit
                pred.Right = cur          // Create thread
                cur = cur.Left            // Go left
            else:                         // Second visit
                pred.Right = nil          // Remove thread
                VISIT(cur)                // Visit now
                cur = cur.Right           // Go right

MORRIS-PREORDER: Same but VISIT at "First visit" instead of "Second visit"
```

## Key Takeaways

1. **O(1) space** — no stack or recursion needed
2. Each edge is traversed at most 3 times → still **O(n)** time
3. **Thread creation** = finding inorder predecessor = rightmost node in left subtree
4. Tree is temporarily modified but fully restored when traversal completes
5. Useful when space is critical: embedded systems, memory-constrained environments
6. Works for any problem solvable by inorder/preorder: BST validation, kth element, mode finding

> **Next up:** Serialization and Deserialization →
