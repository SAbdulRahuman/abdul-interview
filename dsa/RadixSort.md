# Phase 16: Sorting Algorithms — Radix Sort

## Overview

**Radix Sort** sorts integers by processing individual digits, from least significant digit (LSD) to most significant digit (MSD). It uses a stable sub-sort (usually counting sort) at each digit position.

| Aspect | Detail |
|--------|--------|
| **Time** | O(d × (n + k)) where d = digits, k = base |
| **Space** | O(n + k) |
| **Stable** | Yes |
| **Type** | Non-comparison, digit-based |

---

## Example 1: LSD Radix Sort (Least Significant Digit)

```go
package main

import "fmt"

func radixSort(arr []int) {
	if len(arr) == 0 { return }

	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	// Process each digit
	for exp := 1; maxVal/exp > 0; exp *= 10 {
		countingSortByDigit(arr, exp)
	}
}

func countingSortByDigit(arr []int, exp int) {
	n := len(arr)
	output := make([]int, n)
	count := make([]int, 10)

	for _, v := range arr {
		count[(v/exp)%10]++
	}
	for i := 1; i < 10; i++ {
		count[i] += count[i-1]
	}
	for i := n - 1; i >= 0; i-- {
		digit := (arr[i] / exp) % 10
		count[digit]--
		output[count[digit]] = arr[i]
	}
	copy(arr, output)
}

func main() {
	arr := []int{170, 45, 75, 90, 802, 24, 2, 66}
	radixSort(arr)
	fmt.Println(arr) // [2 24 45 66 75 90 170 802]
}
```

**Textual Figure:**

```
LSD Radix Sort: arr = [170, 45, 75, 90, 802, 24, 2, 66]

  Pass 1 — sort by ONES digit:
  ┌───────┬───────────────────┐
  │ Digit │ Elements          │
  ├───────┼───────────────────┤
  │   0   │ 17(0), 9(0)       │
  │   2   │ 80(2), (2)        │
  │   4   │ 2(4)              │
  │   5   │ 4(5), 7(5)        │
  │   6   │ 6(6)              │
  └───────┴───────────────────┘
  → [170, 90, 802, 2, 24, 45, 75, 66]

  Pass 2 — sort by TENS digit:
  → [802, 2, 24, 45, 66, 170, 75, 90]

  Pass 3 — sort by HUNDREDS digit:
  → [2, 24, 45, 66, 75, 90, 170, 802]

  Result: [2, 24, 45, 66, 75, 90, 170, 802]
```

---

## Example 2: Radix Sort with Visualization

```go
package main

import "fmt"

func radixSortVisualize(arr []int) {
	if len(arr) == 0 { return }
	fmt.Println("Initial:", arr)

	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	pass := 1
	for exp := 1; maxVal/exp > 0; exp *= 10 {
		n := len(arr)
		output := make([]int, n)
		count := make([]int, 10)

		for _, v := range arr {
			count[(v/exp)%10]++
		}
		for i := 1; i < 10; i++ {
			count[i] += count[i-1]
		}
		for i := n - 1; i >= 0; i-- {
			d := (arr[i] / exp) % 10
			count[d]--
			output[count[d]] = arr[i]
		}
		copy(arr, output)
		fmt.Printf("Pass %d (digit %d): %v\n", pass, pass, arr)
		pass++
	}
}

func main() {
	arr := []int{170, 45, 75, 90, 802, 24, 2, 66}
	radixSortVisualize(arr)
}
```

**Textual Figure:**

```
Radix Sort Visualization: arr = [170, 45, 75, 90, 802, 24, 2, 66]

  Initial: [170, 45, 75, 90, 802, 24, 2, 66]

  ┌───────┬─────────────┬─────────────────────────────────┐
  │ Pass  │ Digit place │ Result                          │
  ├───────┼─────────────┼─────────────────────────────────┤
  │   1   │ ones (x%10) │ [170,90,802,2,24,45,75,66]      │
  │   2   │ tens        │ [802,2,24,45,66,170,75,90]      │
  │   3   │ hundreds    │ [2,24,45,66,75,90,170,802]      │
  └───────┴─────────────┴─────────────────────────────────┘

  Digit extraction: (value / exp) % 10
  170: ones=(170/1)%10=0, tens=(170/10)%10=7, hundreds=(170/100)%10=1
  Each pass uses stable counting sort on that digit.
```

---

## Example 3: MSD Radix Sort (Most Significant Digit)

```go
package main

import "fmt"

func msdRadixSort(arr []int) []int {
	if len(arr) <= 1 { return arr }

	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	// Find highest digit position
	exp := 1
	for maxVal/exp >= 10 {
		exp *= 10
	}

	return msdSort(arr, exp)
}

func msdSort(arr []int, exp int) []int {
	if len(arr) <= 1 || exp == 0 { return arr }

	// Partition into 10 buckets
	buckets := make([][]int, 10)
	for i := range buckets {
		buckets[i] = []int{}
	}

	for _, v := range arr {
		digit := (v / exp) % 10
		buckets[digit] = append(buckets[digit], v)
	}

	// Recursively sort each bucket on next digit
	result := make([]int, 0, len(arr))
	for _, bucket := range buckets {
		sorted := msdSort(bucket, exp/10)
		result = append(result, sorted...)
	}
	return result
}

func main() {
	arr := []int{170, 45, 75, 90, 802, 24, 2, 66}
	result := msdRadixSort(arr)
	fmt.Println(result) // [2 24 45 66 75 90 170 802]
}
```

**Textual Figure:**

```
MSD Radix Sort: arr = [170, 45, 75, 90, 802, 24, 2, 66]

  max=802, highest exp=100

  Pass 1 — hundreds digit (most significant):
  Bucket 0: [45, 75, 90, 24, 2, 66]  (hundreds=0)
  Bucket 1: [170]                    (hundreds=1)
  Bucket 8: [802]                    (hundreds=8)

  Recurse into Bucket 0 (exp=10, tens digit):
    Bucket 0: [2, 24]    (tens=0)
    Bucket 4: [45]       (tens=4)
    Bucket 6: [66]       (tens=6)
    Bucket 7: [75]       (tens=7)
    Bucket 9: [90]       (tens=9)

    Recurse into [2, 24] (exp=1, ones):
      Bucket 2: [2]      Bucket 4: [24]
      → [2, 24]

  Concatenate all buckets:
  [2, 24, 45, 66, 75, 90] + [170] + [802]

  Result: [2, 24, 45, 66, 75, 90, 170, 802]
  MSD: processes top-down, can short-circuit single-element buckets.
```

---

## Example 4: Radix Sort for Strings (Fixed Length)

```go
package main

import "fmt"

func radixSortStrings(arr []string, maxLen int) {
	if len(arr) == 0 { return }

	// Pad strings to maxLen
	for i := range arr {
		for len(arr[i]) < maxLen {
			arr[i] += " " // pad with spaces (lowest ASCII printable)
		}
	}

	// Sort from last character to first
	for pos := maxLen - 1; pos >= 0; pos-- {
		countSortByChar(arr, pos)
	}

	// Trim padding
	for i := range arr {
		j := len(arr[i]) - 1
		for j >= 0 && arr[i][j] == ' ' { j-- }
		arr[i] = arr[i][:j+1]
	}
}

func countSortByChar(arr []string, pos int) {
	n := len(arr)
	count := make([]int, 128)
	output := make([]string, n)

	for _, s := range arr {
		count[s[pos]]++
	}
	for i := 1; i < 128; i++ {
		count[i] += count[i-1]
	}
	for i := n - 1; i >= 0; i-- {
		ch := arr[i][pos]
		count[ch]--
		output[count[ch]] = arr[i]
	}
	copy(arr, output)
}

func main() {
	words := []string{"cat", "bat", "ant", "dog", "cow"}
	radixSortStrings(words, 3)
	fmt.Println(words) // [ant bat cat cow dog]
}
```

**Textual Figure:**

```
Radix Sort Strings: words = ["cat", "bat", "ant", "dog", "cow"]
  maxLen = 3, pad shorter words with spaces

  Pass 1 — sort by char at position 2 (last char):
  ┌─────┬─────────────────┐
  │ Char│ Words           │
  ├─────┼─────────────────┤
  │  g  │ dog             │
  │  t  │ cat, bat, ant   │
  │  w  │ cow             │
  └─────┴─────────────────┘
  → [dog, cat, bat, ant, cow]

  Pass 2 — sort by char at position 1 (middle):
  → [bat, cat, cow, dog, ant]

  Pass 3 — sort by char at position 0 (first):
  → [ant, bat, cat, cow, dog]

  Result: [ant, bat, cat, cow, dog]
  LSD string sort: process rightmost char first.
```

---

## Example 5: Radix Sort with Negative Numbers

```go
package main

import "fmt"

func radixSortWithNeg(arr []int) {
	// Separate positives and negatives
	var neg, pos []int
	for _, v := range arr {
		if v < 0 {
			neg = append(neg, -v)
		} else {
			pos = append(pos, v)
		}
	}

	// Sort both parts
	radixSortPositive(pos)
	radixSortPositive(neg)

	// Merge: reversed negatives + positives
	idx := 0
	for i := len(neg) - 1; i >= 0; i-- {
		arr[idx] = -neg[i]
		idx++
	}
	for _, v := range pos {
		arr[idx] = v
		idx++
	}
}

func radixSortPositive(arr []int) {
	if len(arr) == 0 { return }
	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}
	for exp := 1; maxVal/exp > 0; exp *= 10 {
		n := len(arr)
		output := make([]int, n)
		count := make([]int, 10)
		for _, v := range arr { count[(v/exp)%10]++ }
		for i := 1; i < 10; i++ { count[i] += count[i-1] }
		for i := n - 1; i >= 0; i-- {
			d := (arr[i] / exp) % 10
			count[d]--
			output[count[d]] = arr[i]
		}
		copy(arr, output)
	}
}

func main() {
	arr := []int{-5, 3, -1, 0, 8, -10, 2}
	radixSortWithNeg(arr)
	fmt.Println(arr) // [-10 -5 -1 0 2 3 8]
}
```

**Textual Figure:**

```
Radix Sort with Negatives: arr = [-5, 3, -1, 0, 8, -10, 2]

  Step 1: Separate positives and negatives
  neg (negate): [5, 1, 10]     pos: [3, 0, 8, 2]

  Step 2: Radix sort each group
  neg sorted: [1, 5, 10]
  pos sorted: [0, 2, 3, 8]

  Step 3: Merge — reverse negatives, negate back
  reversed neg: [10, 5, 1] → [-10, -5, -1]
  ┌─────┬────┬────┬────┬────┬────┬────┐
  │ -10 │ -5 │ -1 │  0 │  2 │  3 │  8 │
  └─────┴────┴────┴────┴────┴────┴────┘
   neg (reversed)     positives

  Result: [-10, -5, -1, 0, 2, 3, 8]
```

---

## Example 6: Base-256 Radix Sort (Byte-based)

```go
package main

import "fmt"

func radixSort256(arr []int) {
	if len(arr) == 0 { return }
	n := len(arr)

	// 4 passes for 32-bit integers (8 bits per pass)
	for shift := uint(0); shift < 32; shift += 8 {
		count := make([]int, 256)
		output := make([]int, n)

		for _, v := range arr {
			bucket := (v >> shift) & 0xFF
			count[bucket]++
		}
		for i := 1; i < 256; i++ {
			count[i] += count[i-1]
		}
		for i := n - 1; i >= 0; i-- {
			bucket := (arr[i] >> shift) & 0xFF
			count[bucket]--
			output[count[bucket]] = arr[i]
		}
		copy(arr, output)
	}
}

func main() {
	arr := []int{170, 45, 75, 90, 802, 24, 2, 66}
	radixSort256(arr)
	fmt.Println(arr) // [2 24 45 66 75 90 170 802]
}
```

**Textual Figure:**

```
Base-256 Radix Sort: process 8 bits (1 byte) per pass
  arr = [170, 45, 75, 90, 802, 24, 2, 66]

  For 32-bit integers: 4 passes (8 bits each)
  ┌──────┬────────┬────────┬────────┬────────┐
  │ Pass │ Bits   │ Mask   │ k      │ Buckets│
  ├──────┼────────┼────────┼────────┼────────┤
  │  1   │ 0-7    │ 0xFF   │ 256    │ 256    │
  │  2   │ 8-15   │ 0xFF   │ 256    │ 256    │
  │  3   │ 16-23  │ 0xFF   │ 256    │ 256    │
  │  4   │ 24-31  │ 0xFF   │ 256    │ 256    │
  └──────┴────────┴────────┴────────┴────────┘

  Digit extraction: (value >> shift) & 0xFF
  170 = 0x000000AA → byte0=0xAA, byte1=0x00, ...

  Base-10: d passes, k=10   → O(d×(n+10))
  Base-256: 4 passes, k=256 → O(4×(n+256))
  Fewer passes with larger base = faster for big n.

  Result: [2, 24, 45, 66, 75, 90, 170, 802]
```

---

## Example 7: Maximum Gap (LC 164) — Radix Sort Approach

```go
package main

import "fmt"

func maximumGap(nums []int) int {
	n := len(nums)
	if n < 2 { return 0 }

	maxVal := nums[0]
	for _, v := range nums {
		if v > maxVal { maxVal = v }
	}

	// Radix sort
	for exp := 1; maxVal/exp > 0; exp *= 10 {
		output := make([]int, n)
		count := make([]int, 10)
		for _, v := range nums { count[(v/exp)%10]++ }
		for i := 1; i < 10; i++ { count[i] += count[i-1] }
		for i := n - 1; i >= 0; i-- {
			d := (nums[i] / exp) % 10
			count[d]--
			output[count[d]] = nums[i]
		}
		copy(nums, output)
	}

	// Find max gap
	maxGap := 0
	for i := 1; i < n; i++ {
		gap := nums[i] - nums[i-1]
		if gap > maxGap { maxGap = gap }
	}
	return maxGap
}

func main() {
	fmt.Println(maximumGap([]int{3, 6, 9, 1}))  // 3
	fmt.Println(maximumGap([]int{10}))            // 0
}
```

**Textual Figure:**

```
Maximum Gap: nums = [3, 6, 9, 1]

  Step 1: Radix sort
  ┌───┬───┬───┬───┐        ┌───┬───┬───┬───┐
  │ 3 │ 6 │ 9 │ 1 │  →  │ 1 │ 3 │ 6 │ 9 │
  └───┴───┴───┴───┘        └───┴───┴───┴───┘

  Step 2: Find max gap between consecutive elements
  ┌───────────┬─────┐
  │ Pair      │ Gap │
  ├───────────┼─────┤
  │ 1 → 3     │  2  │
  │ 3 → 6     │  3  │ ← max
  │ 6 → 9     │  3  │ ← max
  └───────────┴─────┘

  Maximum gap = 3
  Time: O(d×(n+k)) for radix sort + O(n) scan = O(n)
```

---

## Example 8: Sort Large Dataset by ID

```go
package main

import "fmt"

type Record struct {
	ID   int
	Name string
}

func radixSortRecords(records []Record) {
	if len(records) == 0 { return }

	maxID := records[0].ID
	for _, r := range records {
		if r.ID > maxID { maxID = r.ID }
	}

	for exp := 1; maxID/exp > 0; exp *= 10 {
		n := len(records)
		output := make([]Record, n)
		count := make([]int, 10)

		for _, r := range records {
			count[(r.ID/exp)%10]++
		}
		for i := 1; i < 10; i++ {
			count[i] += count[i-1]
		}
		for i := n - 1; i >= 0; i-- {
			d := (records[i].ID / exp) % 10
			count[d]--
			output[count[d]] = records[i]
		}
		copy(records, output)
	}
}

func main() {
	records := []Record{
		{305, "Alice"}, {102, "Bob"}, {201, "Charlie"},
		{450, "Dave"}, {110, "Eve"},
	}
	radixSortRecords(records)
	for _, r := range records {
		fmt.Printf("ID=%d Name=%s\n", r.ID, r.Name)
	}
}
```

**Textual Figure:**

```
Sort Records by ID (Radix Sort):
  records = [(305,Alice),(102,Bob),(201,Charlie),(450,Dave),(110,Eve)]

  Pass 1 — ones digit:
  ┌─────┬────────────────┐
  │ Dig │ Records        │
  ├─────┼────────────────┤
  │  0  │ (110,Eve)      │
  │  0  │ (450,Dave)     │
  │  1  │ (201,Charlie)  │
  │  2  │ (102,Bob)      │
  │  5  │ (305,Alice)    │
  └─────┴────────────────┘
  → [(110,Eve),(450,Dave),(201,Charlie),(102,Bob),(305,Alice)]

  Pass 2 — tens, Pass 3 — hundreds...

  Final sorted by ID:
  102=Bob, 110=Eve, 201=Charlie, 305=Alice, 450=Dave

  Stable: records with same digit keep relative order.
```

---

## Example 9: Binary Radix Sort (Base 2)

```go
package main

import "fmt"

func binaryRadixSort(arr []int) {
	if len(arr) == 0 { return }

	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	// Process each bit
	for bit := 0; (1 << bit) <= maxVal; bit++ {
		// Stable partition: 0-bits first, then 1-bits
		zeros := []int{}
		ones := []int{}
		for _, v := range arr {
			if (v>>bit)&1 == 0 {
				zeros = append(zeros, v)
			} else {
				ones = append(ones, v)
			}
		}
		copy(arr, zeros)
		copy(arr[len(zeros):], ones)
	}
}

func main() {
	arr := []int{170, 45, 75, 90, 802, 24, 2, 66}
	binaryRadixSort(arr)
	fmt.Println(arr) // [2 24 45 66 75 90 170 802]
}
```

**Textual Figure:**

```
Binary Radix Sort: arr = [170, 45, 75, 90, 802, 24, 2, 66]

  Process each bit (LSB to MSB):
  ┌───────┬─────────────────────┬─────────────────────┐
  │ Bit   │ 0-bits              │ 1-bits              │
  ├───────┼─────────────────────┼─────────────────────┤
  │ bit 0 │ 170,90,802,24,2,66  │ 45,75               │
  │ bit 1 │ ...                 │ ...                 │
  │  ...  │                     │                     │
  └───────┴─────────────────────┴─────────────────────┘

  Each bit = 1 pass. max=802 → 10 bits needed.
  Stable partition: 0-bits go first, then 1-bits.

  Base-2 vs Base-10 vs Base-256:
  ┌──────────┬─────────┬──────────┬───────────┐
  │ Base     │ Passes  │ Buckets  │ Trade-off │
  ├──────────┼─────────┼──────────┼───────────┤
  │ 2        │ 32      │ 2        │ Many pass │
  │ 10       │ 10      │ 10       │ Balanced  │
  │ 256      │ 4       │ 256      │ Few pass  │
  └──────────┴─────────┴──────────┴───────────┘

  Result: [2, 24, 45, 66, 75, 90, 170, 802]
```

---

## Example 10: LSD vs MSD Comparison

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Radix Sort: LSD vs MSD ===")
	fmt.Println()

	fmt.Println("LSD (Least Significant Digit First):")
	fmt.Println("  • Process from rightmost to leftmost digit")
	fmt.Println("  • Always processes all d digits")
	fmt.Println("  • Naturally stable")
	fmt.Println("  • Simple iterative implementation")
	fmt.Println("  • Better for fixed-length keys")
	fmt.Println()

	fmt.Println("MSD (Most Significant Digit First):")
	fmt.Println("  • Process from leftmost to rightmost digit")
	fmt.Println("  • Can short-circuit (skip if bucket has 1 element)")
	fmt.Println("  • Requires recursion")
	fmt.Println("  • Better for variable-length strings")
	fmt.Println("  • Can produce sorted order for strings naturally")
	fmt.Println()

	fmt.Println("Time: O(d × (n + k))")
	fmt.Println("  d = number of digits")
	fmt.Println("  n = number of elements")
	fmt.Println("  k = range per digit (10 for decimal, 256 for byte)")
	fmt.Println()

	fmt.Println("Best when:")
	fmt.Println("  • d is small (few digits)")
	fmt.Println("  • n is large")
	fmt.Println("  • k is small relative to n")
	fmt.Println("  • O(d × n) ≈ O(n) when d is constant")
}
```

**Textual Figure:**

```
LSD vs MSD Radix Sort:

  LSD (Least Significant First):     MSD (Most Significant First):
  ┌─────────────────────┐     ┌─────────────────────┐
  │ Process right→left   │     │ Process left→right   │
  │ Iterative             │     │ Recursive             │
  │ Always d passes       │     │ Can short-circuit     │
  │ Naturally stable      │     │ Needs care for       │
  │ Fixed-length keys     │     │ stability             │
  │ Integers              │     │ Variable-length       │
  └─────────────────────┘     │ strings               │
                               └─────────────────────┘

  Example with [170, 45, 75]:

  LSD:                          MSD:
  Pass 1 (ones): [170,45,75]   Pass 1 (hundreds): B0:[45,75]
  Pass 2 (tens):  sorted tens                      B1:[170]
  Pass 3 (hundreds): done       Recurse B0 on tens...

  Time for both: O(d × (n + k))
  d = digits, n = elements, k = base size
```

---

## Key Takeaways

1. Radix sort achieves O(d × n) — linear when d is constant
2. Relies on a stable sub-sort (counting sort) at each digit
3. LSD is simpler and more common; MSD is better for variable-length strings
4. Not comparison-based — breaks the Ω(n log n) lower bound
5. Excellent for sorting integers, fixed-length strings, and records by numeric key

> **Next up:** Bucket Sort →
