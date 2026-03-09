# Phase 20: Bit Manipulation — Bit Counting

## Overview

**Bit counting** (population count / popcount) determines how many 1-bits are in a number's binary representation. This is used in Hamming weight, error detection, similarity measures, and optimization.

| Method | Time | Space |
|--------|------|-------|
| Iterate all bits | O(w) | O(1) |
| Brian Kernighan | O(k), k = set bits | O(1) |
| Lookup table | O(1) | O(256) |
| Parallel (divide-conquer) | O(log w) | O(1) |
| Go `bits.OnesCount` | O(1) | O(1) (POPCNT) |

---

## Example 1: Five Methods to Count Bits

```go
package main

import (
	"fmt"
	"math/bits"
)

func countNaive(n uint) int {
	c := 0
	for n > 0 { c += int(n & 1); n >>= 1 }
	return c
}

func countKernighan(n uint) int {
	c := 0
	for n > 0 { n &= n - 1; c++ }
	return c
}

var lookupTable [256]int

func init() {
	for i := 1; i < 256; i++ {
		lookupTable[i] = lookupTable[i>>1] + int(i&1)
	}
}

func countLookup(n uint) int {
	c := 0
	for n > 0 {
		c += lookupTable[n&0xFF]
		n >>= 8
	}
	return c
}

// Parallel bit counting (divide and conquer)
func countParallel(n uint) int {
	n = n - ((n >> 1) & 0x5555555555555555)
	n = (n & 0x3333333333333333) + ((n >> 2) & 0x3333333333333333)
	n = (n + (n >> 4)) & 0x0F0F0F0F0F0F0F0F
	return int((n * 0x0101010101010101) >> 56)
}

func countStdlib(n uint) int { return bits.OnesCount(n) }

func main() {
	tests := []uint{0, 1, 7, 42, 255, 1023, 0xDEADBEEF}
	for _, n := range tests {
		fmt.Printf("n=%-12d naive=%d kern=%d lookup=%d parallel=%d stdlib=%d\n",
			n, countNaive(n), countKernighan(n), countLookup(n),
			countParallel(n), countStdlib(n))
	}
}
```

---

## Example 2: Counting Bits 0 to N (LeetCode 338)

```go
package main

import "fmt"

// O(n) using dynamic programming
func countBits(n int) []int {
	ans := make([]int, n+1)
	for i := 1; i <= n; i++ {
		// Option 1: ans[i] = ans[i>>1] + (i & 1)
		// Option 2: ans[i] = ans[i&(i-1)] + 1
		ans[i] = ans[i>>1] + (i & 1)
	}
	return ans
}

func main() {
	n := 15
	result := countBits(n)
	for i, c := range result {
		fmt.Printf("  %2d = %04b → %d ones\n", i, i, c)
	}
}
```

---

## Example 3: Hamming Weight Variants

```go
package main

import (
	"fmt"
	"math/bits"
)

func main() {
	// Unsigned hamming weight
	nums := []uint{0, 11, 128, 255, 4294967293}
	for _, n := range nums {
		fmt.Printf("HammingWeight(%d) = %d\n", n, bits.OnesCount(n))
	}

	fmt.Println()

	// Signed integers: count set bits in two's complement
	signed := []int{-1, -2, -128, 0, 7}
	for _, n := range signed {
		// In Go, int can be 64-bit
		fmt.Printf("HammingWeight(%4d) = %d  (as uint64: %064b)\n",
			n, bits.OnesCount64(uint64(n)), uint64(n))
	}
}
```

---

## Example 4: Count Total Set Bits from 1 to N

```go
package main

import "fmt"

// Count total number of 1-bits in all integers from 1 to n
func totalSetBits(n int) int {
	if n <= 0 { return 0 }

	total := 0
	for bit := 0; (1 << bit) <= n; bit++ {
		// For each bit position, count how many numbers have it set
		cycle := 1 << (bit + 1)
		full := n / cycle
		remainder := n % cycle

		total += full * (1 << bit)
		if remainder >= (1 << bit) {
			extra := remainder - (1 << bit) + 1
			total += extra
		}
	}
	return total
}

// Brute force for verification
func totalSetBitsBrute(n int) int {
	total := 0
	for i := 1; i <= n; i++ {
		x := i
		for x > 0 { total += x & 1; x >>= 1 }
	}
	return total
}

func main() {
	for _, n := range []int{1, 5, 10, 15, 100, 1000} {
		fast := totalSetBits(n)
		brute := totalSetBitsBrute(n)
		match := "✓"
		if fast != brute { match = "✗" }
		fmt.Printf("TotalBits(1..%4d) = %6d %s\n", n, fast, match)
	}
}
```

---

## Example 5: Sort by Number of Set Bits (LeetCode 1356)

```go
package main

import (
	"fmt"
	"math/bits"
	"sort"
)

func sortByBits(arr []int) []int {
	sort.Slice(arr, func(i, j int) bool {
		ci := bits.OnesCount(uint(arr[i]))
		cj := bits.OnesCount(uint(arr[j]))
		if ci != cj { return ci < cj }
		return arr[i] < arr[j]
	})
	return arr
}

func main() {
	arr := []int{0, 1, 2, 3, 4, 5, 6, 7, 8}
	sorted := sortByBits(arr)
	fmt.Println("Sorted by bit count:", sorted)
	for _, n := range sorted {
		fmt.Printf("  %d → %04b (%d bits)\n", n, n, bits.OnesCount(uint(n)))
	}
}
```

---

## Example 6: Binary Watch (LeetCode 401)

```go
package main

import (
	"fmt"
	"math/bits"
)

func readBinaryWatch(turnedOn int) []string {
	result := []string{}
	for h := 0; h < 12; h++ {
		for m := 0; m < 60; m++ {
			if bits.OnesCount(uint(h))+bits.OnesCount(uint(m)) == turnedOn {
				result = append(result, fmt.Sprintf("%d:%02d", h, m))
			}
		}
	}
	return result
}

func main() {
	for leds := 0; leds <= 3; leds++ {
		times := readBinaryWatch(leds)
		fmt.Printf("LEDs=%d: %d times (first 5: ", leds, len(times))
		for i := 0; i < 5 && i < len(times); i++ {
			if i > 0 { fmt.Print(", ") }
			fmt.Print(times[i])
		}
		fmt.Println("...)")
	}
}
```

---

## Example 7: Maximum Product of Word Lengths (LeetCode 318)

```go
package main

import (
	"fmt"
)

func maxProduct(words []string) int {
	n := len(words)
	masks := make([]int, n)

	// Convert each word to bitmask of characters
	for i, w := range words {
		for _, ch := range w {
			masks[i] |= 1 << (ch - 'a')
		}
	}

	maxProd := 0
	for i := 0; i < n; i++ {
		for j := i + 1; j < n; j++ {
			// No common letters if masks don't share any bit
			if masks[i]&masks[j] == 0 {
				prod := len(words[i]) * len(words[j])
				if prod > maxProd { maxProd = prod }
			}
		}
	}
	return maxProd
}

func main() {
	words := []string{"abcw", "baz", "foo", "bar", "xtfn", "abcdef"}
	fmt.Println("Max product:", maxProduct(words))
	// "abcw" (4) * "xtfn" (4) = 16 (no common letters)
}
```

---

## Example 8: Number Complement (LeetCode 476)

```go
package main

import (
	"fmt"
	"math/bits"
)

func findComplement(num int) int {
	if num == 0 { return 1 }
	// Create mask with all 1s up to highest bit
	bitLen := bits.Len(uint(num))
	mask := (1 << bitLen) - 1
	return num ^ mask
}

func main() {
	tests := []int{5, 1, 7, 10}
	for _, n := range tests {
		comp := findComplement(n)
		fmt.Printf("n=%d (%08b) → complement=%d (%08b)\n", n, n, comp, comp)
	}
	fmt.Println()
	fmt.Println("bits.Len tells us the bit length, then XOR with all-1s mask")
}
```

---

## Example 9: Prime Number of Set Bits (LeetCode 762)

```go
package main

import (
	"fmt"
	"math/bits"
)

func countPrimeSetBits(left, right int) int {
	// Set bits can be 0..20 for numbers up to 10^6
	// Primes up to 20: 2,3,5,7,11,13,17,19
	primeSet := 0
	for _, p := range []int{2, 3, 5, 7, 11, 13, 17, 19} {
		primeSet |= 1 << p
	}

	count := 0
	for i := left; i <= right; i++ {
		setBits := bits.OnesCount(uint(i))
		if primeSet&(1<<setBits) != 0 {
			count++
		}
	}
	return count
}

func main() {
	tests := [][2]int{{6, 10}, {10, 15}, {1, 100}}
	for _, t := range tests {
		fmt.Printf("[%d, %d] → %d numbers with prime set bits\n",
			t[0], t[1], countPrimeSetBits(t[0], t[1]))
	}
}
```

---

## Example 10: Bit Counting Pattern Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bit Counting Patterns Summary ===")
	fmt.Println()

	patterns := []struct{ name, formula, complexity string }{
		{"Loop & shift", "while n: c += n&1; n >>= 1", "O(w), w = word size"},
		{"Kernighan", "while n: n &= n-1; c++", "O(k), k = set bits"},
		{"Lookup table", "table[byte0] + table[byte1] + ...", "O(1) amortized"},
		{"Parallel", "Divide & conquer with masks", "O(log w)"},
		{"DP: i>>1", "ans[i] = ans[i>>1] + (i&1)", "O(n) for range"},
		{"DP: i&(i-1)", "ans[i] = ans[i&(i-1)] + 1", "O(n) for range"},
		{"Go stdlib", "bits.OnesCount(uint(n))", "O(1) hw POPCNT"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-14s: %-40s %s\n", p.name, p.formula, p.complexity)
	}

	fmt.Println()
	fmt.Println("Applications:")
	fmt.Println("  • Hamming weight / distance")
	fmt.Println("  • Binary watch")
	fmt.Println("  • Sort by popcount")
	fmt.Println("  • Word length products (letter bitmasks)")
	fmt.Println("  • Total set bits in range [1..n]")
}
```

---

## Key Takeaways

1. DP for range: `bits[i] = bits[i>>1] + (i&1)` or `bits[i&(i-1)] + 1`
2. Total set bits in [1..n]: O(log n) by analyzing each bit position's contribution
3. Bitmask of characters enables O(1) "no common letters" check
4. `bits.OnesCount` = Go's built-in POPCNT, fastest for production
5. Bit counting is foundational for Hamming distance, similarity, and subset problems

> **Next up:** Bit Shifting →
