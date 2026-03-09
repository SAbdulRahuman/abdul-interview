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
