# Phase 20: Bit Manipulation — Two's Complement

## Overview

**Two's complement** is how computers represent signed integers. The most significant bit (MSB) is the sign bit: 0 = positive, 1 = negative. Negating a number: flip all bits and add 1.

```
 5 in 8-bit:  00000101
-5 in 8-bit:  11111011  (flip → 11111010, add 1 → 11111011)
```

| Property | Expression |
|----------|-----------|
| Range (n bits) | [-2^(n-1), 2^(n-1) - 1] |
| Negate | `^n + 1` or `-n` |
| Absolute value | `(n ^ (n>>63)) - (n>>63)` |
| Sign bit | `n >> 63` (for int64) |

---

## Example 1: Two's Complement Basics

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Two's Complement (8-bit illustration) ===")
	fmt.Println()

	for _, n := range []int8{0, 1, 2, 127, -1, -2, -128} {
		fmt.Printf("  %4d = %08b\n", n, uint8(n))
	}

	fmt.Println()
	fmt.Println("Key observations:")
	fmt.Println("  -1 = all 1s (11111111)")
	fmt.Println("  MIN = 10000000 (-128), MAX = 01111111 (127)")
	fmt.Println("  Range: [-128, 127] for 8-bit")
}
```

---

## Example 2: Negation and Absolute Value

```go
package main

import "fmt"

func negate(n int) int {
	return ^n + 1  // flip all bits, add 1
}

func absVal(n int) int {
	mask := n >> 63  // all 0s if positive, all 1s if negative
	return (n ^ mask) - mask
}

func main() {
	tests := []int{5, -5, 0, 42, -42, 1, -1}
	for _, n := range tests {
		fmt.Printf("n=%4d → negate=%4d (verify: %4d)  abs=%d\n",
			n, negate(n), -n, absVal(n))
	}
	fmt.Println()
	fmt.Println("Branchless abs: mask = n >> 63")
	fmt.Println("  Positive: mask=0, (n^0)-0 = n")
	fmt.Println("  Negative: mask=-1, (n^-1)-(-1) = ~n+1 = -n")
}
```

---

## Example 3: Overflow Detection

```go
package main

import (
	"fmt"
	"math"
)

func safeAdd(a, b int32) (int32, bool) {
	if b > 0 && a > math.MaxInt32-b { return 0, true }
	if b < 0 && a < math.MinInt32-b { return 0, true }
	return a + b, false
}

func safeMul(a, b int32) (int32, bool) {
	if a == 0 || b == 0 { return 0, false }
	result := a * b
	if result/a != b { return 0, true }
	return result, false
}

func main() {
	fmt.Printf("int32 range: [%d, %d]\n\n", math.MinInt32, math.MaxInt32)

	addTests := [][2]int32{{1000000000, 2000000000}, {-2000000000, -1000000000}, {100, 200}}
	for _, t := range addTests {
		result, overflow := safeAdd(t[0], t[1])
		if overflow {
			fmt.Printf("  %d + %d → OVERFLOW\n", t[0], t[1])
		} else {
			fmt.Printf("  %d + %d = %d\n", t[0], t[1], result)
		}
	}

	fmt.Println()
	mulTests := [][2]int32{{50000, 50000}, {100, 200}}
	for _, t := range mulTests {
		result, overflow := safeMul(t[0], t[1])
		if overflow {
			fmt.Printf("  %d × %d → OVERFLOW\n", t[0], t[1])
		} else {
			fmt.Printf("  %d × %d = %d\n", t[0], t[1], result)
		}
	}
}
```

---

## Example 4: Reverse Integer (LeetCode 7)

```go
package main

import (
	"fmt"
	"math"
)

func reverse(x int) int {
	result := 0
	for x != 0 {
		digit := x % 10
		x /= 10

		// Overflow check before multiply
		if result > math.MaxInt32/10 || (result == math.MaxInt32/10 && digit > 7) {
			return 0
		}
		if result < math.MinInt32/10 || (result == math.MinInt32/10 && digit < -8) {
			return 0
		}
		result = result*10 + digit
	}
	return result
}

func main() {
	tests := []int{123, -123, 120, 0, 1534236469}
	for _, n := range tests {
		fmt.Printf("reverse(%d) = %d\n", n, reverse(n))
	}
}
```

---

## Example 5: Sign Detection Without Branching

```go
package main

import "fmt"

func sign(n int) int {
	// Returns: 1 if positive, -1 if negative, 0 if zero
	return int(int64(n)>>63) | int(uint64(-int64(n))>>63)
}

func copySign(val, signSource int) int {
	// Apply sign of signSource to val
	mask := signSource >> 63
	return (val ^ mask) - mask
}

func max(a, b int) int {
	diff := a - b
	sign := diff >> 63 // 0 if a >= b, -1 if a < b
	return a - (diff & sign)
}

func min(a, b int) int {
	diff := a - b
	sign := diff >> 63
	return b + (diff & sign)
}

func main() {
	fmt.Println("Branchless sign:")
	for _, n := range []int{5, -3, 0, 100, -1} {
		fmt.Printf("  sign(%3d) = %2d\n", n, sign(n))
	}

	fmt.Println("\nBranchless max/min:")
	pairs := [][2]int{{3, 7}, {10, 2}, {5, 5}}
	for _, p := range pairs {
		fmt.Printf("  max(%d,%d)=%d  min(%d,%d)=%d\n",
			p[0], p[1], max(p[0], p[1]), p[0], p[1], min(p[0], p[1]))
	}
}
```

---

## Example 6: Atoi — String to Integer (LeetCode 8)

```go
package main

import (
	"fmt"
	"math"
)

func myAtoi(s string) int {
	i, n := 0, len(s)

	// Skip whitespace
	for i < n && s[i] == ' ' { i++ }
	if i == n { return 0 }

	// Sign
	sign := 1
	if s[i] == '-' { sign = -1; i++ } else if s[i] == '+' { i++ }

	// Digits
	result := 0
	for i < n && s[i] >= '0' && s[i] <= '9' {
		digit := int(s[i] - '0')

		// Overflow check (two's complement bounds)
		if result > math.MaxInt32/10 || (result == math.MaxInt32/10 && digit > 7) {
			if sign == 1 { return math.MaxInt32 }
			return math.MinInt32
		}
		result = result*10 + digit
		i++
	}
	return result * sign
}

func main() {
	tests := []string{"42", "   -42", "4193 with words", "", "+-12", "2147483648", "-2147483649"}
	for _, s := range tests {
		fmt.Printf("  '%s' → %d\n", s, myAtoi(s))
	}
}
```

---

## Example 7: Complement of Base 10 (LeetCode 1009)

```go
package main

import "fmt"

func bitwiseComplement(n int) int {
	if n == 0 { return 1 }
	// Find mask with all 1s covering n's bits
	mask := 1
	for mask <= n { mask <<= 1 }
	return (mask - 1) ^ n
}

func main() {
	tests := []int{5, 7, 10, 0, 1}
	for _, n := range tests {
		comp := bitwiseComplement(n)
		fmt.Printf("n=%d (%08b) → complement=%d (%08b)\n", n, n, comp, comp)
	}
}
```

---

## Example 8: Add Binary Strings (LeetCode 67)

```go
package main

import "fmt"

func addBinary(a, b string) string {
	i, j := len(a)-1, len(b)-1
	carry := 0
	result := []byte{}

	for i >= 0 || j >= 0 || carry > 0 {
		sum := carry
		if i >= 0 { sum += int(a[i] - '0'); i-- }
		if j >= 0 { sum += int(b[j] - '0'); j-- }
		result = append(result, byte('0'+sum%2))
		carry = sum / 2
	}

	// Reverse
	for l, r := 0, len(result)-1; l < r; l, r = l+1, r-1 {
		result[l], result[r] = result[r], result[l]
	}
	return string(result)
}

func main() {
	tests := [][2]string{{"11", "1"}, {"1010", "1011"}, {"0", "0"}, {"111", "111"}}
	for _, t := range tests {
		fmt.Printf("  %s + %s = %s\n", t[0], t[1], addBinary(t[0], t[1]))
	}
}
```

---

## Example 9: UTF-8 Validation (LeetCode 393)

```go
package main

import "fmt"

func validUtf8(data []int) bool {
	remaining := 0 // bytes remaining in current character

	for _, b := range data {
		b &= 0xFF // take lowest 8 bits

		if remaining > 0 {
			// Must be continuation byte: 10xxxxxx
			if b>>6 != 0b10 { return false }
			remaining--
		} else {
			if b>>7 == 0 {       // 0xxxxxxx — 1 byte
				remaining = 0
			} else if b>>5 == 0b110 {  // 110xxxxx — 2 bytes
				remaining = 1
			} else if b>>4 == 0b1110 { // 1110xxxx — 3 bytes
				remaining = 2
			} else if b>>3 == 0b11110 { // 11110xxx — 4 bytes
				remaining = 3
			} else {
				return false
			}
		}
	}
	return remaining == 0
}

func main() {
	tests := []struct{
		name string
		data []int
	}{
		{"ASCII 'A'", []int{65}},
		{"2-byte char", []int{0xC3, 0xA9}},  // é
		{"3-byte char", []int{0xE2, 0x82, 0xAC}},  // €
		{"Invalid continuation", []int{0xC3, 0x28}},
		{"Overlong", []int{0xF8}},
	}
	for _, t := range tests {
		fmt.Printf("  %-25s %v → valid: %v\n", t.name, t.data, validUtf8(t.data))
	}
}
```

---

## Example 10: Two's Complement Patterns

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println("=== Two's Complement Patterns ===")
	fmt.Println()

	patterns := []struct{ name, formula, note string }{
		{"Negate", "^n + 1 or -n", "Flip all bits, add 1"},
		{"Absolute value", "(n ^ (n>>63)) - (n>>63)", "Branchless"},
		{"Sign (0/-1)", "n >> 63", "All 0s or all 1s"},
		{"Min int32", "1 << 31", fmt.Sprintf("%d", math.MinInt32)},
		{"Max int32", "(1<<31) - 1", fmt.Sprintf("%d", math.MaxInt32)},
		{"Overflow check +", "a > MAX-b", "Before addition"},
		{"Overflow check ×", "result/a != b", "After multiplication"},
		{"Lowest set bit", "n & (-n)", "Uses two's complement of -n"},
		{"All 1s", "-1", "11111...1 in binary"},
		{"-n", "^n + 1", "Two's complement negation"},
	}

	for _, p := range patterns {
		fmt.Printf("  %-20s %-30s → %s\n", p.name, p.formula, p.note)
	}
}
```

---

## Key Takeaways

1. Two's complement: negate = flip all bits + 1; MSB is sign bit
2. Range for n bits: [-2^(n-1), 2^(n-1) - 1] — asymmetric!
3. `-n = ^n + 1`: this is why `n & (-n)` isolates the LSB
4. Overflow: check BEFORE the operation using bounds comparison
5. Branchless tricks: sign mask `n>>63` enables conditional operations without if

> **Next up:** Bitwise Tricks →
