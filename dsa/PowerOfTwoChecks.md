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

---

## Key Takeaways

1. `n & (n-1) == 0`: the classic power-of-2 check (one set bit)
2. Power of 4: power of 2 + bit at even position
3. Fast modulo: `hash & (size-1)` when size is power of 2 — used in hash tables
4. Alignment: `(n + align - 1) &^ (align - 1)` — critical in systems programming
5. LSB isolation `n & (-n)`: gives the largest power of 2 that divides n

> **Next up:** Two's Complement →
