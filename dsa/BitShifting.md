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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│       Basic Shift Operations: n = 12            │
├──────────────────────────────────────────────────┤
│  n = 00001100 (12)                               │
│                                                  │
│  LEFT SHIFT (multiply by 2^k):                   │
│  n << 0 : 00001100 =  12  (×1)                   │
│  n << 1 : 00011000 =  24  (×2)    ← bits move left│
│  n << 2 : 00110000 =  48  (×4)      0s fill right│
│  n << 3 : 01100000 =  96  (×8)                   │
│                                                  │
│  RIGHT SHIFT (divide by 2^k):                    │
│  n >> 0 : 00001100 =  12  (÷1)                   │
│  n >> 1 : 00000110 =   6  (÷2)   ← bits move right│
│  n >> 2 : 00000011 =   3  (÷4)     bits fall off  │
│  n >> 3 : 00000001 =   1  (÷8)                   │
│                                                  │
│  Visual:  ←←← left shift     right shift →→→      │
│         [0|0|0|0|1|1|0|0]                        │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Multiply via Shifts: 6 × 7 = 42                │
├──────────────────────────────────────────────────┤
│  b = 7 = 111 in binary                          │
│                                                  │
│  Iter  b(bin)  b&1  action       result  a       │
│  ────  ──────  ───  ───────────  ──────  ─────  │
│   1    111      1   result += 6     6     6     │
│                     a <<= 1               12    │
│                     b >>= 1  → 011               │
│   2    011      1   result += 12   18     12    │
│                     a <<= 1               24    │
│                     b >>= 1  → 001               │
│   3    001      1   result += 24   42     24    │
│                     b >>= 1  → 000  (done)       │
│                                                  │
│  6 × 7 = 6×(4+2+1) = 6×4 + 6×2 + 6×1 = 42      │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│         Reverse Bits (8-bit example)              │
├──────────────────────────────────────────────────┤
│  Input:  1 0 1 0 0 1 0 0                         │
│  Output: 0 0 1 0 0 1 0 1                         │
│         ←───────────────→                         │
│         reversed order                            │
│                                                   │
│  Algorithm (bit-by-bit):                          │
│  Step  n(input)  result     action                │
│   1   10100100  00000000   result<<=1, |=(n&1)=0  │
│   2   01010010  00000000   result<<=1, |=(n&1)=0  │
│   3   00101001  00000001   result<<=1, |=(n&1)=1  │
│   4   00010100  00000010   result<<=1, |=(n&1)=0  │
│   5   00001010  00000100   result<<=1, |=(n&1)=0  │
│   6   00000101  00001001   result<<=1, |=(n&1)=1  │
│   7   00000010  00010010   result<<=1, |=(n&1)=0  │
│   8   00000001  00100101   result<<=1, |=(n&1)=1  │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│    Circular Shift (Rotate) — 8-bit example        │
├──────────────────────────────────────────────────┤
│  Original: [1|0|1|1|0|0|1|1]                     │
│                                                  │
│  Rotate Left by 2:                               │
│  ╭──────────────────╮                               │
│  [1|0] ← [1|1|0|0|1|1] → wraps to right           │
│  Result:  [1|1|0|0|1|1|1|0]                      │
│                                                  │
│  Rotate Right by 2:                              │
│                ╭──────────────────╮              │
│  wraps to left ← [1|0|1|1|0|0] → [1|1]           │
│  Result:  [1|1|1|0|1|1|0|0]                      │
│                                                  │
│  Formula: rotL(n,k) = (n << k) | (n >> (w-k))    │
│          rotR(n,k) = (n >> k) | (n << (w-k))     │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│     Most Significant Bit Position                 │
├──────────────────────────────────────────────────┤
│  Position: 7  6  5  4  3  2  1  0                 │
│                                                   │
│  n=1:      0  0  0  0  0  0  0  1  → MSB pos = 0  │
│  n=4:      0  0  0  0  0  1  0  0  → MSB pos = 2  │
│  n=7:      0  0  0  0  0  1  1  1  → MSB pos = 2  │
│                            ↑ highest 1-bit         │
│  n=100:    0  1  1  0  0  1  0  0  → MSB pos = 6  │
│              ↑ highest 1-bit                       │
│                                                   │
│  Algorithm: shift right until n = 1               │
│  n=100: 1100100 → 110010 → 11001 → 1100           │
│         → 110 → 11 → 1  (6 shifts = position 6)   │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│   Next Power of Two: n = 5                       │
├──────────────────────────────────────────────────┤
│  n = 5 = 00000101     n-- → n = 00000100         │
│                                                  │
│  Step 1: n |= n >> 1                              │
│       00000100 | 00000010 = 00000110              │
│  Step 2: n |= n >> 2                              │
│       00000110 | 00000001 = 00000111              │
│  Step 3: n |= n >> 4                              │
│       00000111 | 00000000 = 00000111              │
│          (no more changes for 8-bit)              │
│                                                  │
│  n + 1 = 00000111 + 1 = 00001000 = 8 ✓           │
│                                                  │
│  Fills all bits below MSB with 1s, then +1        │
│  creates the next power of 2                      │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│  Divide Without / Operator: 10 ÷ 3 = 3           │
├──────────────────────────────────────────────────┤
│  a = 10 (dividend),  b = 3 (divisor)              │
│                                                   │
│  Find largest b << shift ≤ a:                     │
│    b << 0 =  3  ≤ 10  ✓                           │
│    b << 1 =  6  ≤ 10  ✓                           │
│    b << 2 = 12  > 10  ✗  → use shift=1            │
│                                                   │
│  result += 1 << 1 = 2,   a -= 3 << 1 = 6          │
│  a = 10 - 6 = 4                                   │
│                                                   │
│  Repeat: b << 0 = 3 ≤ 4  ✓                        │
│          b << 1 = 6 > 4  ✗  → use shift=0         │
│  result += 1 << 0 = 1,   a -= 3 = 1               │
│  a = 1 < 3  → done                                │
│                                                   │
│  result = 2 + 1 = 3                               │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│   Gray Code: n ^ (n >> 1)                         │
├──────────────────────────────────────────────────┤
│  Binary → Gray:   n XOR (n >> 1)                  │
│                                                   │
│  n=0: 0000 ^ 0000 = 0000  (Gray=0)               │
│  n=1: 0001 ^ 0000 = 0001  (Gray=1)               │
│  n=2: 0010 ^ 0001 = 0011  (Gray=3)               │
│  n=3: 0011 ^ 0001 = 0010  (Gray=2)               │
│  n=4: 0100 ^ 0010 = 0110  (Gray=6)               │
│  n=5: 0101 ^ 0010 = 0111  (Gray=7)               │
│  n=6: 0110 ^ 0011 = 0101  (Gray=5)               │
│  n=7: 0111 ^ 0011 = 0100  (Gray=4)               │
│                                                   │
│  Adjacent Gray codes differ by exactly 1 bit:     │
│  0000 → 0001 → 0011 → 0010 → 0110 → 0111 ...     │
│      ↑     ↑     ↑     ↑     ↑                      │
│    1 bit changed each step                        │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│      Extract/Insert Bit Fields                    │
├──────────────────────────────────────────────────┤
│  n = 11010110 (214)                               │
│  Bit:  7  6  5  4  3  2  1  0                     │
│       [1][1][0][1][0][1][1][0]                    │
│                    └─────┘ bits [2..4]              │
│                                                   │
│  Extract bits [2..4]:                             │
│    mask = (1 << 3) - 1 = 00000111                 │
│    (n >> 2) & mask = 00110101 & 111 = 101 = 5     │
│                                                   │
│  RGB Color Packing:                               │
│  ┌────────┬────────┬────────┐                      │
│  │ R: 255  │ G: 128  │ B:  64  │  = 0xFF8040       │
│  │ bit 23-16│ bit 15-8│ bit 7-0│                      │
│  └────────┴────────┴────────┘                      │
│  Pack: (R << 16) | (G << 8) | B                   │
└──────────────────────────────────────────────────┘
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

**Textual Figure:**
```
┌──────────────────────────────────────────────────┐
│     Shift Patterns Summary                        │
├──────────────────────────────────────────────────┤
│  n = 00001100 (12)                                │
│                                                   │
│  n << 3 = 01100000 =  96  (12 × 2³ = 12×8)      │
│  n >> 2 = 00000011 =   3  (12 ÷ 2² = 12÷4)      │
│  1 << 3 = 00001000 =   8  (mask for bit 3)        │
│  (1<<4)-1= 00001111 = 15  (mask bits 0-3)         │
│                                                   │
│  Multiply 12 × 10 using shifts:                   │
│    12 × 10 = 12×8 + 12×2                          │
│    = (12 << 3) + (12 << 1)                        │
│    = 96 + 24 = 120  ✓                              │
└──────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. Left shift = multiply by 2^k; right shift = floor divide by 2^k
2. Reverse bits: loop or divide-and-conquer swap (16→8→4→2→1)
3. Gray code: `n ^ (n >> 1)` — adjacent codes differ by 1 bit
4. Division without `/`: repeatedly subtract largest (divisor << k)
5. Bit field extract/insert: use masks with shift for packed data

> **Next up:** Power of Two Checks →
