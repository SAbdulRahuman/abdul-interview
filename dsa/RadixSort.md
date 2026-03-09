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

---

## Key Takeaways

1. Radix sort achieves O(d × n) — linear when d is constant
2. Relies on a stable sub-sort (counting sort) at each digit
3. LSD is simpler and more common; MSD is better for variable-length strings
4. Not comparison-based — breaks the Ω(n log n) lower bound
5. Excellent for sorting integers, fixed-length strings, and records by numeric key

> **Next up:** Bucket Sort →
