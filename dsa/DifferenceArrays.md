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

---

## Example 8: Temperature Change Over Time

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

---

## Example 9: Range Set and Range Add Combined

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

---

## Key Takeaways

1. **Difference array** is the inverse of prefix sum
2. **Range update [L,R] by +val**: `diff[L] += val; diff[R+1] -= val` → O(1)
3. **Reconstruct** original: prefix sum of diff → O(n)
4. **Best for**: many range updates followed by one reconstruction
5. **2D difference array**: uses inclusion-exclusion with 4 corners
6. **Common problems**: flight bookings, car pooling, interval overlaps

> **Next up:** Two Pointer Technique →
