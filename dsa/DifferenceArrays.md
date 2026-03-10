# Phase 2: Arrays — Difference Arrays

## What is a Difference Array?

A **difference array** is the inverse of a prefix sum. It stores the difference between consecutive elements, enabling **range updates in O(1)** and reconstruction in O(n).

```
Original:   [a₀, a₁, a₂, a₃, a₄]
Difference: [a₀, a₁-a₀, a₂-a₁, a₃-a₂, a₄-a₃]

Range update [L, R] by +val:
    diff[L] += val
    diff[R+1] -= val

Reconstruct original: prefix sum of diff[]
```

---

## When to Use

| Scenario | Use |
|----------|-----|
| Many range updates, one final read | Difference array |
| Many range reads, few updates | Prefix sum |
| Both updates and reads | Fenwick / Segment tree |

---

## Example 1: Build and Reconstruct

```go
package main

import "fmt"

func buildDiff(nums []int) []int {
    diff := make([]int, len(nums))
    diff[0] = nums[0]
    for i := 1; i < len(nums); i++ {
        diff[i] = nums[i] - nums[i-1]
    }
    return diff
}

func reconstruct(diff []int) []int {
    result := make([]int, len(diff))
    result[0] = diff[0]
    for i := 1; i < len(diff); i++ {
        result[i] = result[i-1] + diff[i]
    }
    return result
}

func main() {
    nums := []int{3, 1, 4, 1, 5, 9}
    diff := buildDiff(nums)
    fmt.Println("Original:  ", nums)
    fmt.Println("Difference:", diff) // [3 -2 3 -3 4 4]

    rebuilt := reconstruct(diff)
    fmt.Println("Rebuilt:   ", rebuilt) // [3 1 4 1 5 9]
}
```

**Textual Figure — Build & Reconstruct Difference Array:**

```
  nums:  [3,   1,   4,   1,   5,   9]
  index:  0    1    2    3    4    5

  Build difference:  diff[i] = nums[i] - nums[i-1]
    diff[0] = 3          (first element as-is)
    diff[1] = 1-3  = -2
    diff[2] = 4-1  =  3
    diff[3] = 1-4  = -3
    diff[4] = 5-1  =  4
    diff[5] = 9-5  =  4

  diff:  [3,  -2,   3,  -3,   4,   4]

  Reconstruct (prefix sum of diff):
    result[0] = 3
    result[1] = 3 + (-2) = 1
    result[2] = 1 + 3    = 4
    result[3] = 4 + (-3) = 1
    result[4] = 1 + 4    = 5
    result[5] = 5 + 4    = 9

  result: [3,  1,  4,  1,  5,  9]  ← matches original! ✓

  ┌────────────────────────────────────┐
  │  diff = differences between elements  │
  │  prefix_sum(diff) = original array     │
  └────────────────────────────────────┘
```

---

## Example 2: Range Update in O(1)

```go
package main

import "fmt"

func main() {
    n := 8
    diff := make([]int, n+1) // extra element for boundary

    // Add 5 to range [2, 5]
    diff[2] += 5
    diff[6] -= 5

    // Add 3 to range [4, 7]
    diff[4] += 3
    diff[8] -= 3 // note: uses n+1 slot

    // Reconstruct
    result := make([]int, n)
    result[0] = diff[0]
    for i := 1; i < n; i++ {
        result[i] = result[i-1] + diff[i]
    }
    fmt.Println("Result:", result) // [0 0 5 5 8 8 3 3]
    //                               indices: 0 1 2 3 4 5 6 7
    // [2,5] gets +5, [4,7] gets +3
}
```

**Textual Figure — Range Update in O(1):**

```
  n=8, start all zeros:  [0, 0, 0, 0, 0, 0, 0, 0]

  Update 1: Add 5 to [2..5]
  diff: [0, 0, +5, 0, 0, 0, -5, 0, 0]
                ↑              ↑
              diff[2]+=5    diff[6]-=5

  Update 2: Add 3 to [4..7]
  diff: [0, 0, +5, 0, +3, 0, -5, 0, -3]
                       ↑              ↑
                    diff[4]+=3    diff[8]-=3

  Reconstruct (prefix sum):
  diff:   [0,  0, +5,  0, +3,  0, -5,  0, -3]
  result: [0,  0,  5,  5,  8,  8,  3,  3]
          │  │        │              │
          │  └── +5 ──┘  +3 added    │
          │     [2..5]     [4..7]     │

          Index: 0  1  2  3  4  5  6  7
  +5 range:            ████████████
  +3 range:                  ████████████
  result:      0  0  5  5  8  8  3  3
```

---

## Example 3: Corporate Flight Bookings (LeetCode 1109)

```go
package main

import "fmt"

// n flights numbered 1..n
// bookings[i] = [first, last, seats]
func corpFlightBookings(bookings [][]int, n int) []int {
    diff := make([]int, n+1)

    for _, b := range bookings {
        first, last, seats := b[0]-1, b[1]-1, b[2]
        diff[first] += seats
        if last+1 < len(diff) {
            diff[last+1] -= seats
        }
    }

    result := make([]int, n)
    result[0] = diff[0]
    for i := 1; i < n; i++ {
        result[i] = result[i-1] + diff[i]
    }
    return result
}

func main() {
    bookings := [][]int{{1, 2, 10}, {2, 3, 20}, {2, 5, 25}}
    fmt.Println(corpFlightBookings(bookings, 5))
    // [10 55 45 25 25]
    // Flight 1: 10
    // Flight 2: 10+20+25 = 55
    // Flight 3: 20+25 = 45
    // Flight 4: 25
    // Flight 5: 25
}
```

**Textual Figure — Corporate Flight Bookings:**

```
  5 flights, 3 bookings:  [1,2,10]  [2,3,20]  [2,5,25]

  diff (0-indexed):
    [1,2,10]: diff[0]+=10, diff[2]-=10
    [2,3,20]: diff[1]+=20, diff[3]-=20
    [2,5,25]: diff[1]+=25, diff[5]-=25

  diff:  [10, 45, -10, -20, 0, -25]

  Reconstruct:
    Flight 1: 10
    Flight 2: 10+45 = 55
    Flight 3: 55+(-10) = 45
    Flight 4: 45+(-20) = 25
    Flight 5: 25+0 = 25

  Visual:
  Flight:   1    2    3    4    5
  +10:     ████ ████
  +20:          ████ ████
  +25:          ████ ████ ████ ████
  Total:   10   55   45   25   25
```

---

## Example 4: Car Pooling (LeetCode 1094)

```go
package main

import "fmt"

func carPooling(trips [][]int, capacity int) bool {
    // trips[i] = [numPassengers, from, to]
    // max possible location
    maxLoc := 0
    for _, t := range trips {
        if t[2] > maxLoc {
            maxLoc = t[2]
        }
    }

    diff := make([]int, maxLoc+1)
    for _, t := range trips {
        passengers, from, to := t[0], t[1], t[2]
        diff[from] += passengers
        if to < len(diff) {
            diff[to] -= passengers // passengers get off at 'to'
        }
    }

    current := 0
    for _, d := range diff {
        current += d
        if current > capacity {
            return false
        }
    }
    return true
}

func main() {
    fmt.Println(carPooling([][]int{{2, 1, 5}, {3, 3, 7}}, 4))  // false
    fmt.Println(carPooling([][]int{{2, 1, 5}, {3, 3, 7}}, 5))  // true
    fmt.Println(carPooling([][]int{{2, 1, 5}, {3, 5, 7}}, 3))  // true (no overlap)
    fmt.Println(carPooling([][]int{{3, 2, 7}, {3, 7, 9}, {8, 3, 9}}, 11)) // true
}
```

**Textual Figure — Car Pooling:**

```
  capacity = 4
  trips: [2, 1, 5]  [3, 3, 7]

  Timeline (location 0..7):
  diff: [0, +2, 0, +3, 0, -2, 0, -3]
         0   1  2   3  4   5  6   7

  Reconstruct (passengers at each location):
  loc 0:  0
  loc 1:  0+2 = 2
  loc 2:  2
  loc 3:  2+3 = 5  ← exceeds capacity 4!  FAIL
  loc 4:  5
  loc 5:  5-2 = 3  (2 passengers get off)
  loc 6:  3
  loc 7:  3-3 = 0  (3 passengers get off)

  Visual:
  Location: 0  1  2  3  4  5  6  7
  Trip 1:      ████████████      (2 pax, loc 1→5)
  Trip 2:            ████████████ (3 pax, loc 3→7)
  Passengers: 0  2  2  5  5  3  3  0
  Capacity:   4  4  4  4  4  4  4  4
                    ✗  ✗           ← over capacity!
```

---

## Example 5: Multiple Range Increments, Then Query

```go
package main

import "fmt"

type RangeUpdater struct {
    diff []int
    n    int
}

func NewRangeUpdater(n int) *RangeUpdater {
    return &RangeUpdater{diff: make([]int, n+1), n: n}
}

func (r *RangeUpdater) AddRange(l, rr, val int) {
    r.diff[l] += val
    if rr+1 <= r.n {
        r.diff[rr+1] -= val
    }
}

func (r *RangeUpdater) Build() []int {
    result := make([]int, r.n)
    result[0] = r.diff[0]
    for i := 1; i < r.n; i++ {
        result[i] = result[i-1] + r.diff[i]
    }
    return result
}

func main() {
    ru := NewRangeUpdater(10)

    // Apply multiple range updates
    ru.AddRange(1, 5, 10)  // [1..5] += 10
    ru.AddRange(3, 8, 20)  // [3..8] += 20
    ru.AddRange(0, 2, 5)   // [0..2] += 5
    ru.AddRange(7, 9, 15)  // [7..9] += 15

    result := ru.Build()
    fmt.Println("Result:", result)
    // Index:  0   1   2   3   4   5   6   7   8   9
    // Value:  5  15  15  30  30  30  20  35  35  15
}
```

**Textual Figure — Multiple Range Updates:**

```
  n = 10, all zeros initially

  Operations:
  [0..4] += 3:   ███████████████
  [2..7] += 5:         ██████████████████
  [5..9] += 2:                  ███████████████
  [1..3] -= 1:      ─────────

  diff array after all operations:
  Index:  0   1   2   3   4   5   6   7   8   9  10
  diff:  [3, -1,  5,  0,  1, -1,  0,  0, -5,  0, -2]

  Prefix sum → result:
  Index:  0   1   2   3   4   5   6   7   8   9
  Value:  3   2   7   7   8   7   7   7   2   2

  All 4 operations applied with just 8 O(1) updates!
```

---

## Example 6: 2D Difference Array

```go
package main

import "fmt"

func main() {
    rows, cols := 5, 5
    diff := make([][]int, rows+1)
    for i := range diff {
        diff[i] = make([]int, cols+1)
    }

    // Add val to submatrix (r1,c1) to (r2,c2)
    addRect := func(r1, c1, r2, c2, val int) {
        diff[r1][c1] += val
        diff[r2+1][c1] -= val
        diff[r1][c2+1] -= val
        diff[r2+1][c2+1] += val
    }

    addRect(1, 1, 3, 3, 10) // inner 3x3 block
    addRect(0, 0, 4, 4, 1)  // entire grid

    // Reconstruct with 2D prefix sum
    result := make([][]int, rows)
    for i := range result {
        result[i] = make([]int, cols)
    }
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            result[i][j] = diff[i][j]
            if i > 0 {
                result[i][j] += result[i-1][j]
            }
            if j > 0 {
                result[i][j] += result[i][j-1]
            }
            if i > 0 && j > 0 {
                result[i][j] -= result[i-1][j-1]
            }
        }
    }

    for _, row := range result {
        fmt.Println(row)
    }
    // [1 1 1 1 1]
    // [1 11 11 11 1]
    // [1 11 11 11 1]
    // [1 11 11 11 1]
    // [1 1 1 1 1]
}
```

**Textual Figure — 2D Difference Array:**

```
  5×5 grid, two operations:
    1) Add 10 to submatrix (1,1)→(3,3)
    2) Add 1  to entire grid (0,0)→(4,4)

  2D diff update for (r1,c1)→(r2,c2) += val:
    diff[r1][c1]     += val
    diff[r2+1][c1]   -= val
    diff[r1][c2+1]   -= val
    diff[r2+1][c2+1]  += val    (inclusion-exclusion)

  After operations, reconstruct with 2D prefix sum:

  Result:
  ┌───┬────┬────┬────┬───┐
  │ 1 │  1 │  1 │  1 │ 1 │   row 0: only +1 from op 2
  ├───┼────┼────┼────┼───┤
  │ 1 │ 11 │ 11 │ 11 │ 1 │   inner: +10 + +1 = 11
  ├───┼────┼────┼────┼───┤
  │ 1 │ 11 │ 11 │ 11 │ 1 │   border: just +1
  ├───┼────┼────┼────┼───┤
  │ 1 │ 11 │ 11 │ 11 │ 1 │
  ├───┼────┼────┼────┼───┤
  │ 1 │  1 │  1 │  1 │ 1 │
  └───┴────┴────┴────┴───┘
```

---

## Example 7: Painting Segments — Counting Overlap

```go
package main

import "fmt"

func main() {
    // Several painters paint segments on a wall [0, 20]
    segments := [][2]int{{2, 8}, {5, 12}, {10, 15}, {1, 3}}

    maxPos := 0
    for _, s := range segments {
        if s[1] > maxPos {
            maxPos = s[1]
        }
    }

    diff := make([]int, maxPos+2)
    for _, s := range segments {
        diff[s[0]]++
        diff[s[1]+1]--
    }

    // Reconstruct: how many painters cover each position
    current := 0
    maxOverlap := 0
    maxPos2 := 0
    for i := 0; i < len(diff); i++ {
        current += diff[i]
        if current > maxOverlap {
            maxOverlap = current
            maxPos2 = i
        }
    }

    fmt.Printf("Max overlap: %d painters at position %d\n", maxOverlap, maxPos2)
    // Max overlap: 2 at position 5 (or 10, depending on segments)

    // Print the coverage map
    current = 0
    for i := 0; i <= 15; i++ {
        current += diff[i]
        fmt.Printf("Pos %2d: %d painters\n", i, current)
    }
}
```

**Textual Figure — Painting Segments (Counting Overlap):**

```
  Segments: [2,8]  [5,12]  [10,15]  [1,3]

  Position:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
  [1, 3]:       █████████
  [2, 8]:          █████████████████████
  [5,12]:                   ████████████████████████
  [10,15]:                               ██████████████████

  Overlap count per position:
  Position: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
  Count:    0  1  2  2  1  2  2  2  2  1  2  2  2  1  1  1

  How: diff[start]++, diff[end+1]--
  Then prefix sum gives overlap count at each position.

  Max overlap = 2 painters at positions 2,3,5,6,7,8,10,11,12
```

---

```go
package main

import "fmt"

func main() {
    hours := 24
    base := make([]int, hours) // base temperature for each hour
    for i := range base {
        base[i] = 20 // 20°C baseline
    }

    diff := make([]int, hours+1)

    // Events that change temperature in ranges
    // Heater on from hour 6-9: +5°C
    diff[6] += 5
    diff[10] -= 5

    // Sun from hour 10-16: +8°C
    diff[10] += 8
    diff[17] -= 8

    // AC from hour 12-18: -3°C
    diff[12] -= 3
    diff[19] += 3

    // Night cooling from 20-23: -4°C
    diff[20] -= 4
    // no end needed if it extends to boundary

    // Reconstruct
    adjustment := 0
    fmt.Println("Hourly temperatures:")
    for h := 0; h < hours; h++ {
        adjustment += diff[h]
        temp := base[h] + adjustment
        fmt.Printf("  Hour %2d: %d°C (adj=%+d)\n", h, temp, adjustment)
    }
}
```

**Textual Figure — Temperature Change Over Time:**

```
  Base: 20°C for all 24 hours

  Events on the difference array:
  Hour:    0  1  2  3  4  5  6  7  8  9 10 11 12 13..16 17 18 19 20..23
  Heater:                       +5───────-5
  Sun:                                      +8────────────-8
  AC:                                              -3─────────+3
  Night:                                                           -4────

  Resulting temperature:
  Hour  0-5:  20°C  (no change)
  Hour  6-9:  25°C  (+5 heater)
  Hour 10-11: 28°C  (+8 sun)
  Hour 12-16: 25°C  (+8 sun, -3 AC)
  Hour 17-18: 17°C  (-3 AC)
  Hour 19:    20°C  (AC off)
  Hour 20-23: 16°C  (-4 night)

  Key: diff array lets us stack changes without touching every hour!
```

---

```go
package main

import "fmt"

func main() {
    n := 10
    // Start with all zeros

    // Operations: add val to range [l, r]
    ops := [][3]int{
        {0, 4, 3},  // [0..4] += 3
        {2, 7, 5},  // [2..7] += 5
        {5, 9, 2},  // [5..9] += 2
        {1, 3, -1}, // [1..3] -= 1
    }

    diff := make([]int, n+1)
    for _, op := range ops {
        l, r, val := op[0], op[1], op[2]
        diff[l] += val
        if r+1 <= n {
            diff[r+1] -= val
        }
    }

    // Reconstruct with prefix sum
    result := make([]int, n)
    result[0] = diff[0]
    for i := 1; i < n; i++ {
        result[i] = result[i-1] + diff[i]
    }
    fmt.Println("Final array:", result)
    // Index: 0  1  2  3  4  5  6  7  8  9
    //        3  2  7  7  8  7  7  7  2  2
}
```

**Textual Figure — Range Set and Range Add:**

```
  n = 10, initial: all zeros

  Operation trace on diff array:

  After [0..4] += 3:
  diff: [+3, 0, 0, 0, 0, -3, 0, 0, 0, 0, 0]

  After [2..7] += 5:
  diff: [+3, 0, +5, 0, 0, -3, 0, 0, -5, 0, 0]

  After [5..9] += 2:
  diff: [+3, 0, +5, 0, 0, -1, 0, 0, -5, 0, -2]

  After [1..3] -= 1:
  diff: [+3, -1, +5, 0, +1, -1, 0, 0, -5, 0, -2]

  Prefix sum:
   i:     0   1   2   3   4   5   6   7   8   9
   val:   3   2   7   7   8   7   7   7   2   2

  Breakdown per index:
  idx 0: +3             = 3
  idx 1: +3 -1          = 2
  idx 2: +3 -1 +5       = 7
  idx 3: +3 -1 +5       = 7
  idx 4: +3    +5       = 8
  idx 5:       +5 +2    = 7
  idx 6:       +5 +2    = 7
  idx 7:       +5 +2    = 7
  idx 8:          +2    = 2
  idx 9:          +2    = 2
```

---

## Example 10: Frequency Histogram with Difference Array

```go
package main

import "fmt"

// Given ranges of ages, build age distribution histogram
func main() {
    maxAge := 100
    diff := make([]int, maxAge+2)

    // People with ages in ranges
    ageRanges := [][2]int{
        {20, 30}, // 50 people aged 20-30
        {25, 35}, // 30 people aged 25-35
        {30, 40}, // 20 people aged 30-40
        {18, 22}, // 40 people aged 18-22
    }
    counts := []int{50, 30, 20, 40}

    for i, r := range ageRanges {
        diff[r[0]] += counts[i]
        diff[r[1]+1] -= counts[i]
    }

    // Find peak age
    current := 0
    maxCount := 0
    peakAge := 0
    for age := 0; age <= maxAge; age++ {
        current += diff[age]
        if current > maxCount {
            maxCount = current
            peakAge = age
        }
    }
    fmt.Printf("Peak age: %d with %d people\n", peakAge, maxCount)

    // Print distribution for ages 18-40
    current = 0
    for age := 0; age <= 40; age++ {
        current += diff[age]
        if age >= 18 {
            bar := ""
            for i := 0; i < current/5; i++ {
                bar += "█"
            }
            fmt.Printf("Age %2d: %3d %s\n", age, current, bar)
        }
    }
}
```

**Textual Figure — Frequency Histogram with Difference Array:**

```
  Age ranges and counts:
    Ages 20-30: 50 people
    Ages 25-35: 30 people
    Ages 30-40: 20 people
    Ages 18-22: 40 people

  diff array updates:
    diff[20]+=50, diff[31]-=50
    diff[25]+=30, diff[36]-=30
    diff[30]+=20, diff[41]-=20
    diff[18]+=40, diff[23]-=40

  Reconstruct histogram:
  Age 18: 40  ████████
  Age 19: 40  ████████
  Age 20: 90  ██████████████████  ← +50
  Age 21: 90  ██████████████████
  Age 22: 90  ██████████████████
  Age 23: 50  ██████████        ← 40 left
  Age 24: 50  ██████████
  Age 25: 80  ████████████████  ← +30
  ...         (and so on)
  Age 30: 100 ████████████████████ ← peak!

  Without diff array: O(people × range_size) updates
  With diff array:    O(ranges) updates + O(max_age) reconstruct
```

---

## Key Takeaways

1. **Difference array** is the inverse of prefix sum
2. **Range update [L,R] by +val**: `diff[L] += val; diff[R+1] -= val` → O(1)
3. **Reconstruct** original: prefix sum of diff → O(n)
4. **Best for**: many range updates followed by one reconstruction
5. **2D difference array**: uses inclusion-exclusion with 4 corners
6. **Common problems**: flight bookings, car pooling, interval overlaps

> **Next up:** Two Pointer Technique →
