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

**Textual Figure:**
```
Huffman Tree Construction: freq = {a:5, b:9, c:12, d:13, e:16, f:45}

Step 1: Merge a(5) + b(9) = 14        Step 2: Merge c(12) + d(13) = 25
  Min-heap: [12,13,14,16,45]            Min-heap: [14,16,25,45]

Step 3: Merge 14 + 16 = 30            Step 4: Merge 25 + 30 = 55
  Min-heap: [25,30,45]                  Min-heap: [45,55]

Step 5: Merge 45 + 55 = 100 (root)

             (100)
            ┏━━━━┓
          0┏     ┓ 1
         (55)    f:45
        ┏━━━┓
      0┏     ┓ 1
     (25)   (30)
    ┏━━┓   ┏━━┓
  0┏  ┓1 0┏  ┓1
  c:12 d:13 (14) e:16
           ┏━━┓
         0┏  ┓1
         a:5  b:9

Root frequency: 100
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

**Textual Figure:**
```
Huffman Codes Generated from tree:

             (100)
            ┏━━━━┓
          0┏     ┓ 1
         (55)    f:45
        ┏━━━┓
      0┏     ┓ 1
     (25)   (30)
    ┏━━┓   ┏━━┓
  0┏  ┓1 0┏  ┓1
  c:12 d:13 (14) e:16
           ┏━━┓
         0┏  ┓1
         a:5  b:9

  Codes:
  ┌──────┬──────┬────────┬───────┐
  │ Char │ Freq │ Code   │ Bits  │
  ├──────┼──────┼────────┼───────┤
  │  f   │  45  │ 1      │  1    │
  │  c   │  12  │ 000    │  3    │
  │  d   │  13  │ 001    │  3    │
  │  a   │   5  │ 0100   │  4    │
  │  b   │   9  │ 0101   │  4    │
  │  e   │  16  │ 011    │  3    │
  └──────┴──────┴────────┴───────┘
  Higher freq → shorter code (f gets 1 bit!)
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

**Textual Figure:**
```
Huffman Encode/Decode: "huffman coding is fun"

Encode process:
  h → code_h | u → code_u | f → code_f | ...
  Each char mapped to variable-length bit string

Decode process (traverse tree):
  bits: 1 0 1 1 0 0 ...
         │
         └── start at root
              0 → go left
              1 → go right
              leaf? → output char, restart at root

  ┌───────────────────────────────────────┐
  │ Original:     21 chars × 8 = 168 bits   │
  │ Huffman:      ~70-90 bits               │
  │ Compression:  ~50% savings               │
  │ Decoded == Original: true ✓              │
  └───────────────────────────────────────┘

Prefix-free property ensures unique decodability
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

**Textual Figure:**
```
Minimum Cost to Merge Files: [4, 3, 2, 6]

Step-by-step (always merge two smallest):

  Heap: [2, 3, 4, 6]

  Step 1: Merge 2+3 = 5 (cost=5)
    Heap: [4, 5, 6]
         (5)
        ┏━┓
       2   3

  Step 2: Merge 4+5 = 9 (cost=9)
    Heap: [6, 9]
           (9)
          ┏━┓
         4  (5)
           ┏━┓
          2   3

  Step 3: Merge 6+9 = 15 (cost=15)
    Heap: [15]
              (15)
             ┏━━┓
            6   (9)
               ┏━┓
              4  (5)
                ┏━┓
               2   3

  Total cost: 5 + 9 + 15 = 29
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

**Textual Figure:**
```
Connect Ropes: [1, 2, 3, 4, 5]

Always connect two shortest ropes:

  Heap: [1, 2, 3, 4, 5]

  Step 1: 1+2 = 3 (cost=3)     Heap: [3, 3, 4, 5]
  Step 2: 3+3 = 6 (cost=6)     Heap: [4, 5, 6]
  Step 3: 4+5 = 9 (cost=9)     Heap: [6, 9]
  Step 4: 6+9 = 15 (cost=15)   Heap: [15]

         (15)
        ┏━━━┓
      (6)   (9)
     ┏━┓   ┏━┓
   (3)  3  4   5
   ┏━┓
  1   2

  Total cost: 3 + 6 + 9 + 15 = 33
  (Same greedy as Huffman tree construction)
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

**Textual Figure:**
```
Huffman Compression Analysis: "aaaaaabbbbbccccdddeef"

Frequencies:
  a:6  b:5  c:4  d:3  e:2  f:1

Tree construction (merge smallest pairs):
  e(2)+f(1)=3 → d(3)+3=6 → c(4)+b(5)=9 → a(6)+6=12 → 9+12=21

  ┌──────┬──────┬────────┬──────────┐
  │ Char │ Freq │ Code   │ Bits Used │
  ├──────┼──────┼────────┼──────────┤
  │  a   │   6  │ ~2 bit │   12     │
  │  b   │   5  │ ~2 bit │   10     │
  │  c   │   4  │ ~2 bit │    8     │
  │  d   │   3  │ ~3 bit │    9     │
  │  e   │   2  │ ~4 bit │    8     │
  │  f   │   1  │ ~4 bit │    4     │
  └──────┴──────┴────────┴──────────┘

  Fixed (8-bit):  21 × 8 = 168 bits
  Huffman:        ~51 bits
  Savings:        ~70% compression
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

**Textual Figure:**
```
Prefix-Free Property Verification

┌─── Huffman codes (prefix-free) ─────────┐
│  Codes: {0, 10, 110, 111}             │
│                                        │
│  0    is NOT a prefix of 10   ✓       │
│  0    is NOT a prefix of 110  ✓       │
│  10   is NOT a prefix of 110  ✓       │
│  10   is NOT a prefix of 111  ✓       │
│  110  is NOT a prefix of 111  ✓       │
│                                        │
│  Decode: 01011010111                   │
│  0|10|110|10|111 → unambiguous ✓      │
└────────────────────────────────────────┘

┌─── Bad codes (NOT prefix-free) ────────┐
│  Codes: {0, 01, 1, 10}                │
│                                        │
│  0 IS a prefix of 01 ✗                │
│  1 IS a prefix of 10 ✗                │
│                                        │
│  "01" = (0)(1) or (01)?               │
│  Ambiguous! Cannot decode uniquely.    │
└────────────────────────────────────────┘
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

**Textual Figure:**
```
Huffman with Priority Queue: "go is great for golang gophers"

Character frequencies (approximate):
  ' ':5  g:4  o:4  r:3  a:2  e:2  ...

Priority Queue (min-heap) processing:
  Round 1: pop 2 least freq, merge
  Round 2: pop 2 least freq, merge
  ...repeat until 1 node remains

  PQ: [○○○○○○○○○○○○○]  (leaf nodes)
       ↓ merge pairs
  PQ: [○○○○○○○]       (some internal nodes)
       ↓ merge pairs
  PQ: [○○○○]            (fewer nodes)
       ↓ merge
  PQ: [●]               (root)

  Result: variable-length codes per character
  Frequent chars (space, g, o) get shorter codes
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

**Textual Figure:**
```
Weighted Path Length: freqs = [5, 9, 12, 13, 16, 45]

Huffman tree minimizes Σ(freq_i × depth_i)

             (100)
            ┏━━━━┓
          0┏     ┓ 1
         (55)    f:45  depth=1, cost=45×1=45
        ┏━━━┓
      0┏     ┓ 1
     (25)   (30)
    ┏━━┓   ┏━━┓
  c:12 d:13 (14) e:16   depth=3, depth=2
           ┏━━┓
         a:5  b:9        depth=4

  Weighted path length:
    f: 45×1 = 45
    c: 12×3 = 36
    d: 13×3 = 39
    e: 16×3 = 48
    a:  5×4 = 20
    b:  9×4 = 36
    Total:    224

  This equals total bits to encode the message
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

**Textual Figure:**
```
Huffman Coding Algorithm Summary

┌─────────────────────────────────────────────┐
│ Step 1: Count frequencies                    │
│   {a:5, b:9, c:12, d:13, e:16, f:45}        │
├─────────────────────────────────────────────┤
│ Step 2: Build min-heap of leaf nodes          │
│   Heap: [5, 9, 12, 13, 16, 45]               │
├─────────────────────────────────────────────┤
│ Step 3: While heap.size > 1:                  │
│   Extract min1, min2                          │
│   Create parent(freq = min1+min2)             │
│   Insert parent back                          │
├─────────────────────────────────────────────┤
│ Step 4: Assign codes                          │
│   Left edge = 0, Right edge = 1               │
│   Traverse root to each leaf                  │
├─────────────────────────────────────────────┤
│ Applications:                                 │
│   gzip, JPEG, MP3, connect ropes               │
└─────────────────────────────────────────────┘
```

---

## Key Takeaways

1. Huffman coding builds an optimal prefix-free code using a greedy min-heap approach
2. Always merge two smallest frequency nodes — greedy choice is provably optimal
3. Produces variable-length codes: frequent chars get shorter codes
4. Same structure appears in connect ropes / merge files problems
5. Time O(n log n), Space O(n)

> **Next up:** Fractional Knapsack →
