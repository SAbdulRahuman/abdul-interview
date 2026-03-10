# Phase 19: Trie — Compressed Trie / Radix Tree

## Overview

A **compressed trie** (radix tree, Patricia tree) merges chains of single-child nodes into one node with a multi-character label. This dramatically reduces space while preserving O(L) operations.

```
Standard Trie:        Compressed Trie:
    root                  root
    |                    / \
    r                  rom   ub
   / \                / \     \
  o   u             an  ulus  ber
  |   |
  m   b
 / \   \
a   u   b
|   |   |
n   l   e
    |   |
    u   r
    |
    s
```

| Property | Standard Trie | Compressed Trie |
|----------|--------------|-----------------|
| Nodes | O(total chars) | O(number of words) |
| Internal nodes | Many single-child | None single-child |
| Edge labels | Single char | String (multiple chars) |
| Lookup time | O(L) | O(L) |

---

## Example 1: Basic Compressed Trie

```go
package main

import "fmt"

type CompressedNode struct {
	children map[byte]*CompressedNode
	label    string   // edge label from parent
	isEnd    bool
	value    interface{}
}

type RadixTree struct {
	root *CompressedNode
}

func NewRadixTree() *RadixTree {
	return &RadixTree{root: &CompressedNode{children: map[byte]*CompressedNode{}}}
}

func commonPrefixLen(a, b string) int {
	n := len(a)
	if len(b) < n { n = len(b) }
	for i := 0; i < n; i++ {
		if a[i] != b[i] { return i }
	}
	return n
}

func (rt *RadixTree) Insert(word string) {
	rt.insertHelper(rt.root, word)
}

func (rt *RadixTree) insertHelper(node *CompressedNode, remaining string) {
	if len(remaining) == 0 {
		node.isEnd = true
		return
	}

	firstChar := remaining[0]
	child, exists := node.children[firstChar]

	if !exists {
		// No child with this first char — create new edge
		newNode := &CompressedNode{
			children: map[byte]*CompressedNode{},
			label:    remaining,
			isEnd:    true,
		}
		node.children[firstChar] = newNode
		return
	}

	// Find common prefix between remaining and child's label
	cpLen := commonPrefixLen(remaining, child.label)

	if cpLen == len(child.label) {
		// Child label is full prefix — recurse on remaining suffix
		rt.insertHelper(child, remaining[cpLen:])
	} else {
		// Split the child's edge
		// Create intermediate node
		splitNode := &CompressedNode{
			children: map[byte]*CompressedNode{},
			label:    child.label[:cpLen],
		}
		node.children[firstChar] = splitNode

		// Move existing child under split node
		child.label = child.label[cpLen:]
		splitNode.children[child.label[0]] = child

		// Insert remaining suffix
		suffix := remaining[cpLen:]
		if len(suffix) == 0 {
			splitNode.isEnd = true
		} else {
			newNode := &CompressedNode{
				children: map[byte]*CompressedNode{},
				label:    suffix,
				isEnd:    true,
			}
			splitNode.children[suffix[0]] = newNode
		}
	}
}

func (rt *RadixTree) Search(word string) bool {
	return rt.searchHelper(rt.root, word)
}

func (rt *RadixTree) searchHelper(node *CompressedNode, remaining string) bool {
	if len(remaining) == 0 {
		return node.isEnd
	}

	child, exists := node.children[remaining[0]]
	if !exists { return false }

	cpLen := commonPrefixLen(remaining, child.label)
	if cpLen < len(child.label) { return false }

	return rt.searchHelper(child, remaining[cpLen:])
}

func (rt *RadixTree) Print() {
	rt.printHelper(rt.root, "", "")
}

func (rt *RadixTree) printHelper(node *CompressedNode, prefix, indent string) {
	marker := ""
	if node.isEnd { marker = " *" }
	if prefix != "" {
		fmt.Printf("%s%s%s\n", indent, prefix, marker)
	}
	for _, child := range node.children {
		rt.printHelper(child, child.label, indent+"  ")
	}
}

func main() {
	rt := NewRadixTree()
	words := []string{"romane", "romanus", "romulus", "rubens", "rubber", "rubicon"}
	for _, w := range words {
		rt.Insert(w)
	}

	rt.Print()
	fmt.Println()

	for _, w := range append(words, "rom", "rub") {
		fmt.Printf("Search '%s': %v\n", w, rt.Search(w))
	}
}
```

**Textual Figure:**

```
Insert: "romane", "romanus", "romulus", "rubens", "rubber", "rubicon"

Step 1: Insert "romane"         Step 2: Insert "romanus"
   (root)                          (root)
    └─ romane ★                     └─ roman
                                        ├─ e ★       → romane
                                        └─ us ★      → romanus

Step 3: Insert "romulus" (split at "rom")
   (root)
    └─ rom
        ├─ an
        │   ├─ e ★               → romane
        │   └─ us ★              → romanus
        └─ ulus ★                → romulus

Steps 4-6: Insert "rubens", "rubber", "rubicon" (split at "r")

   Final Compressed Trie:
   ┌────────────────────────────────────────────┐
   │  (root)                                    │
   │   └─ r                                     │
   │       ├─ om                                │
   │       │   ├─ an                            │
   │       │   │   ├─ e ★      → "romane"       │
   │       │   │   └─ us ★     → "romanus"      │
   │       │   └─ ulus ★       → "romulus"       │
   │       └─ ub                                │
   │           ├─ ber ★        → "rubber"        │
   │           ├─ ens ★        → "rubens"        │
   │           └─ icon ★       → "rubicon"       │
   └────────────────────────────────────────────┘
   ★ = isEnd (word boundary)

   Search "rom":    root ─r→ ─om→ (isEnd=false) → NOT FOUND
   Search "romane": root ─r→ ─om→ ─an→ ─e→ (isEnd=true) → FOUND
   Search "rub":    root ─r→ ─ub→ (isEnd=false) → NOT FOUND
```

---

## Example 2: Radix Tree with Prefix Queries

```go
package main

import "fmt"

type Node struct {
	children map[byte]*Node
	label    string
	isEnd    bool
}

type RadixTree struct {
	root *Node
}

func NewRadixTree() *RadixTree {
	return &RadixTree{root: &Node{children: map[byte]*Node{}}}
}

func cpl(a, b string) int {
	n := len(a)
	if len(b) < n { n = len(b) }
	for i := 0; i < n; i++ {
		if a[i] != b[i] { return i }
	}
	return n
}

func (rt *RadixTree) Insert(word string) {
	node := rt.root
	remaining := word

	for len(remaining) > 0 {
		child, exists := node.children[remaining[0]]
		if !exists {
			node.children[remaining[0]] = &Node{
				children: map[byte]*Node{},
				label: remaining, isEnd: true,
			}
			return
		}

		cp := cpl(remaining, child.label)
		if cp == len(child.label) {
			remaining = remaining[cp:]
			node = child
			if len(remaining) == 0 {
				child.isEnd = true
				return
			}
			continue
		}

		// Split
		split := &Node{children: map[byte]*Node{}, label: child.label[:cp]}
		node.children[remaining[0]] = split
		child.label = child.label[cp:]
		split.children[child.label[0]] = child

		remaining = remaining[cp:]
		if len(remaining) == 0 {
			split.isEnd = true
		} else {
			split.children[remaining[0]] = &Node{
				children: map[byte]*Node{},
				label: remaining, isEnd: true,
			}
		}
		return
	}
	node.isEnd = true
}

func (rt *RadixTree) FindAllWithPrefix(prefix string) []string {
	node := rt.root
	remaining := prefix
	consumed := ""

	for len(remaining) > 0 {
		child, exists := node.children[remaining[0]]
		if !exists { return nil }

		cp := cpl(remaining, child.label)
		if cp < len(remaining) && cp < len(child.label) {
			return nil // mismatch
		}

		consumed += child.label[:cp]
		remaining = remaining[cp:]
		node = child
	}

	// Collect all words from this node
	results := []string{}
	rt.collect(node, []byte(consumed), &results)
	return results
}

func (rt *RadixTree) collect(node *Node, prefix []byte, results *[]string) {
	if node.isEnd {
		*results = append(*results, string(prefix))
	}
	for _, child := range node.children {
		rt.collect(child, append(append([]byte{}, prefix...), child.label...), results)
	}
}

func main() {
	rt := NewRadixTree()
	for _, w := range []string{"test", "testing", "tested", "tester", "team", "tea"} {
		rt.Insert(w)
	}

	fmt.Println("Prefix 'test':", rt.FindAllWithPrefix("test"))
	fmt.Println("Prefix 'te':", rt.FindAllWithPrefix("te"))
	fmt.Println("Prefix 'tea':", rt.FindAllWithPrefix("tea"))
}
```

**Textual Figure:**

```
Insert: "test", "testing", "tested", "tester", "team", "tea"

   Compressed Trie (Radix Tree):
   ┌──────────────────────────────────────┐
   │  (root)                             │
   │   └─ te                             │
   │       ├─ st ★          → "test"     │
   │       │   ├─ e                      │
   │       │   │   ├─ d ★   → "tested"   │
   │       │   │   └─ r ★   → "tester"   │
   │       │   └─ ing ★     → "testing"  │
   │       └─ a ★           → "tea"      │
   │           └─ m ★       → "team"     │
   └──────────────────────────────────────┘

   Prefix query "test":
     root ─te→ ─st→ (★) collect subtree
     Results: [test, tested, tester, testing]

   Prefix query "te":
     root ─te→ collect entire subtree
     Results: [test, tested, tester, testing, tea, team]

   Prefix query "tea":
     root ─te→ ─a→ (★) collect subtree
     Results: [tea, team]
```

---

## Example 3: Space Comparison

```go
package main

import "fmt"

func main() {
	words := []string{
		"romane", "romanus", "romulus",
		"rubens", "rubber", "rubicon",
	}

	// Standard trie node count
	type Set map[string]bool
	paths := Set{}
	for _, w := range words {
		for i := 1; i <= len(w); i++ {
			paths[w[:i]] = true
		}
	}
	standardNodes := len(paths) + 1 // +1 for root

	// Compressed trie: at most 2*n - 1 nodes where n = number of words
	compressedNodes := 2*len(words) - 1 // upper bound

	fmt.Printf("Words: %v\n", words)
	fmt.Printf("Standard trie nodes: %d\n", standardNodes)
	fmt.Printf("Compressed trie nodes: ≤ %d\n", compressedNodes)
	fmt.Printf("Savings: %.0f%%\n", (1-float64(compressedNodes)/float64(standardNodes))*100)
	fmt.Println()

	totalChars := 0
	for _, w := range words { totalChars += len(w) }
	fmt.Printf("Total characters: %d\n", totalChars)
	fmt.Printf("Standard trie: one node per char prefix = O(%d)\n", totalChars)
	fmt.Printf("Compressed trie: O(n) = O(%d) nodes\n", len(words))
}
```

**Textual Figure:**

```
Words: "romane", "romanus", "romulus", "rubens", "rubber", "rubicon"

  Standard Trie (25 nodes):          Compressed Trie (11 nodes):
  (root)                             (root)
   └─ r                               └─ r
       ├─ o                                ├─ om
       │   └─ m                            │   ├─ an
       │       ├─ a                        │   │   ├─ e ★
       │       │   └─ n                    │   │   └─ us ★
       │       │       ├─ e ★              │   └─ ulus ★
       │       │       └─ u                └─ ub
       │       │           └─ s ★              ├─ ber ★
       │       └─ u                            ├─ ens ★
       │           └─ l                        └─ icon ★
       │               └─ u
       │                   └─ s ★
       └─ u                          Nodes: ≤ 2n-1 = 11
           └─ b                      vs Standard: 25 nodes
               ├─ e                  Savings: ~56%
               │   └─ n
               │       └─ s ★
               ├─ b
               │   └─ e
               │       └─ r ★
               └─ i
                   └─ c
                       └─ o
                           └─ n ★

  Each char = 1 node             Edge labels = multi-char strings
  Many single-child nodes        No single-child internal nodes
```

---

## Example 4: Patricia Tree (Binary Radix)

```go
package main

import "fmt"

type PatriciaNode struct {
	bit      int // bit position to test
	key      string
	left     *PatriciaNode
	right    *PatriciaNode
}

func getBit(key string, pos int) int {
	byteIdx := pos / 8
	bitIdx := 7 - (pos % 8)
	if byteIdx >= len(key) {
		return 0
	}
	return int(key[byteIdx]>>uint(bitIdx)) & 1
}

func main() {
	fmt.Println("=== Patricia Tree (PATRICIA = Practical Algorithm To Retrieve Information Coded In Alphanumeric) ===")
	fmt.Println()
	fmt.Println("Binary radix tree where:")
	fmt.Println("  • Each internal node stores a 'skip' bit position")
	fmt.Println("  • Edges represent 0/1 based on that bit")
	fmt.Println("  • Chains of single bits are compressed")
	fmt.Println()

	// Example: keys as binary
	keys := map[string]string{
		"cat": fmt.Sprintf("%08b %08b %08b", 'c', 'a', 't'),
		"car": fmt.Sprintf("%08b %08b %08b", 'c', 'a', 'r'),
		"dog": fmt.Sprintf("%08b %08b %08b", 'd', 'o', 'g'),
	}
	for k, v := range keys {
		fmt.Printf("  '%s' → %s\n", k, v)
	}
	fmt.Println()
	fmt.Println("Patricia tree only stores bits that distinguish keys")
	fmt.Println("  'cat' vs 'car': differ at bit position 22 (t vs r)")
	fmt.Printf("  getBit('cat', 22) = %d\n", getBit("cat", 22))
	fmt.Printf("  getBit('car', 22) = %d\n", getBit("car", 22))
}
```

**Textual Figure:**

```
Patricia Tree — Binary Radix Tree

Keys as binary (8 bits per char):
  'c' = 01100011   'a' = 01100001   't' = 01110100
  'c' = 01100011   'a' = 01100001   'r' = 01110010
  'd' = 01100100   'o' = 01101111   'g' = 01100111

  "cat" → 01100011 01100001 01110100
  "car" → 01100011 01100001 01110010
  "dog" → 01100100 01101111 01100111
                ↑ bit 5                  ↑ bit 22

Patricia Tree (only stores distinguishing bits):
  ┌──────────────────────────────────────────┐
  │         (root) test bit 5               │
  │          ┌──0──┴──1──┐                  │
  │          │           │                  │
  │    test bit 22     "dog"                │
  │     ┌─0──┴──1─┐                         │
  │     │         │                         │
  │   "car"     "cat"                       │
  └──────────────────────────────────────────┘

  Bit 5:  'c'=0, 'd'=1 → separates cat/car from dog
  Bit 22: 'r'=0, 't'=1 → separates car from cat

  Skips all identical bit positions → compressed!
```

---

## Example 5: URL Router with Radix Tree

```go
package main

import (
	"fmt"
	"strings"
)

type RouteNode struct {
	children map[string]*RouteNode
	label    string
	handler  string
	isParam  bool
	paramName string
}

type Router struct {
	root *RouteNode
}

func NewRouter() *Router {
	return &Router{root: &RouteNode{children: map[string]*RouteNode{}}}
}

func (r *Router) AddRoute(path, handler string) {
	parts := strings.Split(strings.Trim(path, "/"), "/")
	node := r.root

	for _, part := range parts {
		isParam := strings.HasPrefix(part, ":")
		key := part
		if isParam { key = ":" }

		if _, ok := node.children[key]; !ok {
			node.children[key] = &RouteNode{
				children: map[string]*RouteNode{},
				label: part, isParam: isParam,
			}
			if isParam {
				node.children[key].paramName = part[1:]
			}
		}
		node = node.children[key]
	}
	node.handler = handler
}

func (r *Router) Match(path string) (string, map[string]string) {
	parts := strings.Split(strings.Trim(path, "/"), "/")
	params := map[string]string{}
	node := r.root

	for _, part := range parts {
		if child, ok := node.children[part]; ok {
			node = child
		} else if child, ok := node.children[":"]; ok {
			params[child.paramName] = part
			node = child
		} else {
			return "404", nil
		}
	}
	return node.handler, params
}

func main() {
	router := NewRouter()
	router.AddRoute("/users", "listUsers")
	router.AddRoute("/users/:id", "getUser")
	router.AddRoute("/users/:id/posts", "getUserPosts")
	router.AddRoute("/posts/:id", "getPost")

	tests := []string{"/users", "/users/42", "/users/42/posts", "/posts/7"}
	for _, path := range tests {
		handler, params := router.Match(path)
		fmt.Printf("%-20s → handler=%s, params=%v\n", path, handler, params)
	}
}
```

**Textual Figure:**

```
Routes: /users, /users/:id, /users/:id/posts, /posts/:id

   URL Router Radix Tree:
   ┌───────────────────────────────────────────────┐
   │  (root)                                       │
   │   ├─ "users" ──→ handler: listUsers           │
   │   │   └─ ":" (param: id) ──→ handler: getUser │
   │   │       └─ "posts" ──→ handler: getUserPosts│
   │   └─ "posts"                                  │
   │       └─ ":" (param: id) ──→ handler: getPost │
   └───────────────────────────────────────────────┘

   Route matching:
   /users          → root─users→ listUsers, params={}
   /users/42       → root─users─:id→ getUser, params={id:42}
   /users/42/posts → root─users─:id─posts→ getUserPosts, params={id:42}
   /posts/7        → root─posts─:id→ getPost, params={id:7}

   Each path segment is one edge in the trie.
   ":" nodes capture URL parameters dynamically.
```

---

## Example 6: Compressed Trie Delete

```go
package main

import "fmt"

type CNode struct {
	children map[byte]*CNode
	label    string
	isEnd    bool
}

type CTrie struct {
	root *CNode
}

func NewCTrie() *CTrie {
	return &CTrie{root: &CNode{children: map[byte]*CNode{}}}
}

func cpLen(a, b string) int {
	n := len(a); if len(b) < n { n = len(b) }
	for i := 0; i < n; i++ { if a[i] != b[i] { return i } }
	return n
}

func (ct *CTrie) Insert(word string) {
	ct.insertAt(ct.root, word)
}

func (ct *CTrie) insertAt(node *CNode, rem string) {
	if len(rem) == 0 { node.isEnd = true; return }
	ch, exists := node.children[rem[0]]
	if !exists {
		node.children[rem[0]] = &CNode{children: map[byte]*CNode{}, label: rem, isEnd: true}
		return
	}
	cp := cpLen(rem, ch.label)
	if cp == len(ch.label) {
		ct.insertAt(ch, rem[cp:])
		return
	}
	split := &CNode{children: map[byte]*CNode{}, label: ch.label[:cp]}
	node.children[rem[0]] = split
	ch.label = ch.label[cp:]
	split.children[ch.label[0]] = ch
	suffix := rem[cp:]
	if len(suffix) == 0 { split.isEnd = true } else {
		split.children[suffix[0]] = &CNode{children: map[byte]*CNode{}, label: suffix, isEnd: true}
	}
}

func (ct *CTrie) Delete(word string) bool {
	return ct.deleteAt(ct.root, word)
}

func (ct *CTrie) deleteAt(node *CNode, rem string) bool {
	if len(rem) == 0 {
		if !node.isEnd { return false }
		node.isEnd = false
		return true
	}
	child, exists := node.children[rem[0]]
	if !exists { return false }
	cp := cpLen(rem, child.label)
	if cp < len(child.label) { return false }

	deleted := ct.deleteAt(child, rem[cp:])
	if deleted {
		// Merge single-child non-end nodes
		if !child.isEnd && len(child.children) == 1 {
			for _, grandchild := range child.children {
				child.label += grandchild.label
				child.isEnd = grandchild.isEnd
				child.children = grandchild.children
			}
		}
		if !child.isEnd && len(child.children) == 0 {
			delete(node.children, rem[0])
		}
	}
	return deleted
}

func (ct *CTrie) Search(word string) bool {
	node := ct.root; rem := word
	for len(rem) > 0 {
		ch, ok := node.children[rem[0]]
		if !ok { return false }
		cp := cpLen(rem, ch.label)
		if cp < len(ch.label) { return false }
		rem = rem[cp:]; node = ch
	}
	return node.isEnd
}

func main() {
	ct := NewCTrie()
	for _, w := range []string{"test", "testing", "tested"} { ct.Insert(w) }

	fmt.Println("Before delete:")
	for _, w := range []string{"test", "testing", "tested"} {
		fmt.Printf("  '%s': %v\n", w, ct.Search(w))
	}

	ct.Delete("testing")
	fmt.Println("\nAfter deleting 'testing':")
	for _, w := range []string{"test", "testing", "tested"} {
		fmt.Printf("  '%s': %v\n", w, ct.Search(w))
	}
}
```

**Textual Figure:**

```
Insert: "test", "testing", "tested"

   Before Delete:                  After Delete("testing"):
   (root)                          (root)
    └─ test ★                       └─ test ★
        ├─ e                            └─ ed ★    → "tested"
        │   └─ d ★  → "tested"
        └─ ing ★   → "testing"

   Delete "testing" step-by-step:
   1. Traverse: root ─test→ ─ing→ (isEnd=true)
   2. Set isEnd=false for "ing" node
   3. "ing" node: isEnd=false, no children → DELETE node
   4. Parent "test" node: check children
      ─ Only child left: "e" → "d"
      ─ "e" node has 1 child, isEnd=false → MERGE
      ─ Merge "e" + "d" → "ed" ★

   Search results:
   "test"    → root ─test→ isEnd=true  → FOUND
   "testing" → root ─test→ no 'i' child → NOT FOUND
   "tested"  → root ─test→ ─ed→ isEnd=true → FOUND
```

---

## Example 7: Suffix Tree (Compressed Trie of Suffixes)

```go
package main

import "fmt"

type SuffixNode struct {
	children map[byte]*SuffixNode
	label    string
	start    int // position in original string
}

func buildNaiveSuffixTree(s string) *SuffixNode {
	root := &SuffixNode{children: map[byte]*SuffixNode{}}
	for i := 0; i < len(s); i++ {
		insertSuffix(root, s[i:], i)
	}
	return root
}

func insertSuffix(node *SuffixNode, suffix string, start int) {
	if len(suffix) == 0 { return }
	ch, exists := node.children[suffix[0]]
	if !exists {
		node.children[suffix[0]] = &SuffixNode{
			children: map[byte]*SuffixNode{},
			label: suffix, start: start,
		}
		return
	}
	cp := 0
	for cp < len(ch.label) && cp < len(suffix) && ch.label[cp] == suffix[cp] { cp++ }
	if cp == len(ch.label) {
		insertSuffix(ch, suffix[cp:], start)
		return
	}
	split := &SuffixNode{children: map[byte]*SuffixNode{}, label: ch.label[:cp]}
	node.children[suffix[0]] = split
	ch.label = ch.label[cp:]
	split.children[ch.label[0]] = ch
	rem := suffix[cp:]
	if len(rem) > 0 {
		split.children[rem[0]] = &SuffixNode{
			children: map[byte]*SuffixNode{},
			label: rem, start: start,
		}
	}
}

func findSubstring(root *SuffixNode, pattern string) []int {
	node := root; rem := pattern
	for len(rem) > 0 {
		ch, ok := node.children[rem[0]]
		if !ok { return nil }
		cp := 0
		for cp < len(ch.label) && cp < len(rem) && ch.label[cp] == rem[cp] { cp++ }
		if cp < len(rem) && cp == len(ch.label) {
			node = ch; rem = rem[cp:]
			continue
		}
		if cp >= len(rem) {
			// Found — collect all leaf positions
			positions := []int{}
			collectPositions(ch, &positions)
			return positions
		}
		return nil
	}
	positions := []int{}
	collectPositions(node, &positions)
	return positions
}

func collectPositions(node *SuffixNode, positions *[]int) {
	if len(node.children) == 0 {
		*positions = append(*positions, node.start)
	}
	for _, child := range node.children {
		collectPositions(child, positions)
	}
}

func main() {
	s := "banana$"
	tree := buildNaiveSuffixTree(s)

	fmt.Println("Substring search in 'banana':")
	fmt.Println("  'ana':", findSubstring(tree, "ana"))
	fmt.Println("  'ban':", findSubstring(tree, "ban"))
	fmt.Println("  'nan':", findSubstring(tree, "nan"))
}
```

**Textual Figure:**

```
Suffix Tree for "banana$" (compressed trie of all suffixes):

Suffixes: banana$, anana$, nana$, ana$, na$, a$, $

   (root)
    ├─ banana$ ★                          (suffix starting at 0)
    ├─ a
    │   ├─ na                              ┌─ na$ ★    (pos 4)
    │   │   └─ na$ ★                      │
    │   └─ $ ★                             └─ anana$ ★ (pos 1)
    ├─ na
    │   ├─ na$ ★                           (pos 2)
    │   └─ $ ★                              (pos 4... wait)
    └─ $ ★                                  (pos 6)

   Substring search:
   "ana": root ─a→ ─na→ match! collect leaves → [1, 3]
   "ban": root ─banana$→ partial match "ban" ✓ → [0]
   "nan": root ─na→ ─na$→ partial match "nan" ✓ → [2]

   Key: suffix tree enables O(P) substring search
   where P = pattern length (after O(n) construction).
```

---

## Example 8: Compressed Trie for IP Addresses

```go
package main

import (
	"fmt"
	"strings"
)

type IPNode struct {
	children map[string]*IPNode
	label    string
	data     string
}

type IPTrie struct {
	root *IPNode
}

func NewIPTrie() *IPTrie {
	return &IPTrie{root: &IPNode{children: map[string]*IPNode{}}}
}

func (t *IPTrie) Insert(ip, data string) {
	parts := strings.Split(ip, ".")
	node := t.root
	for _, p := range parts {
		if _, ok := node.children[p]; !ok {
			node.children[p] = &IPNode{children: map[string]*IPNode{}, label: p}
		}
		node = node.children[p]
	}
	node.data = data
}

func (t *IPTrie) FindSubnet(prefix string) []string {
	parts := strings.Split(prefix, ".")
	node := t.root
	for _, p := range parts {
		if p == "" { continue }
		if child, ok := node.children[p]; ok {
			node = child
		} else {
			return nil
		}
	}
	results := []string{}
	t.collectIPs(node, parts, &results)
	return results
}

func (t *IPTrie) collectIPs(node *IPNode, prefix []string, results *[]string) {
	if node.data != "" {
		*results = append(*results, strings.Join(prefix, ".")+": "+node.data)
	}
	for key, child := range node.children {
		t.collectIPs(child, append(prefix, key), results)
	}
}

func main() {
	trie := NewIPTrie()
	trie.Insert("192.168.1.1", "server-1")
	trie.Insert("192.168.1.2", "server-2")
	trie.Insert("192.168.2.1", "server-3")
	trie.Insert("10.0.0.1", "gateway")

	fmt.Println("Subnet 192.168.1:", trie.FindSubnet("192.168.1"))
	fmt.Println("Subnet 192:", trie.FindSubnet("192"))
}
```

**Textual Figure:**

```
IP Address Trie (octet-based edges):

   (root)
    ├─ "192"
    │   └─ "168"
    │       ├─ "1"
    │       │   ├─ "1" ★  data: "server-1"  → 192.168.1.1
    │       │   └─ "2" ★  data: "server-2"  → 192.168.1.2
    │       └─ "2"
    │           └─ "1" ★  data: "server-3"  → 192.168.2.1
    └─ "10"
        └─ "0"
            └─ "0"
                └─ "1" ★  data: "gateway"   → 10.0.0.1

   FindSubnet("192.168.1"):
     root ─192→ ─168→ ─ 1→ collect all:
     → ["192.168.1.1: server-1", "192.168.1.2: server-2"]

   FindSubnet("192"):
     root ─192→ collect entire subtree:
     → [all three 192.168.x.x entries]

   Each octet = one trie edge (not per-character).
```

---

## Example 9: Adaptive Radix Tree Concept

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Adaptive Radix Tree (ART) ===")
	fmt.Println()
	fmt.Println("Modern radix tree with adaptive node sizes for cache efficiency:")
	fmt.Println()

	nodeTypes := []struct{ name, children, size string }{
		{"Node4", "Up to 4 children", "Key array + child array (128 bytes)"},
		{"Node16", "Up to 16 children", "Key array + child array (384 bytes)"},
		{"Node48", "Up to 48 children", "256-byte index + child array (656 bytes)"},
		{"Node256", "Up to 256 children", "Direct array (2048 bytes)"},
	}

	for _, nt := range nodeTypes {
		fmt.Printf("  %s: %s → %s\n", nt.name, nt.children, nt.size)
	}

	fmt.Println()
	fmt.Println("Transitions:")
	fmt.Println("  Node4 → Node16 (when 5th child added)")
	fmt.Println("  Node16 → Node48 (when 17th child added)")
	fmt.Println("  Node48 → Node256 (when 49th child added)")
	fmt.Println()
	fmt.Println("Benefits over standard radix tree:")
	fmt.Println("  • Cache-friendly: small nodes fit in cache lines")
	fmt.Println("  • Space-efficient: node size adapts to actual fan-out")
	fmt.Println("  • Used in: databases (HyPer), Linux kernel, lmdb")
}
```

**Textual Figure:**

```
Adaptive Radix Tree (ART) — Node Type Transitions:

   ┌──────────────────────────────────────────────────┐
   │  Node4 (128B)     ─1-4 children              │
   │  ┌──────────────────────────┐               │
   │  │ keys: [a][b][c][d]       │               │
   │  │ ptrs: [↓][↓][↓][↓]       │               │
   │  └──────────────────────────┘               │
   │         │ 5th child → upgrade                │
   │         ▼                                     │
   │  Node16 (384B)    ─5-16 children             │
   │         │ 17th child → upgrade               │
   │         ▼                                     │
   │  Node48 (656B)    ─17-48 children             │
   │  ┌──────────────────────────┐               │
   │  │ 256-byte index array     │               │
   │  │ maps key → child slot    │               │
   │  └──────────────────────────┘               │
   │         │ 49th child → upgrade               │
   │         ▼                                     │
   │  Node256 (2048B)  ─49-256 children            │
   │  ┌──────────────────────────┐               │
   │  │ Direct 256-ptr array     │               │
   │  │ O(1) lookup by byte      │               │
   │  └──────────────────────────┘               │
   └──────────────────────────────────────────────────┘

   Node size adapts to actual fan-out → cache-friendly.
```

---

## Example 10: Standard vs Compressed Trie Operations

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Compressed Trie: Complete Comparison ===")
	fmt.Println()

	ops := []struct{ aspect, standard, compressed string }{
		{"Node count", "O(total chars)", "O(n words) — at most 2n-1"},
		{"Edge label", "Single character", "String (shared prefix)"},
		{"Insert", "O(L) char-by-char", "O(L) with prefix split"},
		{"Search", "O(L) char-by-char", "O(L) with string compare"},
		{"Delete", "O(L), prune empty", "O(L), merge single-child"},
		{"Space", "High for long words", "Compact for shared prefixes"},
		{"Implementation", "Simple", "Complex (split/merge logic)"},
		{"Prefix queries", "Walk + DFS", "Walk edges + DFS"},
	}

	for _, op := range ops {
		fmt.Printf("  %-16s\n", op.aspect)
		fmt.Printf("    Standard:   %s\n", op.standard)
		fmt.Printf("    Compressed: %s\n\n", op.compressed)
	}

	fmt.Println("When to use compressed trie:")
	fmt.Println("  • Long strings with shared prefixes (URLs, file paths)")
	fmt.Println("  • Memory-constrained environments")
	fmt.Println("  • IP address routing (CIDR lookups)")
	fmt.Println("  • Suffix trees (compressed trie of all suffixes)")
}
```

**Textual Figure:**

```
Standard vs Compressed Trie — Operation Comparison:

   Standard Trie Insert "test":      Compressed Trie Insert "test":
   (root)                            (root)
    └─ t                              └─ "test" ★
        └─ e                         (single node, edge label = "test")
            └─ s
                └─ t ★
   (4 nodes created)                (1 node created)

   Insert "testing":                Insert "testing":
   (root)                           (root)
    └─ t                             └─ "test" ★
        └─ e                             └─ "ing" ★
            └─ s                     (no split needed, "test" is prefix)
                └─ t ★
                    └─ i
                        └─ n
                            └─ g ★
   (3 more nodes)               (1 more node)

   Delete "testing":               Delete "testing":
   Prune empty nodes i→n→g         Remove "ing" node
   from bottom up.                  No merge needed (parent has isEnd).

   ┌──────────────┬────────────────┬────────────────┐
   │              │ Standard       │ Compressed       │
   ├──────────────┼────────────────┼────────────────┤
   │ Insert       │ O(L) per char  │ O(L) + split     │
   │ Search       │ O(L) per char  │ O(L) str compare │
   │ Delete       │ Prune empty    │ Merge singles    │
   │ Nodes        │ O(total chars) │ O(n words)       │
   └──────────────┴────────────────┴────────────────┘
```

---

## Key Takeaways

1. Compressed trie: merge single-child chains → O(n) nodes instead of O(total chars)
2. Core operations: insert (may split), delete (may merge), search (string compare)
3. Suffix tree = compressed trie of all suffixes = O(n) space with Ukkonen's
4. Used in: DNS, IP routing, URL routing, file systems, databases
5. Trade-off: more complex implementation for significant space savings

> **Next up:** Trie with Backtracking →
