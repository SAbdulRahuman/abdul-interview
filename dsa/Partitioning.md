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
