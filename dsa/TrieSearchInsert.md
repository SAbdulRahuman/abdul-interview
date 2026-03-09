# Phase 19: Trie — Trie Search and Insert

## Overview

**Search** and **Insert** are the fundamental trie operations. Insert creates nodes along the path for a word. Search traverses the path and checks `isEnd`. Both run in O(L) time where L is the word length.

| Operation | Steps |
|-----------|-------|
| Insert | Walk/create path, mark last node as end |
| Search (exact) | Walk path, return isEnd at last node |
| StartsWith | Walk path, return true if path exists |

---

## Example 1: Complete Trie Implementation (LeetCode 208)

```go
package main

import "fmt"

type Trie struct {
	children [26]*Trie
	isEnd    bool
}

func Constructor() Trie {
	return Trie{}
}

func (t *Trie) Insert(word string) {
	node := t
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &Trie{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
}

func (t *Trie) Search(word string) bool {
	node := t.findNode(word)
	return node != nil && node.isEnd
}

func (t *Trie) StartsWith(prefix string) bool {
	return t.findNode(prefix) != nil
}

func (t *Trie) findNode(s string) *Trie {
	node := t
	for _, ch := range s {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return nil
		}
		node = node.children[idx]
	}
	return node
}

func main() {
	trie := Constructor()
	trie.Insert("apple")
	fmt.Println(trie.Search("apple"))   // true
	fmt.Println(trie.Search("app"))     // false
	fmt.Println(trie.StartsWith("app")) // true
	trie.Insert("app")
	fmt.Println(trie.Search("app"))     // true
}
```

---

## Example 2: Replace Words with Trie

```go
package main

import (
	"fmt"
	"strings"
)

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

func (t *Trie) ShortestRoot(word string) string {
	node := t.root
	for i, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return word
		}
		node = node.children[idx]
		if node.isEnd {
			return word[:i+1]
		}
	}
	return word
}

func replaceWords(dictionary []string, sentence string) string {
	trie := NewTrie()
	for _, root := range dictionary {
		trie.Insert(root)
	}

	words := strings.Fields(sentence)
	for i, word := range words {
		words[i] = trie.ShortestRoot(word)
	}
	return strings.Join(words, " ")
}

func main() {
	dict := []string{"cat", "bat", "rat"}
	sentence := "the cattle was rattled by the battery"
	fmt.Println(replaceWords(dict, sentence))
	// "the cat was rat by the bat"
}
```

---

## Example 3: Search Suggestions System

```go
package main

import (
	"fmt"
	"sort"
)

type TrieNode struct {
	children [26]*TrieNode
	words    []string // store matching words (max 3)
}

func suggestedProducts(products []string, searchWord string) [][]string {
	sort.Strings(products) // lexicographic for top-3

	root := &TrieNode{}
	for _, p := range products {
		node := root
		for _, ch := range p {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
			if len(node.words) < 3 {
				node.words = append(node.words, p)
			}
		}
	}

	result := make([][]string, len(searchWord))
	node := root
	for i, ch := range searchWord {
		idx := ch - 'a'
		if node != nil && node.children[idx] != nil {
			node = node.children[idx]
			result[i] = node.words
		} else {
			node = nil
			result[i] = []string{}
		}
	}
	return result
}

func main() {
	products := []string{"mobile", "mouse", "moneypot", "monitor", "mousepad"}
	searchWord := "mouse"
	for i, suggestions := range suggestedProducts(products, searchWord) {
		fmt.Printf("After '%s': %v\n", searchWord[:i+1], suggestions)
	}
}
```

---

## Example 4: Insert and Count with Prefix Tracking

```go
package main

import "fmt"

type TrieNode struct {
	children    [26]*TrieNode
	prefixCount int
	wordCount   int
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
	node.wordCount++
}

func (t *Trie) Erase(word string) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		node = node.children[idx]
		node.prefixCount--
	}
	node.wordCount--
}

func (t *Trie) CountWordsEqualTo(word string) int {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil { return 0 }
		node = node.children[idx]
	}
	return node.wordCount
}

func (t *Trie) CountWordsStartingWith(prefix string) int {
	node := t.root
	for _, ch := range prefix {
		idx := ch - 'a'
		if node.children[idx] == nil { return 0 }
		node = node.children[idx]
	}
	return node.prefixCount
}

func main() {
	trie := NewTrie()
	trie.Insert("apple")
	trie.Insert("apple")
	trie.Insert("app")

	fmt.Println("Count 'apple':", trie.CountWordsEqualTo("apple"))             // 2
	fmt.Println("Prefix 'app':", trie.CountWordsStartingWith("app"))           // 3
	trie.Erase("apple")
	fmt.Println("Count 'apple' after erase:", trie.CountWordsEqualTo("apple")) // 1
	fmt.Println("Prefix 'app' after erase:", trie.CountWordsStartingWith("app")) // 2
}
```

---

## Example 5: Longest Word in Dictionary

```go
package main

import (
	"fmt"
	"sort"
)

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
	word     string
}

func longestWord(words []string) string {
	sort.Strings(words)

	root := &TrieNode{}
	root.isEnd = true // empty prefix is valid
	best := ""

	for _, word := range words {
		node := root
		valid := true
		for i, ch := range word {
			idx := ch - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
			// Each prefix must be a valid word (except at the end)
			if i < len(word)-1 && !node.isEnd {
				valid = false
			}
		}
		node.isEnd = true
		node.word = word

		if valid && (len(word) > len(best) || (len(word) == len(best) && word < best)) {
			best = word
		}
	}
	return best
}

func main() {
	words := []string{"w", "wo", "wor", "worl", "world"}
	fmt.Println(longestWord(words)) // "world"

	words2 := []string{"a", "banana", "app", "appl", "ap", "apply", "apple"}
	fmt.Println(longestWord(words2)) // "apple"
}
```

---

## Example 6: Map Sum Pairs

```go
package main

import "fmt"

type MapSumNode struct {
	children [26]*MapSumNode
	val      int
}

type MapSum struct {
	root *MapSumNode
	keys map[string]int
}

func NewMapSum() *MapSum {
	return &MapSum{root: &MapSumNode{}, keys: map[string]int{}}
}

func (m *MapSum) Insert(key string, val int) {
	delta := val - m.keys[key]
	m.keys[key] = val

	node := m.root
	for _, ch := range key {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &MapSumNode{}
		}
		node = node.children[idx]
		node.val += delta
	}
}

func (m *MapSum) Sum(prefix string) int {
	node := m.root
	for _, ch := range prefix {
		idx := ch - 'a'
		if node.children[idx] == nil {
			return 0
		}
		node = node.children[idx]
	}
	return node.val
}

func main() {
	ms := NewMapSum()
	ms.Insert("apple", 3)
	fmt.Println(ms.Sum("ap")) // 3
	ms.Insert("app", 2)
	fmt.Println(ms.Sum("ap")) // 5
	ms.Insert("apple", 5) // update
	fmt.Println(ms.Sum("ap")) // 7 (5+2)
}
```

---

## Example 7: Stream of Characters

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	isEnd    bool
}

type StreamChecker struct {
	root   *TrieNode
	stream []byte
}

func NewStreamChecker(words []string) *StreamChecker {
	root := &TrieNode{}
	// Insert words in REVERSE
	for _, word := range words {
		node := root
		for i := len(word) - 1; i >= 0; i-- {
			idx := word[i] - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{}
			}
			node = node.children[idx]
		}
		node.isEnd = true
	}
	return &StreamChecker{root: root}
}

func (sc *StreamChecker) Query(letter byte) bool {
	sc.stream = append(sc.stream, letter)
	node := sc.root
	for i := len(sc.stream) - 1; i >= 0; i-- {
		idx := sc.stream[i] - 'a'
		if node.children[idx] == nil {
			return false
		}
		node = node.children[idx]
		if node.isEnd {
			return true
		}
	}
	return false
}

func main() {
	sc := NewStreamChecker([]string{"cd", "f", "kl"})
	for _, ch := range "abcdefghijkl" {
		result := sc.Query(byte(ch))
		if result {
			fmt.Printf("'%c' → found suffix match!\n", ch)
		}
	}
	// 'd' → "cd" ends, 'f' → "f" ends, 'l' → "kl" ends
}
```

---

## Example 8: Palindrome Pairs via Trie

```go
package main

import "fmt"

type TrieNode struct {
	children [26]*TrieNode
	wordIdx  int
	palins   []int // indices of words whose remaining suffix is palindrome
}

func isPalindrome(s string, lo, hi int) bool {
	for lo < hi {
		if s[lo] != s[hi] { return false }
		lo++; hi--
	}
	return true
}

func palindromePairs(words []string) [][]int {
	root := &TrieNode{wordIdx: -1}

	// Insert reversed words
	for wi, word := range words {
		node := root
		for i := len(word) - 1; i >= 0; i-- {
			// If remaining prefix is palindrome, note this word
			if isPalindrome(word, 0, i) {
				node.palins = append(node.palins, wi)
			}
			idx := word[i] - 'a'
			if node.children[idx] == nil {
				node.children[idx] = &TrieNode{wordIdx: -1}
			}
			node = node.children[idx]
		}
		node.palins = append(node.palins, wi)
		node.wordIdx = wi
	}

	result := [][]int{}
	for wi, word := range words {
		node := root
		for i := 0; i < len(word); i++ {
			// If reversed word ends here and remaining of current is palindrome
			if node.wordIdx >= 0 && node.wordIdx != wi && isPalindrome(word, i, len(word)-1) {
				result = append(result, []int{wi, node.wordIdx})
			}
			idx := word[i] - 'a'
			if node.children[idx] == nil {
				node = nil
				break
			}
			node = node.children[idx]
		}
		if node != nil {
			for _, pi := range node.palins {
				if pi != wi {
					result = append(result, []int{wi, pi})
				}
			}
		}
	}
	return result
}

func main() {
	words := []string{"abcd", "dcba", "lls", "s", "sssll"}
	fmt.Println(palindromePairs(words))
	// [[0,1],[1,0],[3,2],[2,4]]
}
```

---

## Example 9: Autocomplete with Frequency

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

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{}}
}

func (t *Trie) Insert(word string, freq int) {
	node := t.root
	for _, ch := range word {
		idx := ch - 'a'
		if node.children[idx] == nil {
			node.children[idx] = &TrieNode{}
		}
		node = node.children[idx]
	}
	node.isEnd = true
	node.freq += freq
}

type WordFreq struct {
	word string
	freq int
}

func (t *Trie) TopK(prefix string, k int) []string {
	node := t.root
	for _, ch := range prefix {
		idx := ch - 'a'
		if node.children[idx] == nil { return nil }
		node = node.children[idx]
	}

	var results []WordFreq
	t.collectFreq(node, []byte(prefix), &results)

	sort.Slice(results, func(i, j int) bool {
		if results[i].freq == results[j].freq {
			return results[i].word < results[j].word
		}
		return results[i].freq > results[j].freq
	})

	top := []string{}
	for i := 0; i < k && i < len(results); i++ {
		top = append(top, results[i].word)
	}
	return top
}

func (t *Trie) collectFreq(node *TrieNode, prefix []byte, results *[]WordFreq) {
	if node.isEnd {
		*results = append(*results, WordFreq{string(prefix), node.freq})
	}
	for i := 0; i < 26; i++ {
		if node.children[i] != nil {
			t.collectFreq(node.children[i], append(prefix, byte('a'+i)), results)
		}
	}
}

func main() {
	trie := NewTrie()
	trie.Insert("google", 100)
	trie.Insert("go", 50)
	trie.Insert("golang", 80)
	trie.Insert("good", 60)
	trie.Insert("goat", 30)

	fmt.Println("Top 3 for 'go':", trie.TopK("go", 3))
	// [google golang good] by frequency
}
```

---

## Example 10: Trie Operations Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Trie Search & Insert: Complete Guide ===")
	fmt.Println()

	ops := []struct{ operation, how, time, notes string }{
		{"Insert", "Walk/create path, mark isEnd", "O(L)", "L = word length"},
		{"Search (exact)", "Walk path, check isEnd", "O(L)", "false if path breaks"},
		{"StartsWith", "Walk path, return true if exists", "O(L)", "No isEnd check"},
		{"Delete", "Recursive: unmark isEnd, prune empty", "O(L)", "Don't delete shared prefixes"},
		{"CountPrefix", "Walk to prefix node, read count", "O(L)", "Need prefixCount field"},
		{"AutoComplete", "Walk to prefix, DFS collect", "O(L + K)", "K = results collected"},
		{"Replace (shortest root)", "Walk, return on first isEnd", "O(L)", "Greedy: first match wins"},
		{"Sum by prefix", "Walk to prefix, read accumulated val", "O(L)", "Store delta at each node"},
	}

	for _, op := range ops {
		fmt.Printf("  %-22s → %s\n", op.operation, op.how)
		fmt.Printf("  %24s Time: %s (%s)\n\n", "", op.time, op.notes)
	}
}
```

---

## Key Takeaways

1. Insert/Search = O(L) — independent of number of stored words
2. `findNode` helper: shared logic for Search and StartsWith
3. Store metadata at nodes: frequency, value, word list for advanced queries
4. Reverse insertion: match suffixes (stream checking, palindrome pairs)
5. Prefix count tracking enables O(L) "count words with prefix" queries

> **Next up:** Prefix Matching →
