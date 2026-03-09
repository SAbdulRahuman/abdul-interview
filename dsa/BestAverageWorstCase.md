# Phase 1: Algorithm Complexity — Best / Average / Worst Case

## What Are Best, Average, and Worst Case?

For any algorithm, the number of operations can vary depending on the **specific input**:

- **Best case**: The input that causes the FEWEST operations
- **Worst case**: The input that causes the MOST operations
- **Average case**: The expected operations over ALL possible inputs

**Big O typically refers to worst case** unless stated otherwise.

---

## Example 1: Linear Search — All Three Cases

```go
package main

import "fmt"

// Linear search: find target in an unsorted array
func linearSearch(nums []int, target int) (int, int) {
    comparisons := 0
    for i, num := range nums {
        comparisons++
        if num == target {
            return i, comparisons
        }
    }
    return -1, comparisons
}

func main() {
    nums := []int{10, 20, 30, 40, 50, 60, 70, 80, 90, 100}

    // Best case: target is the FIRST element → O(1)
    idx, comps := linearSearch(nums, 10)
    fmt.Printf("Best case:  found at %d, comparisons: %d\n", idx, comps) // 1

    // Average case: target is in the MIDDLE → O(n/2) = O(n)
    idx, comps = linearSearch(nums, 50)
    fmt.Printf("Avg case:   found at %d, comparisons: %d\n", idx, comps) // 5

    // Worst case: target is LAST or NOT FOUND → O(n)
    idx, comps = linearSearch(nums, 100)
    fmt.Printf("Worst case: found at %d, comparisons: %d\n", idx, comps) // 10

    idx, comps = linearSearch(nums, 999)
    fmt.Printf("Not found:  found at %d, comparisons: %d\n", idx, comps) // 10
}
```

| Case | When | Comparisons | Big O |
|------|------|-------------|-------|
| Best | Target at index 0 | 1 | O(1) |
| Average | Target at random position | n/2 | O(n) |
| Worst | Target at end or missing | n | O(n) |

---

## Example 2: Bubble Sort — All Three Cases

```go
package main

import "fmt"

func bubbleSort(nums []int) ([]int, int, int) {
    n := len(nums)
    comparisons := 0
    swaps := 0

    for i := 0; i < n; i++ {
        swapped := false
        for j := 0; j < n-1-i; j++ {
            comparisons++
            if nums[j] > nums[j+1] {
                nums[j], nums[j+1] = nums[j+1], nums[j]
                swaps++
                swapped = true
            }
        }
        if !swapped {
            break // optimization: stop if no swaps in a pass
        }
    }
    return nums, comparisons, swaps
}

func main() {
    // Best case: already sorted → O(n) comparisons, 0 swaps
    best := []int{1, 2, 3, 4, 5}
    _, comps, swaps := bubbleSort(best)
    fmt.Printf("Best:  comparisons=%d, swaps=%d\n", comps, swaps)

    // Average case: random order → O(n²)
    avg := []int{3, 1, 4, 5, 2}
    _, comps, swaps = bubbleSort(avg)
    fmt.Printf("Avg:   comparisons=%d, swaps=%d\n", comps, swaps)

    // Worst case: reverse sorted → O(n²) comparisons and swaps
    worst := []int{5, 4, 3, 2, 1}
    _, comps, swaps = bubbleSort(worst)
    fmt.Printf("Worst: comparisons=%d, swaps=%d\n", comps, swaps)
}
```

---

## Example 3: Quick Sort — Best vs Worst Pivot

```go
package main

import "fmt"

var comparisons int

func quickSort(nums []int, low, high int) {
    if low < high {
        pivot := partition(nums, low, high)
        quickSort(nums, low, pivot-1)
        quickSort(nums, pivot+1, high)
    }
}

func partition(nums []int, low, high int) int {
    pivot := nums[high] // last element as pivot
    i := low - 1
    for j := low; j < high; j++ {
        comparisons++
        if nums[j] <= pivot {
            i++
            nums[i], nums[j] = nums[j], nums[i]
        }
    }
    nums[i+1], nums[high] = nums[high], nums[i+1]
    return i + 1
}

func main() {
    // Best case: pivot always splits evenly → O(n log n)
    best := []int{5, 1, 9, 3, 7, 2, 8, 4, 6}
    comparisons = 0
    quickSort(best, 0, len(best)-1)
    fmt.Printf("Random input: %d comparisons\n", comparisons)

    // Worst case: already sorted + last pivot → O(n²)
    worst := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
    comparisons = 0
    quickSort(worst, 0, len(worst)-1)
    fmt.Printf("Sorted input: %d comparisons\n", comparisons)
}
```

| Case | Pivot Choice | Complexity |
|------|-------------|-----------|
| Best | Always median | O(n log n) |
| Average | Random pivot | O(n log n) |
| Worst | Always min/max | O(n²) |

---

## Example 4: Binary Search — All Three Cases

```go
package main

import "fmt"

func binarySearch(nums []int, target int) (int, int) {
    comparisons := 0
    left, right := 0, len(nums)-1

    for left <= right {
        comparisons++
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid, comparisons
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1, comparisons
}

func main() {
    nums := []int{1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 27, 29, 31}
    n := len(nums)

    // Best case: target is at the exact middle → O(1)
    mid := nums[n/2]
    idx, comps := binarySearch(nums, mid)
    fmt.Printf("Best (middle):   idx=%2d, comparisons=%d\n", idx, comps)

    // Average case: ~ log(n)/2 comparisons → O(log n)
    idx, comps = binarySearch(nums, 5)
    fmt.Printf("Average:         idx=%2d, comparisons=%d\n", idx, comps)

    // Worst case: element not found → O(log n)
    idx, comps = binarySearch(nums, 20)
    fmt.Printf("Worst (missing): idx=%2d, comparisons=%d\n", idx, comps)
}
```

---

## Example 5: Insertion Sort — Best Case Advantage

```go
package main

import "fmt"

func insertionSort(nums []int) ([]int, int, int) {
    comparisons := 0
    shifts := 0

    for i := 1; i < len(nums); i++ {
        key := nums[i]
        j := i - 1
        for j >= 0 && nums[j] > key {
            comparisons++
            nums[j+1] = nums[j]
            shifts++
            j--
        }
        if j >= 0 {
            comparisons++ // the comparison that ended the inner loop
        }
        nums[j+1] = key
    }
    return nums, comparisons, shifts
}

func main() {
    // Best case: already sorted → O(n), 0 shifts
    best := []int{1, 2, 3, 4, 5, 6, 7, 8}
    _, comps, shifts := insertionSort(best)
    fmt.Printf("Sorted:  comparisons=%d, shifts=%d\n", comps, shifts)

    // Worst case: reverse sorted → O(n²)
    worst := []int{8, 7, 6, 5, 4, 3, 2, 1}
    _, comps, shifts = insertionSort(worst)
    fmt.Printf("Reverse: comparisons=%d, shifts=%d\n", comps, shifts)

    // Nearly sorted: very efficient → almost O(n)
    nearly := []int{1, 2, 4, 3, 5, 6, 8, 7}
    _, comps, shifts = insertionSort(nearly)
    fmt.Printf("Nearly:  comparisons=%d, shifts=%d\n", comps, shifts)
}
```

**Key insight:** Insertion sort is preferred when data is **nearly sorted** because its best case is O(n).

---

## Example 6: Hash Table Lookup — All Three Cases

```go
package main

import "fmt"

// Hash table collision scenarios
func main() {
    // Best case: O(1) — no collision, direct hit
    m := map[string]int{"apple": 1, "banana": 2, "cherry": 3}
    fmt.Println(m["banana"]) // O(1) average — direct hash lookup

    // Worst case: O(n) — all keys hash to same bucket (pathological)
    // This is theoretical; Go's map handles it well

    // Average case: O(1) — good hash function distributes evenly
    bigMap := make(map[int]int)
    for i := 0; i < 1000000; i++ {
        bigMap[i] = i * 2
    }

    // Lookup is still O(1) average even with 1M entries
    fmt.Println(bigMap[500000]) // 1000000

    fmt.Println("\nHash Table Complexity:")
    fmt.Println("  Best case:    O(1) — no collision")
    fmt.Println("  Average case: O(1) — good hash function")
    fmt.Println("  Worst case:   O(n) — all keys collide")
}
```

---

## Example 7: Measuring All Cases Empirically

```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func linearSearchCount(nums []int, target int) int {
    ops := 0
    for _, num := range nums {
        ops++
        if num == target {
            return ops
        }
    }
    return ops
}

func main() {
    n := 10000
    nums := make([]int, n)
    for i := range nums {
        nums[i] = i
    }

    // Best case: search for first element
    best := linearSearchCount(nums, 0)

    // Worst case: search for last element
    worst := linearSearchCount(nums, n-1)

    // Average case: run 1000 random searches
    rng := rand.New(rand.NewSource(time.Now().UnixNano()))
    totalOps := 0
    trials := 1000
    for i := 0; i < trials; i++ {
        target := rng.Intn(n)
        totalOps += linearSearchCount(nums, target)
    }
    average := totalOps / trials

    fmt.Printf("n = %d\n", n)
    fmt.Printf("Best case:    %d operations\n", best)
    fmt.Printf("Average case: %d operations\n", average)
    fmt.Printf("Worst case:   %d operations\n", worst)
    // Best: 1, Average: ~5000 (n/2), Worst: 10000 (n)
}
```

---

## Example 8: Algorithm Selection Based on Case Analysis

```go
package main

import (
    "fmt"
    "sort"
)

// Different algorithms excel in different cases

// Insertion sort: Best O(n), Worst O(n²)
func insertionSortCopy(nums []int) []int {
    result := make([]int, len(nums))
    copy(result, nums)
    for i := 1; i < len(result); i++ {
        key := result[i]
        j := i - 1
        for j >= 0 && result[j] > key {
            result[j+1] = result[j]
            j--
        }
        result[j+1] = key
    }
    return result
}

// Merge sort: Always O(n log n) — no best/worst variation
func mergeSortArr(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }
    mid := len(nums) / 2
    left := mergeSortArr(nums[:mid])
    right := mergeSortArr(nums[mid:])
    return mergeArrays(left, right)
}

func mergeArrays(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i] <= b[j] {
            result = append(result, a[i]); i++
        } else {
            result = append(result, b[j]); j++
        }
    }
    result = append(result, a[i:]...)
    result = append(result, b[j:]...)
    return result
}

func main() {
    // Nearly sorted data — insertion sort wins
    nearly := []int{1, 2, 3, 5, 4, 6, 7, 9, 8, 10}
    fmt.Println("Nearly sorted:", insertionSortCopy(nearly))

    // Random data — merge sort / built-in sort preferred
    random := []int{38, 27, 43, 3, 9, 82, 10}
    sorted := make([]int, len(random))
    copy(sorted, random)
    sort.Ints(sorted)
    fmt.Println("Random:", sorted)

    // Decision table
    fmt.Println("\n=== When to Use Which Sort ===")
    fmt.Println("Nearly sorted  → Insertion Sort (best=O(n))")
    fmt.Println("Random data    → Quick Sort / Merge Sort (avg=O(n log n))")
    fmt.Println("Guaranteed perf→ Merge Sort (always O(n log n))")
    fmt.Println("Small arrays   → Insertion Sort (low overhead)")
}
```

---

## Example 9: Best/Worst for Tree Operations

```go
package main

import "fmt"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func insert(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    if val < root.Val {
        root.Left = insert(root.Left, val)
    } else {
        root.Right = insert(root.Right, val)
    }
    return root
}

func height(root *TreeNode) int {
    if root == nil {
        return 0
    }
    left := height(root.Left)
    right := height(root.Right)
    if left > right {
        return left + 1
    }
    return right + 1
}

func main() {
    // Best case: balanced BST — height = log(n), search = O(log n)
    var balanced *TreeNode
    for _, v := range []int{50, 25, 75, 12, 37, 62, 87} {
        balanced = insert(balanced, v)
    }
    fmt.Printf("Balanced BST height: %d (ideal: log₂(7) ≈ 3)\n", height(balanced))

    // Worst case: insert in sorted order → becomes linked list
    // height = n, search = O(n)
    var skewed *TreeNode
    for _, v := range []int{1, 2, 3, 4, 5, 6, 7} {
        skewed = insert(skewed, v)
    }
    fmt.Printf("Skewed BST height:   %d (worst: n = 7)\n", height(skewed))

    fmt.Println("\nBST Operation Complexity:")
    fmt.Println("  Best case (balanced):  O(log n)")
    fmt.Println("  Worst case (skewed):   O(n)")
    fmt.Println("  Fix: Use AVL or Red-Black tree → guaranteed O(log n)")
}
```

---

## Example 10: Summary Comparison Table

```go
package main

import "fmt"

func main() {
    type AlgoCase struct {
        Algorithm string
        Best      string
        Average   string
        Worst     string
    }

    algos := []AlgoCase{
        {"Linear Search", "O(1)", "O(n)", "O(n)"},
        {"Binary Search", "O(1)", "O(log n)", "O(log n)"},
        {"Insertion Sort", "O(n)", "O(n²)", "O(n²)"},
        {"Bubble Sort (opt)", "O(n)", "O(n²)", "O(n²)"},
        {"Merge Sort", "O(n log n)", "O(n log n)", "O(n log n)"},
        {"Quick Sort", "O(n log n)", "O(n log n)", "O(n²)"},
        {"Heap Sort", "O(n log n)", "O(n log n)", "O(n log n)"},
        {"Hash Lookup", "O(1)", "O(1)", "O(n)"},
        {"BST Search", "O(1)", "O(log n)", "O(n)"},
        {"Balanced BST", "O(1)", "O(log n)", "O(log n)"},
    }

    fmt.Printf("%-20s %-12s %-12s %-12s\n", "Algorithm", "Best", "Average", "Worst")
    fmt.Println("------------------------------------------------------------")
    for _, a := range algos {
        fmt.Printf("%-20s %-12s %-12s %-12s\n", a.Algorithm, a.Best, a.Average, a.Worst)
    }
}
```

---

## Key Takeaways

1. **Worst case** is what Big O typically represents — used in interviews
2. **Best case** is useful to identify when an algorithm excels (insertion sort on nearly sorted data)
3. **Average case** requires probability analysis — harder to compute
4. **Quick sort**: O(n log n) average but O(n²) worst — randomized pivot fixes this
5. **Merge sort**: always O(n log n) — predictable but uses O(n) space
6. **Choose algorithms based on expected input patterns**
7. **Always state which case** you're analyzing in interviews

> **Next up:** Time-Space Tradeoffs →
