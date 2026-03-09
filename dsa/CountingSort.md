# Phase 16: Sorting Algorithms — Counting Sort

## Overview

**Counting Sort** is a non-comparison sort that counts occurrences of each value, then uses cumulative counts to place elements. It runs in O(n + k) where k is the range of input values.

| Aspect | Detail |
|--------|--------|
| **Time** | O(n + k) |
| **Space** | O(n + k) |
| **Stable** | Yes (with proper implementation) |
| **Type** | Non-comparison, integer-based |

---

## Example 1: Basic Counting Sort

```go
package main

import "fmt"

func countingSort(arr []int) []int {
	if len(arr) == 0 { return arr }

	// Find range
	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	// Count occurrences
	count := make([]int, maxVal+1)
	for _, v := range arr {
		count[v]++
	}

	// Build result
	result := make([]int, 0, len(arr))
	for i, c := range count {
		for j := 0; j < c; j++ {
			result = append(result, i)
		}
	}
	return result
}

func main() {
	arr := []int{4, 2, 2, 8, 3, 3, 1}
	fmt.Println(countingSort(arr)) // [1 2 2 3 3 4 8]
}
```

---

## Example 2: Stable Counting Sort (Preserving Order)

```go
package main

import "fmt"

func stableCountingSort(arr []int) []int {
	if len(arr) == 0 { return arr }
	n := len(arr)

	maxVal := arr[0]
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	count := make([]int, maxVal+1)
	for _, v := range arr {
		count[v]++
	}

	// Cumulative count — prefix sum
	for i := 1; i <= maxVal; i++ {
		count[i] += count[i-1]
	}

	// Place elements in reverse for stability
	output := make([]int, n)
	for i := n - 1; i >= 0; i-- {
		count[arr[i]]--
		output[count[arr[i]]] = arr[i]
	}
	return output
}

func main() {
	arr := []int{4, 2, 2, 8, 3, 3, 1}
	fmt.Println(stableCountingSort(arr)) // [1 2 2 3 3 4 8]
}
```

---

## Example 3: Counting Sort with Negative Numbers

```go
package main

import "fmt"

func countingSortNeg(arr []int) []int {
	if len(arr) == 0 { return arr }

	minVal, maxVal := arr[0], arr[0]
	for _, v := range arr {
		if v < minVal { minVal = v }
		if v > maxVal { maxVal = v }
	}

	rang := maxVal - minVal + 1
	count := make([]int, rang)
	for _, v := range arr {
		count[v-minVal]++
	}

	// Prefix sum
	for i := 1; i < rang; i++ {
		count[i] += count[i-1]
	}

	output := make([]int, len(arr))
	for i := len(arr) - 1; i >= 0; i-- {
		idx := arr[i] - minVal
		count[idx]--
		output[count[idx]] = arr[i]
	}
	return output
}

func main() {
	arr := []int{-5, -10, 0, -3, 8, 5, -1, 10}
	fmt.Println(countingSortNeg(arr)) // [-10 -5 -3 -1 0 5 8 10]
}
```

---

## Example 4: Sort Characters in a String

```go
package main

import "fmt"

func sortString(s string) string {
	count := make([]int, 128) // ASCII
	for _, c := range s {
		count[c]++
	}

	result := make([]byte, 0, len(s))
	for i, c := range count {
		for j := 0; j < c; j++ {
			result = append(result, byte(i))
		}
	}
	return string(result)
}

func main() {
	fmt.Println(sortString("programming")) // aggimnoprrm → sorted
	fmt.Println(sortString("hello"))       // ehllo
}
```

---

## Example 5: Find Maximum Frequency Element

```go
package main

import "fmt"

func maxFrequency(arr []int) (int, int) {
	if len(arr) == 0 { return 0, 0 }

	maxVal := 0
	for _, v := range arr {
		if v > maxVal { maxVal = v }
	}

	count := make([]int, maxVal+1)
	for _, v := range arr {
		count[v]++
	}

	maxFreq, element := 0, 0
	for i, c := range count {
		if c > maxFreq {
			maxFreq = c
			element = i
		}
	}
	return element, maxFreq
}

func main() {
	arr := []int{1, 3, 2, 3, 4, 3, 2, 1, 3}
	elem, freq := maxFrequency(arr)
	fmt.Printf("Element %d appears %d times\n", elem, freq) // 3 appears 4 times
}
```

---

## Example 6: Sort by Digit (Helper for Radix Sort)

```go
package main

import "fmt"

func countingSortByDigit(arr []int, exp int) []int {
	n := len(arr)
	output := make([]int, n)
	count := make([]int, 10)

	// Count occurrences of digit
	for _, v := range arr {
		digit := (v / exp) % 10
		count[digit]++
	}

	// Cumulative
	for i := 1; i < 10; i++ {
		count[i] += count[i-1]
	}

	// Build output (reverse for stability)
	for i := n - 1; i >= 0; i-- {
		digit := (arr[i] / exp) % 10
		count[digit]--
		output[count[digit]] = arr[i]
	}
	return output
}

func main() {
	arr := []int{170, 45, 75, 90, 802, 24, 2, 66}
	fmt.Println("Sort by ones:", countingSortByDigit(arr, 1))
	fmt.Println("Sort by tens:", countingSortByDigit(arr, 10))
}
```

---

## Example 7: Sort 0s, 1s, and 2s (Counting Approach)

```go
package main

import "fmt"

func sortColors(nums []int) {
	count := [3]int{}
	for _, v := range nums {
		count[v]++
	}

	idx := 0
	for color := 0; color < 3; color++ {
		for i := 0; i < count[color]; i++ {
			nums[idx] = color
			idx++
		}
	}
}

func main() {
	nums := []int{2, 0, 2, 1, 1, 0}
	sortColors(nums)
	fmt.Println(nums) // [0 0 1 1 2 2]
}
```

---

## Example 8: Relative Sort Array (LC 1122)

```go
package main

import "fmt"

func relativeSortArray(arr1, arr2 []int) []int {
	maxVal := 0
	for _, v := range arr1 {
		if v > maxVal { maxVal = v }
	}

	count := make([]int, maxVal+1)
	for _, v := range arr1 {
		count[v]++
	}

	result := make([]int, 0, len(arr1))

	// First: elements in arr2 order
	for _, v := range arr2 {
		for count[v] > 0 {
			result = append(result, v)
			count[v]--
		}
	}

	// Then: remaining elements in ascending order
	for i, c := range count {
		for j := 0; j < c; j++ {
			result = append(result, i)
		}
	}
	return result
}

func main() {
	arr1 := []int{2, 3, 1, 3, 2, 4, 6, 7, 9, 2, 19}
	arr2 := []int{2, 1, 4, 3, 9, 6}
	fmt.Println(relativeSortArray(arr1, arr2))
	// [2 2 2 1 4 3 3 9 6 7 19]
}
```

---

## Example 9: H-Index Using Counting Sort

```go
package main

import "fmt"

func hIndex(citations []int) int {
	n := len(citations)
	count := make([]int, n+1)

	// Cap citations at n (anything ≥ n is the same)
	for _, c := range citations {
		if c >= n {
			count[n]++
		} else {
			count[c]++
		}
	}

	// Cumulative from right
	total := 0
	for i := n; i >= 0; i-- {
		total += count[i]
		if total >= i {
			return i
		}
	}
	return 0
}

func main() {
	fmt.Println(hIndex([]int{3, 0, 6, 1, 5})) // 3
	fmt.Println(hIndex([]int{1, 3, 1}))        // 1
}
```

---

## Example 10: When to Use Counting Sort

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Counting Sort — When to Use ===")
	fmt.Println()

	fmt.Println("USE when:")
	fmt.Println("  • Integer values with known, small range (k)")
	fmt.Println("  • k = O(n) — linear time guaranteed")
	fmt.Println("  • Need stable sort")
	fmt.Println("  • Sorting characters, grades, ages, etc.")
	fmt.Println()

	fmt.Println("AVOID when:")
	fmt.Println("  • Large range (k >> n) — wastes memory")
	fmt.Println("  • Floating point numbers")
	fmt.Println("  • Complex objects (use as subroutine with key extraction)")
	fmt.Println()

	fmt.Println("Comparison:")
	fmt.Println("  Counting sort: O(n+k) time, O(n+k) space")
	fmt.Println("  Merge sort:    O(n log n) time, O(n) space")
	fmt.Println("  Quick sort:    O(n log n) avg, O(1) extra space")
	fmt.Println()
	fmt.Println("  When k ≤ n: counting sort wins")
	fmt.Println("  When k >> n: comparison sorts win")
}
```

---

## Key Takeaways

1. O(n + k) time — faster than comparison sorts when k is small
2. Must know the range of input values
3. Stable with prefix sum + reverse traversal
4. Foundation for radix sort
5. Trade-off: speed for memory — needs O(k) space for counts

> **Next up:** Radix Sort →
