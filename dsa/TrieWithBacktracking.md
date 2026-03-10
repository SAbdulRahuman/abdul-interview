# Phase 19: Trie — Trie with Backtracking

## Overview

Combining a **Trie** with **Backtracking** (DFS) creates powerful patterns for searching word grids, solving word puzzles, and efficiently pruning search spaces. The trie guides the search, and backtracking explores all valid paths.

**Core idea**: Instead of checking each word against the board/path, build a trie of all target words and DFS through the search space, following only trie-valid branches.

| Pattern | Description |
|---------|-------------|
| Word Search II | Find all dictionary words in a 2D grid |
| DFS + Trie pruning | Prune DFS branches that don't match any trie prefix |
| Word Break with trie | Check if string can be segmented using trie-guided DFS |
| Boggle solver | Find words in a letter grid by adjacent traversal |

---

## Example 1: Word Search II (LeetCode 212)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	word     string // store complete word at leaf
}

func buildTrie(words []string) *TrieNode {
	root := &TrieNode{}
	for _, w := range words {
		node := root
		for _, ch := range w {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
		}
		node.word = w
	}
	return root
}

func findWords(board [][]byte, words []string) []string {
	root := buildTrie(words)
	result := []string{}
	rows, cols := len(board), len(board[0])

	var dfs func(r, c int, node *TrieNode)
	dfs = func(r, c int, node *TrieNode) {
		if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] == '#' {
			return
		}

		ch := board[r][c]
		next := node.children[ch-'a']
		if next == nil { return } // trie prune!

		if next.word != "" {
			result = append(result, next.word)
			next.word = "" // avoid duplicate
		}

		// Backtrack
		board[r][c] = '#' // mark visited
		dfs(r+1, c, next)
		dfs(r-1, c, next)
		dfs(r, c+1, next)
		dfs(r, c-1, next)
		board[r][c] = ch // restore
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			dfs(r, c, root)
		}
	}
	return result
}

func main() {
	board := [][]byte{
		{'o', 'a', 'a', 'n'},
		{'e', 't', 'a', 'e'},
		{'i', 'h', 'k', 'r'},
		{'i', 'f', 'l', 'v'},
	}
	words := []string{"oath", "pea", "eat", "rain"}
	fmt.Println("Found words:", findWords(board, words))
	// Output: [oath eat]
}
```

**Textual Figure:**

```
Board:              Trie (from words ["oath","pea","eat","rain"]):
┌───┬───┬───┬───┐    (root)
│ o │ a │ a │ n │     ├─ o ─ a ─ t ─ h ★   → "oath"
├───┼───┼───┼───┤     ├─ p ─ e ─ a ★       → "pea"
│ e │ t │ a │ e │     ├─ e ─ a ─ t ★       → "eat"
├───┼───┼───┼───┤     └─ r ─ a ─ i ─ n ★   → "rain"
│ i │ h │ k │ r │
├───┼───┼───┼───┤   DFS from each cell, following trie edges:
│ i │ f │ l │ v │
└───┴───┴───┴───┘   Finding "oath":
                      (0,0)'o' → trie has 'o'
                      (1,0)'e' → no, try (0,1)'a' → trie 'o'─'a' ✓
                      (1,1)'t' → trie 'o'─'a'─'t' ✓
                      (1,1)→(2,1)'h' → trie 'o'─'a'─'t'─'h' ★ FOUND!

                      Trie prune: if cell char has no trie child
                      → immediately stop DFS (no backtracking needed)
                      Result: [oath, eat]
```

---

## Example 2: Word Search II with Trie Pruning Optimization

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	word     string
	count    int // number of words passing through
}

func buildTrie(words []string) *TrieNode {
	root := &TrieNode{}
	for _, w := range words {
		node := root
		node.count++
		for _, ch := range w {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
			node.count++
		}
		node.word = w
	}
	return root
}

func findWordsOptimized(board [][]byte, words []string) []string {
	root := buildTrie(words)
	result := []string{}
	rows, cols := len(board), len(board[0])
	dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

	var dfs func(r, c int, parent, node *TrieNode, idx int)
	dfs = func(r, c int, parent, node *TrieNode, idx int) {
		if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] == '#' {
			return
		}
		ch := board[r][c]
		i := int(ch - 'a')
		next := node.children[i]
		if next == nil || next.count == 0 { return } // aggressive prune

		if next.word != "" {
			result = append(result, next.word)
			next.word = ""
			// Decrement counts up the path (prune found words)
			next.count--
		}

		board[r][c] = '#'
		for _, d := range dirs {
			dfs(r+d[0], c+d[1], node, next, i)
		}
		board[r][c] = ch

		// If subtree is empty, remove this branch
		if next.count == 0 {
			node.children[i] = nil
		}
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			dfs(r, c, nil, root, -1)
		}
	}
	return result
}

func main() {
	board := [][]byte{
		{'o', 'a', 'a', 'n'},
		{'e', 't', 'a', 'e'},
		{'i', 'h', 'k', 'r'},
		{'i', 'f', 'l', 'v'},
	}
	words := []string{"oath", "pea", "eat", "rain", "oat"}
	fmt.Println("Found:", findWordsOptimized(board, words))
}
```

**Textual Figure:**

```
Optimized Trie with count-based pruning:

   (root) cnt=5
    ├─ o cnt=2
    │   └─ a cnt=2
    │       └─ t cnt=2  ★ "oat"
    │           └─ h cnt=1  ★ "oath"
    ├─ p cnt=1 ─ e ─ a ★ "pea"
    ├─ e cnt=1 ─ a ─ t ★ "eat"
    └─ r cnt=1 ─ a ─ i ─ n ★ "rain"

   After finding "oath":
   • Decrement counts: h(0), t(1), a(1), o(1)
   • h.count=0 → remove o─a─t─h branch
   • Node 't' still has count=1 ("oat" remains)

   After finding "oat":
   • Decrement: t(0), a(0), o(0)
   • Entire o-branch pruned (count=0)
   • Future DFS skips 'o' entirely!

   Progressive pruning: each found word shrinks search space.
```

---

## Example 3: Boggle Solver

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	word     string
}

func solveBoggle(grid [][]byte, dictionary []string) []string {
	// Build trie from dictionary
	root := &TrieNode{}
	for _, w := range dictionary {
		node := root
		for _, ch := range w {
			i := ch - 'a'
			if node.children[i] == nil { node.children[i] = &TrieNode{} }
			node = node.children[i]
		}
		node.word = w
	}

	rows, cols := len(grid), len(grid[0])
	found := map[string]bool{}
	dirs := [][2]int{{-1, -1}, {-1, 0}, {-1, 1}, {0, -1}, {0, 1}, {1, -1}, {1, 0}, {1, 1}}

	var dfs func(r, c int, node *TrieNode)
	dfs = func(r, c int, node *TrieNode) {
		if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] == '#' { return }

		ch := grid[r][c]
		next := node.children[ch-'a']
		if next == nil { return }

		if next.word != "" {
			found[next.word] = true
		}

		grid[r][c] = '#'
		for _, d := range dirs {
			dfs(r+d[0], c+d[1], next)
		}
		grid[r][c] = ch
	}

	for r := 0; r < rows; r++ {
		for c := 0; c < cols; c++ {
			dfs(r, c, root)
		}
	}

	result := []string{}
	for w := range found { result = append(result, w) }
	return result
}

func main() {
	grid := [][]byte{
		{'g', 'i', 'z'},
		{'u', 'e', 'k'},
		{'q', 's', 'e'},
	}
	dict := []string{"geeks", "quiz", "seek", "gee", "gig", "key"}
	fmt.Println("Boggle words:", solveBoggle(grid, dict))
}
```

**Textual Figure:**

```
Boggle grid (8-directional):      Dictionary Trie:
┌───┬───┬───┐                      (root)
│ g │ i │ z │                       ├─ g
├───┼───┼───┤                       │   ├─ e ─ e ★ → "gee"
│ u │ e │ k │                       │   │       └─ k ─ s ★ → "geeks"
├───┼───┼───┤                       │   └─ i ─ g ★ → "gig"
│ q │ s │ e │                       ├─ k ─ e ─ y ★ → "key"
└───┴───┴───┘                       ├─ q ─ u ─ i ─ z ★ → "quiz"
                                    └─ s ─ e ─ e ─ k ★ → "seek"
   DFS from (0,0) 'g':
   g(0,0) → e(1,1) → e(2,2) ★ "gee" found!
              └→ k(1,2) → s(2,1) ★ "geeks" found!
   g(0,0) → i(0,1) → no 'g' neighbor with trie match

   8 directions per cell + trie pruning = efficient search.
```

---

## Example 4: Word Break with Trie + Backtracking

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
		i := ch - 'a'
		if node.children[i] == nil { node.children[i] = &TrieNode{} }
		node = node.children[i]
	}
	node.isEnd = true
}

func wordBreak(s string, wordDict []string) []string {
	trie := NewTrie()
	for _, w := range wordDict { trie.Insert(w) }

	result := []string{}
	var backtrack func(pos int, path []string)
	backtrack = func(pos int, path []string) {
		if pos == len(s) {
			sentence := ""
			for i, w := range path {
				if i > 0 { sentence += " " }
				sentence += w
			}
			result = append(result, sentence)
			return
		}

		node := trie.root
		for i := pos; i < len(s); i++ {
			idx := s[i] - 'a'
			if node.children[idx] == nil { break }
			node = node.children[idx]
			if node.isEnd {
				backtrack(i+1, append(path, s[pos:i+1]))
			}
		}
	}

	backtrack(0, nil)
	return result
}

func main() {
	s := "catsanddog"
	dict := []string{"cat", "cats", "and", "sand", "dog"}
	fmt.Println("Word break results:")
	for _, sentence := range wordBreak(s, dict) {
		fmt.Printf("  '%s'\n", sentence)
	}
	// "cats and dog", "cat sand dog"
}
```

**Textual Figure:**

```
s = "catsanddog", dict = ["cat", "cats", "and", "sand", "dog"]

   Dictionary Trie:
   (root)
    ├─ c ─ a ─ t ★ → "cat"
    │           └─ s ★ → "cats"
    ├─ a ─ n ─ d ★ → "and"
    ├─ s ─ a ─ n ─ d ★ → "sand"
    └─ d ─ o ─ g ★ → "dog"

   Backtracking word break:
   pos=0: walk trie with "catsanddog"...
     ├─ c─a─t ★ match "cat" at pos=3
     │   pos=3: walk trie with "sanddog"...
     │     └─ s─a─n─d ★ match "sand" at pos=7
     │         pos=7: walk trie with "dog"...
     │           └─ d─o─g ★ match! → "cat sand dog" ✓
     └─ c─a─t─s ★ match "cats" at pos=4
         pos=4: walk trie with "anddog"...
           └─ a─n─d ★ match "and" at pos=7
               pos=7: walk trie with "dog"...
                 └─ d─o─g ★ match! → "cats and dog" ✓

   Result: ["cats and dog", "cat sand dog"]
```

---

## Example 5: Generate Valid Words from Letter Tiles (LeetCode 1079-style)

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

func countValidWords(tiles string, dictionary []string) []string {
	// Build trie from dictionary
	root := &TrieNode{}
	for _, w := range dictionary {
		node := root
		for _, ch := range w {
			i := ch - 'a'
			if node.children[i] == nil { node.children[i] = &TrieNode{} }
			node = node.children[i]
		}
		node.isEnd = true
	}

	// Count available letters
	freq := [26]int{}
	for _, ch := range tiles { freq[ch-'a']++ }

	found := map[string]bool{}

	var backtrack func(node *TrieNode, word []byte)
	backtrack = func(node *TrieNode, word []byte) {
		if node.isEnd {
			found[string(word)] = true
		}

		for i := 0; i < 26; i++ {
			if freq[i] > 0 && node.children[i] != nil {
				freq[i]--
				backtrack(node.children[i], append(word, byte('a'+i)))
				freq[i]++
			}
		}
	}

	backtrack(root, nil)

	result := []string{}
	for w := range found { result = append(result, w) }
	return result
}

func main() {
	tiles := "aabcct"
	dict := []string{"cat", "bat", "cab", "act", "tac", "abc", "ab", "at"}
	fmt.Println("Valid words from tiles:", countValidWords(tiles, dict))
}
```

**Textual Figure:**

```
Tiles: "aabcct"  → freq: {a:2, b:1, c:2, t:1}
Dict: ["cat","bat","cab","act","tac","abc","ab","at"]

   Dictionary Trie:
   (root)
    ├─ a
    │   ├─ b ★ → "ab"       ├─ c ─ t ★ → "act"
    │   │   └─ c ★ → "abc"  └─ t ★ → "at"
    ├─ b ─ a ─ t ★ → "bat"
    ├─ c ─ a
    │       ├─ b ★ → "cab"
    │       └─ t ★ → "cat"
    └─ t ─ a ─ c ★ → "tac"

   Backtracking with freq array:
   Try 'a' (freq[a]=2→1):
     Try 'b' (freq[b]=1→0): "ab" ★ found!
       Try 'c' (freq[c]=2→1): "abc" ★ found!
     Try 'c' (freq[c]=2→1):
       Try 't' (freq[t]=1→0): "act" ★ found!
     Try 't' (freq[t]=1→0): "at" ★ found!
   Try 'c' (freq[c]=2→1):
     Try 'a' (freq[a]=2→1):
       Try 'b': "cab" ★   Try 't': "cat" ★
   ... (continues for all valid permutations)
```

---

## Example 6: Palindrome Pairs with Trie + Backtracking

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	index    int // word index, -1 if not a word end
	palins   []int // indices of words whose remaining suffix is a palindrome
}

func isPalindrome(s string, lo, hi int) bool {
	for lo < hi {
		if s[lo] != s[hi] { return false }
		lo++; hi--
	}
	return true
}

func palindromePairs(words []string) [][2]int {
	root := &TrieNode{index: -1}

	// Insert reversed words
	for idx, w := range words {
		node := root
		for i := len(w) - 1; i >= 0; i-- {
			c := w[i] - 'a'
			// If remainder s[0..i] is palindrome, record
			if isPalindrome(w, 0, i) {
				node.palins = append(node.palins, idx)
			}
			if node.children[c] == nil {
				node.children[c] = &TrieNode{index: -1}
			}
			node = node.children[c]
		}
		node.palins = append(node.palins, idx)
		node.index = idx
	}

	result := [][2]int{}

	for idx, w := range words {
		node := root
		for i := 0; i < len(w); i++ {
			// If we found a word in trie and rest of w is palindrome
			if node.index >= 0 && node.index != idx && isPalindrome(w, i, len(w)-1) {
				result = append(result, [2]int{idx, node.index})
			}
			c := w[i] - 'a'
			if node.children[c] == nil { node = nil; break }
			node = node.children[c]
		}
		if node != nil {
			for _, j := range node.palins {
				if j != idx {
					result = append(result, [2]int{idx, j})
				}
			}
		}
	}
	return result
}

func main() {
	words := []string{"abcd", "dcba", "lls", "s", "sssll"}
	pairs := palindromePairs(words)
	fmt.Println("Palindrome pairs:")
	for _, p := range pairs {
		fmt.Printf("  [%d,%d] → '%s%s'\n", p[0], p[1], words[p[0]], words[p[1]])
	}
}
```

**Textual Figure:**

```
Words: ["abcd", "dcba", "lls", "s", "sssll"]

   Reversed-word Trie (insert each word reversed):
   (root)
    ├─ d ─ c ─ b ─ a ★ idx=0   rev("abcd")
    ├─ a ─ b ─ c ─ d ★ idx=1   rev("dcba")
    ├─ s
    │   ├─ l ─ l ★       idx=2   rev("lls")
    │   └─ (★)           idx=3   rev("s")
    └─ l ─ l ─ s ─ s ─ s ★ idx=4  rev("sssll")

   Palindrome detection:
   Walk word[i] through reversed trie:
   ┌───────┬───────────────┬─────────────────────────────┐
   │ Pair  │ Concatenation │ Why palindrome?             │
   ├───────┼───────────────┼─────────────────────────────┤
   │ [0,1] │ abcd|dcba     │ word = exact reverse        │
   │ [1,0] │ dcba|abcd     │ word = exact reverse        │
   │ [3,2] │ s|lls         │ "s" + "lls" = "slls"         │
   │ [2,4] │ lls|sssll     │ "lls" + "sssll" = "llssssll" │
   └───────┴───────────────┴─────────────────────────────┘
```

---

## Example 7: Trie-Guided DFS for Maximum XOR

```go
package main

import "fmt"

type BitTrieNode struct {
	children [2]*BitTrieNode
}

func findMaxXOR(nums []int) int {
	if len(nums) == 0 { return 0 }

	root := &BitTrieNode{}

	// Insert all numbers
	for _, num := range nums {
		node := root
		for i := 31; i >= 0; i-- {
			bit := (num >> i) & 1
			if node.children[bit] == nil {
				node.children[bit] = &BitTrieNode{}
			}
			node = node.children[bit]
		}
	}

	maxXOR := 0
	for _, num := range nums {
		node := root
		curXOR := 0
		for i := 31; i >= 0; i-- {
			bit := (num >> i) & 1
			want := 1 - bit // opposite bit gives larger XOR

			if node.children[want] != nil {
				curXOR |= (1 << i)
				node = node.children[want]
			} else {
				node = node.children[bit]
			}
		}
		if curXOR > maxXOR { maxXOR = curXOR }
	}
	return maxXOR
}

func main() {
	nums := []int{3, 10, 5, 25, 2, 8}
	fmt.Printf("Numbers: %v\n", nums)
	fmt.Printf("Maximum XOR: %d\n", findMaxXOR(nums))
	// 5 XOR 25 = 28
}
```

**Textual Figure:**

```
Nums: [3, 10, 5, 25, 2, 8] → find max XOR pair

   Binary Bit-Trie (32-bit, showing last 5 bits):
   (root)
    ├─ 0
    │   ├─ 0
    │   │   ├─ 0
    │   │   │   ├─ 1 ─ 0  → 2  (00010)
    │   │   │   └─ 1 ─ 1  → 3  (00011)
    │   │   └─ 1
    │   │       └─ 0 ─ 1  → 5  (00101)
    │   └─ 1
    │       ├─ 0
    │       │   └─ 0 ─ 0  → 8  (01000)
    │       └─ 0
    │           └─ 1 ─ 0  → 10 (01010)
    └─ 1
        └─ 1
            └─ 0
                └─ 0 ─ 1  → 25 (11001)

   Greedy XOR for num=5 (00101):
   Bit 4: want 1, have 1 → take it! XOR bit set
   Bit 3: want 1, have 1 → take it!
   Bit 2: want 1, have 0 → take 0
   Bit 1: want 0, have 0 → take 0
   Bit 0: want 0, have 1 → take 1
   Result: 5 XOR 25 = 11100 = 28  ← Maximum!
```

---

## Example 8: Word Ladder with Trie Pruning

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
	word     string
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie { return &Trie{root: &TrieNode{}} }

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		i := ch - 'a'
		if node.children[i] == nil { node.children[i] = &TrieNode{} }
		node = node.children[i]
	}
	node.isEnd = true
	node.word = word
}

func (t *Trie) FindNeighbors(word string) []string {
	neighbors := []string{}
	for i := 0; i < len(word); i++ {
		node := t.root
		// Match prefix up to i
		valid := true
		for j := 0; j < i; j++ {
			next := node.children[word[j]-'a']
			if next == nil { valid = false; break }
			node = next
		}
		if !valid { continue }

		// Try all substitutions at position i
		for c := 0; c < 26; c++ {
			if byte(c+'a') == word[i] { continue }
			if node.children[c] == nil { continue }

			// Match suffix after i
			subNode := node.children[c]
			ok := true
			for j := i + 1; j < len(word); j++ {
				next := subNode.children[word[j]-'a']
				if next == nil { ok = false; break }
				subNode = next
			}
			if ok && subNode.isEnd {
				neighbors = append(neighbors, subNode.word)
			}
		}
	}
	return neighbors
}

func wordLadder(begin, end string, wordList []string) int {
	trie := NewTrie()
	for _, w := range wordList { trie.Insert(w) }

	queue := []string{begin}
	visited := map[string]bool{begin: true}
	steps := 1

	for len(queue) > 0 {
		size := len(queue)
		for i := 0; i < size; i++ {
			word := queue[i]
			if word == end { return steps }

			for _, nb := range trie.FindNeighbors(word) {
				if !visited[nb] {
					visited[nb] = true
					queue = append(queue, nb)
				}
			}
		}
		queue = queue[size:]
		steps++
	}
	return 0
}

func main() {
	begin := "hit"
	end := "cog"
	words := []string{"hot", "dot", "dog", "lot", "log", "cog"}
	fmt.Printf("'%s' → '%s': %d steps\n", begin, end, wordLadder(begin, end, words))
}
```

**Textual Figure:**

```
Word Ladder: "hit" → "cog", words=["hot","dot","dog","lot","log","cog"]

   Word Trie (for neighbor lookup):
   (root)
    ├─ h ─ o ─ t ★   ├─ d
    │                │   ├─ o
    │                │   │   ├─ t ★  → "dot"
    │                │   │   └─ g ★  → "dog"
    ├─ l ─ o         │
    │       ├─ t ★   │   → "lot"
    │       └─ g ★   │   → "log"
    └─ c ─ o ─ g ★   │   → "cog"

   BFS with trie-based neighbor finding:
   ┌───────┬─────────────────────────────────┐
   │ Step  │ Word → Neighbors (1 edit)  │
   ├───────┼─────────────────────────────────┤
   │ 1     │ "hit" → ["hot"]            │
   │ 2     │ "hot" → ["dot", "lot"]     │
   │ 3     │ "dot" → ["dog"]            │
   │ 3     │ "lot" → ["log"]            │
   │ 4     │ "dog" → ["cog"] ★ FOUND!  │
   └───────┴─────────────────────────────────┘
   Answer: 5 steps (hit→hot→dot→dog→cog)
```

---

## Example 9: Phone Number Letter Combinations with Trie Filter

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

func phoneCombinations(digits string, validWords []string) []string {
	if len(digits) == 0 { return nil }

	phone := map[byte]string{
		'2': "abc", '3': "def", '4': "ghi", '5': "jkl",
		'6': "mno", '7': "pqrs", '8': "tuv", '9': "wxyz",
	}

	// Build trie for quick validation
	root := &TrieNode{}
	for _, w := range validWords {
		node := root
		for _, ch := range w {
			i := ch - 'a'
			if node.children[i] == nil { node.children[i] = &TrieNode{} }
			node = node.children[i]
		}
		node.isEnd = true
	}

	result := []string{}

	var backtrack func(idx int, node *TrieNode, combo []byte)
	backtrack = func(idx int, node *TrieNode, combo []byte) {
		if idx == len(digits) {
			if node.isEnd {
				result = append(result, string(combo))
			}
			return
		}

		letters := phone[digits[idx]]
		for i := 0; i < len(letters); i++ {
			ch := letters[i]
			next := node.children[ch-'a']
			if next != nil { // trie prune
				backtrack(idx+1, next, append(combo, ch))
			}
		}
	}

	backtrack(0, root, nil)
	return result
}

func main() {
	digits := "228"
	dict := []string{"cat", "bat", "act", "cab", "abt", "bau", "cav"}
	fmt.Printf("Digits '%s' → valid words: %v\n", digits, phoneCombinations(digits, dict))
}
```

**Textual Figure:**

```
Digits: "228"  Phone map: 2→abc, 8→tuv
Valid words: ["cat","bat","act","cab","abt","bau","cav"]

   Dictionary Trie:                  Phone digit mapping:
   (root)                            ┌───┬───────┐
    ├─ a                             │ 2 │ a,b,c │
    │   ├─ b ─ t ★ → "abt"           │ 8 │ t,u,v │
    │   └─ c ─ t ★ → "act"           └───┴───────┘
    ├─ b
    │   └─ a
    │       ├─ t ★ → "bat"
    │       └─ u ★ → "bau"
    └─ c
        └─ a
            ├─ b ★ → "cab"
            ├─ t ★ → "cat"
            └─ v ★ → "cav"

   Digit "2": try a,b,c → check trie children
   Digit "2": try a,b,c → check next level
   Digit "8": try t,u,v → check isEnd

   Trie prune: digit '2' at pos 1 for path 'a'→
     only 'b','c' have trie children (skip rest).
```

---

## Example 10: Crossword Puzzle Solver with Trie

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
	word     string
}

func buildTrie(words []string) *TrieNode {
	root := &TrieNode{}
	for _, w := range words {
		node := root
		for _, ch := range w {
			i := ch - 'a'
			if node.children[i] == nil { node.children[i] = &TrieNode{} }
			node = node.children[i]
		}
		node.isEnd = true
		node.word = w
	}
	return root
}

// Find words that can be placed horizontally starting at (r,c)
func findHorizontalWords(grid [][]byte, root *TrieNode, r, c int) []string {
	cols := len(grid[0])
	results := []string{}

	var dfs func(col int, node *TrieNode)
	dfs = func(col int, node *TrieNode) {
		if node.isEnd {
			results = append(results, node.word)
		}
		if col >= cols { return }

		ch := grid[r][col]
		if ch == '_' {
			// Empty cell — try all
			for i := 0; i < 26; i++ {
				if node.children[i] != nil {
					dfs(col+1, node.children[i])
				}
			}
		} else {
			// Filled cell — must match
			idx := ch - 'a'
			if node.children[idx] != nil {
				dfs(col+1, node.children[idx])
			}
		}
	}

	dfs(c, root)
	return results
}

func main() {
	grid := [][]byte{
		{'c', '_', 't'},
		{'_', '_', '_'},
		{'d', '_', 'g'},
	}
	words := []string{"cat", "car", "cot", "dot", "dog", "dig", "dug"}
	root := buildTrie(words)

	fmt.Println("Grid:")
	for _, row := range grid {
		fmt.Printf("  %s\n", string(row))
	}
	fmt.Println()

	for r := 0; r < len(grid); r++ {
		found := findHorizontalWords(grid, root, r, 0)
		if len(found) > 0 {
			fmt.Printf("Row %d fits: %v\n", r, found)
		}
	}
}
```

**Textual Figure:**

```
Crossword grid with blanks:     Dictionary Trie:
┌───┬───┬───┐                     (root)
│ c │ _ │ t │                      ├─ c
├───┼───┼───┤                      │   ├─ a ─ t ★  → "cat"
│ _ │ _ │ _ │                      │   ├─ a ─ r ★  → "car"
├───┼───┼───┤                      │   └─ o ─ t ★  → "cot"
│ d │ _ │ g │                      └─ d
└───┴───┴───┘                          ├─ o
                                        │   ├─ t ★ → "dot"
  Row 0: c _ t                          │   └─ g ★ → "dog"
  DFS: c → trie has 'c'                 ├─ i ─ g ★ → "dig"
    _ → try all: a,o match              └─ u ─ g ★ → "dug"
    t → check isEnd:
      c-a-t ★ "cat" ✓
      c-o-t ★ "cot" ✓

  Row 2: d _ g
  DFS: d → trie 'd'
    _ → try: i,o,u match
    g → check: d-i-g ★, d-o-g ★, d-u-g ★
  Fits: [dig, dog, dug]
```

---

## Example 11: Auto-Complete with Ranked DFS

```go
package main

import (
	"fmt"
	"sort"
)

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
	freq     int
}

type Trie struct { root *TrieNode }

func NewTrie() *Trie { return &Trie{root: &TrieNode{}} }

func (t *Trie) Insert(word string, freq int) {
	node := t.root
	for _, ch := range word {
		i := ch - 'a'
		if node.children[i] == nil { node.children[i] = &TrieNode{} }
		node = node.children[i]
	}
	node.isEnd = true
	node.freq = freq
}

type WordFreq struct {
	word string
	freq int
}

func (t *Trie) Autocomplete(prefix string, topK int) []string {
	// Navigate to prefix node
	node := t.root
	for _, ch := range prefix {
		i := ch - 'a'
		if node.children[i] == nil { return nil }
		node = node.children[i]
	}

	// DFS collect with backtracking
	var candidates []WordFreq
	var dfs func(n *TrieNode, word []byte)
	dfs = func(n *TrieNode, word []byte) {
		if n.isEnd {
			candidates = append(candidates, WordFreq{string(word), n.freq})
		}
		for i := 0; i < 26; i++ {
			if n.children[i] != nil {
				dfs(n.children[i], append(word, byte('a'+i)))
			}
		}
	}

	dfs(node, []byte(prefix))

	sort.Slice(candidates, func(i, j int) bool {
		return candidates[i].freq > candidates[j].freq
	})

	result := []string{}
	for i := 0; i < topK && i < len(candidates); i++ {
		result = append(result, fmt.Sprintf("%s(%d)", candidates[i].word, candidates[i].freq))
	}
	return result
}

func main() {
	trie := NewTrie()
	words := map[string]int{
		"apple": 100, "application": 80, "apply": 60,
		"apt": 40, "ape": 30, "apex": 20,
	}
	for w, f := range words { trie.Insert(w, f) }

	fmt.Println("Top 3 for 'ap':", trie.Autocomplete("ap", 3))
	fmt.Println("Top 3 for 'app':", trie.Autocomplete("app", 3))
}
```

**Textual Figure:**

```
Insert with frequencies:
   (root)
    └─ a
        └─ p
            ├─ p
            │   ├─ l
            │   │   ├─ e ★  freq=100  → "apple"
            │   │   └─ y ★  freq=60   → "apply"
            │   └─ l ─ i ─ c ─ a ─ t ─ i ─ o ─ n ★
            │                               freq=80 → "application"
            ├─ t ★        freq=40   → "apt"
            └─ e
                └─ x ★    freq=20   → "apex"

   Autocomplete("ap", 3):
   1. Navigate: root─a─p
   2. DFS collect all: apple(100), application(80),
                       apply(60), apt(40), ape(30), apex(20)
   3. Sort by freq: apple, application, apply
   4. Return top-3: [apple(100), application(80), apply(60)]

   Autocomplete("app", 3):
   1. Navigate: root─a─p─p
   2. DFS: apple(100), application(80), apply(60)
   3. Return: [apple(100), application(80), apply(60)]
```

---

## Summary Table

| Pattern | Key Idea | Time Complexity |
|---------|----------|-----------------|
| Word Search II | DFS on grid + trie pruning | O(M·N·4^L) with heavy pruning |
| Boggle | 8-directional DFS + trie | O(M·N·8^L) pruned |
| Word Break | Trie-guided segmentation | O(n²) with memo |
| Palindrome Pairs | Reversed words in trie | O(n·k²) |
| Maximum XOR | Bit trie + greedy | O(n·32) |
| Crossword | Trie on partial grid | Depends on blanks |
| Phone combos | Trie filters invalid combos | Prunes exponential |

## Key Takeaways

1. **Trie + DFS** = powerful grid/string search pattern (Word Search II)
2. **Trie prunes early**: if no prefix match, skip entire branch
3. **Store word at leaf** instead of rebuilding from path — cleaner code
4. **Progressive trie removal**: delete found words to avoid duplicate work
5. **Bit trie + backtracking** extends to numeric problems (XOR)

> **Phase 19 Complete!** Next up: Phase 20 — Bit Manipulation →
