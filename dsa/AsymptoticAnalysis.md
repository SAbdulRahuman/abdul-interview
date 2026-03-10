# Phase 1: Algorithm Complexity — Asymptotic Analysis

## What is Asymptotic Analysis?

Asymptotic analysis studies algorithm behavior **as input size approaches infinity**. It ignores constants and lower-order terms, focusing on the **dominant growth rate**.

"Asymptotic" = "approaching a value as n → ∞"

It answers: **How does your algorithm scale?**

---

## Example 1: Why Constants Don't Matter at Scale

```go
package main

import "fmt"

// Algorithm A: 100n operations
func algorithmA(n int) int {
    ops := 0
    for round := 0; round < 100; round++ { // constant 100 rounds
        for i := 0; i < n; i++ {
            ops++
        }
    }
    return ops // 100n → O(n)
}

// Algorithm B: n² operations
func algorithmB(n int) int {
    ops := 0
    for i := 0; i < n; i++ {
        for j := 0; j < n; j++ {
            ops++
        }
    }
    return ops // n² → O(n²)
}

func main() {
    fmt.Println("n\t| 100n (O(n))\t| n² (O(n²))\t| Winner")
    fmt.Println("--------|---------------|---------------|--------")
    for _, n := range []int{10, 50, 100, 500, 1000, 10000} {
        a := algorithmA(n)
        b := algorithmB(n)
        winner := "A (linear)"
        if a > b {
            winner = "B (quadratic)"
        }
        fmt.Printf("%d\t| %d\t| %d\t| %s\n", n, a, b, winner)
    }
    // For small n, B wins! But as n grows, A always wins.
    // Asymptotic analysis predicts long-term behavior.
}
```

**Textual Figure — Why Constants Vanish at Scale:**

```
  100n (O(n)) vs n² (O(n²)):

  n      100n        n²       Winner
  ────   ────────   ────────  ──────────
  10     1,000       100       n² wins      (constant saves B)
  50     5,000       2,500     n² wins
  100    10,000      10,000    TIE ← crossover point!
  500    50,000      250,000   100n wins    (scaling saves A)
  1000   100,000     1,000,000 100n wins
  10000  1,000,000   10⁸       100n wins by 100×

  ops ▲
      │          · n²
      │        ·
      │      ·
      │    ·
      │   · ──────  100n  (linear always wins eventually)
      │  · ··
      │··
      └──────────────▶ n
        ↑
      crossover at n=100

  Asymptotic analysis ignores constants because
  n² ALWAYS overtakes c·n for large enough n.
```

---

## Example 2: Comparing Growth Rates Visually

```go
package main

import (
    "fmt"
    "math"
    "strings"
)

func main() {
    fmt.Println("Growth rates for increasing n:")
    fmt.Printf("%-6s %-8s %-8s %-10s %-12s %-14s %-10s\n",
        "n", "O(1)", "O(logn)", "O(n)", "O(nlogn)", "O(n²)", "O(2^n)")
    fmt.Println(strings.Repeat("-", 75))

    for _, n := range []int{1, 2, 4, 8, 16, 32, 64, 128, 256} {
        fn := float64(n)
        o1 := 1
        ologn := int(math.Log2(fn)) + 1
        on := n
        onlogn := int(fn * (math.Log2(fn) + 1))
        on2 := n * n
        o2n := "∞"
        if n <= 20 {
            o2n = fmt.Sprintf("%d", int(math.Pow(2, fn)))
        }

        fmt.Printf("%-6d %-8d %-8d %-10d %-12d %-14d %-10s\n",
            n, o1, ologn, on, onlogn, on2, o2n)
    }
}
```

**Textual Figure — Growth Rate Comparison:**

```
  How fast functions grow (for n=16):

  O(1):       1         ■
  O(log n):   4         ■■■■
  O(n):       16        ■■■■■■■■■■■■■■■■
  O(n log n): 64        ■■■■■■... (64 blocks)
  O(n²):      256       ■■■■■■... (256 blocks)
  O(2ⁿ):      65,536    ■■■■■■... (65K blocks!)

  Growth hierarchy (slowest to fastest):
  ┌─────────────────────────────────────────────┐
  │  O(1) < O(log n) < O(√n) < O(n)           │
  │       < O(n log n) < O(n²) < O(n³)        │
  │       < O(2ⁿ) < O(n!) < O(nⁿ)              │
  └─────────────────────────────────────────────┘
                       ↑
         The "efficiency boundary" — polynomial vs exponential
```

--- — Same Big O, Different Constants

```go
package main

import (
    "fmt"
    "time"
)

// All three are O(n) — asymptotically equivalent
func sumV1(nums []int) int {
    s := 0
    for _, n := range nums { s += n }
    return s
}

func sumV2(nums []int) int {
    s := 0
    for _, n := range nums { s += n }
    for _, n := range nums { s += n } // 2 passes
    s /= 2
    return s
}

func sumV3(nums []int) int {
    s := 0
    for _, n := range nums {
        s += n
        _ = n * n // wasteful extra work
        _ = n + 1
    }
    return s
}

func benchmark(name string, f func([]int) int, nums []int) {
    start := time.Now()
    for i := 0; i < 1000; i++ {
        f(nums)
    }
    elapsed := time.Since(start)
    fmt.Printf("%-10s %v\n", name, elapsed)
}

func main() {
    nums := make([]int, 100000)
    for i := range nums { nums[i] = i }

    // All O(n), but real-time differs due to constants
    benchmark("sumV1:", sumV1, nums)
    benchmark("sumV2:", sumV2, nums)
    benchmark("sumV3:", sumV3, nums)

    fmt.Println("\nAll are O(n) asymptotically, but constants affect real performance")
}
```

**Textual Figure — Same Big O, Different Real Performance:**

```
  sumV1: 1 pass, 1 op/element   ──▶ fastest constant
  sumV2: 2 passes, 1 op/element ──▶ 2× slower in practice
  sumV3: 1 pass, 3 ops/element  ──▶ 3× slower in practice

  All three: O(n)

  Real time (n=100,000 × 1000 iterations):
  ┌───────────────────────────────────────────┐
  │  sumV1 ███████              ~1.0×        │
  │  sumV2 ██████████████     ~2.0×        │
  │  sumV3 ██████████████████ ~2.5×        │
  └───────────────────────────────────────────┘

  Key insight: Big O tells you the SHAPE (linear),
  not the exact speed. Constants matter in practice
  but not in asymptotic classification.
```

---

## Example 4: Little-o and Little-ω (Strict Bounds)

```go
package main

import "fmt"

// Little-o: f(n) = o(g(n)) means f grows STRICTLY slower than g
// (Not just ≤, but strictly <)
//
// Little-ω: f(n) = ω(g(n)) means f grows STRICTLY faster than g

// n = o(n²): n grows strictly slower than n²
// n² = ω(n): n² grows strictly faster than n

func main() {
    fmt.Println("Asymptotic Notation Summary:")
    fmt.Println()

    notations := []struct {
        Symbol  string
        Name    string
        Meaning string
        Analogy string
        Example string
    }{
        {"f = O(g)", "Big O", "f ≤ c·g (upper bound)", "≤", "n = O(n²) ✓"},
        {"f = Ω(g)", "Big Omega", "f ≥ c·g (lower bound)", "≥", "n² = Ω(n) ✓"},
        {"f = Θ(g)", "Big Theta", "c₁·g ≤ f ≤ c₂·g (tight)", "=", "2n+3 = Θ(n) ✓"},
        {"f = o(g)", "Little o", "f/g → 0 (strictly less)", "<", "n = o(n²) ✓"},
        {"f = ω(g)", "Little omega", "f/g → ∞ (strictly greater)", ">", "n² = ω(n) ✓"},
    }

    fmt.Printf("%-12s %-14s %-30s %-8s %-20s\n",
        "Notation", "Name", "Meaning", "Like", "Example")
    fmt.Println("------------------------------------------------------------------------------------")
    for _, n := range notations {
        fmt.Printf("%-12s %-14s %-30s %-8s %-20s\n",
            n.Symbol, n.Name, n.Meaning, n.Analogy, n.Example)
    }
}
```

**Textual Figure — Five Asymptotic Notations:**

```
  f(n) compared to g(n):

  O (Big O) — upper bound (f ≤ c·g):
       g(n) ········
       f(n) ──────     "at most this fast"

  Ω (Big Omega) — lower bound (f ≥ c·g):
       f(n) ········
       g(n) ──────     "at least this fast"

  Θ (Big Theta) — tight bound (c₁·g ≤ f ≤ c₂·g):
       c₂·g ·······
       f(n) ──────     "exactly this rate"
       c₁·g ·······

  o (little o) — strict upper (f/g → 0):
       f grows STRICTLY slower than g (like < vs ≤)

  ω (little omega) — strict lower (f/g → ∞):
       f grows STRICTLY faster than g (like > vs ≥)

  Number analogy:
  ┌────────────────────────────────────┐
  │  O    ↔  ≤       o    ↔  <         │
  │  Ω    ↔  ≥       ω    ↔  >         │
  │  Θ    ↔  =                          │
  └────────────────────────────────────┘
```

---

```go
package main

import "fmt"

// Pattern 1: Sequential loops → add
func sequential(n int) int {
    count := 0
    for i := 0; i < n; i++ { count++ }     // O(n)
    for i := 0; i < n*n; i++ { count++ }   // O(n²)
    // Total: O(n + n²) = O(n²) — drop lower term
    return count
}

// Pattern 2: Nested loops → multiply
func nested(n int) int {
    count := 0
    for i := 0; i < n; i++ {       // O(n)
        for j := 0; j < n; j++ {   // × O(n)
            count++
        }
    }
    return count // O(n²)
}

// Pattern 3: Logarithmic loop
func logLoop(n int) int {
    count := 0
    for i := 1; i < n; i *= 2 { // doubles each time
        count++
    }
    return count // O(log n)
}

// Pattern 4: Linear × logarithmic
func nLogN(n int) int {
    count := 0
    for i := 0; i < n; i++ {        // O(n)
        for j := 1; j < n; j *= 2 { // × O(log n)
            count++
        }
    }
    return count // O(n log n)
}

// Pattern 5: Harmonic sum loop
func harmonicLoop(n int) int {
    count := 0
    for i := 1; i <= n; i++ {
        for j := 0; j < n/i; j++ { // n + n/2 + n/3 + ... + 1
            count++
        }
    }
    return count // O(n log n) — harmonic series
}

func main() {
    n := 1000
    fmt.Printf("sequential(%d): %d ops → O(n²)\n", n, sequential(n))
    fmt.Printf("nested(%d):     %d ops → O(n²)\n", n, nested(n))
    fmt.Printf("logLoop(%d):    %d ops → O(log n)\n", n, logLoop(n))
    fmt.Printf("nLogN(%d):      %d ops → O(n log n)\n", n, nLogN(n))
    fmt.Printf("harmonic(%d):   %d ops → O(n log n)\n", n, harmonicLoop(n))
}
```

**Textual Figure — Loop Analysis Patterns:**

```
  Pattern 1: Sequential → ADD complexities
  ────────────────────────────────────
  for i=0..n: ...     O(n)
  for j=0..n²: ...    O(n²)
  Total: O(n) + O(n²) = O(n²)  ← drop lower term

  Pattern 2: Nested → MULTIPLY complexities
  ────────────────────────────────────
  for i=0..n:          O(n)
    for j=0..n:        × O(n)
  Total: O(n) × O(n) = O(n²)

  Pattern 3: Doubling/halving → O(log n)
  ────────────────────────────────────
  for i=1; i<n; i*=2:  O(log n)
   i: 1 → 2 → 4 → 8 → ... → n
   Steps: log₂(n)

  Pattern 4: Outer linear × inner logarithmic
  ────────────────────────────────────
  for i=0..n:            O(n)
    for j=1; j<n; j*=2:  × O(log n)
  Total: O(n log n)

  Pattern 5: Harmonic series
  ────────────────────────────────────
  for i=1..n:
    for j=0; j<n/i:      n/1 + n/2 + n/3 + ... + n/n
                         = n(1 + 1/2 + 1/3 + ... + 1/n)
                         = n × H(n) ≈ n ln n
                         = O(n log n)
```

---

## Example 6: Asymptotic Analysis of Recursive Algorithms

```go
package main

import "fmt"

// Analyzing recursion: count total work across all calls

// Linear recursion: each call does O(1), depth n → O(n)
func linearRec(n int) int {
    if n <= 0 { return 0 }
    return 1 + linearRec(n-1)
}

// Divide recursion: each call does O(1), depth log n → O(log n)
func divideRec(n int) int {
    if n <= 1 { return 1 }
    return 1 + divideRec(n/2)
}

// Branching recursion: 2 calls, each with n-1 → O(2ⁿ)
func branchRec(n int) int {
    if n <= 0 { return 1 }
    return branchRec(n-1) + branchRec(n-1)
}

func main() {
    fmt.Println("Linear recursion (n=20):", linearRec(20))   // O(n) = 20
    fmt.Println("Divide recursion (n=1M):", divideRec(1000000)) // O(log n) ≈ 20
    fmt.Println("Branch recursion (n=10):", branchRec(10))    // O(2ⁿ) = 1024
}
```

**Textual Figure — Three Recursion Patterns:**

```
  Linear Recursion: T(n) = T(n-1) + O(1) → O(n)
  ●─●─●─●─●─...─●   (chain of n calls)
  Depth=n, 1 branch = n nodes = O(n)

  Divide Recursion: T(n) = T(n/2) + O(1) → O(log n)
  ●─●─●─●─●          (chain of log n calls)
  n → n/2 → n/4 → ... → 1
  Depth=log n, 1 branch = log n nodes

  Branching Recursion: T(n) = 2T(n-1) + O(1) → O(2ⁿ)
       ●
      / \
     ●   ●
    / \ / \
   ●  ● ●  ●        (full binary tree, depth n)
  Depth=n, 2 branches = 2ⁿ leaves = O(2ⁿ)

  Summary:
  ┌────────────┬──────────┬─────────┬──────────┐
  │  Pattern   │ Branches │  Depth  │ Nodes    │
  ├────────────┼──────────┼─────────┼──────────┤
  │  Linear   │    1     │   n     │ O(n)     │
  │  Divide   │    1     │ log n   │ O(log n) │
  │  Branch   │    2     │   n     │ O(2ⁿ)    │
  └────────────┴──────────┴─────────┴──────────┘
```

---

## Example 7: Polynomial vs Exponential — The Efficiency Boundary

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    fmt.Println("=== Why Polynomial Algorithms Are 'Efficient' ===")
    fmt.Println()
    fmt.Println("For n = 100:")
    fmt.Printf("  n     = %d\n", 100)
    fmt.Printf("  n²    = %d\n", 10000)
    fmt.Printf("  n³    = %d\n", 1000000)
    fmt.Printf("  n⁵    = %.0f\n", math.Pow(100, 5))
    fmt.Printf("  2ⁿ    = %.2e (impossibly large!)\n", math.Pow(2, 100))
    fmt.Printf("  n!    = way larger than 2ⁿ\n")

    fmt.Println()
    fmt.Println("Tractability classes:")
    fmt.Println("  P     = problems solvable in polynomial time (efficient)")
    fmt.Println("  NP    = problems verifiable in polynomial time")
    fmt.Println("  NP-hard = at least as hard as NP")
    fmt.Println()
    fmt.Println("Polynomial: O(n), O(n²), O(n³) → tractable")
    fmt.Println("Exponential: O(2ⁿ), O(n!) → intractable for large n")
}
```

**Textual Figure — The Polynomial/Exponential Boundary:**

```
  For n = 100:
  ┌──────────┬─────────────────┬──────────────────┐
  │  Class   │    Value        │  Feasible?       │
  ├──────────┼─────────────────┼──────────────────┤
  │  n       │  100            │  ✓ instant       │
  │  n²      │  10,000         │  ✓ instant       │
  │  n³      │  1,000,000      │  ✓ < 1 second    │
  │  n⁵      │  10,000,000,000 │  ⚠ minutes       │
  │──────────┼─────────────────┼ BOUNDARY ────────┤
  │  2ⁿ      │  1.27 × 10³⁰    │  ✘ universe dies  │
  │  n!      │  9.3 × 10¹⁵⁷    │  ✘ impossible     │
  └──────────┴─────────────────┴──────────────────┘

  Polynomial algorithms (P) are considered "efficient"
  even if the degree is high, because exponential
  growth will ALWAYS eventually dwarf any polynomial.
```

---

## Example 8: Input Size Implications

```go
package main

import "fmt"

// What "n" means for different data structures
func main() {
    fmt.Println("=== Input Size 'n' for Different Problems ===")
    fmt.Println()

    cases := []struct {
        Problem   string
        NMeans    string
        Typical   string
    }{
        {"Array operations", "number of elements", "n = len(arr)"},
        {"String operations", "length of string", "n = len(s)"},
        {"Matrix operations", "rows × cols", "n = rows, m = cols"},
        {"Graph operations", "vertices + edges", "V = vertices, E = edges"},
        {"Tree operations", "number of nodes", "n = nodes"},
        {"Number theory", "number of digits", "n = log₁₀(value)"},
        {"Bit manipulation", "number of bits", "n = log₂(value)"},
        {"Sorting", "number of elements", "n = len(arr)"},
    }

    fmt.Printf("%-25s %-25s %-25s\n", "Problem Type", "What n means", "Typical")
    fmt.Println("------------------------------------------------------------------------")
    for _, c := range cases {
        fmt.Printf("%-25s %-25s %-25s\n", c.Problem, c.NMeans, c.Typical)
    }
}
```

**Textual Figure — What Counts as "n"?:**

```
  Different problems measure input differently:

  Array:   nums = [3, 1, 4, 1, 5]    n = 5 (elements)
  String:  s = "hello"                n = 5 (characters)
  Matrix:  3×4 grid                   n = 3, m = 4
  Graph:   5 nodes, 7 edges           V = 5, E = 7
  Number:  n = 1000000                digits = 7

  Why it matters:
  ┌──────────────────────────────────────────┐
  │  isPrime(n):                              │
  │  Input = the number n                     │
  │  Input SIZE = log₂(n) bits = d digits      │
  │                                            │
  │  Trial division: O(√n)                    │
  │  Looks polynomial in n...                  │
  │  But EXPONENTIAL in input size d!           │
  │  √n = √10ᵈ = 10^(d/2) = exponential in d │
  └──────────────────────────────────────────┘
```

--- — What n Can You Handle?

```go
package main

import "fmt"

func main() {
    fmt.Println("=== Maximum n for ~1 second (10⁸ operations) ===")
    fmt.Println()

    limits := []struct {
        Complexity string
        MaxN       string
        Algorithm  string
    }{
        {"O(1)", "any", "Hash lookup, array access"},
        {"O(log n)", "~10^(10⁸)", "Binary search"},
        {"O(n)", "~10⁸", "Linear scan, single pass"},
        {"O(n log n)", "~5×10⁶", "Sorting, priority queue"},
        {"O(n²)", "~10⁴", "Nested loops, brute force pairs"},
        {"O(n³)", "~500", "Triple nested loops"},
        {"O(2ⁿ)", "~25", "Subsets, bitmask DP"},
        {"O(n!)", "~12", "Permutations, TSP brute force"},
    }

    fmt.Printf("%-12s %-12s %-40s\n", "Complexity", "Max n", "Example Algorithm")
    fmt.Println("--------------------------------------------------------------")
    for _, l := range limits {
        fmt.Printf("%-12s %-12s %-40s\n", l.Complexity, l.MaxN, l.Algorithm)
    }

    fmt.Println()
    fmt.Println("Interview tip: If n ≤ 10⁴, O(n²) is fine.")
    fmt.Println("              If n ≤ 10⁶, you need O(n log n) or better.")
    fmt.Println("              If n ≤ 10⁸, you need O(n).")
}
```

**Textual Figure — Practical n Limits:**

```
  Given ~10⁸ operations per second:

  Complexity  Max n     Visual scale
  ─────────── ───────── ────────────────────────────────
  O(n)        10⁸       ██████████████████████████████
  O(n log n)  5×10⁶     ████████████████████
  O(n²)       10⁴       ██████
  O(n³)       500        ██
  O(2ⁿ)       25         ▪
  O(n!)       12         ▪

  Interview decision tree:
  ┌──────────────────────────────────────────┐
  │  n ≤ 20   → O(2ⁿ) or O(n!) OK (brute/DP) │
  │  n ≤ 500  → O(n³) OK                     │
  │  n ≤ 10⁴  → O(n²) OK                     │
  │  n ≤ 10⁶  → need O(n log n)               │
  │  n ≤ 10⁸  → need O(n)                     │
  │  n > 10⁸  → need O(log n) or O(1)         │
  └──────────────────────────────────────────┘
```

---

## Example 10: Complete Asymptotic Analysis of an Algorithm

```go
package main

import "fmt"

// Full analysis of a function
func longestUniqueSubstring(s string) int {
    // Step 1: Identify variables
    // n = len(s)

    n := len(s)
    if n == 0 { return 0 }

    charIndex := make(map[byte]int) // Space: O(min(n, alphabet_size))
    maxLen := 0
    start := 0

    for end := 0; end < n; end++ { // Time: O(n) — each element visited once
        if idx, ok := charIndex[s[end]]; ok && idx >= start {
            start = idx + 1 // O(1) per operation
        }
        charIndex[s[end]] = end // O(1) hash map operation
        if end-start+1 > maxLen {
            maxLen = end - start + 1
        }
    }
    return maxLen
}

func main() {
    tests := []string{
        "abcabcbb",
        "bbbbb",
        "pwwkew",
        "abcdefghijklmnopqrstuvwxyz",
    }

    for _, s := range tests {
        fmt.Printf("'%s' → longest unique substring length: %d\n",
            s, longestUniqueSubstring(s))
    }

    fmt.Println("\n=== Complete Analysis ===")
    fmt.Println("Time Complexity:")
    fmt.Println("  Best case:    O(n) — all unique characters")
    fmt.Println("  Worst case:   O(n) — all same characters")
    fmt.Println("  Average case: O(n)")
    fmt.Println("  → Θ(n) — tight bound")
    fmt.Println()
    fmt.Println("Space Complexity:")
    fmt.Println("  O(min(n, |Σ|)) where |Σ| = alphabet size")
    fmt.Println("  For ASCII: O(128) = O(1)")
    fmt.Println("  For Unicode: O(n)")
}
```

**Textual Figure — Complete Analysis of Sliding Window:**

```
  longestUniqueSubstring("abcabcbb")

  Sliding window trace:
  s = a b c a b c b b
      0 1 2 3 4 5 6 7

  start=0, end=0: 'a' new         window=[a]       max=1
  start=0, end=1: 'b' new         window=[a,b]     max=2
  start=0, end=2: 'c' new         window=[a,b,c]   max=3
  start=0, end=3: 'a' dup at 0    start→1
                                  window=[b,c,a]   max=3
  start=1, end=4: 'b' dup at 1    start→2
                                  window=[c,a,b]   max=3
  start=2, end=5: 'c' dup at 2    start→3
                                  window=[a,b,c]   max=3
  start=3, end=6: 'b' dup at 4    start→5
                                  window=[c,b]     max=3
  start=5, end=7: 'b' dup at 6    start→7
                                  window=[b]       max=3

  Analysis:
  ┌────────────────────────────────────────────┐
  │  Time:  O(n) — each char visited once by 'end' │
  │  Space: O(min(n, |Σ|)) — hash map              │
  │  Best = Worst = Average = O(n) → Θ(n)         │
  └────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Asymptotic analysis** = behavior as n → ∞
2. **Constants don't matter** at scale (100n < n² for large n)
3. **Five notations**: O (≤), Ω (≥), Θ (=), o (<), ω (>)
4. **Polynomial is tractable**, exponential is not
5. **Know your limits**: n ≤ 10⁴ → O(n²) OK, n ≤ 10⁶ → need O(n log n)
6. **Always state**: time complexity, space complexity, which case

> **Phase 1 Complete! Next: Phase 2 — Arrays →**
