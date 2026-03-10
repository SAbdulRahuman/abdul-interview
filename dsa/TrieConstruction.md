# Phase 19: Trie — Trie Construction

## Overview

A **Trie** (prefix tree) is a tree data structure for storing strings where each node represents a character. Common prefixes share the same path from root, making prefix-based operations O(L) where L is the word length.

```
        root
       / | \
      a   b   c
     /       |
    p        a
   / \       |
  p   e      t
  |
  l
  |
  e
```

| Operation | Time | Space |
|-----------|------|-------|
| Insert | O(L) | O(L) |
| Search | O(L) | O(1) |
| Prefix search | O(L) | O(1) |
| Delete | O(L) | O(1) |

---

## Example 1: Basic Trie with Array Children

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{}}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (t *Trie) Search(word string) bool {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return false
		}
		node = node.children[idx]
	}
	return node.isEnd
}

func main() {
	trie := NewTrie()
	words := []string{"apple", "app", "banana", "band"}
	for _, w := range words {
		trie.Insert(w)
	}

	fmt.Println(trie.Search("apple"))  // true
	fmt.Println(trie.Search("app"))    // true
	fmt.Println(trie.Search("ap"))     // false
	fmt.Println(trie.Search("banana")) // true
	fmt.Println(trie.Search("ban"))    // false
}
```

**Textual Figure:**

```
Insert: "apple", "app", "banana", "band"

   Trie (array-based, [26] children per node):
   ┌────────────────────────────────────────┐
   │  (root)                               │
   │   ├─ a                                │
   │   │   └─ p                             │
   │   │       └─ p ★        → "app"       │
   │   │           └─ l                     │
   │   │               └─ e ★  → "apple"   │
   │   └─ b                                │
   │       └─ a                             │
   │           └─ n                          │
   │               ├─ a                     │
   │               │   └─ n                 │
   │               │       └─ a ★ → "banana"│
   │               └─ d ★      → "band"     │
   └────────────────────────────────────────┘
   ★ = isEnd (word boundary)

   Search "apple":  root─a─p─p─l─e → isEnd=true  → FOUND
   Search "app":    root─a─p─p    → isEnd=true  → FOUND
   Search "ap":     root─a─p      → isEnd=false → NOT FOUND
   Search "ban":    root─b─a─n    → isEnd=false → NOT FOUND
```

---

## Example 2: Trie with HashMap Children

```go
package main

import "fmt"

type TrieNode struct {
	children map[rune]*TrieNode
	isEnd    bool
	count    int // words ending here
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{children: map[rune]*TrieNode{}}}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		if _, ok := node.children[ch]; !ok {
			node.children[ch] = &TrieNode{children: map[rune]*TrieNode{}}
		}
		node = node.children[ch]
	}
	node.isEnd = true
	node.count++
}

func (t *Trie) Search(word string) bool {
	node := t.root
	for _, ch := range word {
		if _, ok := node.children[ch]; !ok {
			return false
		}
		node = node.children[ch]
	}
	return node.isEnd
}

func main() {
	trie := NewTrie()
	// Supports any Unicode character
	trie.Insert("hello")
	trie.Insert("café")
	trie.Insert("日本語")

	fmt.Println(trie.Search("hello"))  // true
	fmt.Println(trie.Search("café"))   // true
	fmt.Println(trie.Search("日本語")) // true
	fmt.Println(trie.Search("日本"))   // false
}
```

**Textual Figure:**

```
Insert: "hello", "café", "日本語" (HashMap children → Unicode support)

   Trie (map[rune]*Node children):
   ┌───────────────────────────────────────┐
   │  (root)                              │
   │   ├─ 'h' ─ 'e' ─ 'l' ─ 'l' ─ 'o' ★  │
   │   │                    → "hello"     │
   │   ├─ 'c' ─ 'a' ─ 'f' ─ 'é' ★        │
   │   │                    → "café"      │
   │   └─ '日' ─ '本' ─ '語' ★           │
   │                    → "日本語"         │
   └───────────────────────────────────────┘

   Map-based nodes: each stores only the children that exist.
   Array-based would need [26] fixed slots (no Unicode).
   Map-based supports ANY character set.

   Search "日本": root─'日'─'本' → isEnd=false → NOT FOUND
   Search "日本語": root─'日'─'本'─'語' → isEnd=true → FOUND
```

---

## Example 3: Count Nodes and Words

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type Trie struct {
	root     *TrieNode
	wordCount int
	nodeCount int
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{}, nodeCount: 1}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
			t.nodeCount++
		}
		node = node.children[idx]
	}
	if !node.isEnd {
		node.isEnd = true
		t.wordCount++
	}
}

func main() {
	trie := NewTrie()
	words := []string{"apple", "app", "application", "banana"}
	for _, w := range words {
		trie.Insert(w)
	}

	fmt.Println("Words:", trie.wordCount) // 4
	fmt.Println("Nodes:", trie.nodeCount) // root + unique chars
	// "apple" creates: a,p,p,l,e = 5 new
	// "app" shares a,p,p = 0 new
	// "application" shares a,p,p,l + adds i,c,a,t,i,o,n = 7 new
	// "banana" creates b,a,n,a,n,a = 6 new
	// Total: 1 + 5 + 0 + 7 + 6 = 19
}
```

**Textual Figure:**

```
Insert: "apple", "app", "application", "banana"

   (root)  [nodeCount = 19]
    ├─ a                       Nodes created:
    │   └─ p                    "apple":  a,p,p,l,e     = 5 new
    │       └─ p ★              "app":    shares a,p,p  = 0 new
    │           └─ l            "application": shares a,p,p,l
    │               ├─ e ★                 adds i,c,a,t,i,o,n = 7 new
    │               └─ i      "banana": b,a,n,a,n,a = 6 new
    │                   └─ c
    │                       └─ a          Total: 1(root) + 5+0+7+6 = 19
    │                           └─ t      Words: 4
    │                               └─ i
    │                                   └─ o
    │                                       └─ n ★
    └─ b
        └─ a
            └─ n
                └─ a
                    └─ n
                        └─ a ★

   Shared prefix "appl" saves 4 nodes for "application".
   Each unique character prefix = one node in the trie.
```

---

## Example 4: Trie with Prefix Count

```go
package main

import "fmt"

type TrieNode struct {
	children    [26]*TrieNode
	isEnd       bool
	prefixCount int // how many words pass through
	wordCount   int // how many words end here
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{}}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
		node.prefixCount++
	}
	node.isEnd = true
	node.wordCount++
}

func (t *Trie) CountPrefix(prefix string) int {
	node := t.root
	for _, ch := range prefix {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return 0
		}
		node = node.children[idx]
	}
	return node.prefixCount
}

func (t *Trie) CountWord(word string) int {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return 0
		}
		node = node.children[idx]
	}
	return node.wordCount
}

func main() {
	trie := NewTrie()
	trie.Insert("apple")
	trie.Insert("app")
	trie.Insert("application")
	trie.Insert("apt")

	fmt.Println("Words starting with 'app':", trie.CountPrefix("app")) // 3
	fmt.Println("Words starting with 'ap':", trie.CountPrefix("ap"))   // 4
	fmt.Println("Exact 'app':", trie.CountWord("app"))                 // 1
}
```

**Textual Figure:**

```
Insert: "apple", "app", "application", "apt"

   (root)
    └─ a (pCnt=4)
        └─ p (pCnt=4)       ← CountPrefix("ap") = 4
            ├─ p (pCnt=3)   ← CountPrefix("app") = 3
            │     wCnt=1    ← CountWord("app") = 1 ★
            │   └─ l (pCnt=2)
            │       ├─ e (pCnt=1, wCnt=1) ★ → "apple"
            │       └─ i (pCnt=1)
            │           └─ c ─ a ─ t ─ i ─ o ─ n ★
            │                               → "application"
            └─ t (pCnt=1, wCnt=1) ★ → "apt"

   pCnt = prefixCount (words passing through this node)
   wCnt = wordCount (words ending at this node)

   CountPrefix("app") = 3: apple + app + application
   CountPrefix("ap")  = 4: apple + app + application + apt
   CountWord("app")   = 1: only "app" ends here
```

---

## Example 5: Delete from Trie

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{}}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (t *Trie) Delete(word string) bool {
	return t.deleteHelper(t.root, word, 0)
}

func (t *Trie) deleteHelper(node *TrieNode, word string, depth int) bool {
	if node == nil {
		return false
	}

	if depth == len(word) {
		if !node.isEnd {
			return false
		}
		node.isEnd = false
		return t.isEmpty(node)
	}

	idx := word[depth] - 'a'
	shouldDelete := t.deleteHelper(node.children[idx], word, depth+1)

	if shouldDelete {
		node.children[idx] = nil
		return !node.isEnd && t.isEmpty(node)
	}
	return false
}

func (t *Trie) isEmpty(node *TrieNode) bool {
	for _, child := range node.children {
		if child != nil {
			return false
		}
	}
	return true
}

func (t *Trie) Search(word string) bool {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil { return false }
		node = node.children[idx]
	}
	return node.isEnd
}

func main() {
	trie := NewTrie()
	trie.Insert("apple")
	trie.Insert("app")

	fmt.Println(trie.Search("apple")) // true
	trie.Delete("apple")
	fmt.Println(trie.Search("apple")) // false
	fmt.Println(trie.Search("app"))   // true (prefix still exists)
}
```

**Textual Figure:**

```
Insert: "apple", "app" then Delete("apple")

   Before Delete:               After Delete("apple"):
   (root)                       (root)
    └─ a                        └─ a
        └─ p                       └─ p
            └─ p ★  → "app"          └─ p ★  → "app"
                └─ l                (l, e pruned ↑)
                    └─ e ★ → "apple"

   Delete "apple" traversal (recursive, depth-first):
   ┌──────┬────────┬───────────┬─────────────────────────┐
   │ Depth│ Node   │ Action    │ Reason                  │
   ├──────┼────────┼───────────┼─────────────────────────┤
   │  5   │ 'e'    │ isEnd=F   │ Unmark word end          │
   │  5   │ 'e'    │ DELETE    │ No children, not isEnd   │
   │  4   │ 'l'    │ DELETE    │ No children, not isEnd   │
   │  3   │ 'p'[1] │ KEEP      │ isEnd=true (★ "app")      │
   └──────┴────────┴───────────┴─────────────────────────┘
```

---

## Example 6: Get All Words in Trie

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{}}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (t *Trie) GetAllWords() []string {
	words := []string{}
	t.collect(t.root, []byte{}, &words)
	return words
}

func (t *Trie) collect(node *TrieNode, prefix []byte, words *[]string) {
	if node.isEnd {
		*words = append(*words, string(prefix))
	}
	for i := 0; i < 26; i++ {
		if node.children[i] != nil {
			t.collect(node.children[i], append(prefix, byte('a'+i)), words)
		}
	}
}

func main() {
	trie := NewTrie()
	for _, w := range []string{"cat", "car", "card", "care", "bat", "bar"} {
		trie.Insert(w)
	}
	fmt.Println("All words:", trie.GetAllWords())
	// [bar bat car card care cat] — lexicographic order!
}
```

**Textual Figure:**

```
Insert: "cat", "car", "card", "care", "bat", "bar"

   (root)
    ├─ b
    │   └─ a
    │       ├─ r ★         → "bar"
    │       └─ t ★         → "bat"
    └─ c
        └─ a
            ├─ r ★         → "car"
            │   ├─ d ★     → "card"
            │   └─ e ★     → "care"
            └─ t ★         → "cat"

   DFS collect traversal (lexicographic order):
   root ─b→ ─a→ ─r★ → "bar"
                  ─t★ → "bat"
        ─c→ ─a→ ─r★ → "car"
                   ─d★ → "card"
                   ─e★ → "care"
                  ─t★ → "cat"

   Result: [bar, bat, car, card, care, cat]
   Tries naturally yield lexicographic order via DFS!
```

---

## Example 7: Longest Common Prefix via Trie

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
	count    int // number of children
}

func longestCommonPrefix(strs []string) string {
	if len(strs) == 0 {
		return ""
	}

	root := &TrieNode{}

	// Insert all strings
	for _, s := range strs {
		node := root
		for _, ch := range s {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
				node.count++
			}
			node = node.children[idx]
		}
		node.isEnd = true
	}

	// Walk until branching or word end
	prefix := []byte{}
	node := root
	for node.count == 1 && !node.isEnd {
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				prefix = append(prefix, byte('a'+i))
				node = node.children[i]
				break
			}
		}
	}
	return string(prefix)
}

func main() {
	fmt.Println(longestCommonPrefix([]string{"flower", "flow", "flight"})) // "fl"
	fmt.Println(longestCommonPrefix([]string{"dog", "racecar", "car"}))    // ""
}
```

**Textual Figure:**

```
Longest Common Prefix for ["flower", "flow", "flight"]:

   (root)
    └─ f   ← count=1, single child → continue
        └─ l   ← count=1, single child → continue
            ├─ o   ← count=2 → STOP (branching!)
            │   └─ w ★ → "flow"
            │       └─ e
            │           └─ r ★ → "flower"
            └─ i   ← (second branch)
                └─ g
                    └─ h
                        └─ t ★ → "flight"

   Walk from root while only 1 child and !isEnd:
   root ─f→ (1 child) ─l→ (2 children) → STOP
   LCP = "fl"

   For ["dog", "racecar", "car"]:
   root has 3 children (c, d, r) → STOP immediately
   LCP = ""
```

---

## Example 8: Trie with Value Storage

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
	value    interface{}
}

type TrieMap struct {
	root *TrieNode
}

func NewTrieMap() *TrieMap {
	return &TrieMap{root: &TrieNode{}}
}

func (t *TrieMap) Put(key string, val interface{}) {
	node := t.root
	for _, ch := range key {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
	node.value = val
}

func (t *TrieMap) Get(key string) (interface{}, bool) {
	node := t.root
	for _, ch := range key {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return nil, false
		}
		node = node.children[idx]
	}
	if node.isEnd {
		return node.value, true
	}
	return nil, false
}

func main() {
	m := NewTrieMap()
	m.Put("hello", 42)
	m.Put("help", "assistance")
	m.Put("world", 3.14)

	if v, ok := m.Get("hello"); ok { fmt.Println("hello:", v) }
	if v, ok := m.Get("help"); ok { fmt.Println("help:", v) }
	if _, ok := m.Get("hel"); !ok { fmt.Println("hel: not found") }
}
```

**Textual Figure:**

```
TrieMap: Put("hello", 42), Put("help", "assistance"), Put("world", 3.14)

   (root)
    ├─ h
    │   └─ e
    │       └─ l
    │           ├─ l
    │           │   └─ o ★  value: 42         → "hello"
    │           └─ p ★      value: "assistance" → "help"
    └─ w
        └─ o
            └─ r
                └─ l
                    └─ d ★  value: 3.14      → "world"

   Get("hello"): root─h─e─l─l─o → isEnd=true, value=42
   Get("help"):  root─h─e─l─p   → isEnd=true, value="assistance"
   Get("hel"):   root─h─e─l     → isEnd=false → NOT FOUND

   Trie as a map: O(L) lookup independent of # entries.
```

---

## Example 9: Trie Construction from Sentences

```go
package main

import (
	"fmt"
	"strings"
)

type TrieNode struct {
	children map[string]*TrieNode
	isEnd    bool
	count    int
}

type WordTrie struct {
	root *TrieNode
}

func NewWordTrie() *WordTrie {
	return &WordTrie{root: &TrieNode{children: map[string]*TrieNode{}}}
}

func (t *WordTrie) InsertSentence(sentence string) {
	words := strings.Fields(sentence)
	node := t.root
	for _, word := range words {
		if _, ok := node.children[word]; !ok {
			node.children[word] = &TrieNode{children: map[string]*TrieNode{}}
		}
		node = node.children[word]
	}
	node.isEnd = true
	node.count++
}

func (t *WordTrie) AutoComplete(prefix string) []string {
	words := strings.Fields(prefix)
	node := t.root
	for _, word := range words {
		if _, ok := node.children[word]; !ok {
			return nil
		}
		node = node.children[word]
	}

	results := []string{}
	t.collectSentences(node, words, &results)
	return results
}

func (t *WordTrie) collectSentences(node *TrieNode, prefix []string, results *[]string) {
	if node.isEnd {
		*results = append(*results, strings.Join(prefix, " "))
	}
	for word, child := range node.children {
		t.collectSentences(child, append(prefix, word), results)
	}
}

func main() {
	trie := NewWordTrie()
	trie.InsertSentence("i love go")
	trie.InsertSentence("i love golang")
	trie.InsertSentence("i love programming")
	trie.InsertSentence("i am learning")

	fmt.Println("Autocomplete 'i love':", trie.AutoComplete("i love"))
	fmt.Println("Autocomplete 'i':", trie.AutoComplete("i"))
}
```

**Textual Figure:**

```
Word-level Trie from sentences:
"i love go", "i love golang", "i love programming", "i am learning"

   (root)
    └─ "i"
        ├─ "love"
        │   ├─ "go" ★            → "i love go"
        │   ├─ "golang" ★        → "i love golang"
        │   └─ "programming" ★   → "i love programming"
        └─ "am"
            └─ "learning" ★      → "i am learning"

   Each edge = a whole word (not a character).
   Uses map[string]*Node for word-level children.

   AutoComplete("i love"):
     root ─"i"→ ─"love"→ collect subtree
     → ["i love go", "i love golang", "i love programming"]

   AutoComplete("i"):
     root ─"i"→ collect entire subtree
     → [all 4 sentences]
```

---

## Example 10: Trie Space Analysis

```go
package main

import (
	"fmt"
	"unsafe"
)

type TrieNodeArray struct {
	children [26]*TrieNodeArray
	isEnd    bool
}

type TrieNodeMap struct {
	children map[rune]*TrieNodeMap
	isEnd    bool
}

func main() {
	fmt.Println("=== Trie Construction: Space Analysis ===")
	fmt.Println()

	arrayNodeSize := unsafe.Sizeof(TrieNodeArray{})
	mapNodeSize := unsafe.Sizeof(TrieNodeMap{})

	fmt.Printf("Array-based node size: %d bytes\n", arrayNodeSize)
	fmt.Printf("Map-based node size: %d bytes (+ map overhead)\n", mapNodeSize)
	fmt.Println()

	fmt.Println("Array children [26]*Node:")
	fmt.Println("  + O(1) child lookup")
	fmt.Println("  + Cache-friendly")
	fmt.Println("  - Wastes space for sparse nodes (26 pointers per node)")
	fmt.Println("  Best for: lowercase English letters, dense tries")
	fmt.Println()

	fmt.Println("Map children map[rune]*Node:")
	fmt.Println("  + Space-efficient for sparse nodes")
	fmt.Println("  + Supports any character set")
	fmt.Println("  - O(1) average but with hash overhead")
	fmt.Println("  Best for: Unicode, large alphabets, sparse tries")
	fmt.Println()

	fmt.Println("Space comparison for N words of avg length L:")
	fmt.Println("  Array: O(ALPHABET × total_nodes) — could be O(26 × N × L)")
	fmt.Println("  Map: O(total_unique_chars) — proportional to actual content")
}
```

**Textual Figure:**

```
Trie Space: Array vs Map Children

   Array-based node [26]*Node:      Map-based node map[rune]*Node:
   ┌────────────────────────┐    ┌────────────────────┐
   │ [a][b][c]...[z]        │    │ {a:↓, c:↓}          │
   │  ↓  ·  ↓       ·       │    │  only used keys     │
   │ 26 pointers always    │    │  + hash overhead    │
   └────────────────────────┘    └────────────────────┘
   ~208 bytes/node               Variable, ~8 bytes/entry

   Example: node with 2 children ('a' and 'c')
   Array: 26 slots, 24 wasted    Map: 2 entries, no waste

   ┌─────────────┬───────────────┬───────────────┐
   │             │ Array [26]     │ Map            │
   ├─────────────┼───────────────┼───────────────┤
   │ Lookup      │ O(1) direct   │ O(1) avg hash  │
   │ Space/node  │ 26 × 8 bytes  │ k × ~16 bytes  │
   │ Charset     │ a-z only      │ Any Unicode    │
   │ Cache       │ Friendly      │ Less friendly  │
   └─────────────┴───────────────┴───────────────┘
```

---

## Key Takeaways

1. Trie = tree where each edge represents a character
2. Array children: fast, fixed memory per node (26 pointers for lowercase)
3. Map children: flexible, supports Unicode, space-efficient for sparse data
4. Insert/Search/Delete all O(L) where L = word length
5. Tries naturally produce lexicographic ordering of stored words

> **Next up:** Trie Search and Insert →
