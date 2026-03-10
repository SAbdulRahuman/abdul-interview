# Phase 2: Arrays — Partitioning

## What is Partitioning?

**Partitioning** rearranges elements so that all elements satisfying a condition come before those that don't. It's the foundation of QuickSort and QuickSelect.

---

## Example 1: Basic Partition Around a Pivot (Lomuto's Scheme)

```go
package main

import "fmt"

func lomutoPartition(arr []int, lo, hi int) int {
    pivot := arr[hi]
    i := lo - 1

    for j := lo; j < hi; j++ {
        if arr[j] <= pivot {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }
    arr[i+1], arr[hi] = arr[hi], arr[i+1]
    return i + 1
}

func main() {
    arr := []int{10, 80, 30, 90, 40, 50, 70}
    fmt.Println("Before:", arr)
    pivotIdx := lomutoPartition(arr, 0, len(arr)-1) // pivot = 70
    fmt.Printf("After:  %v (pivot at index %d)\n", arr, pivotIdx)
    // Elements ≤ 70 are left of pivotIdx, > 70 are right
}
```

**Why Lomuto?** Simple, easy to understand. Uses one pointer scanning left-to-right.

**Textual Figure — Lomuto Partition:**

```
  arr = [10, 80, 30, 90, 40, 50, 70]   pivot = arr[hi] = 70
  i = -1 (boundary of ≤ pivot region)

  j=0: arr[0]=10 ≤ 70 → i=0, swap(0,0)
       [10, 80, 30, 90, 40, 50, 70]
        i

  j=1: arr[1]=80 > 70 → skip

  j=2: arr[2]=30 ≤ 70 → i=1, swap(1,2)
       [10, 30, 80, 90, 40, 50, 70]
            i

  j=3: arr[3]=90 > 70 → skip

  j=4: arr[4]=40 ≤ 70 → i=2, swap(2,4)
       [10, 30, 40, 90, 80, 50, 70]
                i

  j=5: arr[5]=50 ≤ 70 → i=3, swap(3,5)
       [10, 30, 40, 50, 80, 90, 70]
                    i

  Final: swap(i+1=4, hi=6)
       [10, 30, 40, 50, 70, 90, 80]
        ≤ pivot region   P  > pivot
                         ↑
                    pivot at index 4
```

---

## Example 2: Hoare's Partition Scheme

```go
package main

import "fmt"

func hoarePartition(arr []int, lo, hi int) int {
    pivot := arr[lo]
    i, j := lo-1, hi+1

    for {
        i++
        for arr[i] < pivot {
            i++
        }
        j--
        for arr[j] > pivot {
            j--
        }
        if i >= j {
            return j
        }
        arr[i], arr[j] = arr[j], arr[i]
    }
}

func main() {
    arr := []int{10, 80, 30, 90, 40, 50, 70}
    fmt.Println("Before:", arr)
    p := hoarePartition(arr, 0, len(arr)-1)
    fmt.Printf("After:  %v (partition at index %d)\n", arr, p)
}
```

**Why Hoare?** Fewer swaps on average than Lomuto. Better for sorted/nearly-sorted data.

**Textual Figure — Hoare’s Partition:**

```
  arr = [10, 80, 30, 90, 40, 50, 70]   pivot = arr[lo] = 10
  i = lo-1 = -1,  j = hi+1 = 7

  Round 1:
    i++ → 0: arr[0]=10 ≥ pivot → stop    (scan right for ≥ pivot)
    j-- → 6: arr[6]=70 > pivot → continue
    j-- → 5: arr[5]=50 > pivot → continue
    j-- → 4: arr[4]=40 > pivot → continue
    j-- → 3: arr[3]=90 > pivot → continue
    j-- → 2: arr[2]=30 > pivot → continue
    j-- → 1: arr[1]=80 > pivot → continue
    j-- → 0: arr[0]=10 ≤ pivot → stop     (scan left for ≤ pivot)
    i ≥ j (0 ≥ 0) → return j = 0

  Result: partition at index 0
  [10 | 80, 30, 90, 40, 50, 70]
  ≤ pivot       > pivot

  Hoare vs Lomuto:
  ┌──────────┬────────────┬──────────────┐
  │          │  Lomuto     │   Hoare        │
  ├──────────┼────────────┼──────────────┤
  │ Scan dir │ L → R only │ Both L→R, R→L │
  │ Swaps    │ More       │ Fewer          │
  │ Pivot    │ In place   │ Not in place   │
  └──────────┴────────────┴──────────────┘
```

---

## Example 3: QuickSort Using Partition

```go
package main

import "fmt"

func quickSort(arr []int, lo, hi int) {
    if lo < hi {
        pivot := arr[hi]
        i := lo - 1
        for j := lo; j < hi; j++ {
            if arr[j] <= pivot {
                i++
                arr[i], arr[j] = arr[j], arr[i]
            }
        }
        arr[i+1], arr[hi] = arr[hi], arr[i+1]
        pi := i + 1

        quickSort(arr, lo, pi-1)
        quickSort(arr, pi+1, hi)
    }
}

func main() {
    arr := []int{38, 27, 43, 3, 9, 82, 10}
    quickSort(arr, 0, len(arr)-1)
    fmt.Println(arr) // [3 9 10 27 38 43 82]

    arr2 := []int{5, 3, 8, 4, 2}
    quickSort(arr2, 0, len(arr2)-1)
    fmt.Println(arr2) // [2 3 4 5 8]
}
```

**Textual Figure — QuickSort Recursion Tree:**

```
  arr = [38, 27, 43, 3, 9, 82, 10]

  Level 0: partition around 10 (last elem)
  [3, 9, 10, 27, 38, 82, 43]
   └──┘  P  └──────────┘

  Level 1L: partition [3, 9] around 9
  [3, 9]  →  [3, 9]   pivot at idx 1

  Level 1R: partition [27, 38, 82, 43] around 43
  [27, 38, 43, 82]   pivot at idx 5

  Level 2: partition [27, 38] around 38 → done
           [82] base case → done

  Recursion tree:
             [38,27,43,3,9,82,10]
                    |  pivot=10
              ┌─────┴─────┐
           [3,9]       [27,38,82,43]
            |              | pivot=43
           done      ┌────┴────┐
                   [27,38]       [82]
                     |           done
                    done

  Result: [3, 9, 10, 27, 38, 43, 82]  ✓
```

---

## Example 4: QuickSelect — Kth Smallest Element

```go
package main

import (
    "fmt"
    "math/rand"
)

func partition(arr []int, lo, hi int) int {
    pivot := arr[hi]
    i := lo - 1
    for j := lo; j < hi; j++ {
        if arr[j] <= pivot {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }
    arr[i+1], arr[hi] = arr[hi], arr[i+1]
    return i + 1
}

func quickSelect(arr []int, lo, hi, k int) int {
    if lo == hi {
        return arr[lo]
    }
    // Random pivot for O(n) average
    randIdx := lo + rand.Intn(hi-lo+1)
    arr[randIdx], arr[hi] = arr[hi], arr[randIdx]

    pivotIdx := partition(arr, lo, hi)

    if k == pivotIdx {
        return arr[k]
    } else if k < pivotIdx {
        return quickSelect(arr, lo, pivotIdx-1, k)
    }
    return quickSelect(arr, pivotIdx+1, hi, k)
}

func main() {
    arr := []int{7, 10, 4, 3, 20, 15}
    k := 2 // 0-indexed: 3rd smallest
    result := quickSelect(arr, 0, len(arr)-1, k)
    fmt.Printf("%dth smallest = %d\n", k+1, result) // 7

    arr2 := []int{12, 3, 5, 7, 19}
    fmt.Printf("1st smallest = %d\n", quickSelect(arr2, 0, len(arr2)-1, 0)) // 3
}
```

**Textual Figure — QuickSelect (Kth Smallest):**

```
  arr = [7, 10, 4, 3, 20, 15]   k = 2 (find 3rd smallest, 0-indexed)

  Step 1: Random pivot selected, say pivot=10
  After partition: [7, 4, 3, 10, 20, 15]
                    0  1  2   3   4   5
                              ↑
                         pivotIdx = 3
  k=2 < pivotIdx=3 → search LEFT half [7, 4, 3]

  Step 2: Partition [7, 4, 3], say pivot=3
  After partition: [3, 4, 7, ...]
                    0  1  2
                    ↑
               pivotIdx = 0
  k=2 > pivotIdx=0 → search RIGHT half [4, 7]

  Step 3: Partition [4, 7], say pivot=7
  After partition: [4, 7]
                    1  2
                       ↑
               pivotIdx = 2
  k=2 == pivotIdx=2 → return arr[2] = 7  ✓

  Unlike QuickSort, we only recurse into ONE side!
  Average O(n), worst O(n²)
```

---

## Example 5: Three-Way Partition (Equal Elements)

```go
package main

import "fmt"

// Three-way partition around value: [< val | == val | > val]
func threeWayPartition(arr []int, val int) (int, int) {
    lo, mid, hi := 0, 0, len(arr)-1

    for mid <= hi {
        if arr[mid] < val {
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo++
            mid++
        } else if arr[mid] > val {
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi--
        } else {
            mid++
        }
    }
    return lo, hi // boundaries of == region
}

func main() {
    arr := []int{4, 9, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 4}
    lo, hi := threeWayPartition(arr, 4)
    fmt.Printf("Result: %v\n", arr)
    fmt.Printf("Equal region: indices %d to %d\n", lo, hi)
    // All 1s first, then 4s, then 9s
}
```

**Textual Figure — Three-Way Partition:**

```
  arr = [4, 9, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 4]   val = 4

  Three regions maintained:
  [0..lo-1]  = elements < val
  [lo..mid-1] = elements == val
  [hi+1..n-1] = elements > val
  [mid..hi]   = unprocessed

  ┌──── < val ────┬─── == val ───┬─ unprocessed ─┬── > val ──┐
  │              │              │              │            │
  0           lo-1  lo      mid-1 mid       hi  hi+1     n-1

  Trace (key steps):
  Start: lo=0, mid=0, hi=12

  mid=0: arr[0]=4 == val → mid++
  mid=1: arr[1]=9 > val → swap(1,12), hi=11
         [4, 4, 4, 4, 1, 9, 4, 4, 9, 4, 4, 1, 9]
  mid=1: arr[1]=4 == val → mid++
  ...continues...

  Final: [1, 1, 4, 4, 4, 4, 4, 4, 4, 4, 9, 9, 9]
          < 4    │──── == 4 ────│   > 4
         lo=2                  hi=9
```

---

## Example 6: Partition by Predicate (Even/Odd)

```go
package main

import "fmt"

func partitionByPredicate(arr []int, pred func(int) bool) int {
    i := 0
    for j := 0; j < len(arr); j++ {
        if pred(arr[j]) {
            arr[i], arr[j] = arr[j], arr[i]
            i++
        }
    }
    return i // first element not satisfying predicate
}

func main() {
    // Even numbers first
    arr1 := []int{1, 2, 3, 4, 5, 6, 7, 8}
    k := partitionByPredicate(arr1, func(x int) bool { return x%2 == 0 })
    fmt.Printf("Evens first: %v (first odd at %d)\n", arr1, k)

    // Negatives first
    arr2 := []int{3, -1, 5, -8, 2, -7, 4}
    k2 := partitionByPredicate(arr2, func(x int) bool { return x < 0 })
    fmt.Printf("Negatives first: %v (first positive at %d)\n", arr2, k2)

    // Multiples of 3 first
    arr3 := []int{1, 3, 6, 7, 9, 12, 5, 15}
    k3 := partitionByPredicate(arr3, func(x int) bool { return x%3 == 0 })
    fmt.Printf("Multiples of 3 first: %v (first non at %d)\n", arr3, k3)
}
```

**Textual Figure — Partition by Predicate:**

```
  arr = [1, 2, 3, 4, 5, 6, 7, 8]   predicate: isEven

  i = write pointer (where next true goes)
  j = read pointer (scanning all elements)

  j=0: 1 odd  → skip         [1, 2, 3, 4, 5, 6, 7, 8]   i=0
  j=1: 2 even → swap(0,1)    [2, 1, 3, 4, 5, 6, 7, 8]   i=1
  j=2: 3 odd  → skip         [2, 1, 3, 4, 5, 6, 7, 8]   i=1
  j=3: 4 even → swap(1,3)    [2, 4, 3, 1, 5, 6, 7, 8]   i=2
  j=4: 5 odd  → skip
  j=5: 6 even → swap(2,5)    [2, 4, 6, 1, 5, 3, 7, 8]   i=3
  j=6: 7 odd  → skip
  j=7: 8 even → swap(3,7)    [2, 4, 6, 8, 5, 3, 7, 1]   i=4

  Result: [2, 4, 6, 8 | 5, 3, 7, 1]
           even group    odd group
           i=0..3        i=4..7

  Note: Order within groups is NOT preserved (unstable).
```

---

## Example 7: Stable Partition (Preserves Relative Order)

```go
package main

import "fmt"

// Stable partition using O(n) extra space
func stablePartition(arr []int, pred func(int) bool) []int {
    var trueGroup, falseGroup []int
    for _, v := range arr {
        if pred(v) {
            trueGroup = append(trueGroup, v)
        } else {
            falseGroup = append(falseGroup, v)
        }
    }
    return append(trueGroup, falseGroup...)
}

func main() {
    arr := []int{1, 2, 3, 4, 5, 6, 7, 8}

    // Stable: evens first, order preserved within each group
    result := stablePartition(arr, func(x int) bool { return x%2 == 0 })
    fmt.Println("Stable partition:", result)
    // [2 4 6 8 1 3 5 7] — order preserved!

    // Compare with unstable:
    arr2 := []int{1, 2, 3, 4, 5, 6, 7, 8}
    i := 0
    for j := 0; j < len(arr2); j++ {
        if arr2[j]%2 == 0 {
            arr2[i], arr2[j] = arr2[j], arr2[i]
            i++
        }
    }
    fmt.Println("Unstable partition:", arr2)
    // Order within groups NOT guaranteed
}
```

**Textual Figure — Stable vs Unstable Partition:**

```
  arr = [1, 2, 3, 4, 5, 6, 7, 8]   predicate: isEven

  Stable Partition (preserves original order):
  ┌────────────────────────────────────┐
  │ Scan and collect into two buckets:   │
  │   trueGroup:  [2, 4, 6, 8]           │
  │   falseGroup: [1, 3, 5, 7]           │
  │   concat: [2, 4, 6, 8, 1, 3, 5, 7]  │
  └────────────────────────────────────┘
  Original order:   2 before 4 before 6 before 8  ✓
  Original order:   1 before 3 before 5 before 7  ✓

  Unstable Partition (swaps may reorder):
  [2, 4, 6, 8, 5, 3, 7, 1]  ← evens ok, odds jumbled

  ┌──────────┬──────┬──────┬──────────┐
  │          │Time  │Space │ Stable?  │
  ├──────────┼──────┼──────┼──────────┤
  │ Swap     │O(n)  │O(1)  │ No       │
  │ 2-bucket │O(n)  │O(n)  │ Yes      │
  └──────────┴──────┴──────┴──────────┘
```

---

## Example 8: Partition Around Median

```go
package main

import (
    "fmt"
    "sort"
)

func partitionAroundMedian(arr []int) int {
    // Find median using sorting (or QuickSelect for O(n))
    sorted := make([]int, len(arr))
    copy(sorted, arr)
    sort.Ints(sorted)
    median := sorted[len(sorted)/2]

    // Partition around median
    i := 0
    for j := 0; j < len(arr); j++ {
        if arr[j] < median {
            arr[i], arr[j] = arr[j], arr[i]
            i++
        }
    }
    return i
}

func main() {
    arr := []int{9, 3, 7, 1, 5, 8, 2, 6, 4}
    medianIdx := partitionAroundMedian(arr)
    fmt.Printf("Partitioned around median: %v (median region starts at %d)\n", arr, medianIdx)
    // All elements < median are to the left
}
```

**Textual Figure — Partition Around Median:**

```
  arr = [9, 3, 7, 1, 5, 8, 2, 6, 4]

  Step 1: Find median
    sorted copy: [1, 2, 3, 4, 5, 6, 7, 8, 9]
    median = sorted[n/2] = sorted[4] = 5

  Step 2: Partition around median=5
    i = write pointer for elements < 5

    j=0: 9 ≥ 5 → skip
    j=1: 3 < 5 → swap(0,1)  [3, 9, 7, 1, 5, 8, 2, 6, 4]  i=1
    j=2: 7 ≥ 5 → skip
    j=3: 1 < 5 → swap(1,3)  [3, 1, 7, 9, 5, 8, 2, 6, 4]  i=2
    j=4: 5 ≥ 5 → skip
    j=5: 8 ≥ 5 → skip
    j=6: 2 < 5 → swap(2,6)  [3, 1, 2, 9, 5, 8, 7, 6, 4]  i=3
    j=7: 6 ≥ 5 → skip
    j=8: 4 < 5 → swap(3,8)  [3, 1, 2, 4, 5, 8, 7, 6, 9]  i=4

  Result: [3, 1, 2, 4 | 5, 8, 7, 6, 9]
           < median     ≥ median
           medianIdx=4

  Useful for: balanced partitioning in QuickSort (ideal pivot).
```

---

## Example 9: Two-Way Partition for Binary Search Prep

```go
package main

import "fmt"

// Partition array so left side is <= threshold, right side > threshold
func partitionByThreshold(arr []int, threshold int) int {
    lo, hi := 0, len(arr)-1
    for lo <= hi {
        if arr[lo] <= threshold {
            lo++
        } else {
            arr[lo], arr[hi] = arr[hi], arr[lo]
            hi--
        }
    }
    return lo // first element > threshold
}

func main() {
    arr := []int{5, 2, 8, 1, 9, 3, 7, 4, 6}
    threshold := 5
    k := partitionByThreshold(arr, threshold)
    fmt.Printf("Threshold %d: %v\n", threshold, arr)
    fmt.Printf("Left (<=%d): %v\n", threshold, arr[:k])
    fmt.Printf("Right (>%d): %v\n", threshold, arr[k:])
}
```

**Textual Figure — Two-Way Partition for Threshold:**

```
  arr = [5, 2, 8, 1, 9, 3, 7, 4, 6]   threshold = 5
  lo = 0,  hi = 8

  lo=0: arr[0]=5 ≤ 5 → lo++           lo=1
  lo=1: arr[1]=2 ≤ 5 → lo++           lo=2
  lo=2: arr[2]=8 > 5 → swap(2,8), hi--
        [5, 2, 6, 1, 9, 3, 7, 4, 8]   hi=7
  lo=2: arr[2]=6 > 5 → swap(2,7), hi--
        [5, 2, 4, 1, 9, 3, 7, 6, 8]   hi=6
  lo=2: arr[2]=4 ≤ 5 → lo++           lo=3
  lo=3: arr[3]=1 ≤ 5 → lo++           lo=4
  lo=4: arr[4]=9 > 5 → swap(4,6), hi--
        [5, 2, 4, 1, 7, 3, 9, 6, 8]   hi=5
  lo=4: arr[4]=7 > 5 → swap(4,5), hi--
        [5, 2, 4, 1, 3, 7, 9, 6, 8]   hi=4
  lo=4: arr[4]=3 ≤ 5 → lo++           lo=5
  lo=5 > hi=4 → STOP

  Result: [5, 2, 4, 1, 3 | 7, 9, 6, 8]
            ≤ 5 (k=5)      > 5
```

---

## Example 10: Multi-Way Partition (Sort by Category)

```go
package main

import "fmt"

// Partition into 3 categories: low, mid, high
func multiPartition(arr []int, lowMax, highMin int) {
    lo, mid, hi := 0, 0, len(arr)-1

    for mid <= hi {
        if arr[mid] <= lowMax {
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo++
            mid++
        } else if arr[mid] >= highMin {
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi--
        } else {
            mid++
        }
    }
}

func main() {
    // Partition temperatures: cold (≤10), warm (11-25), hot (≥26)
    temps := []int{30, 5, 22, 8, 35, 15, 3, 28, 18, 12}
    multiPartition(temps, 10, 26)
    fmt.Println("Partitioned (cold|warm|hot):", temps)
    // Cold (≤10) | Warm (11-25) | Hot (≥26)

    // Partition scores: fail (≤40), pass (41-79), distinction (≥80)
    scores := []int{95, 35, 72, 88, 45, 28, 62, 91, 50, 15}
    multiPartition(scores, 40, 80)
    fmt.Println("Partitioned (fail|pass|dist):", scores)
}
```

**Textual Figure — Multi-Way (3-Category) Partition:**

```
  temps = [30, 5, 22, 8, 35, 15, 3, 28, 18, 12]
  lowMax=10 (cold ≤10), highMin=26 (hot ≥26)

  Three-pointer approach (same as Dutch National Flag):
  lo=0, mid=0, hi=9

  ┌──── cold ───┬──── warm ───┬─ unprocessed ┬── hot ──┐
  │            │            │             │          │
  0         lo-1  lo     mid-1 mid      hi  hi+1    n-1

  Trace (key steps):
  mid=0: 30 ≥ 26(hot) → swap(0,9), hi=8
         [12, 5, 22, 8, 35, 15, 3, 28, 18, 30]
  mid=0: 12 warm → mid++
  mid=1: 5 ≤ 10(cold) → swap(0,1), lo=1, mid=2
         [5, 12, 22, 8, 35, 15, 3, 28, 18, 30]
  ...continues...

  Final: [5, 8, 3 | 22, 15, 18, 12 | 35, 28, 30]
          cold       warm             hot
         (≤10)      (11-25)          (≥26)
```

---

## Comparison Table

| Scheme | Time | Space | Stable? | Swaps |
|--------|------|-------|---------|-------|
| Lomuto | O(n) | O(1) | No | More |
| Hoare | O(n) | O(1) | No | Fewer |
| Three-way | O(n) | O(1) | No | Optimal for duplicates |
| Stable | O(n) | O(n) | Yes | N/A (copy-based) |

---

## Key Takeaways

1. **Lomuto**: simple, good for QuickSort implementation
2. **Hoare**: fewer swaps, better for practical performance
3. **Three-way**: optimal when many duplicates (Dutch National Flag variant)
4. **QuickSelect**: O(n) average for kth element using random pivot
5. **Partition = building block** for sorting, selection, and filtering

> **Next up:** Dutch National Flag Algorithm →
