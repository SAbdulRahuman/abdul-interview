# Phase 8: Binary Trees — Serialization and Deserialization

## Overview

**Serialization** = converting a tree to a string/byte representation. **Deserialization** = reconstructing the tree from that representation. Critical for storing trees in databases, sending over network, or caching.

| Method | Delimiter | Null Marker | Time | Space |
|--------|-----------|-------------|------|-------|
| Preorder DFS | `,` | `#` or `null` | O(n) | O(n) |
| Level-order BFS | `,` | `null` | O(n) | O(n) |
| Parenthesis encoding | `(` `)` | empty `()` | O(n) | O(n) |

---

## Example 1: Serialize/Deserialize with Preorder DFS (LeetCode 297)

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type Codec struct{}

func (c *Codec) serialize(root *TreeNode) string {
    var sb strings.Builder
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil {
            sb.WriteString("#,")
            return
        }
        sb.WriteString(strconv.Itoa(node.Val))
        sb.WriteByte(',')
        dfs(node.Left)
        dfs(node.Right)
    }
    dfs(root)
    return sb.String()
}

func (c *Codec) deserialize(data string) *TreeNode {
    tokens := strings.Split(data, ",")
    idx := 0

    var build func() *TreeNode
    build = func() *TreeNode {
        if idx >= len(tokens) || tokens[idx] == "#" || tokens[idx] == "" {
            idx++
            return nil
        }
        val, _ := strconv.Atoi(tokens[idx])
        idx++
        node := &TreeNode{Val: val}
        node.Left = build()
        node.Right = build()
        return node
    }
    return build()
}

func main() {
    //       1
    //      / \
    //     2   3
    //        / \
    //       4   5
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
    }

    codec := &Codec{}
    s := codec.serialize(root)
    fmt.Println("Serialized:", s)

    tree := codec.deserialize(s)
    fmt.Println("Re-serialized:", codec.serialize(tree))
}
```

**Textual Figure – Example 1:**
```
 Preorder DFS Serialization:
 ─────────────────────────
       1                 Preorder DFS visit:
      / \                1 → 2 → # → # → 3 → 4 → # → # → 5 → # → #
     2   3
        / \              Serialized: "1,2,#,#,3,4,#,#,5,#,#,"
       4   5

 Deserialization (rebuild from token stream):
 ┌──────┬─────────┬─────────────────────────────┐
 │ idx  │ token   │ action                        │
 ├──────┼─────────┼─────────────────────────────┤
 │  0   │  "1"    │ create node(1), recurse L & R │
 │  1   │  "2"    │ create node(2), recurse L & R │
 │  2   │  "#"    │ return nil (2's left)          │
 │  3   │  "#"    │ return nil (2's right)         │
 │  4   │  "3"    │ create node(3), recurse L & R │
 │  5   │  "4"    │ create node(4) → L=#, R=#     │
 │  8   │  "5"    │ create node(5) → L=#, R=#     │
 └──────┴─────────┴─────────────────────────────┘
 Tree rebuilt identically.
```

---

## Example 2: BFS Level-Order Serialization

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func serializeBFS(root *TreeNode) string {
    if root == nil { return "" }
    parts := []string{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        if node == nil {
            parts = append(parts, "null")
        } else {
            parts = append(parts, strconv.Itoa(node.Val))
            queue = append(queue, node.Left)
            queue = append(queue, node.Right)
        }
    }
    // Trim trailing nulls
    for len(parts) > 0 && parts[len(parts)-1] == "null" {
        parts = parts[:len(parts)-1]
    }
    return strings.Join(parts, ",")
}

func deserializeBFS(data string) *TreeNode {
    if data == "" { return nil }
    parts := strings.Split(data, ",")

    val, _ := strconv.Atoi(parts[0])
    root := &TreeNode{Val: val}
    queue := []*TreeNode{root}
    i := 1

    for len(queue) > 0 && i < len(parts) {
        node := queue[0]
        queue = queue[1:]

        // Left child
        if i < len(parts) && parts[i] != "null" {
            v, _ := strconv.Atoi(parts[i])
            node.Left = &TreeNode{Val: v}
            queue = append(queue, node.Left)
        }
        i++

        // Right child
        if i < len(parts) && parts[i] != "null" {
            v, _ := strconv.Atoi(parts[i])
            node.Right = &TreeNode{Val: v}
            queue = append(queue, node.Right)
        }
        i++
    }
    return root
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
    }
    s := serializeBFS(root)
    fmt.Println("BFS:", s) // 1,2,3,null,null,4,5

    tree := deserializeBFS(s)
    fmt.Println("Re-BFS:", serializeBFS(tree))
}
```

**Textual Figure – Example 2:**
```
 BFS Level-Order Serialization:
 ───────────────────────────────
       1          Level 0: [1]
      / \         Level 1: [2, 3]
     2   3        Level 2: [null, null, 4, 5]
        / \
       4   5      BFS queue processes level by level:

 ┌───────┬─────────────┬────────────────────────┐
 │ deque │ output      │ enqueue children       │
 ├───────┼─────────────┼────────────────────────┤
 │  1    │ "1"         │ enqueue 2, 3           │
 │  2    │ "2"         │ enqueue nil, nil       │
 │  3    │ "3"         │ enqueue 4, 5           │
 │ nil   │ "null"      │ (no children)          │
 │ nil   │ "null"      │ (no children)          │
 │  4    │ "4"         │ enqueue nil, nil       │
 │  5    │ "5"         │ enqueue nil, nil       │
 └───────┴─────────────┴────────────────────────┘
 After trimming trailing nulls: "1,2,3,null,null,4,5"
```

---

## Example 3: Parenthesis-Based Serialization

```go
package main

import (
    "fmt"
    "strconv"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Serialize: "1(2)(3(4)(5))"
func serializeParen(root *TreeNode) string {
    if root == nil { return "" }
    s := strconv.Itoa(root.Val)
    if root.Left != nil || root.Right != nil {
        s += "(" + serializeParen(root.Left) + ")"
        if root.Right != nil {
            s += "(" + serializeParen(root.Right) + ")"
        }
    }
    return s
}

func deserializeParen(s string) *TreeNode {
    idx := 0

    var parse func() *TreeNode
    parse = func() *TreeNode {
        if idx >= len(s) || s[idx] == ')' {
            return nil
        }
        // Parse number
        sign := 1
        if s[idx] == '-' {
            sign = -1
            idx++
        }
        num := 0
        for idx < len(s) && s[idx] >= '0' && s[idx] <= '9' {
            num = num*10 + int(s[idx]-'0')
            idx++
        }
        node := &TreeNode{Val: num * sign}

        // Left child
        if idx < len(s) && s[idx] == '(' {
            idx++ // skip '('
            node.Left = parse()
            idx++ // skip ')'
        }
        // Right child
        if idx < len(s) && s[idx] == '(' {
            idx++ // skip '('
            node.Right = parse()
            idx++ // skip ')'
        }
        return node
    }
    return parse()
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
    }
    s := serializeParen(root)
    fmt.Println("Paren:", s) // 1(2)(3(4)(5))

    tree := deserializeParen(s)
    fmt.Println("Re-paren:", serializeParen(tree))
}
```

**Textual Figure – Example 3:**
```
 Parenthesis-Based Serialization:
 ────────────────────────────
       1
      / \         Encoding rule:
     2   3          node(left)(right)
        / \
       4   5      1(2)(3(4)(5))
                   │ │  └──────┘
                   │ │   right subtree of 1
                   │ └─ left subtree of 1
                   └── root value

 Parse trace:
 ┌───────┬────────┬───────────────────────────┐
 │ idx   │ char   │ action                      │
 ├───────┼────────┼───────────────────────────┤
 │  0    │  '1'   │ parse num → node(1)        │
 │  1    │  '('   │ enter left subtree          │
 │  2    │  '2'   │ parse num → node(2)        │
 │  3    │  ')'   │ return from left             │
 │  4    │  '('   │ enter right subtree         │
 │  5    │  '3'   │ parse num → node(3)        │
 │  6-10 │ (4)(5) │ parse 3's children          │
 │  11   │  ')'   │ return, tree complete        │
 └───────┴────────┴───────────────────────────┘
 No null markers needed! Structure encoded in parens.
```

---

## Example 4: Serialize BST with Preorder Only (LeetCode 449)

BSTs can be serialized without null markers — preorder sequence is sufficient.

```go
package main

import (
    "fmt"
    "math"
    "strconv"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func serializeBST(root *TreeNode) string {
    vals := []string{}
    var preorder func(node *TreeNode)
    preorder = func(node *TreeNode) {
        if node == nil { return }
        vals = append(vals, strconv.Itoa(node.Val))
        preorder(node.Left)
        preorder(node.Right)
    }
    preorder(root)
    return strings.Join(vals, ",")
}

func deserializeBST(data string) *TreeNode {
    if data == "" { return nil }
    parts := strings.Split(data, ",")
    vals := make([]int, len(parts))
    for i, p := range parts {
        vals[i], _ = strconv.Atoi(p)
    }

    idx := 0
    var build func(lo, hi int) *TreeNode
    build = func(lo, hi int) *TreeNode {
        if idx >= len(vals) || vals[idx] < lo || vals[idx] > hi {
            return nil
        }
        val := vals[idx]
        idx++
        node := &TreeNode{Val: val}
        node.Left = build(lo, val)
        node.Right = build(val, hi)
        return node
    }
    return build(math.MinInt64, math.MaxInt64)
}

func inorder(root *TreeNode) {
    if root == nil { return }
    inorder(root.Left)
    fmt.Printf("%d ", root.Val)
    inorder(root.Right)
}

func main() {
    root := &TreeNode{5,
        &TreeNode{3, &TreeNode{1, nil, nil}, &TreeNode{4, nil, nil}},
        &TreeNode{7, &TreeNode{6, nil, nil}, &TreeNode{8, nil, nil}},
    }
    s := serializeBST(root)
    fmt.Println("BST serialized:", s) // 5,3,1,4,7,6,8

    tree := deserializeBST(s)
    fmt.Print("Inorder check: "); inorder(tree); fmt.Println()
    // 1 3 4 5 6 7 8
}
```

**Textual Figure – Example 4:**
```
 BST Preorder-Only Serialization (no null markers!):
 ──────────────────────────────────────────
       5               Preorder: [5, 3, 1, 4, 7, 6, 8]
      / \              (no # markers — BST property
     3   7              determines structure)
    / \ / \
   1  4 6  8           Rebuild using bounds (lo, hi):

 ┌─────┬───────┬───────────┬────────────────────────┐
 │ idx │ val   │ bounds    │ action                  │
 ├─────┼───────┼───────────┼────────────────────────┤
 │  0  │  5    │ (-∞,+∞)  │ root=5, L(-∞,5) R(5,+∞) │
 │  1  │  3    │ (-∞, 5)  │ node=3, L(-∞,3) R(3,5) │
 │  2  │  1    │ (-∞, 3)  │ node=1 (leaf)           │
 │  3  │  4    │ (3, 5)    │ node=4 (leaf)           │
 │  4  │  7    │ (5, +∞)  │ node=7, L(5,7) R(7,+∞) │
 │  5  │  6    │ (5, 7)    │ node=6 (leaf)           │
 │  6  │  8    │ (7, +∞)  │ node=8 (leaf)           │
 └─────┴───────┴───────────┴────────────────────────┘
 Inorder check: 1 3 4 5 6 7 8 ✓
```

---

## Example 5: Serialize Using Preorder + Inorder

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func buildTree(preorder, inorder []int) *TreeNode {
    if len(preorder) == 0 { return nil }

    rootVal := preorder[0]
    root := &TreeNode{Val: rootVal}

    // Find root in inorder
    idx := 0
    for i, v := range inorder {
        if v == rootVal { idx = i; break }
    }

    root.Left = buildTree(preorder[1:1+idx], inorder[:idx])
    root.Right = buildTree(preorder[1+idx:], inorder[idx+1:])
    return root
}

func getPreorder(root *TreeNode, out *[]int) {
    if root == nil { return }
    *out = append(*out, root.Val)
    getPreorder(root.Left, out)
    getPreorder(root.Right, out)
}

func getInorder(root *TreeNode, out *[]int) {
    if root == nil { return }
    getInorder(root.Left, out)
    *out = append(*out, root.Val)
    getInorder(root.Right, out)
}

func main() {
    //       3
    //      / \
    //     9  20
    //       /  \
    //      15   7
    root := &TreeNode{3,
        &TreeNode{9, nil, nil},
        &TreeNode{20, &TreeNode{15, nil, nil}, &TreeNode{7, nil, nil}},
    }

    pre, in := []int{}, []int{}
    getPreorder(root, &pre)
    getInorder(root, &in)
    fmt.Println("Preorder:", pre) // [3 9 20 15 7]
    fmt.Println("Inorder:", in)   // [9 3 15 20 7]

    rebuilt := buildTree(pre, in)
    re_pre, re_in := []int{}, []int{}
    getPreorder(rebuilt, &re_pre)
    getInorder(rebuilt, &re_in)
    fmt.Println("Rebuilt preorder:", re_pre)
    fmt.Println("Rebuilt inorder:", re_in)
}
```

**Textual Figure – Example 5:**
```
 Reconstruct Tree from Preorder + Inorder:
 ───────────────────────────────────────
  Preorder: [3, 9, 20, 15, 7]  (root first)
  Inorder:  [9, 3, 15, 20, 7]  (root splits L/R)

 Step 1: root = preorder[0] = 3
         inorder split at 3:
         left = [9]     right = [15, 20, 7]

 Step 2:       3
             / \
           [9] [20,15,7]
            ↓     ↓
            9    root=20
     (leaf)      inorder split at 20:
                 left=[15] right=[7]

 Step 3:       3
             / \
            9  20
              /  \
             15   7

 ┌──────┬─────────────┬──────────────────────┐
 │ Call │ preorder    │ inorder              │
 ├──────┼─────────────┼──────────────────────┤
 │  1   │ [3,9,20..] │ [9,3,15,20,7]        │
 │  2   │ [9]        │ [9] → leaf            │
 │  3   │ [20,15,7]  │ [15,20,7] root=20    │
 │  4   │ [15]       │ [15] → leaf           │
 │  5   │ [7]        │ [7] → leaf            │
 └──────┴─────────────┴──────────────────────┘
 Requires unique values (for finding root in inorder).
```

---

## Example 6: Binary Encoding (Compact Serialization)

```go
package main

import (
    "bytes"
    "encoding/binary"
    "fmt"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func serializeBinary(root *TreeNode) []byte {
    var buf bytes.Buffer
    var encode func(node *TreeNode)
    encode = func(node *TreeNode) {
        if node == nil {
            binary.Write(&buf, binary.LittleEndian, int32(-1)) // sentinel
            return
        }
        binary.Write(&buf, binary.LittleEndian, int32(node.Val))
        encode(node.Left)
        encode(node.Right)
    }
    encode(root)
    return buf.Bytes()
}

func deserializeBinary(data []byte) *TreeNode {
    buf := bytes.NewReader(data)
    var decode func() *TreeNode
    decode = func() *TreeNode {
        var val int32
        err := binary.Read(buf, binary.LittleEndian, &val)
        if err != nil || val == -1 {
            return nil
        }
        return &TreeNode{int(val), decode(), decode()}
    }
    return decode()
}

func printPreorder(root *TreeNode) {
    if root == nil { return }
    fmt.Printf("%d ", root.Val)
    printPreorder(root.Left)
    printPreorder(root.Right)
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, nil},
    }

    data := serializeBinary(root)
    fmt.Printf("Binary size: %d bytes\n", len(data))

    tree := deserializeBinary(data)
    fmt.Print("Reconstructed: "); printPreorder(tree); fmt.Println()
}
```

**Textual Figure – Example 6:**
```
 Binary Encoding (Compact Serialization):
 ────────────────────────────────
       1                 Each int32: 4 bytes
      / \                Sentinel -1: marks nil
     2   3
    /     \
   4      (nil)          Byte stream (little-endian):
       \                 ┌───┬───┬───┬───┬───┬───┬───┬───┐
       (nil)             │ 1 │ 2 │ 4 │-1 │-1 │-1 │ 3 │-1 │
                         └───┴───┴───┴───┴───┴───┴───┴───┘
                           ↑       ↑ nil markers
                          root

 Preorder encoding: 1, 2, 4, -1, -1, -1, 3, -1, -1
                             ↑        ↑      ↑
                           4.L=nil  2.R=nil  3.R=nil
 Total size: 9 x 4 = 36 bytes
 vs string "1,2,4,#,#,#,3,#,#" = 19 chars
 Binary more compact for large values.
```

---

## Example 7: JSON Serialization

```go
package main

import (
    "encoding/json"
    "fmt"
)

type TreeNode struct {
    Val   int       `json:"val"`
    Left  *TreeNode `json:"left,omitempty"`
    Right *TreeNode `json:"right,omitempty"`
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, nil, nil},
        &TreeNode{3, &TreeNode{4, nil, nil}, &TreeNode{5, nil, nil}},
    }

    // Serialize
    data, err := json.Marshal(root)
    if err != nil { panic(err) }
    fmt.Println("JSON:", string(data))

    // Deserialize
    var tree TreeNode
    json.Unmarshal(data, &tree)
    fmt.Printf("Root: %d, Left: %d, Right: %d\n", tree.Val, tree.Left.Val, tree.Right.Val)

    // Pretty print
    pretty, _ := json.MarshalIndent(root, "", "  ")
    fmt.Println(string(pretty))
}
```

**Textual Figure – Example 7:**
```
 JSON Serialization:
 ────────────────
       1            JSON output:
      / \           {"val":1,
     2   3           "left":{"val":2},
        / \          "right":{"val":3,
       4   5           "left":{"val":4},
                       "right":{"val":5}}}

 Struct tags control serialization:
 ┌──────────────────────────────────────┐
 │ Tag              │ Effect              │
 ├──────────────────┼────────────────────┤
 │ json:"val"        │ key name = "val"    │
 │ json:",omitempty" │ skip if nil/zero    │
 └──────────────────┴────────────────────┘
 omitempty prevents nil children from appearing
 as "left":null in the output.

 Pro: human-readable, standard format
 Con: verbose, slower than binary
```

---

## Example 8: Serialize N-ary Tree as Binary Tree

Encode N-ary tree using left-child right-sibling mapping.

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

type NaryNode struct {
    Val      int
    Children []*NaryNode
}

type BinaryNode struct {
    Val   int
    Left  *BinaryNode // first child
    Right *BinaryNode // next sibling
}

func naryToBinary(root *NaryNode) *BinaryNode {
    if root == nil { return nil }
    bNode := &BinaryNode{Val: root.Val}

    if len(root.Children) > 0 {
        bNode.Left = naryToBinary(root.Children[0])
        cur := bNode.Left
        for i := 1; i < len(root.Children); i++ {
            cur.Right = naryToBinary(root.Children[i])
            cur = cur.Right
        }
    }
    return bNode
}

func binaryToNary(root *BinaryNode) *NaryNode {
    if root == nil { return nil }
    nNode := &NaryNode{Val: root.Val}

    cur := root.Left
    for cur != nil {
        nNode.Children = append(nNode.Children, binaryToNary(cur))
        cur = cur.Right
    }
    return nNode
}

func serializeBinary(root *BinaryNode) string {
    if root == nil { return "#" }
    return strconv.Itoa(root.Val) + "," + serializeBinary(root.Left) + "," + serializeBinary(root.Right)
}

func printNary(node *NaryNode, level int) {
    if node == nil { return }
    fmt.Printf("%s%d\n", strings.Repeat("  ", level), node.Val)
    for _, child := range node.Children {
        printNary(child, level+1)
    }
}

func main() {
    //       1
    //      /|\
    //     3 2 4
    //    / \
    //   5   6
    nRoot := &NaryNode{1, []*NaryNode{
        {3, []*NaryNode{{5, nil}, {6, nil}}},
        {2, nil},
        {4, nil},
    }}

    bRoot := naryToBinary(nRoot)
    s := serializeBinary(bRoot)
    fmt.Println("Serialized:", s)

    // Convert back
    nRoot2 := binaryToNary(bRoot)
    fmt.Println("N-ary tree:")
    printNary(nRoot2, 0)
}
```

**Textual Figure – Example 8:**
```
 N-ary → Binary (Left-Child Right-Sibling) Encoding:
 ────────────────────────────────────────────

 N-ary Tree:           Binary Tree (LC-RS):
       1                    1
      /|\                  /
     3 2 4                3          Mapping:
    / \                  / \           Left  = first child
   5   6                5   2         Right = next sibling
                         \   \
                          6   4

 Conversion rules:
 ┌────────────────────────────────────────┐
 │ N-ary           →  Binary              │
 ├────────────────────────────────────────┤
 │ children[0]     →  Left child           │
 │ children[1..n]  →  Right chain from [0] │
 └────────────────────────────────────────┘

 1.Left=3, 3.Right=2, 2.Right=4
 3.Left=5, 5.Right=6

 Binary tree can be serialized normally,
 then deserialized back to N-ary tree.
```

---

## Example 9: Serialize with Bit-Level Structure Encoding

Use a bitmask to encode tree structure, reducing null markers.

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type CompactCodec struct {
    vals  []int
    flags []byte // 2 bits per node: bit0=hasLeft, bit1=hasRight
}

func (c *CompactCodec) encode(root *TreeNode) {
    if root == nil { return }
    c.vals = append(c.vals, root.Val)
    flag := byte(0)
    if root.Left != nil { flag |= 1 }
    if root.Right != nil { flag |= 2 }
    c.flags = append(c.flags, flag)
    c.encode(root.Left)
    c.encode(root.Right)
}

func (c *CompactCodec) decode() *TreeNode {
    idx := 0
    var build func() *TreeNode
    build = func() *TreeNode {
        if idx >= len(c.vals) { return nil }
        node := &TreeNode{Val: c.vals[idx]}
        flag := c.flags[idx]
        idx++
        if flag&1 != 0 { node.Left = build() }
        if flag&2 != 0 { node.Right = build() }
        return node
    }
    return build()
}

func printPreorder(root *TreeNode) {
    if root == nil { return }
    fmt.Printf("%d ", root.Val)
    printPreorder(root.Left)
    printPreorder(root.Right)
}

func main() {
    root := &TreeNode{1,
        &TreeNode{2, &TreeNode{4, nil, nil}, nil},
        &TreeNode{3, nil, &TreeNode{5, nil, nil}},
    }

    codec := &CompactCodec{}
    codec.encode(root)
    fmt.Println("Values:", codec.vals)  // [1 2 4 3 5]
    fmt.Println("Flags:", codec.flags)  // [3 1 0 2 0] (3=both, 1=left, 0=none, 2=right, 0=none)

    tree := codec.decode()
    fmt.Print("Decoded: "); printPreorder(tree); fmt.Println()
}
```

**Textual Figure – Example 9:**
```
 Bit-Level Structure Encoding:
 ────────────────────────────
       1               2 bits per node encode structure:
      / \                bit0 = hasLeft
     2   3               bit1 = hasRight
    /     \
   4       5           ┌──────┬───────┬─────┬────────────────┐
                       │ Node │ L?  R? │ bin │ flag (decimal)│
                       ├──────┼───────┼─────┼────────────────┤
                       │  1   │ Y   Y  │  11 │      3       │
                       │  2   │ Y   N  │  01 │      1       │
                       │  4   │ N   N  │  00 │      0       │
                       │  3   │ N   Y  │  10 │      2       │
                       │  5   │ N   N  │  00 │      0       │
                       └──────┴───────┴─────┴────────────────┘

 Values: [1, 2, 4, 3, 5]  (preorder)
 Flags:  [3, 1, 0, 2, 0]

 No null markers needed — flags tell decoder
 whether to recurse left/right!
 Savings: 2 bits vs full sentinel values.
```

---

## Example 10: Serialize/Deserialize with Verification

Complete solution with equality check and edge cases.

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type Codec struct{}

func (c *Codec) Serialize(root *TreeNode) string {
    var parts []string
    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil {
            parts = append(parts, "N")
            return
        }
        parts = append(parts, strconv.Itoa(node.Val))
        dfs(node.Left)
        dfs(node.Right)
    }
    dfs(root)
    return strings.Join(parts, ",")
}

func (c *Codec) Deserialize(data string) *TreeNode {
    parts := strings.Split(data, ",")
    idx := 0
    var build func() *TreeNode
    build = func() *TreeNode {
        if idx >= len(parts) || parts[idx] == "N" {
            idx++
            return nil
        }
        val, _ := strconv.Atoi(parts[idx])
        idx++
        return &TreeNode{val, build(), build()}
    }
    return build()
}

func isSame(a, b *TreeNode) bool {
    if a == nil && b == nil { return true }
    if a == nil || b == nil { return false }
    return a.Val == b.Val && isSame(a.Left, b.Left) && isSame(a.Right, b.Right)
}

func main() {
    codec := &Codec{}

    tests := []*TreeNode{
        nil,                               // empty
        {Val: 42},                         // single node
        {1, &TreeNode{2, nil, nil}, nil},  // left only
        {1, nil, &TreeNode{3, nil, nil}},  // right only
        {1, &TreeNode{Val: -2}, &TreeNode{3, &TreeNode{Val: 4}, &TreeNode{Val: 5}}},
    }

    for i, root := range tests {
        s := codec.Serialize(root)
        rebuilt := codec.Deserialize(s)
        ok := isSame(root, rebuilt)
        fmt.Printf("Test %d: serialized=%q, match=%v\n", i, s, ok)
    }
}
```

**Textual Figure – Example 10:**
```
 Serialize/Deserialize with Verification:
 ──────────────────────────────────

 Test cases and their serialized forms:
 ┌──────┬───────────────┬────────────────────┬───────┐
 │ Test │ Tree          │ Serialized           │ Match │
 ├──────┼───────────────┼────────────────────┼───────┤
 │  0   │ nil           │ "N"                  │ true  │
 │  1   │ [42]          │ "42,N,N"             │ true  │
 │  2   │ 1─L→2         │ "1,2,N,N,N"          │ true  │
 │  3   │ 1─R→3         │ "1,N,3,N,N"          │ true  │
 │  4   │ 1(-2)(3(4)(5))│ "1,-2,N,N,3,4,N,N,.." │ true  │
 └──────┴───────────────┴────────────────────┴───────┘

 Round-trip verification:
  tree → serialize → string → deserialize → tree'
  isSameTree(tree, tree') must be true for all cases.

 Edge cases handled:
  • nil root        → "N"
  • negative values  → "-2" parsed correctly
  • single node      → "42,N,N"
  • left-only/right-only subtrees
```

---

## Key Takeaways

| Approach | Pros | Cons |
|----------|------|------|
| **Preorder DFS** | Simple, handles all trees | Needs null markers |
| **BFS Level-order** | Matches LeetCode format | Trailing nulls waste space |
| **BST-specific** | No null markers needed | Only works for BSTs |
| **Binary encoding** | Compact | Not human-readable |
| **JSON** | Standard, human-readable | Verbose, slower |
| **Preorder+Inorder** | No null markers | Requires unique values |

1. **DFS preorder** is the most common interview approach — simple serialize/deserialize
2. **BSTs** only need values (no structure encoding) — sorted order reconstructs structure
3. Always handle edge cases: nil root, negative values, single-node trees
4. Verify with `isSameTree()` to confirm round-trip correctness

> **Next up:** Vertical Order Traversal →
