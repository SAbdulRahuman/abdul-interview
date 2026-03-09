# Phase 20: Bit Manipulation — Bitwise Tricks

## Overview

**Bitwise tricks** are clever bit manipulation patterns that replace branching, arithmetic, or loops with O(1) bitwise operations. They appear frequently in competitive programming and systems code.

---

## Example 1: Bit Tricks Collection

```go
package main

import "fmt"

func main() {
	n := 42  // 101010

	fmt.Printf("n = %d = %08b\n\n", n, n)

	// 1. Check if n is odd
	fmt.Printf("Is odd:           %v (n&1=%d)\n", n&1 == 1, n&1)

	// 2. Multiply by 2
	fmt.Printf("n * 2:            %d (n<<1=%d)\n", n*2, n<<1)

	// 3. Divide by 2
	fmt.Printf("n / 2:            %d (n>>1=%d)\n", n/2, n>>1)

	// 4. Toggle case (works for ASCII letters)
	ch := byte('A')
	fmt.Printf("Toggle 'A':       %c (ch^32=%c)\n", ch, ch^32)

	// 5. Uppercase
	ch2 := byte('h')
	fmt.Printf("Upper 'h':        %c (ch&^32=%c)\n", ch2, ch2&^32)

	// 6. Lowercase
	ch3 := byte('H')
	fmt.Printf("Lower 'H':        %c (ch|32=%c)\n", ch3, ch3|32)

	// 7. Check if two have same sign
	a, b := -5, -3
	fmt.Printf("Same sign(%d,%d): %v\n", a, b, (a^b) >= 0)

	// 8. Average without overflow
	x, y := 100, 200
	avg := (x & y) + (x^y)>>1
	fmt.Printf("Avg(%d,%d):       %d (safe: %d)\n", x, y, (x+y)/2, avg)
}
```

---

## Example 2: Swap Bits at Positions

```go
package main

import "fmt"

func swapBits(n, i, j int) int {
	// Only swap if bits differ
	if (n>>i)&1 != (n>>j)&1 {
		mask := (1 << i) | (1 << j)
		n ^= mask  // toggle both bits
	}
	return n
}

func main() {
	n := 0b10110010
	fmt.Printf("n = %08b\n", n)

	for i := 0; i < 8; i++ {
		for j := i + 1; j < 8; j++ {
			if (n>>i)&1 != (n>>j)&1 { // only show swaps that change something
				result := swapBits(n, i, j)
				fmt.Printf("  swap bits %d,%d: %08b → %08b\n", i, j, n, result)
			}
		}
	}
}
```

---

## Example 3: Letter Case Manipulation with XOR

```go
package main

import "fmt"

func toggleCase(s string) string {
	result := []byte(s)
	for i, ch := range result {
		if (ch >= 'A' && ch <= 'Z') || (ch >= 'a' && ch <= 'z') {
			result[i] = ch ^ 32  // toggle bit 5
		}
	}
	return string(result)
}

func toUpper(s string) string {
	result := []byte(s)
	for i, ch := range result {
		if ch >= 'a' && ch <= 'z' {
			result[i] = ch &^ 32  // clear bit 5
		}
	}
	return string(result)
}

func toLower(s string) string {
	result := []byte(s)
	for i, ch := range result {
		if ch >= 'A' && ch <= 'Z' {
			result[i] = ch | 32  // set bit 5
		}
	}
	return string(result)
}

func main() {
	s := "Hello World"
	fmt.Printf("Original:    '%s'\n", s)
	fmt.Printf("Toggle case: '%s'\n", toggleCase(s))
	fmt.Printf("Upper:       '%s'\n", toUpper(s))
	fmt.Printf("Lower:       '%s'\n", toLower(s))

	fmt.Println()
	fmt.Println("ASCII trick: bit 5 (value 32) is the case bit")
	fmt.Printf("  'A' = %08b, 'a' = %08b (differ at bit 5)\n", 'A', 'a')
}
```

---

## Example 4: Branchless Min/Max/Abs/Clamp

```go
package main

import "fmt"

func branchlessMin(a, b int) int {
	return b + ((a - b) & ((a - b) >> 63))
}

func branchlessMax(a, b int) int {
	return a - ((a - b) & ((a - b) >> 63))
}

func branchlessAbs(n int) int {
	mask := n >> 63
	return (n ^ mask) - mask
}

func branchlessClamp(val, lo, hi int) int {
	val = branchlessMax(val, lo)
	val = branchlessMin(val, hi)
	return val
}

func main() {
	fmt.Println("Branchless min/max/abs:")
	pairs := [][2]int{{5, 3}, {-2, 7}, {0, 0}, {-10, -3}}
	for _, p := range pairs {
		fmt.Printf("  min(%3d,%3d)=%3d  max(%3d,%3d)=%3d\n",
			p[0], p[1], branchlessMin(p[0], p[1]),
			p[0], p[1], branchlessMax(p[0], p[1]))
	}

	fmt.Println("\nBranchless abs:")
	for _, n := range []int{5, -5, 0, -100, 42} {
		fmt.Printf("  abs(%4d) = %d\n", n, branchlessAbs(n))
	}

	fmt.Println("\nBranchless clamp [0, 255]:")
	for _, n := range []int{-10, 0, 128, 255, 300} {
		fmt.Printf("  clamp(%4d) = %d\n", n, branchlessClamp(n, 0, 255))
	}
}
```

---

## Example 5: Detect if Subtraction Overflows

```go
package main

import "fmt"

func willSubOverflow(a, b int32) bool {
	// Overflow if signs of a and b differ AND result sign differs from a
	diff := int32(int64(a) - int64(b))
	return ((a ^ b) & (a ^ diff)) < 0
}

func main() {
	tests := [][2]int32{
		{100, 50},
		{-2147483648, 1},   // MinInt32 - 1 → overflow
		{2147483647, -1},   // MaxInt32 - (-1) → overflow
		{0, 0},
	}
	for _, t := range tests {
		fmt.Printf("  %d - %d: overflow=%v\n", t[0], t[1], willSubOverflow(t[0], t[1]))
	}
}
```

---

## Example 6: Next Number with Same Bit Count

```go
package main

import (
	"fmt"
	"math/bits"
)

func nextWithSameBits(n int) int {
	// Gosper's hack
	c := n & (-n)         // lowest set bit
	r := n + c            // turn off rightmost contiguous 1s, set next bit
	diff := ((r ^ n) >> 2) / c  // move rightmost 1s to least significant positions
	return r | diff
}

func prevWithSameBits(n int) int {
	// Mirror of next
	next := nextWithSameBits(^n)
	return ^next
}

func main() {
	fmt.Println("Next number with same number of set bits:")
	for _, n := range []int{6, 12, 7, 14, 19} {
		next := nextWithSameBits(n)
		fmt.Printf("  n=%d (%08b, %d bits) → next=%d (%08b, %d bits)\n",
			n, n, bits.OnesCount(uint(n)),
			next, next, bits.OnesCount(uint(next)))
	}
}
```

---

## Example 7: Compute Modulus Without % Operator

```go
package main

import "fmt"

// n mod d where d is power of 2
func modPow2(n, d int) int {
	return n & (d - 1)
}

// n mod 3 (bitwise approximation)
func mod3(n int) int {
	// Count alternating bit sums
	if n < 0 { n = -n }
	for n > 5 {
		sum := 0
		for n > 0 {
			sum += n & 3    // add pairs of bits
			n >>= 2
		}
		n = sum
	}
	if n >= 3 { n -= 3 }
	return n
}

func main() {
	fmt.Println("Mod by power of 2 (bitwise AND):")
	for _, d := range []int{2, 4, 8, 16} {
		for _, n := range []int{7, 15, 23, 100} {
			fmt.Printf("  %d %% %d = %d (bitwise: %d)\n", n, d, n%d, modPow2(n, d))
		}
		fmt.Println()
	}
}
```

---

## Example 8: Bit Manipulation for Strings

```go
package main

import "fmt"

// Check if string has all unique characters using bitmask
func hasAllUnique(s string) bool {
	if len(s) > 26 { return false }
	seen := 0
	for _, ch := range s {
		bit := 1 << (ch - 'a')
		if seen&bit != 0 { return false }
		seen |= bit
	}
	return true
}

// Count distinct characters using bitmask
func countDistinct(s string) int {
	mask := 0
	for _, ch := range s {
		if ch >= 'a' && ch <= 'z' {
			mask |= 1 << (ch - 'a')
		}
	}
	count := 0
	for mask > 0 { mask &= mask - 1; count++ }
	return count
}

// Check if two strings are anagrams
func areAnagramsBitmask(a, b string) bool {
	if len(a) != len(b) { return false }
	// Simple XOR only works for character presence, not count
	// Use frequency array instead, but bitmask shows unique chars match
	maskA, maskB := 0, 0
	for _, ch := range a { maskA |= 1 << (ch - 'a') }
	for _, ch := range b { maskB |= 1 << (ch - 'a') }
	return maskA == maskB // necessary but not sufficient
}

func main() {
	words := []string{"abcdef", "hello", "world", "unique", "aab"}
	for _, w := range words {
		fmt.Printf("  '%s' → unique:%v distinct:%d\n", w, hasAllUnique(w), countDistinct(w))
	}
}
```

---

## Example 9: Bit Tricks for Numbers

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Number Bit Tricks ===\n")

	// Round to nearest multiple of 8
	for _, n := range []int{1, 7, 8, 9, 15, 16, 17} {
		rounded := (n + 7) &^ 7
		fmt.Printf("  Round %2d up to mult of 8: %2d\n", n, rounded)
	}
	fmt.Println()

	// Check if n is between two powers of 2
	a, b := 4, 16 // both powers of 2
	for n := 0; n <= 20; n++ {
		between := n >= a && n <= b
		bitwise := (n &^ (b*2 - 1)) == 0 && n >= a
		_ = bitwise
		if between {
			fmt.Printf("  %2d is between %d and %d\n", n, a, b)
		}
	}
	fmt.Println()

	// Floor log2
	for _, n := range []int{1, 2, 3, 7, 8, 15, 16, 100} {
		log2 := 0
		temp := n
		for temp > 1 { temp >>= 1; log2++ }
		fmt.Printf("  floor(log2(%3d)) = %d\n", n, log2)
	}
}
```

---

## Example 10: Complete Bit Tricks Reference

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Complete Bitwise Tricks Reference ===")
	fmt.Println()

	tricks := []struct{ trick, formula, example string }{
		{"Toggle case", "ch ^ 32", "'A' ↔ 'a'"},
		{"Uppercase", "ch &^ 32", "'a' → 'A'"},
		{"Lowercase", "ch | 32", "'A' → 'a'"},
		{"Swap without temp", "a^=b; b^=a; a^=b", "a,b swapped"},
		{"Abs without branch", "(n^(n>>63))-(n>>63)", "|-5| = 5"},
		{"Min without branch", "b+((a-b)&((a-b)>>63))", "min(3,5)=3"},
		{"Max without branch", "a-((a-b)&((a-b)>>63))", "max(3,5)=5"},
		{"Average no overflow", "(a&b)+((a^b)>>1)", "avg(100,200)=150"},
		{"Is letter", "(ch|32)-'a' < 26", "Fast alpha check"},
		{"Round up to pow2", "fill + add 1", "5 → 8"},
		{"Isolate rightmost 1", "n & (-n)", "12 → 4"},
		{"Propagate rightmost 1", "n | (n-1)", "Fill right of LSB"},
		{"Turn off rightmost 1", "n & (n-1)", "12 → 8"},
		{"Next same bit count", "Gosper's hack", "Combinatorial"},
		{"Mod by pow2", "n & (d-1)", "n % 16 = n & 15"},
		{"Is same sign", "(a^b) >= 0", "Sign comparison"},
	}

	for _, t := range tricks {
		fmt.Printf("  %-25s %-30s → %s\n", t.trick, t.formula, t.example)
	}
}
```

---

## Key Takeaways

1. **Case toggle**: XOR with 32; upper = AND NOT 32; lower = OR 32
2. **Branchless**: min/max/abs/clamp using sign-bit mask (`n >> 63`)
3. **Average without overflow**: `(a & b) + ((a ^ b) >> 1)`
4. **Gosper's hack**: next number with same popcount for combinatorial iteration
5. **All unique chars**: bitmask of 26 bits replaces hash set for lowercase

> **Phase 20 Complete!** Next up: Phase 21 — Math & Geometry →
