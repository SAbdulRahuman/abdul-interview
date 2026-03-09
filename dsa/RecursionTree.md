# Phase 14: Backtracking — Recursion Tree

## Overview

A **recursion tree** is the conceptual tree that represents all recursive calls made by a backtracking or divide-and-conquer algorithm. Each node represents a function call, and branches represent choices.

| Concept | Description |
|---------|------------|
| **Root** | Initial call with full problem |
| **Branch** | A decision/choice at each step |
| **Leaf** | Base case — complete solution or dead end |
| **Depth** | Length of the decision sequence |
| **Width** | Number of choices at each level |

**Total nodes** ≈ branching_factor^depth

---

## Example 1: Visualizing a Recursion Tree (Fibonacci)

```go
package main

import (
	"fmt"
	"strings"
)

func fibTree(n, depth int) int {
	indent := strings.Repeat("  ", depth)
	fmt.Printf("%sfib(%d)\n", indent, n)

	if n <= 1 { return n }
	return fibTree(n-1, depth+1) + fibTree(n-2, depth+1)
}

func main() {
	fmt.Println("Recursion tree for fib(5):")
	result := fibTree(5, 0)
	fmt.Println("Result:", result)
}
```

---

## Example 2: Subsets Recursion Tree

```go
package main

import (
	"fmt"
	"strings"
)

func subsetsTree(nums []int, idx int, current []int, depth int) {
	indent := strings.Repeat("  ", depth)
	fmt.Printf("%ssubset(%v, idx=%d)\n", indent, current, idx)

	if idx == len(nums) {
		fmt.Printf("%s→ LEAF: %v\n", indent, current)
		return
	}

	// Branch 1: exclude nums[idx]
	subsetsTree(nums, idx+1, current, depth+1)

	// Branch 2: include nums[idx]
	subsetsTree(nums, idx+1, append(append([]int{}, current...), nums[idx]), depth+1)
}

func main() {
	fmt.Println("Recursion tree for subsets([1,2,3]):")
	subsetsTree([]int{1, 2, 3}, 0, []int{}, 0)
}
```

---

## Example 3: Counting Nodes in Recursion Tree

```go
package main

import "fmt"

func countNodes(branching, depth int) int {
	if depth == 0 { return 1 }
	total := 1
	for i := 0; i < branching; i++ {
		total += countNodes(branching, depth-1)
	}
	return total
}

func main() {
	fmt.Println("=== Recursion Tree Node Counts ===")
	fmt.Println()
	fmt.Printf("%-12s %-8s %-12s\n", "Branching", "Depth", "Total Nodes")
	fmt.Println("------------------------------------")

	cases := []struct{ b, d int }{
		{2, 3}, {2, 5}, {2, 10},
		{3, 3}, {3, 5},
		{4, 3},
	}
	for _, c := range cases {
		fmt.Printf("%-12d %-8d %-12d\n", c.b, c.d, countNodes(c.b, c.d))
	}
}
```

---

## Example 4: Permutation Recursion Tree

```go
package main

import (
	"fmt"
	"strings"
)

func permuteTree(nums []int, chosen []int, used []bool, depth int) {
	indent := strings.Repeat("  ", depth)
	if len(chosen) == len(nums) {
		fmt.Printf("%s→ LEAF: %v\n", indent, chosen)
		return
	}

	fmt.Printf("%spermute(chosen=%v)\n", indent, chosen)
	for i := 0; i < len(nums); i++ {
		if used[i] { continue }
		used[i] = true
		permuteTree(nums, append(append([]int{}, chosen...), nums[i]), used, depth+1)
		used[i] = false
	}
}

func main() {
	fmt.Println("Recursion tree for permutations([1,2,3]):")
	permuteTree([]int{1, 2, 3}, []int{}, make([]bool, 3), 0)
}
```

---

## Example 5: Binary Tree as Recursion Tree

```go
package main

import "fmt"

// Each node in a recursion tree can be modeled as a tree node
type TreeNode struct {
	Label    string
	Children []*TreeNode
}

func buildSubsetTree(nums []int, idx int, path string) *TreeNode {
	node := &TreeNode{Label: fmt.Sprintf("idx=%d, path=%s", idx, path)}
	if idx == len(nums) { return node }

	// Exclude
	node.Children = append(node.Children, buildSubsetTree(nums, idx+1, path))
	// Include
	node.Children = append(node.Children, buildSubsetTree(nums, idx+1, path+fmt.Sprintf("%d", nums[idx])))
	return node
}

func printTree(node *TreeNode, prefix string, isLast bool) {
	connector := "├── "
	if isLast { connector = "└── " }
	fmt.Println(prefix + connector + node.Label)

	childPrefix := prefix + "│   "
	if isLast { childPrefix = prefix + "    " }

	for i, child := range node.Children {
		printTree(child, childPrefix, i == len(node.Children)-1)
	}
}

func main() {
	root := buildSubsetTree([]int{1, 2}, 0, "")
	fmt.Println("Subset recursion tree [1,2]:")
	printTree(root, "", true)
}
```

---

## Example 6: Time Complexity from Recursion Tree

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println("=== Deriving Time Complexity from Recursion Trees ===")
	fmt.Println()

	problems := []struct {
		name     string
		branch   int
		depth    string
		nodes    string
		workNode string
		total    string
	}{
		{"Fibonacci", 2, "n", "2^n", "O(1)", "O(2^n)"},
		{"Subsets", 2, "n", "2^n", "O(n) copy", "O(n·2^n)"},
		{"Permutations", 0, "n", "n!", "O(n) copy", "O(n·n!)"},
		{"Merge Sort", 2, "log n", "n", "O(n) merge", "O(n log n)"},
		{"Binary Search", 1, "log n", "log n", "O(1)", "O(log n)"},
		{"N-Queens", 0, "n", "≤n!", "O(n) check", "O(n·n!)"},
		{"Combination Sum", 0, "target/min", "varies", "O(n)", "exponential"},
	}

	fmt.Printf("%-20s %-8s %-8s %-10s %-12s %-15s\n",
		"Problem", "Branch", "Depth", "Nodes", "Work/Node", "Total")
	fmt.Println(strings._Repeat("-", 75))
	for _, p := range problems {
		b := fmt.Sprintf("%d", p.branch)
		if p.branch == 0 { b = "varies" }
		fmt.Printf("%-20s %-8s %-8s %-10s %-12s %-15s\n",
			p.name, b, p.depth, p.nodes, p.workNode, p.total)
	}
}
```

Wait — let me fix that:

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Deriving Time Complexity from Recursion Trees ===")
	fmt.Println()

	problems := []struct{ name, branch, depth, nodes, workNode, total string }{
		{"Fibonacci", "2", "n", "2^n", "O(1)", "O(2^n)"},
		{"Subsets", "2", "n", "2^n", "O(n) copy", "O(n·2^n)"},
		{"Permutations", "n→1", "n", "n!", "O(n) copy", "O(n·n!)"},
		{"Merge Sort", "2", "log n", "n", "O(n) merge", "O(n log n)"},
		{"Binary Search", "1", "log n", "log n", "O(1)", "O(log n)"},
		{"N-Queens", "≤n", "n", "≤n!", "O(n) check", "O(n·n!)"},
		{"Comb. Sum", "varies", "target/min", "varies", "O(n)", "exponential"},
	}

	fmt.Printf("%-17s %-8s %-10s %-8s %-12s %-12s\n",
		"Problem", "Branch", "Depth", "Nodes", "Work/Node", "Total")
	for i := 0; i < 70; i++ { fmt.Print("-") }
	fmt.Println()
	for _, p := range problems {
		fmt.Printf("%-17s %-8s %-10s %-8s %-12s %-12s\n",
			p.name, p.branch, p.depth, p.nodes, p.workNode, p.total)
	}

	fmt.Println()
	fmt.Println("Formula: Total = Σ (work at each node)")
	fmt.Println("       ≈ nodes × work_per_node")
	fmt.Println("       = branching^depth × work_per_node")
	_ = math.Pow(2, 10) // use math package
}
```

Hmm, let me just provide a clean version:

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Deriving Time Complexity from Recursion Trees ===")
	fmt.Println()

	problems := []struct{ name, branch, depth, nodes, workNode, total string }{
		{"Fibonacci", "2", "n", "2^n", "O(1)", "O(2^n)"},
		{"Subsets", "2", "n", "2^n", "O(n) copy", "O(n·2^n)"},
		{"Permutations", "n→1", "n", "n!", "O(n) copy", "O(n·n!)"},
		{"Merge Sort", "2", "log n", "n", "O(n) merge", "O(n log n)"},
		{"Binary Search", "1", "log n", "log n", "O(1)", "O(log n)"},
		{"N-Queens", "≤n", "n", "≤n!", "O(n) check", "O(n·n!)"},
		{"Comb. Sum", "varies", "t/min", "varies", "O(n)", "exponential"},
	}

	fmt.Printf("%-17s %-8s %-8s %-8s %-13s %-12s\n",
		"Problem", "Branch", "Depth", "Nodes", "Work/Node", "Total")
	for i := 0; i < 68; i++ {
		fmt.Print("-")
	}
	fmt.Println()
	for _, p := range problems {
		fmt.Printf("%-17s %-8s %-8s %-8s %-13s %-12s\n",
			p.name, p.branch, p.depth, p.nodes, p.workNode, p.total)
	}

	fmt.Println()
	fmt.Println("Total time ≈ (number of nodes) × (work per node)")
}
```

---

## Example 7: Recursion Tree for Combination Sum

```go
package main

import (
	"fmt"
	"strings"
)

func combSumTree(candidates []int, target, start, depth int, path []int) {
	indent := strings.Repeat("  ", depth)
	fmt.Printf("%starget=%d, path=%v\n", indent, target, path)

	if target == 0 {
		fmt.Printf("%s→ FOUND: %v\n", indent, path)
		return
	}
	if target < 0 { return }

	for i := start; i < len(candidates); i++ {
		combSumTree(candidates, target-candidates[i], i, depth+1,
			append(append([]int{}, path...), candidates[i]))
	}
}

func main() {
	fmt.Println("Recursion tree for combinationSum([2,3,6,7], 7):")
	combSumTree([]int{2, 3, 6, 7}, 7, 0, 0, []int{})
}
```

---

## Example 8: Recursion Tree for N-Queens

```go
package main

import (
	"fmt"
	"strings"
)

var nodeCount int

func nQueensTree(n, row int, cols, diag1, diag2 map[int]bool, depth int) {
	indent := strings.Repeat("  ", depth)
	nodeCount++

	if row == n {
		fmt.Printf("%s→ SOLUTION FOUND (node #%d)\n", indent, nodeCount)
		return
	}

	for col := 0; col < n; col++ {
		if cols[col] || diag1[row-col] || diag2[row+col] { continue }
		fmt.Printf("%sPlace Q at (%d,%d)\n", indent, row, col)
		cols[col] = true; diag1[row-col] = true; diag2[row+col] = true
		nQueensTree(n, row+1, cols, diag1, diag2, depth+1)
		cols[col] = false; diag1[row-col] = false; diag2[row+col] = false
	}
}

func main() {
	nodeCount = 0
	fmt.Println("4-Queens recursion tree:")
	nQueensTree(4, 0, map[int]bool{}, map[int]bool{}, map[int]bool{}, 0)
	fmt.Println("Total nodes explored:", nodeCount)
}
```

---

## Example 9: Memoized vs Unmemoized Tree Size

```go
package main

import "fmt"

func fibCallCount(n int, memo map[int]bool, useMemo bool) int {
	if n <= 1 { return 1 }
	if useMemo && memo[n] { return 1 } // memo hit = 1 call
	memo[n] = true
	return 1 + fibCallCount(n-1, memo, useMemo) + fibCallCount(n-2, memo, useMemo)
}

func main() {
	fmt.Println("=== Recursion Tree Size: Memoized vs Unmemoized ===")
	fmt.Println()
	fmt.Printf("%-4s %-15s %-15s\n", "n", "Without memo", "With memo")
	for _, n := range []int{5, 10, 15, 20, 25} {
		without := fibCallCount(n, map[int]bool{}, false)
		with := fibCallCount(n, map[int]bool{}, true)
		fmt.Printf("%-4d %-15d %-15d\n", n, without, with)
	}
}
```

---

## Example 10: Recursion Tree Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Recursion Tree Patterns ===")
	fmt.Println()

	patterns := []struct{ pattern, example, shape string }{
		{"Include/Exclude", "Subsets", "Binary tree, depth n"},
		{"Choose from remaining", "Permutations", "n-ary tree, shrinking branches"},
		{"Choose with repetition", "Combination Sum", "n-ary tree, variable depth"},
		{"Divide in half", "Merge Sort", "Binary tree, depth log n"},
		{"Pick one", "Binary Search", "Single path, depth log n"},
		{"Place & validate", "N-Queens, Sudoku", "Pruned n-ary tree"},
		{"Partition string", "Palindrome Partition", "n-ary tree, depth n"},
		{"Two overlapping calls", "Fibonacci", "Binary tree, many repeated nodes"},
		{"Grid exploration", "Word Search", "4-ary tree in grid"},
		{"Generate valid", "Generate Parentheses", "Binary tree with constraints"},
	}

	for i, p := range patterns {
		fmt.Printf("%2d. %-25s %-25s %s\n", i+1, p.pattern, p.example, p.shape)
	}

	fmt.Println()
	fmt.Println("Key: Draw the tree to understand time complexity!")
}
```

---

## Key Takeaways

1. Every backtracking algorithm has an implicit recursion tree
2. Time complexity = total nodes × work per node
3. Branching factor and depth determine tree size
4. Pruning reduces the effective tree size
5. Draw the tree for the first few levels to understand the algorithm

> **Next up:** State Space Search →
