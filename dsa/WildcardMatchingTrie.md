# Phase 19: Trie — Wildcard Matching

## Overview

**Wildcard matching in tries** supports pattern searches with special characters like `.` (any single character) and `*` (any sequence). When encountering a wildcard, we branch into multiple trie paths simultaneously.

| Wildcard | Meaning | Trie Approach |
|----------|---------|---------------|
| `.` | Any single char | Try all 26 children |
| `*` | Any sequence | Recursive: skip 0+ chars |
| `?` | Same as `.` | Try all children |

---

## Example 1: Design Add and Search Words (LeetCode 211)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type WordDictionary struct {
	root *TrieNode
}

func Constructor() WordDictionary {
	return WordDictionary{root: &TrieNode{}}
}

func (wd *WordDictionary) AddWord(word string) {
	node := wd.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (wd *WordDictionary) Search(word string) bool {
	return wd.searchHelper(wd.root, word, 0)
}

func (wd *WordDictionary) searchHelper(node *TrieNode, word string, idx int) bool {
	if idx == len(word) {
		return node.isEnd
	}

	ch := word[idx]
	if ch == '.' {
		// Try all children
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				if wd.searchHelper(node.children[i], word, idx+1) {
					return true
				}
			}
		}
		return false
	}

	child := node.children[ch-'a']
	if child == nil {
		return false
	}
	return wd.searchHelper(child, word, idx+1)
}

func main() {
	wd := Constructor()
	wd.AddWord("bad")
	wd.AddWord("dad")
	wd.AddWord("mad")

	fmt.Println(wd.Search("pad"))  // false
	fmt.Println(wd.Search("bad"))  // true
	fmt.Println(wd.Search(".ad"))  // true
	fmt.Println(wd.Search("b.."))  // true
	fmt.Println(wd.Search("..."))  // true (matches bad, dad, mad)
	fmt.Println(wd.Search("....")) // false (no 4-letter words)
}
```

**Textual Figure:**

```
AddWord: "bad", "dad", "mad"

   Trie:
   (root)
    ├─ b ─ a ─ d ★   → "bad"
    ├─ d ─ a ─ d ★   → "dad"
    └─ m ─ a ─ d ★   → "mad"

   Search with '.' wildcard (try all 26 children):
   ┌─────────┬───────────────────────────────────┐
   │ Pattern │ Trie traversal                    │
   ├─────────┼───────────────────────────────────┤
   │ "pad"   │ p→no child → false               │
   │ "bad"   │ b─a─d ★ → true                  │
   │ ".ad"   │ .→try b,d,m: b─a─d★ → true       │
   │ "b.."   │ b─.→a─.→d★ → true               │
   │ "..."   │ .→b/d/m, .→a, .→d ★ → true      │
   │ "...."  │ 4 dots but words are length 3 → F │
   └─────────┴───────────────────────────────────┘
   '.' = branch into all 26 possible children.
```

---

## Example 2: Wildcard Match with Star

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type WildcardTrie struct {
	root *TrieNode
}

func NewWildcardTrie() *WildcardTrie {
	return &WildcardTrie{root: &TrieNode{}}
}

func (wt *WildcardTrie) Insert(word string) {
	node := wt.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (wt *WildcardTrie) Match(pattern string) []string {
	results := []string{}
	wt.matchHelper(wt.root, pattern, 0, []byte{}, &results)
	return results
}

func (wt *WildcardTrie) matchHelper(node *TrieNode, pattern string, pi int, current []byte, results *[]string) {
	if pi == len(pattern) {
		if node.isEnd {
			*results = append(*results, string(current))
		}
		return
	}

	ch := pattern[pi]
	switch ch {
	case '.', '?':
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				wt.matchHelper(node.children[i], pattern, pi+1,
					append(append([]byte{}, current...), byte('a'+i)), results)
			}
		}
	case '*':
		// Match 0 characters
		wt.matchHelper(node, pattern, pi+1, current, results)
		// Match 1+ characters
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				wt.matchHelper(node.children[i], pattern, pi,
					append(append([]byte{}, current...), byte('a'+i)), results)
			}
		}
	default:
		idx := ch - 'a'
		if node.children[idx] != nil {
			wt.matchHelper(node.children[idx], pattern, pi+1,
				append(append([]byte{}, current...), ch), results)
		}
	}
}

func main() {
	wt := NewWildcardTrie()
	for _, w := range []string{"cat", "car", "card", "care", "bat", "bar", "cattle"} {
		wt.Insert(w)
	}

	fmt.Println("ca.:", wt.Match("ca."))    // [car cat]
	fmt.Println("c*:", wt.Match("c*"))      // [car card care cat cattle]
	fmt.Println("ca*e:", wt.Match("ca*e"))  // [care cattle]
}
```

**Textual Figure:**

```
Words: ["cat","car","card","care","bat","bar","cattle"]

   Trie:
   (root)
    ├─ b ─ a
    │       ├─ r ★  → "bar"
    │       └─ t ★  → "bat"
    └─ c ─ a
            ├─ r ★  → "car"
            │   ├─ d ★ → "card"
            │   └─ e ★ → "care"
            └─ t ★  → "cat"
                └─ t ─ l ─ e ★ → "cattle"

   Pattern matching:
   "ca." ('.'→try all):  c─a─[r★, t★] → [car, cat]

   "c*" ('*'→match 0+ chars):
     c → match 0: not isEnd
     c─a → match 1: not isEnd
     c─a─r ★, c─a─r─d ★, c─a─r─e ★, c─a─t ★, c─a─t─t─l─e ★
     → [car, card, care, cat, cattle]

   "ca*e" ('*'→any seq ending in 'e'):
     c─a → *matches r─e → "care" ★
     c─a → *matches t─t─l─e → "cattle" ★
     → [care, cattle]
```

---

## Example 3: Regex-Like Search (Character Classes)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type RegexTrie struct {
	root *TrieNode
}

func NewRegexTrie() *RegexTrie {
	return &RegexTrie{root: &TrieNode{}}
}

func (rt *RegexTrie) Insert(word string) {
	node := rt.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

// Pattern supports [abc] character classes and . wildcard
func (rt *RegexTrie) Search(pattern string) []string {
	results := []string{}
	rt.searchDFS(rt.root, pattern, 0, []byte{}, &results)
	return results
}

func (rt *RegexTrie) searchDFS(node *TrieNode, pattern string, pi int, current []byte, results *[]string) {
	if pi >= len(pattern) {
		if node.isEnd {
			*results = append(*results, string(current))
		}
		return
	}

	if pattern[pi] == '[' {
		// Parse character class [abc]
		end := pi + 1
		for end < len(pattern) && pattern[end] != ']' {
			end++
		}
		chars := pattern[pi+1 : end]
		for _, ch := range chars {
			idx := ch - 'a'
			if idx >= 0 && idx < 26 && node.children[idx] != nil {
				cp := make([]byte, len(current))
				copy(cp, current)
				rt.searchDFS(node.children[idx], pattern, end+1,
					append(cp, byte(ch)), results)
			}
		}
	} else if pattern[pi] == '.' {
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				cp := make([]byte, len(current))
				copy(cp, current)
				rt.searchDFS(node.children[i], pattern, pi+1,
					append(cp, byte('a'+i)), results)
			}
		}
	} else {
		idx := pattern[pi] - 'a'
		if node.children[idx] != nil {
			rt.searchDFS(node.children[idx], pattern, pi+1,
				append(current, pattern[pi]), results)
		}
	}
}

func main() {
	rt := NewRegexTrie()
	for _, w := range []string{"cat", "car", "cut", "cup", "bat", "but"} {
		rt.Insert(w)
	}

	fmt.Println("[cb]at:", rt.Search("[cb]at"))   // [bat cat]
	fmt.Println("c[au]t:", rt.Search("c[au]t"))   // [cat cut]
	fmt.Println(".ut:", rt.Search(".ut"))          // [but cut]
}
```

**Textual Figure:**

```
Words: ["cat","car","cut","cup","bat","but"]

   Trie:
   (root)
    ├─ b
    │   ├─ a ─ t ★  → "bat"
    │   └─ u ─ t ★  → "but"
    └─ c
        ├─ a
        │   ├─ r ★     → "car"
        │   └─ t ★     → "cat"
        └─ u
            ├─ p ★     → "cup"
            └─ t ★     → "cut"

   Regex-like pattern matching:
   "[cb]at": parse [cb] → try 'c' and 'b' at pos 0
     c─a─t ★ → "cat"     b─a─t ★ → "bat"
     Result: [bat, cat]

   "c[au]t": c → parse [au] → try 'a' and 'u' at pos 1
     c─a─t ★ → "cat"     c─u─t ★ → "cut"
     Result: [cat, cut]

   ".ut": '.' at pos 0 → try all 26
     b─u─t ★ → "but"     c─u─t ★ → "cut"
     Result: [but, cut]
```

---

## Example 4: Fuzzy Search (One Edit Distance)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type FuzzyTrie struct {
	root *TrieNode
}

func NewFuzzyTrie() *FuzzyTrie {
	return &FuzzyTrie{root: &TrieNode{}}
}

func (ft *FuzzyTrie) Insert(word string) {
	node := ft.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (ft *FuzzyTrie) FuzzySearch(word string, maxEdits int) []string {
	results := map[string]bool{}
	ft.fuzzyDFS(ft.root, word, 0, maxEdits, []byte{}, results)
	res := []string{}
	for w := range results { res = append(res, w) }
	return res
}

func (ft *FuzzyTrie) fuzzyDFS(node *TrieNode, target string, pos, editsLeft int, current []byte, results map[string]bool) {
	if editsLeft < 0 { return }

	if pos == len(target) {
		if node.isEnd {
			results[string(current)] = true
		}
		// Allow deletions at end
		if editsLeft > 0 {
			for i := 0; i < 26; i++ {
				if node.children[i] != nil {
					ft.fuzzyDFS(node.children[i], target, pos, editsLeft-1,
						append(append([]byte{}, current...), byte('a'+i)), results)
				}
			}
		}
		return
	}

	// Exact match
	idx := target[pos] - 'a'
	if node.children[idx] != nil {
		ft.fuzzyDFS(node.children[idx], target, pos+1, editsLeft,
			append(append([]byte{}, current...), target[pos]), results)
	}

	if editsLeft > 0 {
		// Substitution: try all other characters
		for i := 0; i < 26; i++ {
			if i != int(idx) && node.children[i] != nil {
				ft.fuzzyDFS(node.children[i], target, pos+1, editsLeft-1,
					append(append([]byte{}, current...), byte('a'+i)), results)
			}
		}
		// Insertion: skip a trie character
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				ft.fuzzyDFS(node.children[i], target, pos, editsLeft-1,
					append(append([]byte{}, current...), byte('a'+i)), results)
			}
		}
		// Deletion: skip a target character
		ft.fuzzyDFS(node, target, pos+1, editsLeft-1, current, results)
	}
}

func main() {
	ft := NewFuzzyTrie()
	for _, w := range []string{"hello", "help", "heap", "heal", "hero", "halo"} {
		ft.Insert(w)
	}
	fmt.Println("Fuzzy 'helo' (1 edit):", ft.FuzzySearch("helo", 1))
	// [hello hero help heal]
}
```

**Textual Figure:**

```
Words: ["hello","help","heap","heal","hero","halo"]

   Trie:
   (root)
    └─ h
        ├─ e
        │   ├─ a
        │   │   ├─ l ★  → "heal"
        │   │   └─ p ★  → "heap"
        │   ├─ l
        │   │   ├─ l ─ o ★  → "hello"
        │   │   └─ p ★      → "help"
        │   └─ r ─ o ★      → "hero"
        └─ a ─ l ─ o ★      → "halo"

   Fuzzy search "helo" with maxEdits=1:
   ┌──────────┬──────────────────────────────────┐
   │ Match    │ Edit operation               │
   ├──────────┼──────────────────────────────────┤
   │ "hello"  │ Insert 'l': helo → hel+l+o    │
   │ "hero"   │ Substitute: helo → hero (l→r) │
   │ "help"   │ Substitute: helo → help (o→p) │
   │ "heal"   │ Substitute: helo → heal (l→a) │
   └──────────┴──────────────────────────────────┘
   DFS tries all 3 edit types at each position.
```

---

## Example 5: Pattern Search with Fixed Length

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

// searchPattern: letters and dots, fixed length
func (t *Trie) searchPattern(pattern string) int {
	return t.countMatches(t.root, pattern, 0)
}

func (t *Trie) countMatches(node *TrieNode, pattern string, idx int) int {
	if idx == len(pattern) {
		if node.isEnd { return 1 }
		return 0
	}

	count := 0
	if pattern[idx] == '.' {
		for i := 0; i < 26; i++ {
			if node.children[i] != nil {
				count += t.countMatches(node.children[i], pattern, idx+1)
			}
		}
	} else {
		ci := pattern[idx] - 'a'
		if node.children[ci] != nil {
			count += t.countMatches(node.children[ci], pattern, idx+1)
		}
	}
	return count
}

func main() {
	trie := NewTrie()
	for _, w := range []string{"cat", "car", "bat", "bar", "cap", "cup"} {
		trie.Insert(w)
	}

	fmt.Println("c.t matches:", trie.searchPattern("c.t")) // 1 (cat)
	fmt.Println("..t matches:", trie.searchPattern("..t")) // 2 (cat, bat)
	fmt.Println("ca. matches:", trie.searchPattern("ca.")) // 3 (cat, car, cap)
	fmt.Println("... matches:", trie.searchPattern("...")) // 6 (all)
}
```

**Textual Figure:**

```
Words: ["cat","car","bat","bar","cap","cup"]

   Trie:
   (root)
    ├─ b ─ a
    │       ├─ r ★  → "bar"
    │       └─ t ★  → "bat"
    └─ c
        ├─ a
        │   ├─ p ★  → "cap"
        │   ├─ r ★  → "car"
        │   └─ t ★  → "cat"
        └─ u ─ p ★  → "cup"

   Pattern search (fixed length, '.' = any char):
   "c.t": c → .(try a,u) → t
          c─a─t ★ → count 1  ("cat")
          c─u─(no t) → skip
          Result: 1

   "..t": .(b,c) → .(a,u) → t
          b─a─t ★, c─a─t ★ → count 2
          Result: 2

   "ca.": c → a → .(p,r,t)
          all 3 are isEnd → count 3  (cap, car, cat)

   "...": all 26 → all 26 → all 26
          count every 3-letter word = 6
```

---

## Example 6: Multi-Pattern Search (Aho-Corasick Simplified)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	output   []string
}

func multiPatternSearch(text string, patterns []string) map[string][]int {
	// Build trie from patterns
	root := &TrieNode{}
	for _, p := range patterns {
		node := root
		for _, ch := range p {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
		}
		node.output = append(node.output, p)
	}

	// Naive search: at each position, try all patterns via trie
	result := map[string][]int{}
	for i := 0; i < len(text); i++ {
		node := root
		for j := i; j < len(text); j++ {
			idx := text[j] - 'a'
			if node.children[idx] == nil {
				break
			}
			node = node.children[idx]
			for _, word := range node.output {
				result[word] = append(result[word], i)
			}
		}
	}
	return result
}

func main() {
	text := "ahishershe"
	patterns := []string{"he", "she", "his", "hers"}
	result := multiPatternSearch(text, patterns)
	for pattern, positions := range result {
		fmt.Printf("'%s' found at: %v\n", pattern, positions)
	}
}
```

**Textual Figure:**

```
Text: "ahishershe", Patterns: ["he","she","his","hers"]

   Pattern Trie:
   (root)
    ├─ h
    │   ├─ e ★        → "he"
    │   │   └─ r ─ s ★ → "hers"
    │   └─ i ─ s ★    → "his"
    └─ s ─ h ─ e ★    → "she"

   Scan text, at each position walk trie:
   pos 0: 'a' → no trie match
   pos 1: 'h'─'i'─'s' ★ "his" found at 1
   pos 2: 'i' → no match
   pos 3: 's'─'h'─'e' ★ "she" found at 3
   pos 4: 'h'─'e' ★ "he" found at 4
          'h'─'e'─'r'─'s' ★ "hers" found at 4
   pos 5: 'e' → no match
   pos 6: 'r' → no match
   pos 7: 's'─'h'─'e' ★ "she" found at 7
   pos 8: 'h'─'e' ★ "he" found at 8

   Result: he:[4,8] she:[3,7] his:[1] hers:[4]
```

---

## Example 7: Glob Pattern Matching

```go
package main

import "fmt"

func globMatch(pattern, text string) bool {
	return globHelper(pattern, text, 0, 0)
}

func globHelper(pattern, text string, pi, ti int) bool {
	if pi == len(pattern) {
		return ti == len(text)
	}

	if pattern[pi] == '*' {
		// * matches zero or more characters
		// Try matching 0 chars, 1 char, 2 chars, etc
		for i := ti; i <= len(text); i++ {
			if globHelper(pattern, text, pi+1, i) {
				return true
			}
		}
		return false
	}

	if ti == len(text) {
		return false
	}

	if pattern[pi] == '?' || pattern[pi] == text[ti] {
		return globHelper(pattern, text, pi+1, ti+1)
	}

	return false
}

// Optimized with DP
func globMatchDP(pattern, text string) bool {
	m, n := len(pattern), len(text)
	dp := make([][]bool, m+1)
	for i := range dp { dp[i] = make([]bool, n+1) }

	dp[0][0] = true
	for i := 1; i <= m; i++ {
		if pattern[i-1] == '*' {
			dp[i][0] = dp[i-1][0]
		}
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if pattern[i-1] == '*' {
				dp[i][j] = dp[i-1][j] || dp[i][j-1]
			} else if pattern[i-1] == '?' || pattern[i-1] == text[j-1] {
				dp[i][j] = dp[i-1][j-1]
			}
		}
	}
	return dp[m][n]
}

func main() {
	fmt.Println(globMatchDP("he*o", "hello"))      // true
	fmt.Println(globMatchDP("he?lo", "hello"))      // true
	fmt.Println(globMatchDP("h*l?o", "hello"))      // true
	fmt.Println(globMatchDP("*", "anything"))        // true
	fmt.Println(globMatchDP("h*xyz", "hello"))       // false
}
```

**Textual Figure:**

```
Glob Pattern Matching (pattern vs text):

   DP table for pattern="he*o" text="hello":
        ""  h   e   l   l   o
   ""  [ T   F   F   F   F   F ]
   h   [ F   T   F   F   F   F ]
   e   [ F   F   T   F   F   F ]
   *   [ F   F   T   T   T   T ]  ← * propagates right
   o   [ F   F   F   F   F   T ]  ← MATCH!

   * matches 0+ chars: "" or "l" or "ll"
   he*o matches "hello" → * consumes "ll"

   Pattern matching rules:
   ┌─────────┬─────────────────────────────┐
   │ Pattern │ Rule                        │
   ├─────────┼─────────────────────────────┤
   │ *       │ dp[i][j] = dp[i-1][j]       │
   │         │         OR dp[i][j-1]       │
   │ ?       │ dp[i][j] = dp[i-1][j-1]     │
   │ char    │ dp[i][j] = dp[i-1][j-1]     │
   │         │   if p[i]==t[j]              │
   └─────────┴─────────────────────────────┘
```

---

## Example 8: File System with Wildcard Paths

```go
package main

import (
	"fmt"
	"strings"
)

type FSNode struct {
	children map[string]*FSNode
	isFile   bool
	content  string
}

type FileSystem struct {
	root *FSNode
}

func NewFileSystem() *FileSystem {
	return &FileSystem{root: &FSNode{children: map[string]*FSNode{}}}
}

func (fs *FileSystem) Create(path, content string) {
	parts := strings.Split(strings.Trim(path, "/"), "/")
	node := fs.root
	for _, p := range parts {
		if _, ok := node.children[p]; !ok {
			node.children[p] = &FSNode{children: map[string]*FSNode{}}
		}
		node = node.children[p]
	}
	node.isFile = true
	node.content = content
}

func (fs *FileSystem) Glob(pattern string) []string {
	parts := strings.Split(strings.Trim(pattern, "/"), "/")
	results := []string{}
	fs.globDFS(fs.root, parts, 0, "/", &results)
	return results
}

func (fs *FileSystem) globDFS(node *FSNode, parts []string, pi int, path string, results *[]string) {
	if pi == len(parts) {
		*results = append(*results, path)
		return
	}

	part := parts[pi]
	if part == "*" {
		for name, child := range node.children {
			newPath := path + name
			if pi < len(parts)-1 { newPath += "/" }
			fs.globDFS(child, parts, pi+1, newPath, results)
		}
	} else {
		if child, ok := node.children[part]; ok {
			newPath := path + part
			if pi < len(parts)-1 { newPath += "/" }
			fs.globDFS(child, parts, pi+1, newPath, results)
		}
	}
}

func main() {
	fs := NewFileSystem()
	fs.Create("/src/main.go", "package main")
	fs.Create("/src/utils.go", "package utils")
	fs.Create("/test/main_test.go", "test")
	fs.Create("/README.md", "readme")

	fmt.Println("Glob /src/*:", fs.Glob("src/*"))
	fmt.Println("Glob /*:", fs.Glob("*"))
}
```

**Textual Figure:**

```
File System Trie (path segment edges):

   (root)
    ├─ "src"
    │   ├─ "main.go" ★     content: "package main"
    │   └─ "utils.go" ★    content: "package utils"
    ├─ "test"
    │   └─ "main_test.go" ★  content: "test"
    └─ "README.md" ★        content: "readme"

   Glob "src/*" ('*' matches any one segment):
     root ─src→ '*' matches all children:
       ├─ "main.go"   → /src/main.go
       └─ "utils.go"  → /src/utils.go

   Glob "*" ('*' matches any one segment at root):
     root → '*' matches all children:
       ├─ "src"        → /src
       ├─ "test"       → /test
       └─ "README.md"  → /README.md

   Each path segment = one trie edge (not per-char).
```

---

## Example 9: Wildcard DNS Matching

```go
package main

import (
	"fmt"
	"strings"
)

type DNSNode struct {
	children map[string]*DNSNode
	ip       string
}

type DNSResolver struct {
	root *DNSNode
}

func NewDNSResolver() *DNSResolver {
	return &DNSResolver{root: &DNSNode{children: map[string]*DNSNode{}}}
}

func (d *DNSResolver) AddRecord(domain, ip string) {
	parts := strings.Split(domain, ".")
	// Reverse to build trie from TLD
	node := d.root
	for i := len(parts) - 1; i >= 0; i-- {
		if _, ok := node.children[parts[i]]; !ok {
			node.children[parts[i]] = &DNSNode{children: map[string]*DNSNode{}}
		}
		node = node.children[parts[i]]
	}
	node.ip = ip
}

func (d *DNSResolver) Resolve(domain string) string {
	parts := strings.Split(domain, ".")
	return d.resolveHelper(d.root, parts, len(parts)-1)
}

func (d *DNSResolver) resolveHelper(node *DNSNode, parts []string, idx int) string {
	if idx < 0 {
		return node.ip
	}

	// Exact match
	if child, ok := node.children[parts[idx]]; ok {
		if result := d.resolveHelper(child, parts, idx-1); result != "" {
			return result
		}
	}

	// Wildcard match
	if child, ok := node.children["*"]; ok {
		if result := d.resolveHelper(child, parts, idx-1); result != "" {
			return result
		}
	}

	return ""
}

func main() {
	dns := NewDNSResolver()
	dns.AddRecord("api.example.com", "1.1.1.1")
	dns.AddRecord("*.example.com", "2.2.2.2")
	dns.AddRecord("example.com", "3.3.3.3")

	fmt.Println("api.example.com →", dns.Resolve("api.example.com"))    // 1.1.1.1
	fmt.Println("web.example.com →", dns.Resolve("web.example.com"))    // 2.2.2.2
	fmt.Println("test.example.com →", dns.Resolve("test.example.com"))  // 2.2.2.2
	fmt.Println("example.com →", dns.Resolve("example.com"))            // 3.3.3.3
}
```

**Textual Figure:**

```
DNS Trie (reversed domain, TLD-first):

   Records:
   api.example.com → 1.1.1.1
   *.example.com   → 2.2.2.2
   example.com     → 3.3.3.3

   Trie (built from TLD inward):
   (root)
    └─ "com"
        └─ "example"  ip: "3.3.3.3"
            ├─ "api" ★  ip: "1.1.1.1"
            └─ "*" ★    ip: "2.2.2.2"  (wildcard)

   Resolution (walk reversed domain parts):
   ┌────────────────────┬─────────────────────────┬─────────┐
   │ Domain             │ Path                    │ IP      │
   ├────────────────────┼─────────────────────────┼─────────┤
   │ api.example.com    │ com─example─api (exact) │ 1.1.1.1 │
   │ web.example.com    │ com─example─* (wildcard)│ 2.2.2.2 │
   │ test.example.com   │ com─example─* (wildcard)│ 2.2.2.2 │
   │ example.com        │ com─example (direct)   │ 3.3.3.3 │
   └────────────────────┴─────────────────────────┴─────────┘
   Exact match tried first; wildcard "*" as fallback.
```

---

## Example 10: Wildcard Matching Complexity Analysis

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Wildcard Matching in Tries: Summary ===")
	fmt.Println()

	approaches := []struct{ wildcard, approach, time, space string }{
		{
			". (single char)",
			"Try all 26 children recursively",
			"O(26^d × L) worst case, d=dot count",
			"O(L) stack depth",
		},
		{
			"* (any sequence)",
			"Recursive: skip 0,1,2,... chars in trie",
			"O(N) where N=total nodes (can visit all)",
			"O(L) stack depth",
		},
		{
			"[abc] (char class)",
			"Try each char in class",
			"O(K × L) where K=class size",
			"O(L) stack depth",
		},
		{
			"Fuzzy (edit dist ≤ k)",
			"Branch: substitute, insert, delete",
			"O(26^k × N) exponential in k",
			"O(L+k) stack depth",
		},
		{
			"Glob (file pattern)",
			"Trie on path segments, * per segment",
			"O(N × P) N=nodes, P=pattern parts",
			"O(depth) stack",
		},
	}

	for _, a := range approaches {
		fmt.Printf("Wildcard: %s\n", a.wildcard)
		fmt.Printf("  Approach: %s\n", a.approach)
		fmt.Printf("  Time: %s\n", a.time)
		fmt.Printf("  Space: %s\n\n", a.space)
	}

	fmt.Println("Tips:")
	fmt.Println("  • . wildcard: DFS with branching at wildcard positions")
	fmt.Println("  • * wildcard: tricky — consider DP for string matching")
	fmt.Println("  • Limit results (top-K) to avoid exponential blowup")
	fmt.Println("  • Reverse insertion helps with suffix matching")
}
```

**Textual Figure:**

```
Wildcard Matching in Tries — Complexity Summary:

   Trie with words: ["cat","car","cut"]
   (root)
    └─ c
        ├─ a ─ t ★ / r ★
        └─ u ─ t ★

   ┌───────────┬─────────────────┐
   │ Wildcard  │ Search branching │
   ├───────────┼─────────────────┤
   │ c.t       │ c→[a,u]→t       │
   │           │  2 branches     │
   ├───────────┼─────────────────┤
   │ c*        │ c→all subtree   │
   │           │  visits all     │
   ├───────────┼─────────────────┤
   │ [au]      │ try a,u only    │
   │           │  K branches     │
   ├───────────┼─────────────────┤
   │ fuzzy(1)  │ sub+ins+del     │
   │           │  26^k branches  │
   └───────────┴─────────────────┘

   Key trade-off: more wildcards = exponential branching.
   Trie structure limits branching to existing paths only.
```

---

## Key Takeaways

1. `.` wildcard: try all 26 children — exponential in # of dots
2. `*` wildcard: recursive match 0 or more characters
3. For string-vs-string wildcard matching, DP is often better than trie
4. Trie wildcards excel when searching a dictionary of stored words
5. Fuzzy search (edit distance) via trie: powerful but exponential in edit count

> **Next up:** Compressed Trie / Radix Tree →
