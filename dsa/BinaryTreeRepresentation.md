# Phase 8: Binary Trees — Binary Tree Representation

## Overview

A binary tree can be stored in two fundamental ways:

| Representation | Structure | Access | Memory |
|---------------|-----------|--------|--------|
| **Pointer-based** (linked) | Each node has left/right pointers | O(h) traversal | O(n) + pointer overhead |
| **Array-based** (implicit) | Level-order in contiguous array | O(1) by index | O(2^h) worst case |

Array indexing (0-based): parent = `(i-1)/2`, left child = `2i+1`, right child = `2i+2`

---

## Example 1: Pointer-Based Binary Tree

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    // Level-order insertion (complete tree)
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if node.Left == nil {
            node.Left = &TreeNode{Val: val}
            return root
        }
        queue = append(queue, node.Left)

        if node.Right == nil {
            node.Right = &TreeNode{Val: val}
            return root
        }
        queue = append(queue, node.Right)
    }
    return root
}

func printInorder(root *TreeNode) {
    if root == nil {
        return
    }
    printInorder(root.Left)
    fmt.Printf("%d ", root.Val)
    printInorder(root.Right)
}

func main() {
    var root *TreeNode
    for _, v := range []int{1, 2, 3, 4, 5, 6, 7} {
        root = insert(root, v)
    }
    fmt.Print("Inorder: ")
    printInorder(root) // 4 2 5 1 6 3 7
    fmt.Println()
}
```

---

## Example 2: Array-Based Binary Tree

```go
package main

import "fmt"

type ArrayTree struct {
    data []int
    null int // sentinel for "no node"
}

func NewArrayTree(capacity int) *ArrayTree {
    data := make([]int, capacity)
    for i := range data {
        data[i] = -1 // -1 = empty
    }
    return &ArrayTree{data: data, null: -1}
}

func (t *ArrayTree) SetRoot(val int)              { t.data[0] = val }
func (t *ArrayTree) SetLeft(parentIdx, val int)    { t.data[2*parentIdx+1] = val }
func (t *ArrayTree) SetRight(parentIdx, val int)   { t.data[2*parentIdx+2] = val }
func (t *ArrayTree) Parent(i int) int              { return (i - 1) / 2 }
func (t *ArrayTree) LeftChild(i int) int           { return 2*i + 1 }
func (t *ArrayTree) RightChild(i int) int          { return 2*i + 2 }

func (t *ArrayTree) Print() {
    for i, v := range t.data {
        if v != t.null {
            fmt.Printf("Index %d: %d", i, v)
            if i > 0 {
                fmt.Printf(" (parent=%d)", t.data[t.Parent(i)])
            }
            fmt.Println()
        }
    }
}

func (t *ArrayTree) Inorder(i int) {
    if i >= len(t.data) || t.data[i] == t.null {
        return
    }
    t.Inorder(t.LeftChild(i))
    fmt.Printf("%d ", t.data[i])
    t.Inorder(t.RightChild(i))
}

func main() {
    //       1
    //      / \
    //     2   3
    //    / \
    //   4   5
    tree := NewArrayTree(15)
    tree.SetRoot(1)
    tree.SetLeft(0, 2)
    tree.SetRight(0, 3)
    tree.SetLeft(1, 4)
    tree.SetRight(1, 5)

    tree.Print()
    fmt.Print("Inorder: ")
    tree.Inorder(0) // 4 2 5 1 3
    fmt.Println()
}
```

---

## Example 3: Convert Between Representations

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Array → Linked
func arrayToLinked(arr []int, i int) *TreeNode {
    if i >= len(arr) || arr[i] == -1 {
        return nil
    }
    return &TreeNode{
        Val:   arr[i],
        Left:  arrayToLinked(arr, 2*i+1),
        Right: arrayToLinked(arr, 2*i+2),
    }
}

// Linked → Array
func linkedToArray(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := []int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if node == nil {
            result = append(result, -1)
            continue
        }
        result = append(result, node.Val)
        queue = append(queue, node.Left, node.Right)
    }

    // Trim trailing -1s
    for len(result) > 0 && result[len(result)-1] == -1 {
        result = result[:len(result)-1]
    }
    return result
}

func main() {
    // Array → Linked
    arr := []int{1, 2, 3, 4, 5, -1, 6}
    root := arrayToLinked(arr, 0)
    fmt.Println("Built from array:", root.Val, root.Left.Val, root.Right.Val)

    // Linked → Array
    result := linkedToArray(root)
    fmt.Println("Back to array:", result) // [1 2 3 4 5 -1 6]
}
```

---

## Example 4: N-ary Tree Representation

```go
package main

import "fmt"

type NaryNode struct {
    Val      int
    Children []*NaryNode
}

func newNode(val int, children ...*NaryNode) *NaryNode {
    return &NaryNode{Val: val, Children: children}
}

func printTree(node *NaryNode, indent string) {
    if node == nil {
        return
    }
    fmt.Println(indent + fmt.Sprint(node.Val))
    for _, child := range node.Children {
        printTree(child, indent+"  ")
    }
}

func countNodes(node *NaryNode) int {
    if node == nil {
        return 0
    }
    count := 1
    for _, child := range node.Children {
        count += countNodes(child)
    }
    return count
}

func main() {
    //       1
    //     / | \
    //    2  3  4
    //   / \    |
    //  5   6   7
    root := newNode(1,
        newNode(2, newNode(5), newNode(6)),
        newNode(3),
        newNode(4, newNode(7)),
    )

    printTree(root, "")
    fmt.Println("Total nodes:", countNodes(root)) // 7
}
```

---

## Example 5: Tree with Parent Pointers

```go
package main

import "fmt"

type TreeNode struct {
    Val    int
    Left   *TreeNode
    Right  *TreeNode
    Parent *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
    node := &TreeNode{Val: val}
    if root == nil {
        return node
    }
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]
        if cur.Left == nil {
            cur.Left = node
            node.Parent = cur
            return root
        }
        queue = append(queue, cur.Left)
        if cur.Right == nil {
            cur.Right = node
            node.Parent = cur
            return root
        }
        queue = append(queue, cur.Right)
    }
    return root
}

func pathToRoot(node *TreeNode) []int {
    path := []int{}
    for node != nil {
        path = append(path, node.Val)
        node = node.Parent
    }
    return path
}

func main() {
    var root *TreeNode
    for _, v := range []int{1, 2, 3, 4, 5} {
        root = insert(root, v)
    }

    // Find node 5
    node5 := root.Left.Right // 5 is right child of 2
    fmt.Println("Path from 5 to root:", pathToRoot(node5)) // [5 2 1]
}
```

---

## Example 6: Threaded Binary Tree (Morris Threading)

```go
package main

import "fmt"

type TreeNode struct {
    Val     int
    Left    *TreeNode
    Right   *TreeNode
    IsThread bool // true if Right points to inorder successor
}

func createThreaded(root *TreeNode) {
    var prev *TreeNode
    var thread func(node *TreeNode)
    thread = func(node *TreeNode) {
        if node == nil {
            return
        }
        thread(node.Left)
        if prev != nil && prev.Right == nil {
            prev.Right = node
            prev.IsThread = true
        }
        prev = node
        thread(node.Right)
    }
    thread(root)
}

func inorderThreaded(root *TreeNode) {
    cur := root
    // Go to leftmost
    for cur.Left != nil {
        cur = cur.Left
    }

    for cur != nil {
        fmt.Printf("%d ", cur.Val)
        if cur.IsThread {
            cur = cur.Right // follow thread
        } else {
            cur = cur.Right
            if cur != nil {
                for cur.Left != nil {
                    cur = cur.Left
                }
            }
        }
    }
    fmt.Println()
}

func main() {
    //       4
    //      / \
    //     2   6
    //    / \ / \
    //   1  3 5  7
    root := &TreeNode{Val: 4,
        Left: &TreeNode{Val: 2,
            Left:  &TreeNode{Val: 1},
            Right: &TreeNode{Val: 3},
        },
        Right: &TreeNode{Val: 6,
            Left:  &TreeNode{Val: 5},
            Right: &TreeNode{Val: 7},
        },
    }

    createThreaded(root)
    fmt.Print("Threaded inorder: ")
    inorderThreaded(root) // 1 2 3 4 5 6 7
}
```

---

## Example 7: Left-Child Right-Sibling (LCRS) Representation

Convert any N-ary tree to binary using the LCRS encoding.

```go
package main

import "fmt"

type NaryNode struct {
    Val      int
    Children []*NaryNode
}

type BinaryNode struct {
    Val   int
    Left  *BinaryNode // first child
    Right *BinaryNode // next sibling
}

func naryToBinary(nary *NaryNode) *BinaryNode {
    if nary == nil {
        return nil
    }
    bin := &BinaryNode{Val: nary.Val}

    if len(nary.Children) > 0 {
        bin.Left = naryToBinary(nary.Children[0]) // first child
        cur := bin.Left
        for i := 1; i < len(nary.Children); i++ {
            cur.Right = naryToBinary(nary.Children[i]) // sibling chain
            cur = cur.Right
        }
    }
    return bin
}

func binaryToNary(bin *BinaryNode) *NaryNode {
    if bin == nil {
        return nil
    }
    nary := &NaryNode{Val: bin.Val}
    child := bin.Left
    for child != nil {
        nary.Children = append(nary.Children, binaryToNary(child))
        child = child.Right
    }
    return nary
}

func printBinary(node *BinaryNode, indent string) {
    if node == nil {
        return
    }
    fmt.Printf("%sVal=%d\n", indent, node.Val)
    if node.Left != nil {
        fmt.Printf("%s  child→\n", indent)
        printBinary(node.Left, indent+"    ")
    }
    if node.Right != nil {
        fmt.Printf("%s  sibling→\n", indent)
        printBinary(node.Right, indent)
    }
}

func main() {
    //  N-ary:    1
    //          / | \
    //         2  3  4
    //        / \
    //       5   6
    nary := &NaryNode{1, []*NaryNode{
        {2, []*NaryNode{{5, nil}, {6, nil}}},
        {3, nil},
        {4, nil},
    }}

    bin := naryToBinary(nary)
    fmt.Println("Binary (LCRS) representation:")
    printBinary(bin, "")

    // Convert back
    back := binaryToNary(bin)
    fmt.Printf("\nBack to N-ary: root=%d, children=%d\n", back.Val, len(back.Children))
}
```

---

## Example 8: Tree to Parenthesis String

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// LeetCode 606: Construct String from Binary Tree
func tree2str(root *TreeNode) string {
    if root == nil {
        return ""
    }
    s := fmt.Sprint(root.Val)

    if root.Left == nil && root.Right == nil {
        return s
    }

    s += "(" + tree2str(root.Left) + ")"

    if root.Right != nil {
        s += "(" + tree2str(root.Right) + ")"
    }

    return s
}

// Reconstruct from parenthesis string
func str2tree(s string) *TreeNode {
    if len(s) == 0 {
        return nil
    }
    idx := 0
    return parse(s, &idx)
}

func parse(s string, idx *int) *TreeNode {
    if *idx >= len(s) {
        return nil
    }

    // Parse number
    start := *idx
    if s[*idx] == '-' {
        (*idx)++
    }
    for *idx < len(s) && s[*idx] >= '0' && s[*idx] <= '9' {
        (*idx)++
    }
    val := 0
    fmt.Sscanf(s[start:*idx], "%d", &val)
    node := &TreeNode{Val: val}

    // Parse left child
    if *idx < len(s) && s[*idx] == '(' {
        (*idx)++ // skip '('
        node.Left = parse(s, idx)
        (*idx)++ // skip ')'
    }

    // Parse right child
    if *idx < len(s) && s[*idx] == '(' {
        (*idx)++
        node.Right = parse(s, idx)
        (*idx)++
    }

    return node
}

func main() {
    //       1
    //      / \
    //     2   3
    //    /
    //   4
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, nil},
    }

    s := tree2str(root)
    fmt.Println("Parenthesis:", s) // 1(2(4))(3)

    rebuilt := str2tree(s)
    fmt.Println("Rebuilt:", tree2str(rebuilt)) // 1(2(4))(3)
}
```

---

## Example 9: Adjacency List Representation for Generic Trees

```go
package main

import "fmt"

type Tree struct {
    n        int
    children [][]int
    root     int
}

func NewTree(n, root int) *Tree {
    return &Tree{
        n:        n,
        children: make([][]int, n),
        root:     root,
    }
}

func (t *Tree) AddEdge(parent, child int) {
    t.children[parent] = append(t.children[parent], child)
}

func (t *Tree) DFS(node int, depth int) {
    indent := ""
    for i := 0; i < depth; i++ {
        indent += "  "
    }
    fmt.Printf("%sNode %d\n", indent, node)
    for _, child := range t.children[node] {
        t.DFS(child, depth+1)
    }
}

func (t *Tree) Height(node int) int {
    if len(t.children[node]) == 0 {
        return 0
    }
    maxH := 0
    for _, child := range t.children[node] {
        h := t.Height(child)
        if h > maxH {
            maxH = h
        }
    }
    return maxH + 1
}

func main() {
    //       0
    //      /|\
    //     1 2 3
    //    / \   |
    //   4   5  6
    tree := NewTree(7, 0)
    tree.AddEdge(0, 1)
    tree.AddEdge(0, 2)
    tree.AddEdge(0, 3)
    tree.AddEdge(1, 4)
    tree.AddEdge(1, 5)
    tree.AddEdge(3, 6)

    tree.DFS(0, 0)
    fmt.Println("Height:", tree.Height(0)) // 2
}
```

---

## Example 10: Euler Tour Representation (for LCA/RMQ)

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func eulerTour(root *TreeNode) (tour []int, depth []int, first map[int]int) {
    first = map[int]int{}
    var dfs func(node *TreeNode, d int)
    dfs = func(node *TreeNode, d int) {
        if node == nil {
            return
        }
        if _, ok := first[node.Val]; !ok {
            first[node.Val] = len(tour)
        }
        tour = append(tour, node.Val)
        depth = append(depth, d)

        if node.Left != nil {
            dfs(node.Left, d+1)
            tour = append(tour, node.Val)
            depth = append(depth, d)
        }
        if node.Right != nil {
            dfs(node.Right, d+1)
            tour = append(tour, node.Val)
            depth = append(depth, d)
        }
    }
    dfs(root, 0)
    return
}

// Find LCA using Euler Tour + RMQ (simple scan)
func lcaEuler(tour, depth []int, first map[int]int, u, v int) int {
    l, r := first[u], first[v]
    if l > r {
        l, r = r, l
    }

    minDepth := depth[l]
    lcaNode := tour[l]
    for i := l; i <= r; i++ {
        if depth[i] < minDepth {
            minDepth = depth[i]
            lcaNode = tour[i]
        }
    }
    return lcaNode
}

func main() {
    //         1
    //        / \
    //       2   3
    //      / \
    //     4   5
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
        &TreeNode{3, nil, nil},
    }

    tour, depth, first := eulerTour(root)
    fmt.Println("Tour: ", tour)
    fmt.Println("Depth:", depth)
    fmt.Println("First:", first)

    fmt.Println("LCA(4,5):", lcaEuler(tour, depth, first, 4, 5)) // 2
    fmt.Println("LCA(4,3):", lcaEuler(tour, depth, first, 4, 3)) // 1
    fmt.Println("LCA(2,5):", lcaEuler(tour, depth, first, 2, 5)) // 2
}
```

---

## Representation Comparison

| Feature | Pointer-based | Array-based | Adjacency List | LCRS |
|---------|-------------|------------|----------------|------|
| Memory | 2 ptrs/node | Contiguous | 1 list/node | 2 ptrs/node |
| Random access | No | O(1) by index | No | No |
| Supports N-ary | Via child list | Wasteful | Natural | Yes (binary encoding) |
| Cache friendly | Poor | Excellent | Moderate | Poor |
| Insertion/deletion | O(1) local | O(n) shift | O(1) local | O(1) local |

## Key Takeaways

1. **Pointer-based** is the default for interviews — simple, intuitive, flexible
2. **Array-based** is used for heaps and segment trees — O(1) parent/child access
3. **LCRS** converts any N-ary tree to binary — first child as left, next sibling as right
4. **Euler tour** enables LCA via RMQ — visit every node on enter and return
5. **Parent pointers** simplify ancestor queries but add memory overhead
6. Array index formula: `left = 2i+1`, `right = 2i+2`, `parent = (i-1)/2`

> **Next up:** Tree Traversal →
