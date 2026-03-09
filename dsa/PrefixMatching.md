# Phase 19: Trie — Prefix Matching

## Overview

**Prefix matching** finds all words sharing a common prefix. Tries excel at this: walk to the prefix node in O(L), then DFS/BFS to enumerate all matches. This is the basis for autocomplete, spell checkers, and IP routing.

| Pattern | Approach |
|---------|----------|
| All words with prefix | Walk to node, DFS collect |
| Shortest prefix match | Walk, return at first isEnd |
| Longest prefix match | Walk, track last isEnd position |
| K-prefix matches | Walk, BFS/DFS with limit |

---

## Example 1: Find All Words with Prefix

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

func (t *Trie) FindAllWithPrefix(prefix string) []string {
	node := t.root
	for _, ch := range prefix {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return nil
		}
		node = node.children[idx]
	}

	result := []string{}
	t.dfs(node, []byte(prefix), &result)
	return result
}

func (t *Trie) dfs(node *TrieNode, prefix []byte, result *[]string) {
	if node.isEnd {
		*result = append(*result, string(prefix))
	}
	for i := 0; i < 26; i++ {
		if node.children[i] != nil {
			t.dfs(node.children[i], append(prefix, byte('a'+i)), result)
		}
	}
}

func main() {
	trie := NewTrie()
	for _, w := range []string{"app", "apple", "application", "apt", "bat", "bar"} {
		trie.Insert(w)
	}
	fmt.Println("Words with 'app':", trie.FindAllWithPrefix("app"))
	// [app apple application]
	fmt.Println("Words with 'ba':", trie.FindAllWithPrefix("ba"))
	// [bar bat]
}
```

---

## Example 2: Longest Prefix Match (IP Routing)

```go
package main

import "fmt"

type TrieNode struct {
	children [2]*TrieNode // binary trie: 0, 1
	value    string        // route destination
}

type IPTrie struct {
	root *TrieNode
}

func NewIPTrie() *IPTrie {
	return &IPTrie{root: &TrieNode{}}
}

func (t *IPTrie) InsertRoute(prefix string, dest string) {
	node := t.root
	for _, bit := range prefix {
		idx := bit - '0'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.value = dest
}

func (t *IPTrie) LongestMatch(ip string) string {
	node := t.root
	lastMatch := ""

	if node.value != "" {
		lastMatch = node.value // default route
	}

	for _, bit := range ip {
		idx := bit - '0'
		if node.children[idx] == nil {
			break
		}
		node = node.children[idx]
		if node.value != "" {
			lastMatch = node.value
		}
	}
	return lastMatch
}

func main() {
	trie := NewIPTrie()
	trie.InsertRoute("", "default")
	trie.InsertRoute("1", "network-A")
	trie.InsertRoute("10", "network-B")
	trie.InsertRoute("101", "network-C")
	trie.InsertRoute("1010", "network-D")

	fmt.Println(trie.LongestMatch("10100110")) // network-D (prefix "1010")
	fmt.Println(trie.LongestMatch("10110000")) // network-C (prefix "101")
	fmt.Println(trie.LongestMatch("11000000")) // network-A (prefix "1")
	fmt.Println(trie.LongestMatch("01000000")) // default
}
```

---

## Example 3: Shortest Unique Prefix

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	count    int // words passing through
}

func shortestUniquePrefixes(words []string) []string {
	root := &TrieNode{}

	// Insert all words with count
	for _, word := range words {
		node := root
		for _, ch := range word {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
			node.count++
		}
	}

	// Find shortest prefix with count=1
	result := make([]string, len(words))
	for i, word := range words {
		node := root
		prefix := []byte{}
		for _, ch := range word {
			idx := ch - 'a'
			node = node.children[idx]
			prefix = append(prefix, byte(ch))
			if node.count == 1 {
				break
			}
		}
		result[i] = string(prefix)
	}
	return result
}

func main() {
	words := []string{"zebra", "dog", "duck", "dove"}
	fmt.Println(shortestUniquePrefixes(words))
	// [z, dog, du, dov]

	words2 := []string{"apple", "application", "ape"}
	fmt.Println(shortestUniquePrefixes(words2))
}
```

---

## Example 4: Prefix Matching for Word Search

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	word     string
}

func findWords(board [][]byte, words []string) []string {
	root := &TrieNode{}
	for _, word := range words {
		node := root
		for _, ch := range word {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
		}
		node.word = word
	}

	result := []string{}
	rows, cols := len(board), len(board[0])
	dirs := [][]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

	var dfs func(r, c int, node *TrieNode)
	dfs = func(r, c int, node *TrieNode) {
		if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] == '#' {
			return
		}

		ch := board[r][c]
		next := node.children[ch-'a']
		if next == nil {
			return // prefix not in trie
		}

		if next.word != "" {
			result = append(result, next.word)
			next.word = "" // avoid duplicates
		}

		board[r][c] = '#' // mark visited
		for _, d := range dirs {
			dfs(r+d[0], c+d[1], next)
		}
		board[r][c] = ch // unmark
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
	fmt.Println(findWords(board, words)) // [oath eat]
}
```

---

## Example 5: CamelCase Matching

```go
package main

import "fmt"

func camelMatch(queries []string, pattern string) []bool {
	result := make([]bool, len(queries))
	for i, query := range queries {
		result[i] = matches(query, pattern)
	}
	return result
}

func matches(query, pattern string) bool {
	pi := 0
	for _, ch := range query {
		if pi < len(pattern) && byte(ch) == pattern[pi] {
			pi++
		} else if ch >= 'A' && ch <= 'Z' {
			return false // unmatched uppercase
		}
	}
	return pi == len(pattern)
}

func main() {
	queries := []string{"FooBar", "FooBarTest", "FootBall", "FrameBuffer", "ForceFeedBack"}
	pattern := "FB"
	fmt.Println(camelMatch(queries, pattern))
	// [true false true true false]

	queries2 := []string{"FooBar", "FooBarTest", "FootBall", "FrameBuffer", "ForceFeedBack"}
	pattern2 := "FoBa"
	fmt.Println(camelMatch(queries2, pattern2))
	// [true false true false false]
}
```

---

## Example 6: Maximum XOR Using Trie (Prefix Match)

```go
package main

import "fmt"

type BitNode struct {
	children [2]*BitNode
}

func findMaxXOR(nums []int) int {
	root := &BitNode{}

	insert := func(num int) {
		node := root
		for i := 31; i >= 0; i-- {
			bit := (num >> i) & 1
			if node.children[bit] == nil {
				node.children[bit] = &BitNode{}
			}
			node = node.children[bit]
		}
	}

	query := func(num int) int {
		node := root
		xor := 0
		for i := 31; i >= 0; i-- {
			bit := (num >> i) & 1
			want := 1 - bit // opposite bit for max XOR
			if node.children[want] != nil {
				xor |= 1 << i
				node = node.children[want]
			} else {
				node = node.children[bit]
			}
		}
		return xor
	}

	for _, num := range nums {
		insert(num)
	}

	maxXor := 0
	for _, num := range nums {
		xor := query(num)
		if xor > maxXor {
			maxXor = xor
		}
	}
	return maxXor
}

func main() {
	nums := []int{3, 10, 5, 25, 2, 8}
	fmt.Println("Max XOR:", findMaxXOR(nums)) // 28 (5^25)
}
```

---

## Example 7: Count Distinct Substrings

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
}

func countDistinctSubstrings(s string) int {
	root := &TrieNode{}
	count := 0

	for i := 0; i < len(s); i++ {
		node := root
		for j := i; j < len(s); j++ {
			idx := s[j] - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
				count++
			}
			node = node.children[idx]
		}
	}
	return count + 1 // +1 for empty string
}

func main() {
	fmt.Println(countDistinctSubstrings("abab")) // 8: "", a, ab, aba, abab, b, ba, bab
	fmt.Println(countDistinctSubstrings("aaa"))   // 4: "", a, aa, aaa
}
```

---

## Example 8: Phone Directory with Prefix

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	contacts []string
}

type PhoneDirectory struct {
	root *TrieNode
}

func NewPhoneDirectory() *PhoneDirectory {
	return &PhoneDirectory{root: &TrieNode{}}
}

func (pd *PhoneDirectory) AddContact(name string) {
	node := pd.root
	for _, ch := range name {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
		node.contacts = append(node.contacts, name)
	}
}

func (pd *PhoneDirectory) SearchPrefix(prefix string) []string {
	node := pd.root
	for _, ch := range prefix {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return []string{"No contacts found"}
		}
		node = node.children[idx]
	}
	return node.contacts
}

func (pd *PhoneDirectory) DisplayContacts(query string) {
	for i := 1; i <= len(query); i++ {
		prefix := query[:i]
		results := pd.SearchPrefix(prefix)
		fmt.Printf("'%s': %v\n", prefix, results)
	}
}

func main() {
	pd := NewPhoneDirectory()
	contacts := []string{"giksu", "geeks", "geeksforgeeks", "game"}
	for _, c := range contacts {
		pd.AddContact(c)
	}
	pd.DisplayContacts("gee")
}
```

---

## Example 9: Prefix Match for Spell Check Suggestions

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type SpellChecker struct {
	root *TrieNode
}

func NewSpellChecker() *SpellChecker {
	return &SpellChecker{root: &TrieNode{}}
}

func (sc *SpellChecker) AddWord(word string) {
	node := sc.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (sc *SpellChecker) Suggest(word string, maxDist int) []string {
	results := []string{}
	sc.suggestDFS(sc.root, []byte{}, word, 0, maxDist, &results)
	return results
}

func (sc *SpellChecker) suggestDFS(node *TrieNode, current []byte, target string, pos, maxDist int, results *[]string) {
	if len(*results) >= 5 { return } // limit suggestions

	dist := 0
	minLen := len(current)
	if pos < minLen { minLen = pos }

	// Simple prefix distance check
	if pos > len(target)+maxDist { return }

	if node.isEnd && pos >= len(target)-maxDist {
		// Check edit distance (simplified: length difference)
		diff := len(current) - len(target)
		if diff < 0 { diff = -diff }
		if diff <= maxDist {
			_ = dist
			*results = append(*results, string(current))
		}
	}

	for i := 0; i < 26; i++ {
		if node.children[i] != nil {
			sc.suggestDFS(node.children[i], append(current, byte('a'+i)), target, pos+1, maxDist, results)
		}
	}
}

func main() {
	sc := NewSpellChecker()
	for _, w := range []string{"hello", "help", "heap", "helm", "hero", "heal"} {
		sc.AddWord(w)
	}
	fmt.Println("Suggestions for 'helo':", sc.Suggest("helo", 1))
}
```

---

## Example 10: Prefix Matching Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Prefix Matching Patterns ===")
	fmt.Println()

	patterns := []struct{ pattern, timeComplexity, useCase string }{
		{
			"Find all words with prefix",
			"O(P + K) — P=prefix len, K=results",
			"Autocomplete, search suggestions",
		},
		{
			"Longest prefix match",
			"O(L) — L=query length",
			"IP routing, URL matching",
		},
		{
			"Shortest unique prefix",
			"O(L) per word with count tracking",
			"Abbreviations, unique identifiers",
		},
		{
			"Prefix pruning in search",
			"O(L) to check if prefix viable",
			"Word search II, backtracking with trie",
		},
		{
			"Max XOR via bit prefix",
			"O(32) per query",
			"Maximum XOR of two numbers",
		},
		{
			"Count distinct substrings",
			"O(n²) insert all suffixes",
			"Substring counting, suffix trie",
		},
	}

	for _, p := range patterns {
		fmt.Printf("  %s\n", p.pattern)
		fmt.Printf("    Time: %s\n", p.timeComplexity)
		fmt.Printf("    Use: %s\n\n", p.useCase)
	}
}
```

---

## Key Takeaways

1. Prefix matching = walk to prefix node + DFS/BFS to collect results
2. Longest prefix match: track last isEnd seen during traversal
3. Shortest unique prefix: count how many words pass through each node
4. Binary trie: prefix match on bits for XOR problems
5. Trie + backtracking: prune search space by checking if prefix exists

> **Next up:** Wildcard Matching →
