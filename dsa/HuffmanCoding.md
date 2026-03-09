# Phase 17: Greedy Algorithms — Huffman Coding

## Overview

**Huffman Coding** builds an optimal prefix-free binary code for data compression. It assigns shorter codes to more frequent characters using a greedy algorithm with a min-heap.

| Aspect | Detail |
|--------|--------|
| **Type** | Greedy, prefix-free coding |
| **Time** | O(n log n) |
| **Space** | O(n) |
| **Optimality** | Optimal for character-level encoding |

---

## Example 1: Build Huffman Tree

```go
package main

import (
	"container/heap"
	"fmt"
)

type HuffNode struct {
	Char  byte
	Freq  int
	Left  *HuffNode
	Right *HuffNode
}

type MinHeap []*HuffNode
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i].Freq < h[j].Freq }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*HuffNode)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func buildHuffmanTree(freq map[byte]int) *HuffNode {
	h := &MinHeap{}
	for ch, f := range freq {
		heap.Push(h, &HuffNode{Char: ch, Freq: f})
	}

	for h.Len() > 1 {
		left := heap.Pop(h).(*HuffNode)
		right := heap.Pop(h).(*HuffNode)
		parent := &HuffNode{
			Freq:  left.Freq + right.Freq,
			Left:  left,
			Right: right,
		}
		heap.Push(h, parent)
	}
	return heap.Pop(h).(*HuffNode)
}

func main() {
	freq := map[byte]int{'a': 5, 'b': 9, 'c': 12, 'd': 13, 'e': 16, 'f': 45}
	root := buildHuffmanTree(freq)
	fmt.Println("Root frequency:", root.Freq) // 100
}
```

---

## Example 2: Generate Huffman Codes

```go
package main

import (
	"container/heap"
	"fmt"
)

type HuffNode struct {
	Char  byte
	Freq  int
	Left  *HuffNode
	Right *HuffNode
}

type MinHeap []*HuffNode
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i].Freq < h[j].Freq }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*HuffNode)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func buildTree(freq map[byte]int) *HuffNode {
	h := &MinHeap{}
	for ch, f := range freq {
		heap.Push(h, &HuffNode{Char: ch, Freq: f})
	}
	for h.Len() > 1 {
		l := heap.Pop(h).(*HuffNode)
		r := heap.Pop(h).(*HuffNode)
		heap.Push(h, &HuffNode{Freq: l.Freq + r.Freq, Left: l, Right: r})
	}
	return heap.Pop(h).(*HuffNode)
}

func generateCodes(node *HuffNode, code string, codes map[byte]string) {
	if node == nil { return }
	if node.Left == nil && node.Right == nil {
		if code == "" { code = "0" } // single character case
		codes[node.Char] = code
		return
	}
	generateCodes(node.Left, code+"0", codes)
	generateCodes(node.Right, code+"1", codes)
}

func main() {
	freq := map[byte]int{'a': 5, 'b': 9, 'c': 12, 'd': 13, 'e': 16, 'f': 45}
	root := buildTree(freq)

	codes := map[byte]string{}
	generateCodes(root, "", codes)

	for ch, code := range codes {
		fmt.Printf("'%c': %s (freq=%d)\n", ch, code, freq[ch])
	}
}
```

---

## Example 3: Encode and Decode

```go
package main

import (
	"container/heap"
	"fmt"
	"strings"
)

type HuffNode struct {
	Char  byte
	Freq  int
	Left  *HuffNode
	Right *HuffNode
}

type MinHeap []*HuffNode
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i].Freq < h[j].Freq }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*HuffNode)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func buildTree(text string) *HuffNode {
	freq := map[byte]int{}
	for i := 0; i < len(text); i++ { freq[text[i]]++ }
	h := &MinHeap{}
	for ch, f := range freq { heap.Push(h, &HuffNode{Char: ch, Freq: f}) }
	for h.Len() > 1 {
		l, r := heap.Pop(h).(*HuffNode), heap.Pop(h).(*HuffNode)
		heap.Push(h, &HuffNode{Freq: l.Freq + r.Freq, Left: l, Right: r})
	}
	return heap.Pop(h).(*HuffNode)
}

func getCodes(node *HuffNode, code string, codes map[byte]string) {
	if node == nil { return }
	if node.Left == nil && node.Right == nil {
		if code == "" { code = "0" }
		codes[node.Char] = code; return
	}
	getCodes(node.Left, code+"0", codes)
	getCodes(node.Right, code+"1", codes)
}

func encode(text string, codes map[byte]string) string {
	var sb strings.Builder
	for i := 0; i < len(text); i++ {
		sb.WriteString(codes[text[i]])
	}
	return sb.String()
}

func decode(bits string, root *HuffNode) string {
	var sb strings.Builder
	node := root
	for i := 0; i < len(bits); i++ {
		if bits[i] == '0' { node = node.Left } else { node = node.Right }
		if node.Left == nil && node.Right == nil {
			sb.WriteByte(node.Char)
			node = root
		}
	}
	return sb.String()
}

func main() {
	text := "huffman coding is fun"
	root := buildTree(text)

	codes := map[byte]string{}
	getCodes(root, "", codes)

	encoded := encode(text, codes)
	fmt.Println("Original:", text)
	fmt.Println("Encoded:", encoded)
	fmt.Printf("Compression: %d bits vs %d bits (fixed 8-bit)\n",
		len(encoded), len(text)*8)

	decoded := decode(encoded, root)
	fmt.Println("Decoded:", decoded)
	fmt.Println("Match:", text == decoded)
}
```

---

## Example 4: Minimum Cost to Merge Files (LC 1167 variant)

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func minCostToMerge(files []int) int {
	h := &IntHeap{}
	for _, f := range files {
		heap.Push(h, f)
	}

	totalCost := 0
	for h.Len() > 1 {
		a := heap.Pop(h).(int)
		b := heap.Pop(h).(int)
		cost := a + b
		totalCost += cost
		heap.Push(h, cost)
	}
	return totalCost
}

func main() {
	fmt.Println(minCostToMerge([]int{4, 3, 2, 6})) // 29
	// Merge 2+3=5 (cost 5), 4+5=9 (cost 9), 6+9=15 (cost 15) = 29

	fmt.Println(minCostToMerge([]int{1, 2, 5, 10, 35, 89})) // 224

	fmt.Println()
	fmt.Println("Same structure as Huffman: always merge two smallest")
}
```

---

## Example 5: Connect Ropes with Minimum Cost

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func connectRopes(ropes []int) int {
	h := &IntHeap{}
	for _, r := range ropes {
		heap.Push(h, r)
	}

	cost := 0
	for h.Len() > 1 {
		first := heap.Pop(h).(int)
		second := heap.Pop(h).(int)
		merged := first + second
		cost += merged
		heap.Push(h, merged)
	}
	return cost
}

func main() {
	fmt.Println(connectRopes([]int{1, 2, 3, 4, 5}))  // 33
	fmt.Println(connectRopes([]int{2, 4, 3}))          // 14
}
```

---

## Example 6: Huffman Compression Ratio

```go
package main

import (
	"container/heap"
	"fmt"
)

type HuffNode struct {
	Char  byte
	Freq  int
	Left  *HuffNode
	Right *HuffNode
}

type MinHeap []*HuffNode
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i].Freq < h[j].Freq }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(*HuffNode)) }
func (h *MinHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

func compressionAnalysis(text string) {
	freq := map[byte]int{}
	for i := 0; i < len(text); i++ { freq[text[i]]++ }

	h := &MinHeap{}
	for ch, f := range freq { heap.Push(h, &HuffNode{Char: ch, Freq: f}) }
	for h.Len() > 1 {
		l, r := heap.Pop(h).(*HuffNode), heap.Pop(h).(*HuffNode)
		heap.Push(h, &HuffNode{Freq: l.Freq + r.Freq, Left: l, Right: r})
	}
	root := heap.Pop(h).(*HuffNode)

	codes := map[byte]string{}
	var gen func(*HuffNode, string)
	gen = func(n *HuffNode, code string) {
		if n == nil { return }
		if n.Left == nil && n.Right == nil {
			if code == "" { code = "0" }
			codes[n.Char] = code; return
		}
		gen(n.Left, code+"0")
		gen(n.Right, code+"1")
	}
	gen(root, "")

	totalBits := 0
	for ch, f := range freq {
		totalBits += f * len(codes[ch])
	}

	fixedBits := len(text) * 8
	fmt.Printf("Text length: %d chars\n", len(text))
	fmt.Printf("Fixed (8-bit): %d bits\n", fixedBits)
	fmt.Printf("Huffman:       %d bits\n", totalBits)
	fmt.Printf("Compression:   %.1f%%\n", float64(fixedBits-totalBits)/float64(fixedBits)*100)
}

func main() {
	compressionAnalysis("aaaaaabbbbbccccdddeef")
}
```

---

## Example 7: Prefix-Free Property Verification

```go
package main

import (
	"fmt"
	"strings"
)

func isPrefixFree(codes []string) bool {
	for i := 0; i < len(codes); i++ {
		for j := 0; j < len(codes); j++ {
			if i != j && strings.HasPrefix(codes[j], codes[i]) {
				fmt.Printf("  '%s' is prefix of '%s'\n", codes[i], codes[j])
				return false
			}
		}
	}
	return true
}

func main() {
	// Huffman codes are always prefix-free
	huffman := []string{"0", "10", "110", "111"}
	fmt.Println("Huffman codes:", huffman)
	fmt.Println("Prefix-free:", isPrefixFree(huffman)) // true

	// Non-prefix-free example
	bad := []string{"0", "01", "1", "10"}
	fmt.Println("\nBad codes:", bad)
	fmt.Println("Prefix-free:", isPrefixFree(bad)) // false

	fmt.Println()
	fmt.Println("Why prefix-free matters:")
	fmt.Println("  No code is a prefix of another")
	fmt.Println("  → Can decode without delimiters")
	fmt.Println("  → Unique decodability guaranteed")
}
```

---

## Example 8: Huffman with Priority Queue (Generic)

```go
package main

import (
	"container/heap"
	"fmt"
)

type Node struct {
	Symbol rune
	Freq   int
	Left   *Node
	Right  *Node
}

type PQ []*Node
func (pq PQ) Len() int           { return len(pq) }
func (pq PQ) Less(i, j int) bool { return pq[i].Freq < pq[j].Freq }
func (pq PQ) Swap(i, j int)      { pq[i], pq[j] = pq[j], pq[i] }
func (pq *PQ) Push(x interface{}) { *pq = append(*pq, x.(*Node)) }
func (pq *PQ) Pop() interface{} {
	old := *pq; n := len(old); x := old[n-1]; *pq = old[:n-1]; return x
}

func huffman(text string) map[rune]string {
	freq := map[rune]int{}
	for _, ch := range text { freq[ch]++ }

	pq := &PQ{}
	for ch, f := range freq {
		heap.Push(pq, &Node{Symbol: ch, Freq: f})
	}

	for pq.Len() > 1 {
		a := heap.Pop(pq).(*Node)
		b := heap.Pop(pq).(*Node)
		heap.Push(pq, &Node{Freq: a.Freq + b.Freq, Left: a, Right: b})
	}

	codes := map[rune]string{}
	var walk func(*Node, string)
	walk = func(n *Node, code string) {
		if n == nil { return }
		if n.Left == nil && n.Right == nil {
			if code == "" { code = "0" }
			codes[n.Symbol] = code
			return
		}
		walk(n.Left, code+"0")
		walk(n.Right, code+"1")
	}
	walk(heap.Pop(pq).(*Node), "")
	return codes
}

func main() {
	codes := huffman("go is great for golang gophers")
	for ch, code := range codes {
		fmt.Printf("'%c' → %s\n", ch, code)
	}
}
```

---

## Example 9: Weighted Path Length (Optimal Code Property)

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() interface{} {
	old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// Weighted external path length = sum of freq*depth for all leaves
// Huffman minimizes this
func minWeightedPathLength(freqs []int) int {
	h := &IntHeap{}
	for _, f := range freqs {
		heap.Push(h, f)
	}

	total := 0
	for h.Len() > 1 {
		a := heap.Pop(h).(int)
		b := heap.Pop(h).(int)
		merged := a + b
		total += merged
		heap.Push(h, merged)
	}
	return total
}

func main() {
	freqs := []int{5, 9, 12, 13, 16, 45}
	fmt.Println("Min weighted path length:", minWeightedPathLength(freqs))
	// This equals the total number of bits needed

	fmt.Println()
	fmt.Println("Insight: Huffman minimizes Σ(freq_i × depth_i)")
	fmt.Println("which is exactly the total encoded message length")
}
```

---

## Example 10: Huffman Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Huffman Coding Summary ===")
	fmt.Println()

	fmt.Println("Algorithm:")
	fmt.Println("  1. Count character frequencies")
	fmt.Println("  2. Create leaf node for each character")
	fmt.Println("  3. Build min-heap from leaf nodes")
	fmt.Println("  4. While heap has > 1 node:")
	fmt.Println("     a. Extract two minimum nodes")
	fmt.Println("     b. Create parent with sum of frequencies")
	fmt.Println("     c. Insert parent back into heap")
	fmt.Println("  5. Root of resulting tree gives optimal codes")
	fmt.Println()

	fmt.Println("Properties:")
	fmt.Println("  • Prefix-free: no code is prefix of another")
	fmt.Println("  • Optimal: minimizes weighted path length")
	fmt.Println("  • Greedy: always merge two smallest frequencies")
	fmt.Println("  • Time: O(n log n) where n = unique characters")
	fmt.Println()

	fmt.Println("Applications:")
	fmt.Println("  • File compression (gzip, bzip2)")
	fmt.Println("  • JPEG, MP3 (as part of pipeline)")
	fmt.Println("  • Network data compression")
	fmt.Println("  • Connect ropes / merge files problems")
}
```

---

## Key Takeaways

1. Huffman coding builds an optimal prefix-free code using a greedy min-heap approach
2. Always merge two smallest frequency nodes — greedy choice is provably optimal
3. Produces variable-length codes: frequent chars get shorter codes
4. Same structure appears in connect ropes / merge files problems
5. Time O(n log n), Space O(n)

> **Next up:** Fractional Knapsack →
