# Phase 20: Bit Manipulation — Bitwise Operators

## Overview

Bitwise operators work directly on the binary representation of integers. They are fundamental to all bit manipulation techniques.

| Operator | Symbol | Description |
|----------|--------|-------------|
| AND | `&` | 1 only if both bits are 1 |
| OR | `\|` | 1 if at least one bit is 1 |
| XOR | `^` | 1 if bits differ |
| NOT | `^` (Go) | Flips all bits (unary) |
| Left Shift | `<<` | Shift bits left (multiply by 2) |
| Right Shift | `>>` | Shift bits right (divide by 2) |
| AND NOT | `&^` | Clear bits (Go-specific) |

---

## Example 1: All Bitwise Operators Demo

```go
package main

import "fmt"

func main() {
	a, b := 12, 10
	fmt.Printf("a = %d = %08b\n", a, a)
	fmt.Printf("b = %d = %08b\n\n", b, b)

	fmt.Printf("a & b  = %08b = %d  (AND)\n", a&b, a&b)
	fmt.Printf("a | b  = %08b = %d  (OR)\n", a|b, a|b)
	fmt.Printf("a ^ b  = %08b = %d  (XOR)\n", a^b, a^b)
	fmt.Printf("a &^ b = %08b = %d  (AND NOT)\n", a&^b, a&^b)
	fmt.Printf("a << 1 = %08b = %d  (Left Shift)\n", a<<1, a<<1)
	fmt.Printf("a >> 1 = %08b = %d  (Right Shift)\n", a>>1, a>>1)
	fmt.Printf("^a     = %d (bitwise NOT, depends on int size)\n", ^a)
}
```

---

## Example 2: Set, Clear, Toggle, Check a Bit

```go
package main

import "fmt"

func setBit(n, pos int) int    { return n | (1 << pos) }
func clearBit(n, pos int) int  { return n &^ (1 << pos) }
func toggleBit(n, pos int) int { return n ^ (1 << pos) }
func checkBit(n, pos int) bool { return (n>>pos)&1 == 1 }

func main() {
	n := 0b10101010 // 170
	fmt.Printf("n = %08b (%d)\n\n", n, n)

	for pos := 0; pos < 8; pos++ {
		fmt.Printf("Bit %d: %v\n", pos, checkBit(n, pos))
	}

	fmt.Printf("\nSet bit 2:    %08b → %08b\n", n, setBit(n, 2))
	fmt.Printf("Clear bit 3:  %08b → %08b\n", n, clearBit(n, 3))
	fmt.Printf("Toggle bit 5: %08b → %08b\n", n, toggleBit(n, 5))
}
```

---

## Example 3: Check Odd/Even Using AND

```go
package main

import "fmt"

func isEven(n int) bool { return n&1 == 0 }

func main() {
	for i := 0; i <= 10; i++ {
		if isEven(i) {
			fmt.Printf("%d is even  (binary: %04b, last bit: 0)\n", i, i)
		} else {
			fmt.Printf("%d is odd   (binary: %04b, last bit: 1)\n", i, i)
		}
	}
}
```

---

## Example 4: Swap Two Numbers Without Temp (XOR)

```go
package main

import "fmt"

func main() {
	a, b := 5, 9
	fmt.Printf("Before: a=%d, b=%d\n", a, b)

	a = a ^ b
	b = a ^ b // b = (a^b) ^ b = a
	a = a ^ b // a = (a^b) ^ a = b

	fmt.Printf("After:  a=%d, b=%d\n", a, b)

	// Also works for multiple pairs
	pairs := [][2]int{{1, 2}, {100, 200}, {-5, 5}}
	for _, p := range pairs {
		x, y := p[0], p[1]
		x, y = x^y, x^y^y // wait, need 3 steps
		x = p[0]; y = p[1]
		x ^= y; y ^= x; x ^= y
		fmt.Printf("  Swapped (%d,%d) → (%d,%d)\n", p[0], p[1], x, y)
	}
}
```

---

## Example 5: Extract Lowest Set Bit

```go
package main

import "fmt"

func lowestSetBit(n int) int { return n & (-n) }

func main() {
	nums := []int{12, 10, 18, 7, 16, 1}
	for _, n := range nums {
		lsb := lowestSetBit(n)
		fmt.Printf("n=%3d (%08b) → lowest set bit = %d (%08b)\n", n, n, lsb, lsb)
	}
	fmt.Println()
	fmt.Println("Trick: n & (-n) isolates the rightmost 1-bit")
	fmt.Println("  -n = ~n + 1 (two's complement)")
	fmt.Println("  All bits to right of rightmost 1 flip, so AND isolates it")
}
```

---

## Example 6: Turn Off Rightmost Set Bit

```go
package main

import "fmt"

func turnOffRightmost(n int) int { return n & (n - 1) }

func main() {
	nums := []int{12, 10, 7, 16, 15}
	for _, n := range nums {
		result := turnOffRightmost(n)
		fmt.Printf("n=%3d (%08b) → n&(n-1) = %3d (%08b)\n", n, n, result, result)
	}
	fmt.Println()
	fmt.Println("Key insight: n-1 flips all bits from rightmost 1 to end")
	fmt.Println("  So n & (n-1) clears the rightmost 1")
	fmt.Println("  If result == 0, n was a power of 2!")
}
```

---

## Example 7: AND of Range (LeetCode 201)

```go
package main

import "fmt"

// Bitwise AND of all numbers in [left, right]
func rangeBitwiseAnd(left, right int) int {
	// Find common prefix of left and right
	shift := 0
	for left != right {
		left >>= 1
		right >>= 1
		shift++
	}
	return left << shift
}

func main() {
	tests := [][2]int{{5, 7}, {0, 0}, {1, 2147483647}, {26, 30}}
	for _, t := range tests {
		result := rangeBitwiseAnd(t[0], t[1])
		fmt.Printf("AND of [%d, %d] = %d\n", t[0], t[1], result)
	}
	fmt.Println()
	fmt.Println("Why? AND with any 0 bit makes result 0")
	fmt.Println("  Only common prefix bits survive through the range")
}
```

---

## Example 8: Single Number (XOR All)

```go
package main

import "fmt"

// Every element appears twice except one
func singleNumber(nums []int) int {
	result := 0
	for _, n := range nums {
		result ^= n
	}
	return result
}

func main() {
	tests := [][]int{
		{2, 2, 1},
		{4, 1, 2, 1, 2},
		{1},
	}
	for _, nums := range tests {
		fmt.Printf("nums=%v → single=%d\n", nums, singleNumber(nums))
	}
	fmt.Println()
	fmt.Println("XOR properties: a^a=0, a^0=a, commutative & associative")
}
```

---

## Example 9: Determine Sign of Product

```go
package main

import "fmt"

// Return 1 if product is positive, -1 if negative, 0 if zero
func signOfProduct(nums []int) int {
	negCount := 0
	for _, n := range nums {
		if n == 0 { return 0 }
		if n < 0 { negCount++ }
	}
	if negCount&1 == 0 { return 1 }
	return -1
}

func main() {
	tests := [][]int{
		{-1, -2, -3, -4, 3, 2, 1},
		{1, 5, 0, 2, -3},
		{-1, 1, -1, 1, -1},
	}
	for _, nums := range tests {
		fmt.Printf("nums=%v → sign=%d\n", nums, signOfProduct(nums))
	}
}
```

---

## Example 10: Bitwise Operations Cheat Sheet

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bitwise Operations Cheat Sheet ===")
	fmt.Println()

	ops := []struct{ name, formula, use string }{
		{"Check if bit i is set", "(n >> i) & 1", "Bit testing"},
		{"Set bit i", "n | (1 << i)", "Turn on"},
		{"Clear bit i", "n &^ (1 << i)", "Turn off"},
		{"Toggle bit i", "n ^ (1 << i)", "Flip"},
		{"Get lowest set bit", "n & (-n)", "BIT/Fenwick tree"},
		{"Clear lowest set bit", "n & (n-1)", "Count bits, power of 2"},
		{"Check power of 2", "n > 0 && n&(n-1) == 0", "Validation"},
		{"Check odd/even", "n & 1", "Parity check"},
		{"Multiply by 2^k", "n << k", "Fast multiply"},
		{"Divide by 2^k", "n >> k", "Fast divide"},
		{"Set all bits 0..i", "(1 << (i+1)) - 1", "Mask creation"},
		{"Swap a,b", "a^=b; b^=a; a^=b", "No temp variable"},
		{"Absolute value", "(n ^ (n>>31)) - (n>>31)", "Branchless abs"},
	}

	for _, op := range ops {
		fmt.Printf("  %-25s %-25s → %s\n", op.name, op.formula, op.use)
	}
}
```

---

## Key Takeaways

1. **AND** masks/clears bits, **OR** sets bits, **XOR** toggles/finds differences
2. `n & (n-1)` clears rightmost set bit — used in power-of-2 check and bit counting
3. `n & (-n)` isolates rightmost set bit — used in Fenwick trees
4. XOR is self-inverse (`a^a=0`) — basis for single-number problems
5. Go-specific: `&^` (AND NOT) for clearing bits; `^` prefix for bitwise NOT

> **Next up:** Bit Masking →
