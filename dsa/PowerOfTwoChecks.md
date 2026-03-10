# Phase 20: Bit Manipulation — Power of Two Checks

## Overview

A number is a **power of two** if it has exactly one set bit: `1, 2, 4, 8, 16, ...` This property extends to checking powers of other values and is used extensively in memory allocation, hash table sizing, and optimization.

| Check | Expression | Why |
|-------|-----------|-----|
| Power of 2 | `n > 0 && n&(n-1) == 0` | Single set bit cleared by n-1 |
| Power of 4 | Above + `n & 0x55555555 != 0` | Bit at even position |
| Power of 8 | Above + `n & 0x49249249 != 0` | Bit at position 0,3,6,9,... |

---

## Example 1: Power of Two — All Methods

```go
package main

import (
	"fmt"
	"math/bits"
)

func isPow2_and(n int) bool { return n > 0 && n&(n-1) == 0 }
func isPow2_popcount(n int) bool { return n > 0 && bits.OnesCount(uint(n)) == 1 }
func isPow2_lowbit(n int) bool { return n > 0 && n&(-n) == n }

func main() {
	fmt.Println("Three methods to check power of 2:")
	for _, n := range []int{0, 1, 2, 3, 4, 6, 8, 15, 16, 32, 100, 128, 256} {
		fmt.Printf("  n=%3d → AND:%v  POPCOUNT:%v  LOWBIT:%v\n",
			n, isPow2_and(n), isPow2_popcount(n), isPow2_lowbit(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Two: Three Methods                     │
├────────────────────────────────────────────────┤
│  Method 1: n & (n-1) == 0                        │
│    16 = 10000,  16-1 = 01111                     │
│    10000 & 01111 = 00000 → 0 ✓ power of 2        │
│                                                  │
│    6 = 0110,  6-1 = 0101                         │
│    0110 & 0101 = 0100 ≠ 0 ✗ not power of 2       │
│                                                  │
│  Method 2: OnesCount(n) == 1                     │
│    16=10000 has 1 set bit ✓                       │
│     6=0110  has 2 set bits ✗                      │
│                                                  │
│  Method 3: n & (-n) == n                         │
│    16 & (-16) = 10000 & 10000 = 10000 == 16 ✓    │
│     6 & (-6)  = 0110  & 1010  = 0010  ≠ 6  ✗    │
└────────────────────────────────────────────────┘
```

---

## Example 2: Power of Three

```go
package main

import "fmt"

func isPowerOfThree(n int) bool {
	if n <= 0 { return false }
	// Largest power of 3 in 32-bit int: 3^19 = 1162261467
	return 1162261467%n == 0
}

func isPowerOfThreeLoop(n int) bool {
	if n <= 0 { return false }
	for n%3 == 0 { n /= 3 }
	return n == 1
}

func main() {
	for _, n := range []int{0, 1, 3, 9, 12, 27, 81, 100, 243} {
		fmt.Printf("  n=%3d → math:%v  loop:%v\n", n, isPowerOfThree(n), isPowerOfThreeLoop(n))
	}
	fmt.Println()
	fmt.Println("Trick: 3^19 is max power of 3 for int32")
	fmt.Println("  If n divides 3^19, n must be a power of 3")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Three Check                            │
├────────────────────────────────────────────────┤
│  3^19 = 1162261467 (largest pow3 in int32)        │
│                                                  │
│  Trick: 1162261467 % n == 0                      │
│  3^19 = 3×3×3×...×3 (19 factors of 3)             │
│                                                  │
│  n=27:  1162261467 % 27 = 0  ✓ (27 = 3³)        │
│  n=12:  1162261467 % 12 ≠ 0  ✗ (12 = 4×3)        │
│                                                  │
│  Loop method: keep dividing by 3                 │
│  27 ÷ 3 = 9 ÷ 3 = 3 ÷ 3 = 1 ✓                    │
│  12 ÷ 3 = 4 (not divisible by 3) ✗               │
└────────────────────────────────────────────────┘
```

---

## Example 3: Power of Four (LeetCode 342)

```go
package main

import "fmt"

func isPowerOfFour(n int) bool {
	return n > 0 && n&(n-1) == 0 && n&0x55555555 != 0
}

func isPowerOfFourMod(n int) bool {
	return n > 0 && n&(n-1) == 0 && n%3 == 1
}

func main() {
	fmt.Println("Powers of 4:")
	for i := 1; i <= 1<<20; i <<= 1 {
		if isPowerOfFour(i) {
			fmt.Printf("  %6d = 4^? ✓  (binary: %022b)\n", i, i)
		}
	}
	fmt.Println()
	fmt.Println("0x55555555 = ...01010101 → even bit positions only")
	fmt.Println("Powers of 4 have their single bit at position 0, 2, 4, 6, ...")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Four Check                             │
├────────────────────────────────────────────────┤
│  Step 1: Must be power of 2 (n & (n-1) == 0)     │
│  Step 2: Bit must be at even position             │
│                                                  │
│  0x55555555 = ...01010101010101010101              │
│  Position:      ...10  8  6  4  2  0              │
│                                                  │
│  n=4:   00000100  (bit 2, even ✓)               │
│    & 01010101 = 00000100 ≠ 0  ✓ power of 4       │
│                                                  │
│  n=8:   00001000  (bit 3, odd)                   │
│    & 01010101 = 00000000 = 0  ✗ not power of 4   │
│                                                  │
│  Powers of 4: 1, 4, 16, 64, 256, 1024...         │
│  (bits at positions 0, 2, 4, 6, 8, 10...)        │
└────────────────────────────────────────────────┘
```

---

## Example 4: Nearest Power of Two

```go
package main

import (
	"fmt"
	"math/bits"
)

func nextPowerOf2(n int) int {
	if n <= 1 { return 1 }
	return 1 << bits.Len(uint(n-1))
}

func prevPowerOf2(n int) int {
	if n <= 0 { return 0 }
	return 1 << (bits.Len(uint(n)) - 1)
}

func nearestPowerOf2(n int) int {
	next := nextPowerOf2(n)
	prev := prevPowerOf2(n)
	if next-n < n-prev { return next }
	return prev
}

func main() {
	tests := []int{1, 3, 5, 7, 10, 15, 17, 100, 127, 129}
	for _, n := range tests {
		fmt.Printf("n=%3d → prev=%3d, next=%3d, nearest=%3d\n",
			n, prevPowerOf2(n), nextPowerOf2(n), nearestPowerOf2(n))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Nearest Power of Two                            │
├────────────────────────────────────────────────┤
│  Powers: 1  2  4  8  16  32  64  128  256 ...    │
│                                                  │
│  n=10:  ──8──────10──────16─                     │
│         prev    n     next                       │
│         nearest = 8  (10-8=2 < 16-10=6)          │
│                                                  │
│  nextPow2: 1 << bits.Len(n-1)                    │
│    n=10: bits.Len(9) = 4 → 1<<4 = 16             │
│                                                  │
│  prevPow2: 1 << (bits.Len(n) - 1)                │
│    n=10: bits.Len(10) = 4 → 1<<3 = 8             │
└────────────────────────────────────────────────┘
```

---

## Example 5: Align to Power of Two

```go
package main

import "fmt"

// Align n up to nearest multiple of alignment (must be power of 2)
func alignUp(n, align int) int {
	return (n + align - 1) &^ (align - 1)
}

// Align n down
func alignDown(n, align int) int {
	return n &^ (align - 1)
}

func main() {
	fmt.Println("Memory alignment examples:")
	for _, align := range []int{4, 8, 16, 64} {
		fmt.Printf("\n  Alignment = %d:\n", align)
		for _, n := range []int{1, 5, 13, 16, 33, 100} {
			fmt.Printf("    n=%3d → alignUp=%3d, alignDown=%3d\n",
				n, alignUp(n, align), alignDown(n, align))
		}
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Align to Power of Two (Memory Alignment)        │
├────────────────────────────────────────────────┤
│  alignUp(n, 8) = (n + 7) &^ 7                    │
│                                                  │
│  n = 13 = 00001101,  align = 8                   │
│    n + 7     = 20 = 00010100                      │
│    align - 1 =  7 = 00000111                      │
│    &^ (AND NOT):                                 │
│    00010100  &^  00000111  = 00010000 = 16  ✓    │
│    (clears lower 3 bits)                         │
│                                                  │
│  alignDown(n, 8) = n &^ 7                        │
│    00001101  &^  00000111  = 00001000 = 8   ✓    │
│                                                  │
│  0  8  16  24  32  ...   (multiples of 8)        │
│     ↑      ↑                                     │
│  down=8  up=16  for n=13                         │
└────────────────────────────────────────────────┘
```

---

## Example 6: Hash Table Size (Always Power of 2)

```go
package main

import (
	"fmt"
	"math/bits"
)

func optimalTableSize(minSize int) int {
	if minSize <= 1 { return 1 }
	return 1 << bits.Len(uint(minSize-1))
}

func fastModPow2(hash, tableSize int) int {
	return hash & (tableSize - 1) // works only when tableSize is power of 2
}

func main() {
	fmt.Println("Hash table sizing:")
	for _, n := range []int{5, 10, 50, 100, 1000} {
		size := optimalTableSize(n)
		fmt.Printf("  Need %4d slots → table size = %4d (%.0f%% utilization)\n",
			n, size, float64(n)/float64(size)*100)
	}

	fmt.Println("\nFast modulo (hash & (size-1)):")
	tableSize := 16
	for _, hash := range []int{5, 18, 33, 100} {
		fmt.Printf("  hash=%3d %% %d = %d (bitwise: %d)\n",
			hash, tableSize, hash%tableSize, fastModPow2(hash, tableSize))
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Hash Table Size (Power of 2 + Fast Mod)         │
├────────────────────────────────────────────────┤
│  Table size must be power of 2 for fast mod      │
│                                                  │
│  Need 50 slots → next pow2 = 64                  │
│  bits.Len(49) = 6 → 1 << 6 = 64                 │
│                                                  │
│  Fast modulo: hash & (size - 1)                  │
│  size = 16 = 10000,  size-1 = 01111              │
│                                                  │
│  hash = 33:  00100001                            │
│           &  00001111  (16-1)                     │
│              ────────                            │
│              00000001 = 1   (33 % 16 = 1 ✓)     │
│                                                  │
│  Same as modulo, but single AND instruction!     │
└────────────────────────────────────────────────┘
```

---

## Example 7: Count Trailing Zeros

```go
package main

import (
	"fmt"
	"math/bits"
)

// Trailing zeros = log2 of the lowest set bit
func trailingZeros(n int) int {
	if n == 0 { return -1 }
	count := 0
	for n&1 == 0 {
		count++
		n >>= 1
	}
	return count
}

func main() {
	tests := []int{1, 2, 4, 6, 8, 12, 16, 48, 64, 128, 192}
	for _, n := range tests {
		tz := trailingZeros(n)
		tzStd := bits.TrailingZeros(uint(n))
		isPow2 := n > 0 && n&(n-1) == 0
		fmt.Printf("n=%3d (%08b) → trailing zeros=%d (stdlib=%d) pow2=%v\n",
			n, n, tz, tzStd, isPow2)
	}
	fmt.Println()
	fmt.Println("Power of 2 → trailing zeros = log2(n)")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Trailing Zeros = log2(lowest set bit)            │
├────────────────────────────────────────────────┤
│  n=12:  00001100  trailing zeros = 2              │
│                ↑↑ two zeros before first 1        │
│                                                  │
│  n=48:  00110000  trailing zeros = 4              │
│              ↑↑↑↑ four zeros                      │
│                                                  │
│  n=16:  00010000  trailing zeros = 4              │
│         pow2 ✓ so log2(16) = 4                   │
│                                                  │
│  Algorithm: shift right until bit 0 is 1         │
│  12: 1100 → 110 → 11 (bit0=1, count=2)           │
└────────────────────────────────────────────────┘
```

---

## Example 8: Largest Power of 2 Dividing N (Factor of 2)

```go
package main

import "fmt"

func largestPow2Factor(n int) int {
	if n == 0 { return 0 }
	return n & (-n) // isolate lowest set bit
}

func main() {
	tests := []int{12, 18, 24, 36, 48, 64, 100, 128}
	for _, n := range tests {
		factor := largestPow2Factor(n)
		fmt.Printf("n=%3d → largest power of 2 dividing n = %d  (%d = %d × %d)\n",
			n, factor, n, factor, n/factor)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Largest Power of 2 Dividing N: n & (-n)         │
├────────────────────────────────────────────────┤
│  n=12: 00001100                                  │
│  -12:  11110100  (two's complement)               │
│        ────────  AND                            │
│        00000100  = 4                             │
│  12 = 4 × 3  (largest pow2 factor = 4)            │
│                                                  │
│  n=48: 00110000                                  │
│  -48:  11010000                                   │
│        00010000  = 16                             │
│  48 = 16 × 3  (largest pow2 factor = 16)          │
│                                                  │
│  n=64: 01000000 & 11000000 = 01000000 = 64       │
│  64 is itself a power of 2                       │
└────────────────────────────────────────────────┘
```

---

## Example 9: Power of Two Sum Decomposition

```go
package main

import "fmt"

// Decompose n into sum of distinct powers of 2
func decompose(n int) []int {
	powers := []int{}
	for bit := 0; n > 0; bit++ {
		if n&1 == 1 {
			powers = append(powers, 1<<bit)
		}
		n >>= 1
	}
	return powers
}

func main() {
	tests := []int{1, 7, 13, 42, 100, 255}
	for _, n := range tests {
		powers := decompose(n)
		fmt.Printf("n=%3d = ", n)
		for i, p := range powers {
			if i > 0 { fmt.Print(" + ") }
			fmt.Print(p)
		}
		fmt.Println()
	}
	fmt.Println()
	fmt.Println("Every integer = unique sum of distinct powers of 2")
	fmt.Println("  This IS the binary representation!")
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Two Decomposition: n = 42              │
├────────────────────────────────────────────────┤
│  42 = 00101010                                   │
│  Bit:  5  4  3  2  1  0                          │
│        1  0  1  0  1  0                          │
│        ↑     ↑     ↑                              │
│       2⁵   2³   2¹                              │
│       32    8    2                               │
│                                                  │
│  42 = 32 + 8 + 2                                 │
│     = 2⁵ + 2³ + 2¹                                │
│                                                  │
│  Every integer is a unique sum of distinct        │
│  powers of 2 — this IS binary representation!   │
└────────────────────────────────────────────────┘
```

---

## Example 10: Power of Two Patterns Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Power of Two Patterns ===")
	fmt.Println()

	checks := []struct{ name, test, note string }{
		{"Power of 2", "n>0 && n&(n-1)==0", "Exactly one set bit"},
		{"Power of 4", "pow2 && n&0x55555555!=0", "Set bit at even position"},
		{"Power of 3", "1162261467 % n == 0", "Divides max pow3"},
		{"Next pow2", "1 << bits.Len(n-1)", "Round up"},
		{"Prev pow2", "1 << (bits.Len(n)-1)", "Round down"},
		{"Align up", "(n+a-1) &^ (a-1)", "Memory alignment"},
		{"Fast mod pow2", "n & (m-1)", "Hash table index"},
		{"Trailing zeros", "bits.TrailingZeros(n)", "Log2 of pow2"},
		{"Largest pow2 factor", "n & (-n)", "Isolate LSB"},
		{"Decompose", "Binary representation", "Sum of pow2s"},
	}

	for _, c := range checks {
		fmt.Printf("  %-20s %-30s → %s\n", c.name, c.test, c.note)
	}
}
```

**Textual Figure:**
```
┌────────────────────────────────────────────────┐
│  Power of Two Patterns Summary                   │
├──────────────┬─────────────────┬────────────────┤
│  Check         │  Expression      │  Binary          │
├──────────────┼─────────────────┼────────────────┤
│  Power of 2    │  n&(n-1)==0      │  10000&01111=0  │
│  Power of 4    │  +n&0x55!=0      │  bit at even pos│
│  Next pow2     │  1<<bits.Len    │  round up        │
│  Prev pow2     │  1<<(Len-1)     │  round down      │
│  Align up      │  (n+a-1)&^(a-1) │  clear low bits  │
│  Fast mod      │  n&(size-1)     │  keep low bits   │
│  Trail zeros   │  TrailingZeros  │  count low 0s    │
│  Largest fact  │  n&(-n)         │  isolate LSB     │
└──────────────┴─────────────────┴────────────────┘
```

---

## Key Takeaways

1. `n & (n-1) == 0`: the classic power-of-2 check (one set bit)
2. Power of 4: power of 2 + bit at even position
3. Fast modulo: `hash & (size-1)` when size is power of 2 — used in hash tables
4. Alignment: `(n + align - 1) &^ (align - 1)` — critical in systems programming
5. LSB isolation `n & (-n)`: gives the largest power of 2 that divides n

> **Next up:** Two's Complement →
