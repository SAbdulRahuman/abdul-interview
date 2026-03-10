# Phase 24: Divide & Conquer — Karatsuba Multiplication

## Overview

Standard multiplication of two n-digit numbers takes **O(n²)**. Karatsuba's algorithm reduces this to **O(n^1.585)** by computing **3 multiplications** instead of 4 in each recursive step.

### Key Idea
For x = a·Bᵐ + b and y = c·Bᵐ + d (B = base, m = n/2):
- Naive: x·y = ac·B²ᵐ + (ad+bc)·Bᵐ + bd → **4 multiplications**
- Karatsuba: compute ac, bd, and (a+b)(c+d) → ad+bc = (a+b)(c+d) - ac - bd → **3 multiplications**

| Method | Multiplications per level | Complexity |
|--------|--------------------------|------------|
| Grade school | 4 | O(n²) |
| Karatsuba | 3 | O(n^log₂3) ≈ O(n^1.585) |

---

## Example 1: Grade School Multiplication

```go
package main

import "fmt"

// Simulate grade-school multiplication on digit arrays
// Least significant digit first

func gradeSchool(a, b []int) []int {
	n, m := len(a), len(b)
	result := make([]int, n+m)

	for i := 0; i < n; i++ {
		carry := 0
		for j := 0; j < m; j++ {
			product := a[i]*b[j] + result[i+j] + carry
			result[i+j] = product % 10
			carry = product / 10
		}
		result[i+m] += carry
	}

	// Remove trailing zeros
	for len(result) > 1 && result[len(result)-1] == 0 {
		result = result[:len(result)-1]
	}
	return result
}

func toDigits(n int) []int {
	if n == 0 { return []int{0} }
	var digits []int
	for n > 0 { digits = append(digits, n%10); n /= 10 }
	return digits
}

func fromDigits(d []int) int {
	result, place := 0, 1
	for _, digit := range d { result += digit * place; place *= 10 }
	return result
}

func main() {
	a, b := 1234, 5678
	da, db := toDigits(a), toDigits(b)
	result := gradeSchool(da, db)

	fmt.Printf("%d × %d = %d\n", a, b, fromDigits(result))
	fmt.Printf("Expected: %d\n", a*b)
	fmt.Println("\nGrade school: O(n²) digit multiplications")
}
```

**Textual Figure:**

```
  Grade School Multiplication: 1234 × 5678

  Each digit-pair multiplied (4 × 4 = 16 digit muls):
  ┌────────────────────────────────────┐
  │          1  2  3  4                │
  │        × 5  6  7  8                │
  │      ────────────                  │
  │  8×:     8 16 24 32   (1234×8)     │
  │  7×:    7 14 21 28    (1234×7×10)  │
  │  6×:   6 12 18 24     (1234×6×100) │
  │  5×:  5 10 15 20      (1234×5×1000)│
  │      ────────────                  │
  │      7  0  0  6  6  5  2          │
  └────────────────────────────────────┘

  n digits × n digits = n² single-digit muls
  Complexity: O(n²)
```

---

## Example 2: Karatsuba for Integers

```go
package main

import (
	"fmt"
	"math"
)

func karatsuba(x, y int64) int64 {
	if x < 10 || y < 10 { return x * y }

	// Determine number of digits
	n := int(math.Max(float64(numDigits(x)), float64(numDigits(y))))
	m := n / 2

	// Split: x = a * 10^m + b, y = c * 10^m + d
	pow := int64(math.Pow(10, float64(m)))
	a, b := x/pow, x%pow
	c, d := y/pow, y%pow

	// 3 multiplications instead of 4
	ac := karatsuba(a, c)
	bd := karatsuba(b, d)
	abcd := karatsuba(a+b, c+d) // (a+b)(c+d) = ac + ad + bc + bd

	// ad + bc = (a+b)(c+d) - ac - bd
	adbc := abcd - ac - bd

	return ac*pow*pow + adbc*pow + bd
}

func numDigits(n int64) int {
	if n == 0 { return 1 }
	count := 0
	for n > 0 { count++; n /= 10 }
	return count
}

func main() {
	tests := [][2]int64{{1234, 5678}, {12, 34}, {999, 111}, {123456, 789012}}

	for _, t := range tests {
		result := karatsuba(t[0], t[1])
		expected := t[0] * t[1]
		fmt.Printf("%d × %d = %d (expected %d) ✓=%v\n",
			t[0], t[1], result, expected, result == expected)
	}

	fmt.Println("\nKaratsuba: O(n^1.585) — 3 muls per level")
}
```

**Textual Figure:**

```
  Karatsuba: 1234 × 5678

  Split: x = 12·10² + 34,  y = 56·10² + 78
         a=12  b=34       c=56  d=78

  3 sub-multiplications (instead of 4):
  ┌────────────────────────────────────────────┐
  │  ac   = 12 × 56         = 672            │
  │  bd   = 34 × 78         = 2652           │
  │  (a+b)(c+d) = 46 × 134 = 6164           │
  │  ad+bc = 6164 - 672 - 2652 = 2840       │
  └────────────────────────────────────────────┘

  Combine: ac·10⁴ + (ad+bc)·10² + bd
           672·10000 + 2840·100 + 2652
         = 6720000 + 284000 + 2652 = 7006652 ✓

  Recursion tree (a=3 branches per level):
           n
        /  |  \
      n/2 n/2 n/2      3 sub-multiplications
     /|\ /|\ /|\
     .........          9 sub-multiplications

  T(n) = 3T(n/2) + O(n) → O(n^log₂ 3) ≈ O(n^1.585)
```

---

## Example 3: Big Number Karatsuba (String-Based)

```go
package main

import (
	"fmt"
	"strings"
)

// Big number represented as string

func addBig(a, b string) string {
	// Ensure a is longer
	if len(a) < len(b) { a, b = b, a }
	
	// Pad b
	b = strings.Repeat("0", len(a)-len(b)) + b
	
	result := make([]byte, len(a)+1)
	carry := 0
	for i := len(a) - 1; i >= 0; i-- {
		sum := int(a[i]-'0') + int(b[i]-'0') + carry
		result[i+1] = byte(sum%10) + '0'
		carry = sum / 10
	}
	result[0] = byte(carry) + '0'
	
	s := string(result)
	s = strings.TrimLeft(s, "0")
	if s == "" { return "0" }
	return s
}

func subBig(a, b string) string {
	// Assume a >= b (no negative results here)
	b = strings.Repeat("0", len(a)-len(b)) + b
	
	result := make([]byte, len(a))
	borrow := 0
	for i := len(a) - 1; i >= 0; i-- {
		diff := int(a[i]-'0') - int(b[i]-'0') - borrow
		if diff < 0 {
			diff += 10
			borrow = 1
		} else {
			borrow = 0
		}
		result[i] = byte(diff) + '0'
	}
	s := strings.TrimLeft(string(result), "0")
	if s == "" { return "0" }
	return s
}

func mulSingle(a string, d byte) string {
	carry := 0
	result := make([]byte, len(a)+1)
	for i := len(a) - 1; i >= 0; i-- {
		prod := int(a[i]-'0')*int(d-'0') + carry
		result[i+1] = byte(prod%10) + '0'
		carry = prod / 10
	}
	result[0] = byte(carry) + '0'
	s := strings.TrimLeft(string(result), "0")
	if s == "" { return "0" }
	return s
}

func karatsubaBig(x, y string) string {
	x = strings.TrimLeft(x, "0")
	y = strings.TrimLeft(y, "0")
	if x == "" || y == "" { return "0" }
	
	if len(x) == 1 || len(y) == 1 {
		if len(x) == 1 {
			return mulSingle(y, x[0])
		}
		return mulSingle(x, y[0])
	}
	
	// Make same length
	n := len(x)
	if len(y) > n { n = len(y) }
	x = strings.Repeat("0", n-len(x)) + x
	y = strings.Repeat("0", n-len(y)) + y
	
	m := n / 2
	
	a, b := x[:len(x)-m], x[len(x)-m:]
	c, d := y[:len(y)-m], y[len(y)-m:]
	
	ac := karatsubaBig(a, c)
	bd := karatsubaBig(b, d)
	ab := addBig(a, b)
	cd := addBig(c, d)
	abcd := karatsubaBig(ab, cd)
	
	adbc := subBig(subBig(abcd, ac), bd)
	
	// ac * 10^(2m) + adbc * 10^m + bd
	acShifted := ac + strings.Repeat("0", 2*m)
	adbcShifted := adbc + strings.Repeat("0", m)
	
	return addBig(addBig(acShifted, adbcShifted), bd)
}

func main() {
	tests := [][2]string{
		{"1234", "5678"},
		{"999999999999", "888888888888"},
		{"123456789012345678", "987654321098765432"},
	}
	
	for _, t := range tests {
		result := karatsubaBig(t[0], t[1])
		fmt.Printf("%s × %s = %s\n", t[0], t[1], result)
	}
}
```

**Textual Figure:**

```
  Big Number Karatsuba (String-Based)

  For numbers too large for int64:
  999999999999 × 888888888888

  Recursion trace (simplified):
  ┌──────────────────────────────────────────────┐
  │  x = "999999999999"   (12 digits)           │
  │  y = "888888888888"   (12 digits)           │
  │                                             │
  │  Split at m=6:                              │
  │    a = "999999"  b = "999999"               │
  │    c = "888888"  d = "888888"               │
  │                                             │
  │  3 recursive calls on 6-digit numbers:      │
  │    ac = karatsuba("999999","888888")         │
  │    bd = karatsuba("999999","888888")         │
  │    (a+b)(c+d) = karatsuba("1999998",...)    │
  │                                             │
  │  All string arithmetic: addBig, subBig       │
  │  Shift = concatenate "0" strings             │
  └──────────────────────────────────────────────┘

  Recursion depth = log₂(12) ≈ 4 levels
  Handles arbitrarily large numbers!
```

---

## Example 4: Karatsuba for Polynomials

```go
package main

import "fmt"

// Multiply two polynomials using Karatsuba-style D&C
// p(x) = a0 + a1*x + a2*x² + ...

func polyMul(a, b []int) []int {
	n := len(a)
	m := len(b)
	if n == 0 || m == 0 { return nil }
	
	result := make([]int, n+m-1)
	if n == 1 || m == 1 {
		for i := 0; i < n; i++ {
			for j := 0; j < m; j++ {
				result[i+j] += a[i] * b[j]
			}
		}
		return result
	}
	
	// Make same length (pad with zeros)
	maxLen := n
	if m > maxLen { maxLen = m }
	if maxLen%2 != 0 { maxLen++ }
	for len(a) < maxLen { a = append(a, 0) }
	for len(b) < maxLen { b = append(b, 0) }
	
	half := maxLen / 2
	
	// a = a_low + a_high * x^half
	aLow, aHigh := a[:half], a[half:]
	bLow, bHigh := b[:half], b[half:]
	
	// 3 multiplications
	p1 := polyMul(aLow, bLow)            // aL * bL
	p2 := polyMul(aHigh, bHigh)          // aH * bH
	aSum := polyAdd(aLow, aHigh)
	bSum := polyAdd(bLow, bHigh)
	p3 := polyMul(aSum, bSum)            // (aL+aH)(bL+bH)
	
	// middle = p3 - p1 - p2
	middle := polySub(polySub(p3, p1), p2)
	
	// result = p1 + middle * x^half + p2 * x^(2*half)
	result = make([]int, n+m-1)
	for i, v := range p1 { result[i] += v }
	for i, v := range middle { result[i+half] += v }
	for i, v := range p2 { result[i+2*half] += v }
	
	return result
}

func polyAdd(a, b []int) []int {
	n := len(a)
	if len(b) > n { n = len(b) }
	result := make([]int, n)
	for i := 0; i < len(a); i++ { result[i] += a[i] }
	for i := 0; i < len(b); i++ { result[i] += b[i] }
	return result
}

func polySub(a, b []int) []int {
	n := len(a)
	if len(b) > n { n = len(b) }
	result := make([]int, n)
	for i := 0; i < len(a); i++ { result[i] += a[i] }
	for i := 0; i < len(b); i++ { result[i] -= b[i] }
	return result
}

func printPoly(p []int) string {
	s := ""
	for i, c := range p {
		if c == 0 { continue }
		if s != "" && c > 0 { s += " + " }
		if i == 0 { s += fmt.Sprintf("%d", c) } else {
			s += fmt.Sprintf("%dx^%d", c, i)
		}
	}
	if s == "" { return "0" }
	return s
}

func main() {
	// (1 + 2x + 3x²) * (4 + 5x)
	a := []int{1, 2, 3}  // 1 + 2x + 3x²
	b := []int{4, 5}     // 4 + 5x

	result := polyMul(a, b)
	fmt.Printf("(%s) × (%s) = %s\n", printPoly(a), printPoly(b), printPoly(result))
	// Expected: 4 + 13x + 22x² + 15x³
}
```

**Textual Figure:**

```
  Polynomial Karatsuba: (1+2x+3x²) × (4+5x)

  Split polynomials at half-degree:
  ┌────────────────────────────────────────────┐
  │  a = aLow + aHigh·x^half                   │
  │  aLow = [1, 2]     aHigh = [3, 0]          │
  │  bLow = [4, 5]     bHigh = [0, 0]          │
  └────────────────────────────────────────────┘

  3 sub-multiplications:
    p1 = aLow × bLow    = [4, 13, 10]
    p2 = aHigh × bHigh  = [0]
    p3 = (aLow+aHigh) × (bLow+bHigh)
       = [4, 7] × [4, 5] = [16, 48, 35]

  middle = p3 - p1 - p2 = [12, 35, 25]

  Combine: p1 + middle·x^half + p2·x^(2·half)
  Result:  4 + 13x + 22x² + 15x³ ✓

  Same trick: 3 poly-muls instead of 4
```

---

## Example 5: Visualizing Karatsuba's Trick

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Karatsuba's Key Insight ===\n")

	fmt.Println("Multiply X = 12 × Y = 34:")
	fmt.Println()
	fmt.Println("Split: X = 1·10 + 2, Y = 3·10 + 4")
	fmt.Println("       a=1, b=2, c=3, d=4\n")

	a, b, c, d := 1, 2, 3, 4

	fmt.Println("--- Naive: 4 multiplications ---")
	ac := a * c
	ad := a * d
	bc := b * c
	bd := b * d
	naive := ac*100 + (ad+bc)*10 + bd
	fmt.Printf("ac=%d, ad=%d, bc=%d, bd=%d\n", ac, ad, bc, bd)
	fmt.Printf("Result = %d·100 + %d·10 + %d = %d\n\n", ac, ad+bc, bd, naive)

	fmt.Println("--- Karatsuba: 3 multiplications ---")
	p1 := a * c        // ac = 3
	p2 := b * d        // bd = 8
	p3 := (a + b) * (c + d) // (1+2)(3+4) = 21
	mid := p3 - p1 - p2     // ad + bc = 21 - 3 - 8 = 10
	kara := p1*100 + mid*10 + p2
	fmt.Printf("p1 = ac = %d\n", p1)
	fmt.Printf("p2 = bd = %d\n", p2)
	fmt.Printf("p3 = (a+b)(c+d) = %d\n", p3)
	fmt.Printf("ad+bc = p3 - p1 - p2 = %d\n", mid)
	fmt.Printf("Result = %d·100 + %d·10 + %d = %d\n", p1, mid, p2, kara)

	fmt.Printf("\n12 × 34 = %d ✓\n", 12*34)
	fmt.Println("\nSaved 1 multiplication (25% fewer)!")
	fmt.Println("Recursively: O(n^1.585) vs O(n²)")
}
```

**Textual Figure:**

```
  Karatsuba's Trick: 12 × 34

  Split: x=12 → a=1, b=2    y=34 → c=3, d=4

  Naive (4 multiplications):        Karatsuba (3 multiplications):
  ┌──────────────────────┐      ┌───────────────────────┐
  │ ac = 1×3 = 3          │      │ p1 = ac = 1×3 = 3       │
  │ ad = 1×4 = 4          │      │ p2 = bd = 2×4 = 8       │
  │ bc = 2×3 = 6          │      │ p3 = (1+2)(3+4) = 21   │
  │ bd = 2×4 = 8          │      │ ad+bc = 21-3-8 = 10    │
  └──────────────────────┘      └───────────────────────┘
  4 muls                         3 muls  ★

  Both yield: 3×100 + 10×10 + 8 = 300+100+8 = 408
  Verify: 12 × 34 = 408 ✓

  ┌────────────────────────────────────────────┐
  │  Saved: 1 mul per level = 25% fewer        │
  │  Recursive savings:                        │
  │    Naive: 4^k muls at depth k → O(n²)      │
  │    Kara:  3^k muls at depth k → O(n^1.585) │
  └────────────────────────────────────────────┘
```

---

## Example 6: Toom-Cook 3-Way (Generalization)

```go
package main

import "fmt"

// Toom-Cook generalizes Karatsuba to k-way splits
// Karatsuba = Toom-2
// Toom-3: split into 3 parts, 5 multiplications instead of 9

func main() {
	fmt.Println("=== Toom-Cook Generalization ===\n")

	methods := []struct {
		name   string
		parts  int
		muls   int
		exp    string
	}{
		{"Karatsuba (Toom-2)", 2, 3, "log₂3 ≈ 1.585"},
		{"Toom-3", 3, 5, "log₃5 ≈ 1.465"},
		{"Toom-4", 4, 7, "log₄7 ≈ 1.404"},
		{"Toom-k", 0, 0, "→ approaches O(n log n)"},
		{"FFT (limit)", 0, 0, "O(n log n log log n)"},
	}

	fmt.Printf("%-22s %s → %s → %s\n", "Method", "Parts", "Muls", "Exponent")
	fmt.Println("----------------------------------------------")
	for _, m := range methods {
		if m.parts == 0 {
			fmt.Printf("%-22s %4s   %4s   %s\n", m.name, "-", "-", m.exp)
		} else {
			naive := m.parts * m.parts
			fmt.Printf("%-22s %4d   %d/%d   %s\n",
				m.name, m.parts, m.muls, naive, m.exp)
		}
	}

	fmt.Println("\nWhy not always use higher Toom?")
	fmt.Println("• Higher Toom requires more additions/subtractions")
	fmt.Println("• Overhead grows: only worth it for very large numbers")
	fmt.Println("• In practice: grade school → Karatsuba → Toom-3 → FFT")
}
```

**Textual Figure:**

```
  Toom-Cook Generalization (Karatsuba = Toom-2)

  Method           Parts  Muls  Naive muls  Exponent
  ────────────────────────────────────────────────
  Karatsuba(Toom-2)  2      3       4        log₂ 3 ≈ 1.585
  Toom-3             3      5       9        log₃ 5 ≈ 1.465
  Toom-4             4      7      16        log₄ 7 ≈ 1.404
  Toom-k             k    2k-1     k²       → approaches 1
  FFT (limit)        —      —       —        O(n log n log log n)

  Recursion tree comparison:
  Karatsuba:  3 branches/level   Toom-3:  5 branches/level
       n                              n
     / | \                        / | | | \
   n/2 n/2 n/2                  n/3 .... n/3
  O(n^1.585)                   O(n^1.465)

  Practical crossover thresholds:
  ┌─────────────┬────────────────────────┐
  │ < 20 digits │ Grade school O(n²)       │
  │ 20-1000     │ Karatsuba O(n^1.585)     │
  │ 1000-10000  │ Toom-3 O(n^1.465)        │
  │ > 10000     │ FFT O(n log n log log n) │
  └─────────────┴────────────────────────┘
```

---

## Example 7: Karatsuba with Counting Operations

```go
package main

import "fmt"

var mulCount, addCount int

func karatsubaCount(x, y int64, depth int) int64 {
	if x < 10 || y < 10 {
		mulCount++
		return x * y
	}

	n := numDigits(x)
	m := n / 2
	pow := int64(1)
	for i := 0; i < m; i++ { pow *= 10 }

	a, b := x/pow, x%pow
	c, d := y/pow, y%pow

	ac := karatsubaCount(a, c, depth+1)
	bd := karatsubaCount(b, d, depth+1)
	
	addCount += 2 // a+b, c+d
	abcd := karatsubaCount(a+b, c+d, depth+1)
	
	addCount += 2 // abcd - ac - bd
	adbc := abcd - ac - bd

	return ac*pow*pow + adbc*pow + bd
}

func numDigits(n int64) int {
	if n == 0 { return 1 }
	count := 0; for n > 0 { count++; n /= 10 }; return count
}

func main() {
	tests := [][2]int64{{12, 34}, {1234, 5678}, {12345678, 87654321}}

	for _, t := range tests {
		mulCount, addCount = 0, 0
		result := karatsubaCount(t[0], t[1], 0)

		naiveMuls := numDigits(t[0]) * numDigits(t[1])
		fmt.Printf("%d × %d = %d\n", t[0], t[1], result)
		fmt.Printf("  Karatsuba: %d muls, %d adds\n", mulCount, addCount)
		fmt.Printf("  Naive would need: %d muls\n\n", naiveMuls)
	}
}
```

**Textual Figure:**

```
  Counting Operations: Karatsuba vs Naive

  12 × 34 (2-digit):
  ┌───────────────────────────────────────────┐
  │  Karatsuba: 3 muls, 4 adds               │
  │  Naive:     4 muls (2×2)                  │
  └───────────────────────────────────────────┘

  1234 × 5678 (4-digit):
  ┌───────────────────────────────────────────┐
  │  Karatsuba: 9 muls, 12 adds               │
  │  Naive:     16 muls (4×4)                  │
  └───────────────────────────────────────────┘

  Recursion tree for 4-digit multiply:
          1234 × 5678         ← 3 muls of 2-digit
         /     |     \
    12×56  34×78  46×134    ← each: 3 muls of 1-digit
   /|\     /|\     /|\
  1d 1d..1d 1d..1d 1d       = 9 single-digit muls

  Growth: 3^(log₂ n) = n^(log₂ 3) ≈ n^1.585
```

---

## Example 8: Application — Large Number Squaring

```go
package main

import (
	"fmt"
	"math"
)

// Squaring is slightly faster than general multiplication
// x² = (a·B^m + b)² = a²·B^2m + 2ab·B^m + b²
// Only 3 products: a², b², ab (but ab is general so still 3)
// With Karatsuba trick: compute a², b², (a+b)²
// 2ab = (a+b)² - a² - b²  → still 3 squarings!

func karatsubaSquare(x int64) int64 {
	if x < 10 { return x * x }

	n := numDigits(x)
	m := n / 2
	pow := int64(math.Pow(10, float64(m)))

	a, b := x/pow, x%pow

	aSq := karatsubaSquare(a)
	bSq := karatsubaSquare(b)
	abSq := karatsubaSquare(a + b)

	twoAB := abSq - aSq - bSq

	return aSq*pow*pow + twoAB*pow + bSq
}

func numDigits(n int64) int {
	if n == 0 { return 1 }
	count := 0; for n > 0 { count++; n /= 10 }; return count
}

func main() {
	tests := []int64{12, 99, 1234, 9999, 12345}

	for _, x := range tests {
		result := karatsubaSquare(x)
		expected := x * x
		fmt.Printf("%d² = %d (expected %d) ✓=%v\n", x, result, expected, result == expected)
	}

	fmt.Println("\nSquaring: same recursive structure")
	fmt.Println("Saves one multiplication per level for general case")
}
```

**Textual Figure:**

```
  Karatsuba Squaring: x² = (a·B^m + b)²

  Expansion: a²·B^(2m) + 2ab·B^m + b²

  ┌────────────────────────────────────────────┐
  │  Trick: 2ab = (a+b)² - a² - b²          │
  │  Only 3 squarings needed!              │
  └────────────────────────────────────────────┘

  Example: 1234²
    a=12, b=34, m=2
    a²  = karatsubaSquare(12)  = 144
    b²  = karatsubaSquare(34)  = 1156
    (a+b)² = karatsubaSquare(46) = 2116
    2ab = 2116 - 144 - 1156 = 816

    Result = 144·10000 + 816·100 + 1156
           = 1440000 + 81600 + 1156 = 1522756
    Verify: 1234² = 1522756 ✓

  Recursion: same T(n) = 3T(n/2) + O(n) = O(n^1.585)
```

---

## Example 9: Complex Number Multiplication (3 muls instead of 4)

```go
package main

import "fmt"

// (a + bi)(c + di) = (ac - bd) + (ad + bc)i
// Naive: 4 real multiplications
// Karatsuba-style: 3 real multiplications

type Complex struct{ Re, Im float64 }

func mulNaive(x, y Complex) Complex {
	// 4 multiplications
	return Complex{
		Re: x.Re*y.Re - x.Im*y.Im,
		Im: x.Re*y.Im + x.Im*y.Re,
	}
}

func mulKaratsuba(x, y Complex) Complex {
	// 3 multiplications (same Karatsuba trick!)
	p1 := x.Re * y.Re        // ac
	p2 := x.Im * y.Im        // bd
	p3 := (x.Re + x.Im) * (y.Re + y.Im) // (a+b)(c+d)

	return Complex{
		Re: p1 - p2,           // ac - bd
		Im: p3 - p1 - p2,     // ad + bc = (a+b)(c+d) - ac - bd
	}
}

func main() {
	x := Complex{3, 4}
	y := Complex{1, 2}

	naive := mulNaive(x, y)
	kara := mulKaratsuba(x, y)

	fmt.Printf("(%g + %gi) × (%g + %gi)\n", x.Re, x.Im, y.Re, y.Im)
	fmt.Printf("Naive:     %g + %gi\n", naive.Re, naive.Im)
	fmt.Printf("Karatsuba: %g + %gi\n", kara.Re, kara.Im)
	fmt.Println("\nSame trick: 3 real muls instead of 4!")
	fmt.Println("Also known as Gauss's complex multiplication trick")
}
```

**Textual Figure:**

```
  Complex Multiplication: (3+4i)(1+2i)

  Naive (4 real multiplications):
  ┌──────────────────────────────────────────┐
  │  Re = ac - bd = 3×1 - 4×2 = 3-8 = -5    │
  │  Im = ad + bc = 3×2 + 4×1 = 6+4 = 10    │
  │  Muls: ac, bd, ad, bc = 4                │
  └──────────────────────────────────────────┘

  Karatsuba/Gauss (3 real multiplications):
  ┌──────────────────────────────────────────┐
  │  p1 = ac = 3×1 = 3                       │
  │  p2 = bd = 4×2 = 8                       │
  │  p3 = (a+b)(c+d) = 7×3 = 21             │
  │  Re = p1 - p2     = 3-8    = -5          │
  │  Im = p3 - p1 - p2 = 21-3-8 = 10        │
  │  Muls: 3 only! ★                          │
  └──────────────────────────────────────────┘

  Result: (3+4i)(1+2i) = -5 + 10i ✓
  Same Karatsuba pattern: ad+bc = (a+b)(c+d) - ac - bd
```

---

## Example 10: Patterns & Comparison Summary

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Karatsuba & D&C Multiplication Patterns ===\n")

	fmt.Println("--- The Core Pattern ---")
	fmt.Println("  Split operands:    x = aB^m + b,  y = cB^m + d")
	fmt.Println("  Naive product:     xy = ac·B²ᵐ + (ad+bc)·Bᵐ + bd   [4 muls]")
	fmt.Println("  Karatsuba trick:   ad+bc = (a+b)(c+d) - ac - bd     [3 muls]")
	fmt.Println()

	fmt.Println("--- Same Trick in Different Contexts ---")
	contexts := []struct{ context, savings string }{
		{"Integer multiplication", "3 half-size muls instead of 4"},
		{"Polynomial multiplication", "3 half-degree muls instead of 4"},
		{"Matrix multiplication (Strassen)", "7 half-size muls instead of 8"},
		{"Complex number multiplication", "3 real muls instead of 4"},
	}
	for _, c := range contexts {
		fmt.Printf("  %-38s %s\n", c.context, c.savings)
	}

	fmt.Println("\n--- Practical Hierarchy ---")
	hierarchy := []struct{ range_, method string }{
		{"< 20 digits", "Grade school O(n²)"},
		{"20-1000 digits", "Karatsuba O(n^1.585)"},
		{"1000-10000 digits", "Toom-Cook 3 O(n^1.465)"},
		{"> 10000 digits", "FFT-based O(n log n log log n)"},
	}
	for _, h := range hierarchy {
		fmt.Printf("  %-20s → %s\n", h.range_, h.method)
	}

	fmt.Println("\n--- Interview Tips ---")
	tips := []string{
		"Know the 3-vs-4-muls trick cold — explain in 30 seconds",
		"Master theorem: T(n) = 3T(n/2) + O(n) → O(n^log₂3)",
		"Same idea as Strassen: save one multiplication per level",
		"Complex number variant appears in signal processing",
		"FFT is the practical champion for very large numbers",
	}
	for _, t := range tips { fmt.Println("  •", t) }
}
```

**Textual Figure:**

```
  Karatsuba Pattern Applied Everywhere

  The Core Trick:
  ┌───────────────────────────────────────────────┐
  │  Need: ac, bd, ad+bc  (4 products)         │
  │  Compute: ac, bd, (a+b)(c+d)  (3 products) │
  │  Derive: ad+bc = (a+b)(c+d) - ac - bd      │
  └───────────────────────────────────────────────┘

  Where it appears:
  ┌────────────────────┬────────────┬───────────────┐
  │  Context           │  Muls saved  │  Complexity     │
  ├────────────────────┼────────────┼───────────────┤
  │  Integers           │  4 → 3       │  O(n^1.585)    │
  │  Polynomials         │  4 → 3       │  O(n^1.585)    │
  │  Complex numbers     │  4 → 3       │  constant      │
  │  Matrices (Strassen) │  8 → 7       │  O(n^2.807)    │
  └────────────────────┴────────────┴───────────────┘
```

---

## Key Takeaways

1. **3 vs 4 multiplications**: the fundamental insight — compute `ac`, `bd`, `(a+b)(c+d)` to get all four products
2. **O(n^1.585)**: from T(n) = 3·T(n/2) + O(n), log₂3 ≈ 1.585
3. **Same trick everywhere**: integers, polynomials, complex numbers, matrices (Strassen)
4. **Practical crossover**: Karatsuba beats grade school around 20-80 digits
5. **Hierarchy**: grade school → Karatsuba → Toom-Cook → FFT for increasingly large inputs
6. **Interview**: explain the trick clearly; relate to Strassen for bonus points

> **Phase 24 Divide & Conquer Complete! Next up: Phase 25 — Advanced Sliding Window →**
