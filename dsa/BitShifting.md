# Phase 20: Bit Manipulation — Bit Shifting

## Overview

**Bit shifting** moves all bits in a number left or right. Left shift multiplies by powers of 2; right shift divides. Go has two shift operators: `<<` (left) and `>>` (right, arithmetic for signed, logical for unsigned).

| Operation | Expression | Effect |
|-----------|-----------|--------|
| Left shift by k | `n << k` | n × 2^k |
| Right shift by k | `n >> k` | n / 2^k (floor) |
| Set bit i | `1 << i` | Create mask with bit i |
| Double | `n << 1` | n × 2 |
| Halve | `n >> 1` | n / 2 |

---

## Example 1: Basic Shift Operations

```go
package main

import "fmt"

func main() {
	n := 12 // 1100

	fmt.Printf("n = %d = %08b\n\n", n, n)

	for shift := 0; shift <= 4; shift++ {
		left := n << shift
		right := n >> shift
		fmt.Printf("n << %d = %3d (%08b)    n >> %d = %3d (%08b)\n",
			shift, left, left, shift, right, right)
	}

	fmt.Println()
	fmt.Println("Left shift by k = multiply by 2^k")
	fmt.Println("Right shift by k = divide by 2^k (floor)")
}
```

---

## Example 2: Multiply and Divide Using Shifts

```go
package main

import "fmt"

func multiplyByShift(a, b int) int {
	result := 0
	for b > 0 {
		if b&1 == 1 {
			result += a
		}
		a <<= 1
		b >>= 1
	}
	return result
}

func main() {
	tests := [][2]int{{6, 7}, {12, 5}, {100, 3}, {0, 999}}
	for _, t := range tests {
		product := multiplyByShift(t[0], t[1])
		fmt.Printf("%d × %d = %d (verify: %d)\n", t[0], t[1], product, t[0]*t[1])
	}
	fmt.Println()

	// Fast multiply/divide by known constants
	n := 100
	fmt.Printf("%d * 8  = %d (shift: %d)\n", n, n*8, n<<3)
	fmt.Printf("%d / 4  = %d (shift: %d)\n", n, n/4, n>>2)
	fmt.Printf("%d * 10 = %d (shift: %d)\n", n, n*10, (n<<3)+(n<<1))
}
```

---

## Example 3: Reverse Bits (LeetCode 190)

```go
package main

import "fmt"

func reverseBits(n uint32) uint32 {
	var result uint32
	for i := 0; i < 32; i++ {
		result <<= 1
		result |= n & 1
		n >>= 1
	}
	return result
}

// Divide and conquer approach
func reverseBitsDAC(n uint32) uint32 {
	n = (n >> 16) | (n << 16)
	n = ((n & 0xFF00FF00) >> 8) | ((n & 0x00FF00FF) << 8)
	n = ((n & 0xF0F0F0F0) >> 4) | ((n & 0x0F0F0F0F) << 4)
	n = ((n & 0xCCCCCCCC) >> 2) | ((n & 0x33333333) << 2)
	n = ((n & 0xAAAAAAAA) >> 1) | ((n & 0x55555555) << 1)
	return n
}

func main() {
	tests := []uint32{43261596, 4294967293, 1}
	for _, n := range tests {
		r := reverseBits(n)
		fmt.Printf("n=%032b\n  %032b (reversed = %d)\n\n", n, r, r)
	}
}
```

---

## Example 4: Circular Shift (Rotate)

```go
package main

import (
	"fmt"
	"math/bits"
)

func rotateLeft32(n uint32, k int) uint32 {
	k %= 32
	return (n << k) | (n >> (32 - k))
}

func rotateRight32(n uint32, k int) uint32 {
	k %= 32
	return (n >> k) | (n << (32 - k))
}

func main() {
	n := uint32(0b10110011_00000000_00000000_00000001)
	fmt.Printf("Original: %032b\n\n", n)

	for k := 1; k <= 8; k++ {
		l := rotateLeft32(n, k)
		r := rotateRight32(n, k)
		fmt.Printf("RotL(%d): %032b\n", k, l)
		fmt.Printf("RotR(%d): %032b\n\n", k, r)
	}

	// Using stdlib
	fmt.Printf("bits.RotateLeft32: %032b\n", bits.RotateLeft32(n, 4))
}
```

---

## Example 5: Find Position of Most Significant Bit

```go
package main

import (
	"fmt"
	"math/bits"
)

func msbPosition(n int) int {
	if n == 0 { return -1 }
	pos := 0
	for n > 1 {
		n >>= 1
		pos++
	}
	return pos
}

func msbPositionStdlib(n int) int {
	if n == 0 { return -1 }
	return bits.Len(uint(n)) - 1
}

func main() {
	tests := []int{1, 2, 4, 7, 8, 15, 16, 100, 1024}
	for _, n := range tests {
		fmt.Printf("n=%4d (%08b) → MSB at position %d (stdlib: %d)\n",
			n, n, msbPosition(n), msbPositionStdlib(n))
	}
}
```

---

## Example 6: Next Power of Two

```go
package main

import "fmt"

func nextPowerOfTwo(n int) int {
	if n <= 1 { return 1 }
	n--
	n |= n >> 1
	n |= n >> 2
	n |= n >> 4
	n |= n >> 8
	n |= n >> 16
	n |= n >> 32
	return n + 1
}

func main() {
	tests := []int{0, 1, 2, 3, 5, 7, 9, 15, 16, 17, 100}
	for _, n := range tests {
		fmt.Printf("Next power of 2 ≥ %3d = %d\n", n, nextPowerOfTwo(n))
	}
	fmt.Println()
	fmt.Println("Key: fill all bits below MSB with 1s, then add 1")
}
```

---

## Example 7: Divide Two Integers Without Division (LeetCode 29)

```go
package main

import (
	"fmt"
	"math"
)

func divide(dividend, divisor int) int {
	if dividend == math.MinInt32 && divisor == -1 {
		return math.MaxInt32 // overflow
	}

	negative := (dividend < 0) != (divisor < 0)
	a := abs(dividend)
	b := abs(divisor)
	result := 0

	for a >= b {
		shift := 0
		for a >= (b << (shift + 1)) {
			shift++
		}
		result += 1 << shift
		a -= b << shift
	}

	if negative { return -result }
	return result
}

func abs(n int) int {
	if n < 0 { return -n }
	return n
}

func main() {
	tests := [][2]int{{10, 3}, {7, -2}, {100, 7}, {0, 1}}
	for _, t := range tests {
		fmt.Printf("%d / %d = %d (actual: %d)\n",
			t[0], t[1], divide(t[0], t[1]), t[0]/t[1])
	}
}
```

---

## Example 8: Gray Code

```go
package main

import "fmt"

func grayCode(n int) []int {
	result := make([]int, 1<<n)
	for i := 0; i < (1 << n); i++ {
		result[i] = i ^ (i >> 1)
	}
	return result
}

func grayToBinary(gray int) int {
	binary := gray
	for gray > 0 {
		gray >>= 1
		binary ^= gray
	}
	return binary
}

func main() {
	n := 4
	codes := grayCode(n)
	fmt.Printf("Gray code (n=%d):\n", n)
	for i, g := range codes {
		fmt.Printf("  %2d → Gray: %04b → Binary: %04b (%d)\n",
			i, g, grayToBinary(g), grayToBinary(g))
	}
	fmt.Println()
	fmt.Println("Gray code: adjacent values differ by exactly 1 bit")
	fmt.Println("  Formula: gray = n ^ (n >> 1)")
}
```

---

## Example 9: Extract and Insert Bit Fields

```go
package main

import "fmt"

// Extract bits [lo..hi] from n
func extractBits(n, lo, hi int) int {
	mask := (1<<(hi-lo+1)) - 1
	return (n >> lo) & mask
}

// Insert val into bits [lo..hi] of n
func insertBits(n, val, lo, hi int) int {
	mask := (1<<(hi-lo+1)) - 1
	n &^= mask << lo          // clear target bits
	n |= (val & mask) << lo   // set new value
	return n
}

func main() {
	n := 0b11010110
	fmt.Printf("n = %08b (%d)\n\n", n, n)

	// Extract bits 2..4
	extracted := extractBits(n, 2, 4)
	fmt.Printf("Extract bits [2..4]: %03b (%d)\n", extracted, extracted)

	// Insert 0b101 at bits 2..4
	inserted := insertBits(n, 0b101, 2, 4)
	fmt.Printf("Insert 101 at [2..4]: %08b (%d)\n", inserted, inserted)

	// Color packing: RGB into single int
	r, g, b := 255, 128, 64
	color := (r << 16) | (g << 8) | b
	fmt.Printf("\nRGB(%d,%d,%d) packed = 0x%06X\n", r, g, b, color)
	fmt.Printf("  R = %d, G = %d, B = %d (extracted)\n",
		extractBits(color, 16, 23), extractBits(color, 8, 15), extractBits(color, 0, 7))
}
```

---

## Example 10: Shift Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Bit Shifting Patterns ===")
	fmt.Println()

	patterns := []struct{ name, formula, use string }{
		{"Multiply by 2^k", "n << k", "Fast multiply"},
		{"Divide by 2^k", "n >> k", "Fast floor division"},
		{"Create mask for bit i", "1 << i", "Single bit mask"},
		{"All 1s up to bit i", "(1 << (i+1)) - 1", "Range mask"},
		{"Keep lower k bits", "n & ((1<<k) - 1)", "Modulo 2^k"},
		{"Next power of 2", "Fill + add 1", "Buffer sizing"},
		{"Reverse bits", "Swap nibbles → pairs → singles", "Network protocols"},
		{"Gray code", "n ^ (n >> 1)", "Error-resilient encoding"},
		{"Rotate left", "(n<<k) | (n>>(w-k))", "Cryptography"},
		{"MSB position", "bits.Len(n) - 1", "Log2 floor"},
		{"Multiply by 10", "(n<<3) + (n<<1)", "Avoid multiply instruction"},
		{"Sign extension", "Automatic for signed >>", "Signed arithmetic"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-22s %-30s → %s\n", p.name, p.formula, p.use)
	}
}
```

---

## Key Takeaways

1. Left shift = multiply by 2^k; right shift = floor divide by 2^k
2. Reverse bits: loop or divide-and-conquer swap (16→8→4→2→1)
3. Gray code: `n ^ (n >> 1)` — adjacent codes differ by 1 bit
4. Division without `/`: repeatedly subtract largest (divisor << k)
5. Bit field extract/insert: use masks with shift for packed data

> **Next up:** Power of Two Checks →
