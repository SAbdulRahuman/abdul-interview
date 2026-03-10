# Phase 1: Algorithm Complexity — Space Complexity

## What is Space Complexity?

Space complexity measures **how much extra memory** an algorithm uses as the input size (`n`) grows. Like time complexity, we express it using **Big O notation**.

**Important:** We typically measure **auxiliary space** — the extra space used **beyond the input itself**.

```
Total Space = Input Space + Auxiliary Space
Space Complexity usually refers to → Auxiliary Space
```

---

## Why Does Space Complexity Matter?

- Memory is **finite** — your program can run out of it
- In interviews, you're often asked: *"Can you solve it in-place?"* (O(1) space)
- There's frequently a **time-space tradeoff** — you can use more memory to run faster, or use less memory but run slower
- Recursive solutions consume **stack space** — this counts!

---

## Space Complexity Cheat Sheet

| Big O | Name | Example |
|-------|------|---------|
| O(1) | Constant | Swap two variables |
| O(log n) | Logarithmic | Recursive binary search (call stack) |
| O(n) | Linear | Creating a copy of an array |
| O(n²) | Quadratic | 2D matrix (n×n) |
| O(2ⁿ) | Exponential | All subsets stored in memory |

---

## Example 1: O(1) — Constant Space

No extra memory is allocated regardless of input size.

```go
package main

import "fmt"

// O(1) space — only uses a fixed number of variables
// The variables total, i do NOT grow with input size
func sum(nums []int) int {
    total := 0                    // 1 variable
    for i := 0; i < len(nums); i++ { // 1 variable (i)
        total += nums[i]
    }
    return total
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    fmt.Println(sum(nums)) // 55
}
```

**Why O(1)?** — We only use `total` and `i` — two variables. Whether the slice has 10 elements or 10 million, we still use the same 2 variables. The input slice itself doesn't count as auxiliary space.

```
  Memory Layout:

  INPUT (not counted as auxiliary):
  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ 8  │ 9  │ 10 │  ← nums (given)
  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

  AUXILIARY (this is what we count):
  ┌──────────┐  ┌──────┐
  │ total=55 │  │ i=10 │   ← always exactly 2 variables
  └──────────┘  └──────┘

  n=10       → 2 variables (O(1))
  n=1,000    → 2 variables (O(1))
  n=1,000,000 → 2 variables (O(1))  ← same!
```

---

## Example 2: O(1) — In-Place Swap / Reverse

```go
package main

import "fmt"

// O(1) space — reverses the array IN-PLACE
// No new array is created; we only swap elements
func reverseInPlace(nums []int) {
    left, right := 0, len(nums)-1  // 2 variables

    for left < right {
        nums[left], nums[right] = nums[right], nums[left] // swap in-place
        left++
        right--
    }
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    reverseInPlace(nums)
    fmt.Println(nums) // [5 4 3 2 1]
}
```

**Why O(1)?** — We modify the original array directly. No matter how big the array is, we only ever use `left` and `right` — constant extra space.

```
  In-Place Reverse of [1, 2, 3, 4, 5]:

  Step 1: left=0, right=4 → swap 1↔5
  ┌───┬───┬───┬───┬───┐
  │ 5 │ 2 │ 3 │ 4 │ 1 │
  └───┴───┴───┴───┴───┘
    ↑               ↑
   left           right

  Step 2: left=1, right=3 → swap 2↔4
  ┌───┬───┬───┬───┬───┐
  │ 5 │ 4 │ 3 │ 2 │ 1 │
  └───┴───┴───┴───┴───┘
        ↑       ↑
       left   right

  Step 3: left=2, right=2 → left ≥ right, DONE
  ┌───┬───┬───┬───┬───┐
  │ 5 │ 4 │ 3 │ 2 │ 1 │  ← reversed, no extra array!
  └───┴───┴───┴───┴───┘

  Extra memory: only 2 index variables = O(1)
```

**Key insight:** "In-place" algorithms are O(1) space because they don't create new data structures.

---

## Example 3: O(n) — Creating a New Array

```go
package main

import "fmt"

// O(n) space — creates a new slice of the same size
func doubleValues(nums []int) []int {
    result := make([]int, len(nums)) // allocates n elements ← O(n)

    for i, v := range nums {
        result[i] = v * 2
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println(doubleValues(nums)) // [2 4 6 8 10]
}
```

**Why O(n)?** — We allocate a new slice `result` with `n` elements. If `n` doubles, the memory used doubles.

```
  Memory: O(1) in-place  vs  O(n) new array

  ┌─────────── O(n) approach ───────────┐
  │                                     │
  │  nums:    [1] [2] [3] [4] [5]       │  ← input
  │  result:  [2] [4] [6] [8] [10]      │  ← NEW array (O(n) extra)
  │                                     │
  └─────────────────────────────────────┘

  ┌─────────── O(1) approach ───────────┐
  │                                     │
  │  nums:    [2] [4] [6] [8] [10]      │  ← modified in place
  │  (no extra array!)                  │
  │                                     │
  └─────────────────────────────────────┘
```

**Compare with in-place version (O(1) space):**

```go
// O(1) space — modifies original slice
func doubleInPlace(nums []int) {
    for i := range nums {
        nums[i] *= 2
    }
}
```

---

## Example 4: O(n) — Hash Map / Frequency Counter

```go
package main

import "fmt"

// O(n) space — hash map can store up to n entries
func firstUnique(s string) byte {
    freq := make(map[byte]int) // worst case: n unique characters → O(n)

    for i := 0; i < len(s); i++ {
        freq[s[i]]++
    }

    for i := 0; i < len(s); i++ {
        if freq[s[i]] == 1 {
            return s[i]
        }
    }
    return 0
}

func main() {
    fmt.Printf("%c\n", firstUnique("leetcode"))  // l
    fmt.Printf("%c\n", firstUnique("aabb"))       // (null)
}
```

**Why O(n)?** — The map `freq` could store up to `n` unique keys in the worst case (when all characters are different). The map grows proportionally to the input.

```
  Hash Map growth for "leetcode":

  Input: l  e  e  t  c  o  d  e

  Map after processing:
  ┌─────┬───────┐
  │ key │ count │
  ├─────┼───────┤
  │  l  │   1   │  ← first unique!
  │  e  │   3   │
  │  t  │   1   │
  │  c  │   1   │
  │  o  │   1   │
  │  d  │   1   │
  └─────┴───────┘
  Map size = 6 unique chars

  Worst case: all different → map size = n
  Best case:  all same     → map size = 1
  Bounded input (a-z only) → map size ≤ 26 → O(1)!
```

> **Note:** If the input is limited (e.g., only lowercase English letters), the map is bounded at 26 entries → O(1) space. **Always clarify constraints!**

---

## Example 5: O(n) — Call Stack in Recursion

```go
package main

import "fmt"

// O(n) space — due to the call stack
// Each recursive call adds a frame to the stack
func factorial(n int) int {
    if n <= 1 {        // base case
        return 1
    }
    return n * factorial(n-1) // recursive call → stack frame
}

func main() {
    fmt.Println(factorial(5))  // 120
    fmt.Println(factorial(10)) // 3628800
}
```

**Why O(n)?** — Each call to `factorial` waits for the next one to return, stacking frames:

```
  Call Stack for factorial(5):

  ┌─────────────────────────────────┐
  │ factorial(1)  → returns 1       │  ← top of stack (base case)
  ├─────────────────────────────────┤
  │ factorial(2)  → 2 * factorial(1)│
  ├─────────────────────────────────┤
  │ factorial(3)  → 3 * factorial(2)│
  ├─────────────────────────────────┤
  │ factorial(4)  → 4 * factorial(3)│
  ├─────────────────────────────────┤
  │ factorial(5)  → 5 * factorial(4)│  ← bottom of stack (first call)
  └─────────────────────────────────┘

  Stack depth = n = 5 frames → O(n) space

  Each frame stores: function args + local vars + return address
  n frames × constant per frame = O(n)
```

Maximum depth = `n` frames on the stack → O(n) space.

**Iterative version (O(1) space):**

```go
func factorialIterative(n int) int {
    result := 1
    for i := 2; i <= n; i++ {
        result *= i
    }
    return result
}
```

> **Key insight:** Every recursive call uses stack space. Always count the **maximum recursion depth** for space complexity.

---

## Example 6: O(log n) — Recursive Binary Search (Stack Space)

```go
package main

import "fmt"

// O(log n) space — recursion depth is log₂(n)
func binarySearchRecursive(nums []int, target, left, right int) int {
    if left > right {
        return -1
    }

    mid := left + (right-left)/2

    if nums[mid] == target {
        return mid
    } else if nums[mid] < target {
        return binarySearchRecursive(nums, target, mid+1, right) // right half
    } else {
        return binarySearchRecursive(nums, target, left, mid-1)  // left half
    }
}

func main() {
    nums := []int{1, 3, 5, 7, 9, 11, 13, 15}
    fmt.Println(binarySearchRecursive(nums, 7, 0, len(nums)-1)) // 3
}
```

**Why O(log n)?** — Each recursive call halves the search space. The maximum recursion depth is log₂(n).

```
  Recursive Binary Search — Call Stack:

  Searching for 7 in [1,3,5,7,9,11,13,15]

  ┌──────────────────────────────────────┐
  │ search([7], 7)          → FOUND!  │  ← frame 3
  ├──────────────────────────────────────┤
  │ search([1,3,5,7], 7)    → go right│  ← frame 2
  ├──────────────────────────────────────┤
  │ search([1..15], 7)      → go left │  ← frame 1
  └──────────────────────────────────────┘

  Stack depth = log₂(8) = 3 frames → O(log n)

  Compare: iterative version uses 0 stack frames → O(1) space!
```

```
n = 16: depth = 4 (16 → 8 → 4 → 2 → 1)
n = 1,000,000: depth = ~20
```

**Iterative version (O(1) space):**

```go
func binarySearchIterative(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}
```

> **Interview tip:** Iterative binary search is O(1) space vs O(log n) for recursive. Interviewers notice this difference.

---

## Example 7: O(n) — Stack-Based Algorithm

```go
package main

import "fmt"

// O(n) space — stack can hold up to n elements
// Monotonic stack to find next greater element
func nextGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)     // O(n) space
    stack := []int{}              // O(n) worst case

    for i := 0; i < n; i++ {
        result[i] = -1
    }

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[top] = nums[i]
        }
        stack = append(stack, i)
    }
    return result
}

func main() {
    nums := []int{4, 5, 2, 25}
    fmt.Println(nextGreaterElement(nums)) // [5 25 25 -1]

    nums2 := []int{13, 7, 6, 12}
    fmt.Println(nextGreaterElement(nums2)) // [-1 12 12 -1]
}
```

**Why O(n)?** — In the worst case (e.g., a strictly decreasing array like `[5, 4, 3, 2, 1]`), all `n` elements are pushed onto the stack before any are popped. The `result` array also takes O(n).

```
  Monotonic Stack for [4, 5, 2, 25]:

  i=0: push 4       stack: [4]           result: [-1,-1,-1,-1]
  i=1: 5>4, pop 4   stack: []            result: [5,-1,-1,-1]
       push 5       stack: [5]
  i=2: 2<5          stack: [5,2]         result: [5,-1,-1,-1]
       push 2
  i=3: 25>2, pop 2  stack: [5]           result: [5,-1,25,-1]
       25>5, pop 5  stack: []            result: [5,25,25,-1]
       push 25      stack: [25]

  Worst case memory (descending input [5,4,3,2,1]):
  ┌─────┐
  │  1  │  ← top
  ├─────┤
  │  2  │
  ├─────┤
  │  3  │
  ├─────┤
  │  4  │
  ├─────┤
  │  5  │  ← bottom     All n elements on stack = O(n)
  └─────┘
```

**Total:** O(n) for result + O(n) for stack = O(2n) → **O(n)**

---

## Example 8: O(n²) — 2D Matrix Allocation

```go
package main

import "fmt"

// O(n²) space — allocates an n×n matrix
func createMatrix(n int) [][]int {
    matrix := make([][]int, n)        // n rows
    for i := 0; i < n; i++ {
        matrix[i] = make([]int, n)    // n columns per row
    }

    // Fill with multiplication table
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            matrix[i][j] = (i + 1) * (j + 1)
        }
    }
    return matrix
}

func main() {
    m := createMatrix(4)
    for _, row := range m {
        fmt.Println(row)
    }
    // [1 2 3 4]
    // [2 4 6 8]
    // [3 6 9 12]
    // [4 8 12 16]
}
```

**Why O(n²)?** — We create `n` rows, each with `n` columns = n × n cells total.

```
  Memory layout for n=4:

  matrix[0] → [1] [2] [3] [4]     ← 4 ints
  matrix[1] → [2] [4] [6] [8]     ← 4 ints
  matrix[2] → [3] [6] [9] [12]    ← 4 ints
  matrix[3] → [4] [8] [12] [16]   ← 4 ints
                                    ────────
                                    16 cells = n² = 4²

  Space
  ▲
  │                    · O(n²)
  │                ·
  │            ·
  │        ·
  │     · ╱ O(n)
  │   ·╱
  │  ╱
  │╱
  └────────────────────────▶ n

  O(n²) grows much faster than O(n).
```

```
n = 10    → 100 cells
n = 1000  → 1,000,000 cells
n = 10000 → 100,000,000 cells ← ~400 MB for int32!
```

---

## Example 9: O(n) — BFS Using a Queue

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// O(n) space — queue can hold up to n/2 nodes (last level of complete tree)
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }

    var result [][]int
    queue := []*TreeNode{root} // O(n/2) worst case ≈ O(n)

    for len(queue) > 0 {
        levelSize := len(queue)
        level := make([]int, 0, levelSize)

        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \   \
    //   4   5   6
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, nil, nil},
            &TreeNode{5, nil, nil}},
        &TreeNode{3, nil,
            &TreeNode{6, nil, nil}}}

    fmt.Println(levelOrder(root))
    // [[1] [2 3] [4 5 6]]
}
```

**Why O(n)?** — The queue holds at most one full level at a time. In a complete binary tree, the last level has ~n/2 nodes → O(n/2) → **O(n)**.

```
  BFS Queue State at Each Level:

  Tree:        1
              / \
             2   3
            / \   \
           4   5   6

  Level 0: queue = [1]       → 1 node in queue
  Level 1: queue = [2, 3]    → 2 nodes in queue
  Level 2: queue = [4, 5, 6] → 3 nodes in queue  ← maximum!

  Complete binary tree with n nodes:
  Last level ≈ n/2 nodes

  Level 0:        ●                  → 1 node
  Level 1:      ●   ●                → 2 nodes
  Level 2:    ● ● ● ●               → 4 nodes
  Level 3:  ●●●●●●●●                 → 8 nodes  ← n/2
                                      ──────────
  Queue max = n/2 at last level → O(n)
```

The `result` also stores all `n` nodes → O(n).

---

## Example 10: O(h) — DFS Using Recursion on a Tree

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

// Space: O(h) where h = height of tree
// Balanced tree: h = log n → O(log n)
// Skewed tree:   h = n    → O(n)
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    left := maxDepth(root.Left)    // recurse left
    right := maxDepth(root.Right)  // recurse right
    return int(math.Max(float64(left), float64(right))) + 1
}

func main() {
    //       1
    //      / \
    //     2   3
    //    /
    //   4
    root := &TreeNode{1,
        &TreeNode{2,
            &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil}}

    fmt.Println(maxDepth(root)) // 3
}
```

**Why O(h)?** — The recursion goes as deep as the tree height. Each call adds one stack frame. The maximum number of frames on the stack at any point equals the height `h`.

```
Balanced tree (n = 15): h = 4  → O(log n) space
Skewed tree  (n = 15): h = 15 → O(n) space  ← worst case!

    1               1
   / \               \
  2   3               2
 / \ / \               \
4  5 6  7               3
                          \
                           4    ← skewed (linked list)
```

---

## Example 11: O(n) — DFS with Explicit Stack (Avoiding Recursion)

```go
package main

import "fmt"

// O(n) space — explicit stack replaces call stack
// Iterative DFS for graph traversal
func dfsIterative(graph map[int][]int, start int) []int {
    visited := make(map[int]bool) // O(n) — tracks visited nodes
    stack := []int{start}         // O(n) worst case
    var result []int

    for len(stack) > 0 {
        // Pop
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if visited[node] {
            continue
        }
        visited[node] = true
        result = append(result, node)

        // Push neighbors
        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                stack = append(stack, neighbor)
            }
        }
    }
    return result
}

func main() {
    graph := map[int][]int{
        0: {1, 2},
        1: {0, 3},
        2: {0, 4},
        3: {1},
        4: {2, 5},
        5: {4},
    }
    fmt.Println(dfsIterative(graph, 0)) // [0 2 4 5 1 3]
}
```

**Why O(n)?**
- `visited` map: stores up to `n` entries → O(n)
- `stack`: in worst case holds up to `n` nodes → O(n)
- `result`: stores all `n` nodes → O(n)
- Total: O(n) + O(n) + O(n) = **O(n)**

```
  Graph DFS — Stack and Visited State:

  Graph:  0 ── 1 ── 3
          │
          2 ── 4 ── 5

  Step 1: stack=[0]      visited={}          result=[]
  Step 2: stack=[1,2]    visited={0}         result=[0]
  Step 3: stack=[1,4]    visited={0,2}       result=[0,2]
  Step 4: stack=[1,5]    visited={0,2,4}     result=[0,2,4]
  Step 5: stack=[1]      visited={0,2,4,5}   result=[0,2,4,5]
  Step 6: stack=[3]      visited={0,2,4,5,1} result=[0,2,4,5,1]
  Step 7: stack=[]       visited={all}       result=[0,2,4,5,1,3]

  Memory breakdown:
  ┌──────────────┤
  │ visited: 6  │  O(n)
  │ stack:  ≤2  │  O(n) worst case
  │ result:  6  │  O(n)
  └──────────────┘
```

---

## Example 12: O(2ⁿ) — Generating All Subsets

```go
package main

import "fmt"

// O(2ⁿ × n) space — stores all subsets, each up to size n
func subsets(nums []int) [][]int {
    var result [][]int
    var current []int

    var backtrack func(start int)
    backtrack = func(start int) {
        // Copy current subset and add to result
        subset := make([]int, len(current))
        copy(subset, current)
        result = append(result, subset)

        for i := start; i < len(nums); i++ {
            current = append(current, nums[i])  // choose
            backtrack(i + 1)                     // explore
            current = current[:len(current)-1]   // un-choose (backtrack)
        }
    }

    backtrack(0)
    return result
}

func main() {
    nums := []int{1, 2, 3}
    result := subsets(nums)
    for _, s := range result {
        fmt.Println(s)
    }
    // [] [1] [1 2] [1 2 3] [1 3] [2] [2 3] [3]
}
```

**Why O(2ⁿ × n)?**
- There are 2ⁿ subsets of a set with `n` elements
- Each subset can be up to size `n`
- Storing all subsets: 2ⁿ × n

```
  Subset Generation Tree for [1, 2, 3]:

                        []
                      /    \
                 include 1   exclude 1
                  /             \
               [1]              []
              /   \            /   \
         inc 2   exc 2    inc 2   exc 2
          /       \        /       \
       [1,2]     [1]     [2]      []
       /  \     /  \    /  \    /  \
  [1,2,3][1,2][1,3][1][2,3][2] [3]  []

  8 subsets = 2³  (each stored in result)

  Memory used:
  Output: 2ⁿ subsets × avg size n/2 ≈ O(2ⁿ × n)
  Stack:  max recursion depth = n   ≈ O(n)
```

```
n = 3  → 8 subsets
n = 10 → 1,024 subsets
n = 20 → 1,048,576 subsets
n = 30 → 1,073,741,824 subsets ← ~1 billion!
```

**Recursion stack depth:** O(n) — we go at most `n` levels deep.

---

## Example 13: O(n) — Dynamic Programming Table

```go
package main

import "fmt"

// O(n) space — DP table of size n+1
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }

    dp := make([]int, n+1) // O(n) space
    dp[1] = 1
    dp[2] = 2

    for i := 3; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}

// O(1) space — optimized! Only track last two values
func climbStairsOptimized(n int) int {
    if n <= 2 {
        return n
    }

    prev2, prev1 := 1, 2 // only 2 variables instead of array

    for i := 3; i <= n; i++ {
        curr := prev1 + prev2
        prev2 = prev1
        prev1 = curr
    }
    return prev1
}

func main() {
    fmt.Println(climbStairs(10))          // 89
    fmt.Println(climbStairsOptimized(10)) // 89
}
```

**Why O(n) → O(1)?**

| Version | Space | Why |
|---------|-------|-----|
| DP table | O(n) | Array of size n+1 |
| Optimized | O(1) | Only 2 variables (prev1, prev2) |

> **Interview tip:** After building a DP solution, always ask yourself: *"Do I really need the entire table, or can I keep only the last few values?"* This is called **space optimization** or **rolling array**.

---

## Space Complexity of Common Data Structures

| Data Structure | Space |
|---------------|-------|
| Array / Slice | O(n) |
| Hash Map | O(n) |
| Hash Set | O(n) |
| Stack | O(n) |
| Queue | O(n) |
| Linked List | O(n) |
| Binary Tree | O(n) |
| Graph (adjacency list) | O(V + E) |
| 2D Matrix | O(n × m) |
| Trie | O(alphabet × total_chars) |

---

## Space Complexity of Common Algorithms

| Algorithm | Time | Space |
|-----------|------|-------|
| Binary search (iterative) | O(log n) | O(1) |
| Binary search (recursive) | O(log n) | O(log n) |
| Merge sort | O(n log n) | O(n) |
| Quick sort | O(n log n) avg | O(log n) avg |
| BFS | O(V + E) | O(V) |
| DFS (recursive) | O(V + E) | O(V) |
| Dynamic programming | varies | O(n) to O(n²) |
| Backtracking | varies | O(n) stack + output |

---

## Time vs Space Tradeoffs

This is a critical concept for interviews. You can often trade one for the other.

| Problem | Brute Force | Optimized | Tradeoff |
|---------|------------|-----------|----------|
| Two Sum | O(n²) time, O(1) space | O(n) time, O(n) space | Use hash map → faster but more memory |
| Fibonacci | O(2ⁿ) time, O(n) space | O(n) time, O(1) space | DP with rolling variables |
| Duplicate detection | O(n²) time, O(1) space | O(n) time, O(n) space | Use hash set |
| Anagram check | O(n log n) time, O(1) space | O(n) time, O(1) space | Frequency array (bounded) |

```go
// TRADEOFF EXAMPLE: Two Sum

// Approach 1: O(n²) time, O(1) space — brute force
func twoSumBrute(nums []int, target int) []int {
    for i := 0; i < len(nums); i++ {
        for j := i + 1; j < len(nums); j++ {
            if nums[i]+nums[j] == target {
                return []int{i, j}
            }
        }
    }
    return nil
}

// Approach 2: O(n) time, O(n) space — hash map (trade space for time)
func twoSumHash(nums []int, target int) []int {
    seen := make(map[int]int) // O(n) extra space
    for i, num := range nums {
        complement := target - num
        if j, ok := seen[complement]; ok {
            return []int{j, i}
        }
        seen[num] = i
    }
    return nil
}
```

---

## How to Identify Space Complexity — Quick Rules

| Pattern | Space Complexity |
|---------|-----------------|
| Fixed number of variables | O(1) |
| Recursion with depth d | O(d) |
| New array of size n | O(n) |
| Hash map with up to n keys | O(n) |
| n × m matrix | O(n × m) |
| Storing all subsets | O(2ⁿ) |
| Storing all permutations | O(n × n!) |

---

## Practice: Analyze These

```go
// Problem 1: What is the space complexity?
func reverseString(s []byte) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i]
    }
}
// Answer: O(1) — in-place, only two index variables


// Problem 2: What is the space complexity?
func flatten(matrix [][]int) []int {
    var result []int
    for _, row := range matrix {
        result = append(result, row...)
    }
    return result
}
// Answer: O(n × m) — result holds all elements from the matrix


// Problem 3: What is the space complexity?
func power(base, exp int) int {
    if exp == 0 {
        return 1
    }
    if exp%2 == 0 {
        half := power(base, exp/2)
        return half * half
    }
    return base * power(base, exp-1)
}
// Answer: O(log n) — recursion depth is log₂(exp), each call uses O(1) stack space
```

---

## Key Takeaways

1. **Space = auxiliary memory** — don't count the input unless you modify it or the problem says otherwise
2. **Recursion uses stack space** — always count the maximum recursion depth
3. **Hash maps, arrays, stacks, queues** all contribute to space complexity
4. **In-place algorithms = O(1) space** — they modify the input directly
5. **DP space optimization** — often you can reduce O(n) → O(1) by keeping only the last few values
6. **Time-space tradeoff** — using more memory (e.g., hash maps) often lets you reduce time complexity
7. **Always mention both time AND space** complexity in interviews

> **Next up:** Big O, Big Theta, Big Omega Notation →
